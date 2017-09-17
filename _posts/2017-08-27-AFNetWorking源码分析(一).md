---
layout:     post
title:      "AFNetWorking源码分析"
subtitle:   ""
date:       2017-08-25 15:11:00
author:     "Renchao"
header-img: "img/2017.05.07.jpg"
catalog:    true
tags: 
   - 源码分析
---

## AFNetworking源码分析

AFNetworking是iOS开发者不会不知道的网络库，可能没用过原生NSUrlSession，但一定用过AFN来请求。它简单好用的接口受到开发者的肯定，从github上马上30k的star就能看出来。那么为什么放着原生库不用，而都选择这个库？既然人尽皆知，作为iOSer却不知道内部实现就说不过去了，我们来分析分析源码（基于3.x）。

### 整体架构

![afntree](http://ov8ee4i4b.bkt.clouddn.com/afntree.png)

从上图来看，afn的整体框架应该分为五大模块：

- **网络通信模块**：AFURLSessionManager、AFHTTPSessionManger
- **网络状态监听模块**：Reachability
- **网络通信安全策略模块**：SecurityPolicy
- **网络通信信息序列化、反序列化模块**：Serialization
- **UIKit库的拓展**：UIKit+AFNetworking

其中核心模块是AFURLSessionManager，也就是基于NSURLSessionManager封装的请求类。其余四个模块是为了配合网络通信而做的拓展类。AFHTTPSessionManger只是简单封装自NSURLSessionManager的类，简单的http请求一般就用这个类就足够了。

关系图如下：

![](http://ov8ee4i4b.bkt.clouddn.com/关系图.png)

### 初始化

先写一个简单的get请求：

```objective-c
AFHTTPSessionManager *manager = [[AFHTTPSessionManager alloc]init];

[manager GET:@"http://123.com" parameters:nil progress:nil success:^(NSURLSessionDataTask * _Nonnull task, id  _Nullable responseObject) {

} failure:^(NSURLSessionDataTask * _Nullable task, NSError * _Nonnull error) {

}];
```

初始化manager的内部实现如下：

```objective-c
+ (instancetype)manager {
    return [[[self class] alloc] initWithBaseURL:nil];
}

- (instancetype)init {
    return [self initWithBaseURL:nil];
}

- (instancetype)initWithBaseURL:(NSURL *)url {
    return [self initWithBaseURL:url sessionConfiguration:nil];
}

- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    return [self initWithBaseURL:nil sessionConfiguration:configuration];
}

//都进入此方法中
- (instancetype)initWithBaseURL:(NSURL *)url
           sessionConfiguration:(NSURLSessionConfiguration *)configuration
{
    self = [super initWithSessionConfiguration:configuration];
    if (!self) {
        return nil;
    }

    if ([[url path] length] > 0 && ![[url absoluteString] hasSuffix:@"/"]) {
        url = [url URLByAppendingPathComponent:@""];
    }

    self.baseURL = url;

    self.requestSerializer = [AFHTTPRequestSerializer serializer];
    self.responseSerializer = [AFJSONResponseSerializer serializer];

    return self;
}
```

进入初始化父类sessionConfiguration的方法：

```objective-c
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }

    self.sessionConfiguration = configuration;

    self.operationQueue = [[NSOperationQueue alloc] init];
    //并发线程设置为1
    self.operationQueue.maxConcurrentOperationCount = 1;

  	//注意代理，代理的继承，实际上NSURLSession去判断了，你实现了哪个方法会去调用，包括子代理方法
    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];

    //转码
    self.responseSerializer = [AFJSONResponseSerializer serializer];
	//安全策略
    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

#if !TARGET_OS_WATCH
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
#endif
	//设置存储NSURL task与AFURLSessionManagerTaskDelegate的词典（重点，在AFNet中，每一个task都会被匹配一个AFURLSessionManagerTaskDelegate 来做task的delegate事件处理）
    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

 	//设置AFURLSessionManagerTaskDelegate 词典的锁，确保词典在多线程访问时的线程安全
    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;

  	//置空task关联的代理
    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task uploadProgress:nil downloadProgress:nil completionHandler:nil];
        }

        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

    return self;
}
```

需要说明三点：

- `self.operationQueue.maxConcurrentOperationCount = 1;`**这个operationQueue就是我们代理回调的queue。这里把代理回调的线程并发数设置为1了。**至于这里为什么要这么做，我们先留一个坑，等我们讲完AF2.x之后再来分析这一块。

- 我们初始化了一些属性，其中包括`self.mutableTaskDelegatesKeyedByTaskIdentifier`，这个是用来让每一个请求task和我们自定义的AF代理来建立映射用的，其实AF对task的代理进行了一个封装，并且转发代理到AF自定义的代理，这是AF比较重要的一部分，接下来我们会具体讲这一块。

- `[self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) { }];`

  首先说说这个方法是干什么用的：这个方法用来异步的获取当前session的所有未完成的task。其实讲道理来说在初始化中调用这个方法应该里面一个task都不会有。我们打断点去看，也确实如此，里面的数组都是空的。但是想想也知道，AF大神不会把一段没用的代码放在这吧。辗转多处，终于从AF的issue中找到了结论：[github](https://github.com/AFNetworking/AFNetworking/issues/3499)。原来这是为了从后台回来，重新初始化session，防止一些之前的后台请求任务，导致程序的crash。

初始化方法到这里就全部完成了。

### 网络请求

```objective-c
- (NSURLSessionDataTask *)GET:(NSString *)URLString
                   parameters:(id)parameters
                     progress:(void (^)(NSProgress * _Nonnull))downloadProgress
                      success:(void (^)(NSURLSessionDataTask * _Nonnull, id _Nullable))success
                      failure:(void (^)(NSURLSessionDataTask * _Nullable, NSError * _Nonnull))failure
{
	
    //生成一个NSURLSessionDataTask实例开始网络请求
    NSURLSessionDataTask *dataTask = [self dataTaskWithHTTPMethod:@"GET"
                                                        URLString:URLString
                                                       parameters:parameters
                                                   uploadProgress:nil
                                                 downloadProgress:downloadProgress
                                                          success:success
                                                          failure:failure];

    [dataTask resume];

    return dataTask;
}
```

进入该方法：

```objective-c
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                  uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgress
                                downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgress
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
    NSError *serializationError = nil;
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];
    if (serializationError) {
        if (failure) {
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
        }

        return nil;
    }

    __block NSURLSessionDataTask *dataTask = nil;
    dataTask = [self dataTaskWithRequest:request
                          uploadProgress:uploadProgress
                        downloadProgress:downloadProgress
                       completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];

    return dataTask;
}
```

这个方法做了两件事：

- 用`self.requestSerializer`和各种参数去获取了一个我们最终请求网络需要的`NSMutableURLRequest`实例。
- 调用另外一个方法`dataTaskWithRequest`去拿到我们最终需要的`NSURLSessionDataTask`实例，并且在完成的回调里，调用我们传过来的成功和失败的回调。

说到底这个方法还是没有做实事，我们继续到`requestSerializer`方法里去看，看看AF到底如何拼接成我们需要的`request`的：

```objective-c
- (NSMutableURLRequest *)requestWithMethod:(NSString *)method
                                 URLString:(NSString *)URLString
                                parameters:(id)parameters
                                     error:(NSError *__autoreleasing *)error
{
    //断言，debug模式下，如果缺少改参数，crash                                
    NSParameterAssert(method);
    NSParameterAssert(URLString);

    NSURL *url = [NSURL URLWithString:URLString];

    NSParameterAssert(url);

    NSMutableURLRequest *mutableRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    mutableRequest.HTTPMethod = method;
	
  	//将request的各种属性循环遍历
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
      	//如果自己观察到的发生变化的属性，在这些方法里
        if ([self.mutableObservedChangedKeyPaths containsObject:keyPath]) {
           	//把给自己设置的属性给request设置
            [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
        }
    }

    //将传入的parameters进行编码，并添加到request中
    mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];

	return mutableRequest;
}
```

这个方法做了三件事：

- 设置request的请求类型，如get、post、put等

- 向request里添加一些参数设置，其中`AFHTTPRequestSerializerObservedKeyPaths()`是一个C函数，返回一个数组，这个函数的内部：

  ```objective-c
  static NSArray * AFHTTPRequestSerializerObservedKeyPaths() {
    static NSArray *_AFHTTPRequestSerializerObservedKeyPaths = nil;
    static dispatch_once_t onceToken;
    // 此处需要observer的keypath为allowsCellularAccess、cachePolicy、HTTPShouldHandleCookies
    // HTTPShouldUsePipelining、networkServiceType、timeoutInterval
    dispatch_once(&onceToken, ^{
        _AFHTTPRequestSerializerObservedKeyPaths = @[NSStringFromSelector(@selector(allowsCellularAccess)), NSStringFromSelector(@selector(cachePolicy)), NSStringFromSelector(@selector(HTTPShouldHandleCookies)), NSStringFromSelector(@selector(HTTPShouldUsePipelining)), NSStringFromSelector(@selector(networkServiceType)), NSStringFromSelector(@selector(timeoutInterval))];
    });
    //就是一个数组里装了很多方法的名字,
    return _AFHTTPRequestSerializerObservedKeyPaths;
  }
  ```

  其实这个函数就是封装了一些属性的名字，这些都是`NSURLRequest`的属性。

  再来看看`self.mutableObservedChangedKeyPaths`，这个是当前类的一个属性：

  ```objective-c
  @property (readwrite, nonatomic, strong) NSMutableSet *mutableObservedChangedKeyPaths;
  ```

  在`init`方法对这个集合进行了初始化，并且对当前类的和`NSURLRequest`相关的那些属性添加了KVO监听：

  ```objective-c
  //每次都会重置变化
    self.mutableObservedChangedKeyPaths = [NSMutableSet set];

    //给这自己些方法添加观察者为自己，就是request的各种属性，set方法
    for (NSString *keyPath in AFHTTPRequestSerializerObservedKeyPaths()) {
        if ([self respondsToSelector:NSSelectorFromString(keyPath)]) {
            [self addObserver:self forKeyPath:keyPath options:NSKeyValueObservingOptionNew context:AFHTTPRequestSerializerObserverContext];
        }
    }
  ```

  KVO触发的方法：

  ```objective-c
  -(void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(__unused id)object
                        change:(NSDictionary *)change
                       context:(void *)context
  {
    //当观察到这些set方法被调用了，而且不为Null就会添加到集合里，否则移除
    if (context == AFHTTPRequestSerializerObserverContext) {
        if ([change[NSKeyValueChangeNewKey] isEqual:[NSNull null]]) {
            [self.mutableObservedChangedKeyPaths removeObject:keyPath];
        } else {
            [self.mutableObservedChangedKeyPaths addObject:keyPath];
        }
    }
  }
  ```

  至此我们知道了`self.mutableObservedChangedKeyPaths`其实就是我们自己设置的`request`属性值的集合，接下来调用：

  ```objective-c
  [mutableRequest setValue:[self valueForKeyPath:keyPath] forKey:keyPath];
  ```

  用KVC的方式，把属性值都设置到我们请求的request中去。

- 把需要传递的参数进行编码，并且设置到request中去：

  ```objective-c
  //将传入的parameters进行编码，并添加到request中
  mutableRequest = [[self requestBySerializingRequest:mutableRequest withParameters:parameters error:error] mutableCopy];
  ```

  进入该方法：

  ```objective-c
  - (NSURLRequest *)requestBySerializingRequest:(NSURLRequest *)request
                               withParameters:(id)parameters
                                        error:(NSError *__autoreleasing *)error
  {
    NSParameterAssert(request);

    NSMutableURLRequest *mutableRequest = [request mutableCopy];

    //从自己的head里去遍历，如果有值则设置给request的head
    [self.HTTPRequestHeaders enumerateKeysAndObjectsUsingBlock:^(id field, id value, BOOL * __unused stop) {
        if (![request valueForHTTPHeaderField:field]) {
            [mutableRequest setValue:value forHTTPHeaderField:field];
        }
    }];

    //来把各种类型的参数，array dic set转化成字符串，给request
    NSString *query = nil;
    if (parameters) {
        //自定义的解析方式
        if (self.queryStringSerialization) {
            NSError *serializationError;
            query = self.queryStringSerialization(request, parameters, &serializationError);

            if (serializationError) {
                if (error) {
                    *error = serializationError;
                }

                return nil;
            }
        } else {
            //默认解析方式
            switch (self.queryStringSerializationStyle) {
                case AFHTTPRequestQueryStringDefaultStyle:
                    query = AFQueryStringFromParameters(parameters);
                    break;
            }
        }
    }

    //最后判断该request中是否包含了GET、HEAD、DELETE（都包含在HTTPMethodsEncodingParametersInURI）。因为这几个method的quey是拼接到url后面的。而POST、PUT是把query拼接到http body中的。
    if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
        if (query && query.length > 0) {
            mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
        }
    } else {
        //post put请求

        // #2864: an empty string is a valid x-www-form-urlencoded payload
        if (!query) {
            query = @"";
        }
        if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
            [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
        }
        //设置请求体
        [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
    }

    return mutableRequest;
  }
  ```

  这个方法做了三件事：

  1. 从`self.HTTPRequestHeaders`中拿到设置的参数，赋值要请求的`request`里去

  2. 把请求网络的参数，从array dic set这些容器类型转换为字符串，具体转码方式，我们可以使用自定义的方式，也可以用AF默认的转码方式。自定义的方式没什么好说的，想怎么去解析由你自己来决定。我们可以来看看默认的方式：

     ```objective-c
     NSString * AFQueryStringFromParameters(NSDictionary *parameters) {
       NSMutableArray *mutablePairs = [NSMutableArray array];

       //把参数给AFQueryStringPairsFromDictionary，拿到AF的一个类型的数据就一个key，value对象，在URLEncodedStringValue拼接keyValue，一个加到数组里
       for (AFQueryStringPair *pair in AFQueryStringPairsFromDictionary(parameters)) {
           [mutablePairs addObject:[pair URLEncodedStringValue]];
       }

       //拆分数组返回参数字符串
       return [mutablePairs componentsJoinedByString:@"&"];
     }
     NSArray * AFQueryStringPairsFromDictionary(NSDictionary *dictionary) {
       //往下调用
       return AFQueryStringPairsFromKeyAndValue(nil, dictionary);
     }
     NSArray * AFQueryStringPairsFromKeyAndValue(NSString *key, id value) {
       NSMutableArray *mutableQueryStringComponents = [NSMutableArray array];

       // 根据需要排列的对象的description来进行升序排列，并且selector使用的是compare:
       // 因为对象的description返回的是NSString，所以此处compare:使用的是NSString的compare函数
       // 即@[@"foo", @"bar", @"bae"] ----> @[@"bae", @"bar",@"foo"]
       NSSortDescriptor *sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"description" ascending:YES selector:@selector(compare:)];

       //判断vaLue是什么类型的，然后去递归调用自己，直到解析的是除了array dic set以外的元素，然后把得到的参数数组返回。
       if ([value isKindOfClass:[NSDictionary class]]) {
           NSDictionary *dictionary = value;
           // Sort dictionary keys to ensure consistent ordering in query string, which is important when deserializing potentially ambiguous sequences, such as an array of dictionaries

           //拿到
           for (id nestedKey in [dictionary.allKeys sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
               id nestedValue = dictionary[nestedKey];
               if (nestedValue) {
                   [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue((key ? [NSString stringWithFormat:@"%@[%@]", key, nestedKey] : nestedKey), nestedValue)];
               }
           }
       } else if ([value isKindOfClass:[NSArray class]]) {
           NSArray *array = value;
           for (id nestedValue in array) {
               [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue([NSString stringWithFormat:@"%@[]", key], nestedValue)];
           }
       } else if ([value isKindOfClass:[NSSet class]]) {
           NSSet *set = value;
           for (id obj in [set sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
               [mutableQueryStringComponents addObjectsFromArray:AFQueryStringPairsFromKeyAndValue(key, obj)];
           }
       } else {
           [mutableQueryStringComponents addObject:[[AFQueryStringPair alloc] initWithField:key value:value]];
       }

       return mutableQueryStringComponents;
     }
     ```

     转码主要是以上三个函数，配合着注释应该也很好理解：主要是在递归调用`AFQueryStringPairsFromKeyAndValue`。判断vaLue是什么类型的，然后去递归调用自己，直到解析的是除了array dic set以外的元素，然后把得到的参数数组返回。

     其中有个`AFQueryStringPair`对象，其只有两个属性和两个方法：

     ```objective-c
     @property (readwrite, nonatomic, strong) id field;
     @property (readwrite, nonatomic, strong) id value;

     - (instancetype)initWithField:(id)field value:(id)value {
     	self = [super init];
     	if (!self) {
         	return nil;
     	}

     	self.field = field;
     	self.value = value;

     	return self;
     }

     - (NSString *)URLEncodedStringValue {
     	if (!self.value || [self.value isEqual:[NSNull null]]) {
         	return AFPercentEscapedStringFromString([self.field description]);
     	} else {
        		return [NSString stringWithFormat:@"%@=%@", AFPercentEscapedStringFromString([self.field description]), AFPercentEscapedStringFromString([self.value description])];
     	}
     }
     ```

     方法很简单，现在我们也很容易理解这整个转码过程了，我们举个例子梳理下，就是以下这3步：

     ```objective-c
     @{ 
      @"name" : @"bang", 
      @"phone": @{@"mobile": @"xx", @"home": @"xx"}, 
      @"families": @[@"father", @"mother"], 
      @"nums": [NSSet setWithObjects:@"1", @"2", nil] 
     } 
     -> 
     @[ 
      field: @"name", value: @"bang", 
      field: @"phone[mobile]", value: @"xx", 
      field: @"phone[home]", value: @"xx", 
      field: @"families[]", value: @"father", 
      field: @"families[]", value: @"mother", 
      field: @"nums", value: @"1", 
      field: @"nums", value: @"2", 
     ] 
     -> 
     name=bang&phone[mobile]=xx&phone[home]=xx&families[]=father&families[]=mother&nums=1&num=2
     ```

     至此，我们原来的容器类型的参数，就这样变成字符串类型了。

  3. 紧接着这个方法还根据该request中请求类型，来判断参数字符串应该如何设置到request中去。如果是GET、HEAD、DELETE，则把参数quey是拼接到url后面的。而POST、PUT是把query拼接到http body中的：

     ```objective-c
     if ([self.HTTPMethodsEncodingParametersInURI containsObject:[[request HTTPMethod] uppercaseString]]) {
       if (query && query.length > 0) {
           mutableRequest.URL = [NSURL URLWithString:[[mutableRequest.URL absoluteString] stringByAppendingFormat:mutableRequest.URL.query ? @"&%@" : @"?%@", query]];
       }
     } else {
       //post put请求

       // #2864: an empty string is a valid x-www-form-urlencoded payload
       if (!query) {
           query = @"";
       }
       if (![mutableRequest valueForHTTPHeaderField:@"Content-Type"]) {
           [mutableRequest setValue:@"application/x-www-form-urlencoded" forHTTPHeaderField:@"Content-Type"];
       }
       //设置请求体
       [mutableRequest setHTTPBody:[query dataUsingEncoding:self.stringEncoding]];
     }
     ```

至此，我们生成了一个request。

### 再回到`AFHTTPSessionManager`来

```objective-c
- (NSURLSessionDataTask *)dataTaskWithHTTPMethod:(NSString *)method
                                       URLString:(NSString *)URLString
                                      parameters:(id)parameters
                                  uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgress
                                downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgress
                                         success:(void (^)(NSURLSessionDataTask *, id))success
                                         failure:(void (^)(NSURLSessionDataTask *, NSError *))failure
{
    NSError *serializationError = nil;
    //把参数，还有各种东西转化为一个request
    NSMutableURLRequest *request = [self.requestSerializer requestWithMethod:method URLString:[[NSURL URLWithString:URLString relativeToURL:self.baseURL] absoluteString] parameters:parameters error:&serializationError];

    if (serializationError) {
        if (failure) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wgnu"
            //如果解析错误，直接返回
            dispatch_async(self.completionQueue ?: dispatch_get_main_queue(), ^{
                failure(nil, serializationError);
            });
#pragma clang diagnostic pop
        }

        return nil;
    }

    __block NSURLSessionDataTask *dataTask = nil;
    dataTask = [self dataTaskWithRequest:request
                          uploadProgress:uploadProgress
                        downloadProgress:downloadProgress
                       completionHandler:^(NSURLResponse * __unused response, id responseObject, NSError *error) {
        if (error) {
            if (failure) {
                failure(dataTask, error);
            }
        } else {
            if (success) {
                success(dataTask, responseObject);
            }
        }
    }];
    return dataTask;
}
```

继续往下看，当解析错误，我们直接调用传进来的fauler的Block失败返回了，这里有一个`self.completionQueue`,这个是我们自定义的，这个是一个GCD的`Queue`如果设置了那么从这个Queue中回调结果，否则从主队列回调。

实际上这个`Queue`还是挺有用的，之前还用到过。我们公司有自己的一套数据加解密的解析模式，所以我们回调回来的数据并不想是主线程，我们可以设置这个`Queue`,在分线程进行解析数据，然后自己再调回到主线程去刷新`UI`。

言归正传，我们接着调用了父类的生成task的方法，并且执行了一个成功和失败的回调，我们接着去父类AFURLSessionManger里看（总算到我们的核心类了..）：

```objective-c
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    //第一件事，创建NSURLSessionDataTask，里面适配了Ios8以下taskIdentifiers，函数创建task对象。
    //其实现应该是因为iOS 8.0以下版本中会并发地创建多个task对象，而同步有没有做好，导致taskIdentifiers 不唯一…这边做了一个串行处理
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```

我们注意到这个方法非常简单，就调用了一个`url_session_manager_create_task_safely()`函数，传了一个Block进去，Block里就是iOS原生生成dataTask的方法。此外，还调用了一个`addDelegateForDataTask`的方法。

我们到这先到这个函数里去看看：

```objective-c
static void url_session_manager_create_task_safely(dispatch_block_t block) {
  if (NSFoundationVersionNumber < NSFoundationVersionNumber_With_Fixed_5871104061079552_bug) {
      // Fix of bug
      // Open Radar:http://openradar.appspot.com/radar?id=5871104061079552 (status: Fixed in iOS8)
      // Issue about:https://github.com/AFNetworking/AFNetworking/issues/2093

    //理解下，第一为什么用sync，因为是想要主线程等在这，等执行完，在返回，因为必须执行完dataTask才有数据，传值才有意义。
    //第二，为什么要用串行队列，因为这块是为了防止ios8以下内部的dataTaskWithRequest是并发创建的，
    //这样会导致taskIdentifiers这个属性值不唯一，因为后续要用taskIdentifiers来作为Key对应delegate。
      dispatch_sync(url_session_manager_creation_queue(), block);
  } else {
      block();
  }
}
static dispatch_queue_t url_session_manager_creation_queue() {
  static dispatch_queue_t af_url_session_manager_creation_queue;
  static dispatch_once_t onceToken;
  //保证了即使是在多线程的环境下，也不会创建其他队列
  dispatch_once(&onceToken, ^{
      af_url_session_manager_creation_queue = dispatch_queue_create("com.alamofire.networking.session.manager.creation", DISPATCH_QUEUE_SERIAL);
  });

  return af_url_session_manager_creation_queue;
}
```

方法非常简单，关键是理解这么做的目的：为什么我们不直接去调用
`dataTask = [self.session dataTaskWithRequest:request];`
非要绕这么一圈，我们点进去bug日志里看看，**原来这是为了适配iOS8的以下，创建session的时候，偶发的情况会出现session的属性taskIdentifier这个值不唯一**，而这个taskIdentifier是我们后面来映射delegate的key,所以它必须是唯一的。

**具体原因应该是NSURLSession内部去生成task的时候是用多线程并发去执行的。**想通了这一点，我们就很好解决了，我们只需要在iOS8以下**同步串行**的去生成task就可以防止这一问题发生（如果还是不理解同步串行的原因，可以看看注释）。

题外话：很多同学都会抱怨为什么sync我从来用不到，看，有用到的地方了吧，**很多东西不是没用，而只是你想不到怎么用**。

我们接着看到：

```objective-c
[self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];
```

调用到：

```objective-c
- (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];

    // AFURLSessionManagerTaskDelegate与AFURLSessionManager建立相互关系
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    //这个taskDescriptionForSessionTasks用来发送开始和挂起通知的时候会用到,就是用这个值来Post通知，来两者对应
    dataTask.taskDescription = self.taskDescriptionForSessionTasks;

    // ***** 将AF delegate对象与 dataTask建立关系
    [self setDelegate:delegate forTask:dataTask];

    // 设置AF delegate的上传进度，下载进度块。
    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
}
```

总结一下：

1. 这个方法，生成了一个`AFURLSessionManagerTaskDelegate`,这个其实就是AF的自定义代理。我们请求传来的参数，都赋值给这个AF的代理了。

2. `delegate.manager = self;`代理把AFURLSessionManager这个类作为属性了,我们可以看到：

   ```objective-c
   @property (nonatomic, weak) AFURLSessionManager *manager;
   ```

   这个属性是弱引用的，所以不会存在循环引用的问题。

3. 我们调用了`[self setDelegate:delegate forTask:dataTask];`，进去看看：

   ```objective-c
   - (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
               forTask:(NSURLSessionTask *)task
   {
       //断言，如果没有这个参数，debug下crash在这
       NSParameterAssert(task);
       NSParameterAssert(delegate);

       //加锁保证字典线程安全
       [self.lock lock];
       // 将AF delegate放入以taskIdentifier标记的词典中（同一个NSURLSession中的taskIdentifier是唯一的）
       self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;

       // 为AF delegate 设置task 的progress监听
       [delegate setupProgressForTask:task];

       //添加task开始和暂停的通知
       [self addNotificationObserverForTask:task];
       [self.lock unlock];
   }
   ```

   - 这个方法主要就是把AF代理和task建立映射，存在了一个我们事先声明好的字典里。

   - 而要加锁的原因是因为本身我们这个字典属性是mutable的，是线程不安全的。而我们对这些方法的调用，确实是会在复杂的多线程环境中，后面会仔细提到线程问题。

   - 还有个`[delegate setupProgressForTask:task];`我们到方法里去看看：

     ```objective-c
     - (void)setupProgressForTask:(NSURLSessionTask *)task {

         __weak __typeof__(task) weakTask = task;

         //拿到上传下载期望的数据大小
         self.uploadProgress.totalUnitCount = task.countOfBytesExpectedToSend;
         self.downloadProgress.totalUnitCount = task.countOfBytesExpectedToReceive;
     ```


         //将上传与下载进度和 任务绑定在一起，直接cancel suspend resume进度条，可以cancel...任务
         [self.uploadProgress setCancellable:YES];
         [self.uploadProgress setCancellationHandler:^{
             __typeof__(weakTask) strongTask = weakTask;
             [strongTask cancel];
         }];
         [self.uploadProgress setPausable:YES];
         [self.uploadProgress setPausingHandler:^{
             __typeof__(weakTask) strongTask = weakTask;
             [strongTask suspend];
         }];
    
         if ([self.uploadProgress respondsToSelector:@selector(setResumingHandler:)]) {
             [self.uploadProgress setResumingHandler:^{
                 __typeof__(weakTask) strongTask = weakTask;
                 [strongTask resume];
             }];
         }
    
         [self.downloadProgress setCancellable:YES];
         [self.downloadProgress setCancellationHandler:^{
             __typeof__(weakTask) strongTask = weakTask;
             [strongTask cancel];
         }];
         [self.downloadProgress setPausable:YES];
         [self.downloadProgress setPausingHandler:^{
             __typeof__(weakTask) strongTask = weakTask;
             [strongTask suspend];
         }];
    
         if ([self.downloadProgress respondsToSelector:@selector(setResumingHandler:)]) {
             [self.downloadProgress setResumingHandler:^{
                 __typeof__(weakTask) strongTask = weakTask;
                 [strongTask resume];
             }];
         }
    
         //观察task的这些属性
         [task addObserver:self
                forKeyPath:NSStringFromSelector(@selector(countOfBytesReceived))
                   options:NSKeyValueObservingOptionNew
                   context:NULL];
         [task addObserver:self
                forKeyPath:NSStringFromSelector(@selector(countOfBytesExpectedToReceive))
                   options:NSKeyValueObservingOptionNew
                   context:NULL];
    
         [task addObserver:self
                forKeyPath:NSStringFromSelector(@selector(countOfBytesSent))
                   options:NSKeyValueObservingOptionNew
                   context:NULL];
         [task addObserver:self
                forKeyPath:NSStringFromSelector(@selector(countOfBytesExpectedToSend))
                   options:NSKeyValueObservingOptionNew
                   context:NULL];
    
         //观察progress这两个属性
         [self.downloadProgress addObserver:self
                                 forKeyPath:NSStringFromSelector(@selector(fractionCompleted))
                                    options:NSKeyValueObservingOptionNew
                                    context:NULL];
         [self.uploadProgress addObserver:self
                               forKeyPath:NSStringFromSelector(@selector(fractionCompleted))
                                  options:NSKeyValueObservingOptionNew
                                  context:NULL];
     }
     ```
    
     这个方法也非常简单，主要做了以下几件事：
    
     - 设置`downloadProgress`与`uploadProgress`的一些属性，并且把两者和task的任务状态绑定在了一起。注意这两者都是NSProgress的实例对象，（这里可能又一群小伙伴楞在这了，这是个什么...）简单来说，这就是iOS7引进的一个用来管理进度的类，可以开始，暂停，取消，完整的对应了task的各种状态，当progress进行各种操作的时候，task也会引发对应操作。
    
     - 给task和progress的各个属及添加KVO监听，至于监听了干什么用，我们接着往下看：
    
       ```objective-c
       - (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSString *,id> *)change context:(void *)context {
    
         //是task
         if ([object isKindOfClass:[NSURLSessionTask class]] || [object isKindOfClass:[NSURLSessionDownloadTask class]]) {
             //给进度条赋新值
             if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesReceived))]) {
                 self.downloadProgress.completedUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
             } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesExpectedToReceive))]) {
                 self.downloadProgress.totalUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
             } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesSent))]) {
                 self.uploadProgress.completedUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
             } else if ([keyPath isEqualToString:NSStringFromSelector(@selector(countOfBytesExpectedToSend))]) {
                 self.uploadProgress.totalUnitCount = [change[NSKeyValueChangeNewKey] longLongValue];
             }
         }
         //上面的赋新值会触发这两个，调用block回调，用户拿到进度
         else if ([object isEqual:self.downloadProgress]) {
             if (self.downloadProgressBlock) {
                 self.downloadProgressBlock(object);
             }
         }
         else if ([object isEqual:self.uploadProgress]) {
             if (self.uploadProgressBlock) {
                 self.uploadProgressBlock(object);
             }
         }
       }
       ```
    
       方法非常简单直观，主要就是如果task触发KVO,则给progress进度赋值，应为赋值了，所以会触发progress的KVO，也会调用到这里，然后去执行我们传进来的`downloadProgressBlock`和`uploadProgressBlock`。主要的作用就是为了让进度实时的传递。
    
       还有一点需要注意：我们之前的setProgress和这个KVO监听，都是在我们AF自定义的delegate内的，是**有一个task就会有一个delegate的。所以说我们是每个task都会去监听这些属性，分别在各自的AF代理内。**看到这，可能有些小伙伴会有点乱，没关系。等整个讲完之后我们还会详细的去讲捋一捋manager、task、还有AF自定义代理三者之前的对应关系。

   到这里我们整个对task的处理就完成了。

###  task请求网络

接着task就开始请求网络了，还记得我们初始化方法中：

```objective-c
self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
```

我们把AFUrlSessionManager作为了所有的task的delegate。当我们请求网络的时候，这些代理开始调用了：

![img](http://upload-images.jianshu.io/upload_images/2702646-5a404cc7d92fb8ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

AFUrlSessionManager一共实现了如上图所示这么一大堆NSUrlSession相关的代理。

而只转发了其中3条到AF自定义的delegate中：

![img](http://upload-images.jianshu.io/upload_images/2702646-e6469f92ca6a550e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这就是我们一开始说的，AFUrlSessionManager对这一大堆代理做了一些公共的处理，而转发到AF自定义代理的3条，则负责把每个task对应的数据回调出去。

又有小伙伴问了，我们设置的这个代理不是`NSURLSessionDelegate`吗？怎么能响应NSUrlSession这么多代理呢？我们点到类的声明文件中去看看：

```objective-c
@protocol NSURLSessionDelegate <NSObject>
@protocol NSURLSessionTaskDelegate <NSURLSessionDelegate>
@protocol NSURLSessionDataDelegate <NSURLSessionTaskDelegate>
@protocol NSURLSessionDownloadDelegate <NSURLSessionTaskDelegate>
@protocol NSURLSessionStreamDelegate <NSURLSessionTaskDelegate>
```

我们可以看到这些代理都是继承关系，而在`NSURLSession`实现中，只要设置了这个代理，它会去判断这些所有的代理，是否`respondsToSelector`这些代理中的方法，如果响应了就会去调用。

而AF还重写了`respondsToSelector`方法：

```objective-c
- (BOOL)respondsToSelector:(SEL)selector {

  //复写了selector的方法，这几个方法是在本类有实现的，但是如果外面的Block没赋值的话，则返回NO，相当于没有实现！
  if (selector == @selector(URLSession:task:willPerformHTTPRedirection:newRequest:completionHandler:)) {
      return self.taskWillPerformHTTPRedirection != nil;
  } else if (selector == @selector(URLSession:dataTask:didReceiveResponse:completionHandler:)) {
      return self.dataTaskDidReceiveResponse != nil;
  } else if (selector == @selector(URLSession:dataTask:willCacheResponse:completionHandler:)) {
      return self.dataTaskWillCacheResponse != nil;
  } else if (selector == @selector(URLSessionDidFinishEventsForBackgroundURLSession:)) {
      return self.didFinishEventsForBackgroundURLSession != nil;
  }
  return [[self class] instancesRespondToSelector:selector];
}
```

这样如果没实现这些我们自定义的Block也不会去回调这些代理。因为本身某些代理，只执行了这些自定义的Block，如果Block都没有赋值，那我们调用代理也没有任何意义。
讲到这，我们顺便看看AFUrlSessionManager的一些自定义Block：

```objective-c
@property (readwrite, nonatomic, copy) AFURLSessionDidBecomeInvalidBlock sessionDidBecomeInvalid;
@property (readwrite, nonatomic, copy) AFURLSessionDidReceiveAuthenticationChallengeBlock sessionDidReceiveAuthenticationChallenge;
@property (readwrite, nonatomic, copy) AFURLSessionDidFinishEventsForBackgroundURLSessionBlock didFinishEventsForBackgroundURLSession;
@property (readwrite, nonatomic, copy) AFURLSessionTaskWillPerformHTTPRedirectionBlock taskWillPerformHTTPRedirection;
@property (readwrite, nonatomic, copy) AFURLSessionTaskDidReceiveAuthenticationChallengeBlock taskDidReceiveAuthenticationChallenge;
@property (readwrite, nonatomic, copy) AFURLSessionTaskNeedNewBodyStreamBlock taskNeedNewBodyStream;
@property (readwrite, nonatomic, copy) AFURLSessionTaskDidSendBodyDataBlock taskDidSendBodyData;
@property (readwrite, nonatomic, copy) AFURLSessionTaskDidCompleteBlock taskDidComplete;
@property (readwrite, nonatomic, copy) AFURLSessionDataTaskDidReceiveResponseBlock dataTaskDidReceiveResponse;
@property (readwrite, nonatomic, copy) AFURLSessionDataTaskDidBecomeDownloadTaskBlock dataTaskDidBecomeDownloadTask;
@property (readwrite, nonatomic, copy) AFURLSessionDataTaskDidReceiveDataBlock dataTaskDidReceiveData;
@property (readwrite, nonatomic, copy) AFURLSessionDataTaskWillCacheResponseBlock dataTaskWillCacheResponse;
@property (readwrite, nonatomic, copy) AFURLSessionDownloadTaskDidFinishDownloadingBlock downloadTaskDidFinishDownloading;
@property (readwrite, nonatomic, copy) AFURLSessionDownloadTaskDidWriteDataBlock downloadTaskDidWriteData;
@property (readwrite, nonatomic, copy) AFURLSessionDownloadTaskDidResumeBlock downloadTaskDidResume;
```

各自对应的还有一堆这样的set方法：

```objective-c
- (void)setSessionDidBecomeInvalidBlock:(void (^)(NSURLSession *session, NSError *error))block {
  self.sessionDidBecomeInvalid = block;
}
```

方法都是一样的，就不重复粘贴占篇幅了。主要谈谈这个设计思路：

作者用@property把这个些Block属性在.m文件中声明,然后复写了set方法，然后在.h中去声明这些set方法：

```objective-c
- (void)setSessionDidBecomeInvalidBlock:(nullable void (^)(NSURLSession *session, NSError *error))block;
```

为什么要绕这么一大圈呢？**原来这是为了我们这些用户使用起来方便，调用set方法去设置这些Block，能很清晰的看到Block的各个参数与返回值。**大神的精髓的编程思想无处不体现...

下一节就讲讲这些代理方法做了些什么

本篇END
#### crash问题几大类

##### 空指针

指针指向的内存空间已经释放，或者指针根本没有初始化和赋值。

空指针是没有存储任何内存地址的指针。如`Student *s1 = NULL;`和`Student *s2 = nil;`

而野指针是指指向一个已删除的对象（"垃圾"内存既不可用内存）或未申请访问受限内存区域的指针。野指针是比较危险的。因为野指针指向的对象已经被释放了，不能用了，你再给被释放的对象发送消息就是违法的，所以会崩溃。

##### 内存不足

通过instrument工具进行模拟。

##### 数组越界

##### 找不到方法

##### 多线程

A进程先删除，B进程后访问；

异步网络操作，服务器响应前需要处理的对象已经被释放，这时如果服务器响应，则找不到对象而crash。

##### iOS新老版本不兼容

系统动态链接库或framework无法找到



## 解决方法

#### 基本和通用的手段：日志（Log）

日志基本上是各类问题的最基本和最通用的收集信息方式，放在别的问题领域同样适用。

记得以前折腾Linux的时候，不去认真检查日志，真的是想不到问题是出现在哪里。

在iOS开发中呢，NSLog应该是人人都在用了。

然而你也会发现，单纯的NSLog用多了，也会有些问题，比如总是输出一大堆无用的信息，除非你在解决完这个问题或解决别的问题时先把无关的NSLog注释掉。

这种时候，日志框架就派上用场了，它们能帮你控制日志的级别、分类和对应的存放位置，有的还提供了工具帮你检索它。

可以关注 [CocoaLumberjack](https://github.com/CocoaLumberjack/CocoaLumberjack) 和 [NSLogger](https://github.com/fpillet/NSLogger)，或者是这个列表：<https://github.com/vsouza/awesome-ios#logging>

#### 面对崩溃(Crash)

崩溃是很烦人的问题，很多崩溃的实质大体就是“尝试访问一个不存在的东西，然后没有访问到”。

如果是在调试环境下运行，可以看调试器(Debbuger)在命令行下输出的东西，通常它的错误信息已经足够完备。

值得强调的是，一个全局异常断点是很必要，它让程序崩溃的时候指向抛出异常的代码，不然程序一崩溃，xcode就直接跳到main函数去了。

具体方法见[文末附录](https://my.oschina.net/zzdjk6/blog/540492#OSC_h2_8)。

如果是在非调试环境下，就要借助一些崩溃信息收集的工具了。

苹果自己有崩溃信息收集的平台，但我还是更心怡Twitter的[Fabric](https://www.fabric.io/)(以前的Crashlytics)。

## 调试UI(User Interface)

自XCode6以后，XCode集成了一个简易的UI调试工具，不过功能上确实不能和第三方的工具相比。

第三方工具推荐两个，[SparkInspector](http://sparkinspector.com/) 和 [Reveal](http://revealapp.com/)。

如果想要完全不改动XCode就启用第三方工具调试UI，那么可以自行通过LLDB动态加载库，或者在UIApplecationDelegate里下一个Symbolic断点，然后自动执行加载第三方库的命令。

## 调试网络请求(Network)

网络请求是单纯的，可以通过监听工具查看HTTP/HTTPS请求和响应的所有内容，这样一来就再也不用问你的请求不通是什么问题了，直接观察请求和响应吧。

这里两个工具是必备的：[Charles](http://www.charlesproxy.com/) 和 [Postman](https://www.getpostman.com/)。

Charles能帮助你在Mac上监听HTTP/HTTPS请求和响应的详细内容，而Postman则可以帮助你轻松地生成想要的HTTP/HTTPS请求来测试后端API。

## 调试技巧(Tips)

开发调试通常都是 Edit->Compile->Run->Edit...这样一个循环。

其实你也可以 Edit->Refresh->Edit...

这就需要借助 [Injection for Xcode](http://injectionforxcode.com/) 这样一个插件，来进行注入更新，它支持iOS和Mac双平台，OC和Swift双语言。

注入更新能加速程序开发和调试的迭代速度，但似乎不能和UI调试工具如SparkInspector一起用。

## 附：添加全局异常断点，并让所有工程共享

![GlobalExceptionBreakpoint](https://static.oschina.net/uploads/img/201512/08001510_2Tnt.png)
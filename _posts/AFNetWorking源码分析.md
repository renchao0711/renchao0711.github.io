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

- 网络通信模块：AFURLSessionManager、AFHTTPSessionManger
- 网络状态监听模块：Reachability
- 网络通信安全策略模块：SecurityPolicy
- 网络通信信息序列化、反序列化模块：Serialization
- UIKit库的拓展：UIKit+AFNetworking

其中核心模块是AFURLSessionManager，也就是基于NSURLSessionManager封装的请求类。其余四个模块是为了配合网络通信而做的拓展类。AFHTTPSessionManger只是简单封装自NSURLSessionManager的类，简单的http请求一般就用这个类就足够了。

关系图如下：

![](http://ov8ee4i4b.bkt.clouddn.com/关系图.png)




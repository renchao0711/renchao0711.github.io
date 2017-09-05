---
layout:     post
title:      "前端小白对React Native的理解"
subtitle:   "Learn once, write anywhere"
date:       2017-09-05 15:11:00
author:     "Renchao"
header-img: "img/2017.05.07.jpg"
catalog:    true
tags: 
   - iOS
   - 前端
---

## 前端小白对React Native的理解

从2015年4月Facebook开源了跨平台移动应用开发框架React Native开始，用JS编写iOS和Android应用变得越来越流行。

Learn once, write anywhere.只使用前端语言就可以编写三个平台上的应用，提高了开发效率，同时即时更新的特点收到了更广泛的关注。

### 动态配置

使用原生开发iOS客户端的开发者都知道，审核时间的限制使得bug问题不能及时得到解决，除非加快审核。所以如何动态的更改app成为了永恒的话题。但是无论采用什么方式，流程都会是：**从Server获取配置->解析->执行native代码**。

我们使用JSON文件实现动态配置的效果，会是这样的流程：

1. 通过 HTTP 请求获取 JSON 格式的配置文件。
2. 配置文件中标记了每一个元素的属性，比如位置，颜色，图片 URL 等。
3. 解析完 JSON 后，我们调用 Objective-C 的代码，完成 UI 控件的渲染。

通过这种方法，我们实现了在后台配置 app 的展示样式。从本质上来说，移动端和服务端约定了一套协议，但是协议内容严重依赖于应用内要展示的内容，不利于拓展。也就是说，如果业务要求频繁的增加或修改页面，这套协议很难应付。

最重要的是，JSON 只是一种数据交换的格式，说白了，我们就是在解析文本数据。这就意味着它只适合提供一些配置信息，而不方便提供逻辑信息。举个例子，我们从后台可以配置颜色，位置等信息，但如果想要控制 app 内的业务逻辑，就非常复杂了。

记住，我们只是在解析字符串，它完全不具备运行和调试的能力。

### React

#### 背景

作为前端小白，我以前对前端的理解是这样的：

- 用 HTML 创建 DOM，构建整个网页的布局、结构
- 用 CSS 控制 DOM 的样式，比如字体、字号、颜色、居中等
- 用 JavaScript 接受用户事件，动态的操控 DOM

在这三者的配合下，几乎所有页面上的功能都能实现。但也有比较不爽地方，比如我想动态修改一个按钮的文字，我需要这样写：

```html
<button type="button" id="button" onclick="onClick()">old button</button>
```

然后在 JavaScript 中操作 DOM：

```javascript
<script>
function onClick() {
  document.getElementById('button').innerHTML='new button';
}
</script>
```

可以看到，在 HTML 和 JavaScript 代码中，`id` 和 `onclick` 事件触发的函数必须完全对应，否则就无法正确的响应事件。如果想知道一个 HTML 标签会如何被响应，我们还得跑去 JavaScript 代码中查找，这种原始的配置方式让我觉得非常不爽。

#### 初识React

随着 FaceBook 推出了 React 框架，这个问题得到了大幅度改善。我们可以把一组相关的 HTML 标签，也就是 app 内的 UI 控件，封装进一个组件(Component)中，我从[阮一峰的 React 教程](http://www.ruanyifeng.com/blog/2015/03/react.html)中摘录了一段代码：

```javascript
var MyComponent = React.createClass({
  handleClick: function() {
    this.refs.myTextInput.focus();
  },
  render: function() {
    return (
      <div>
        <input type="text" ref="myTextInput" />
        <input type="button" value="Focus the text input" onClick={this.handleClick} />
      </div>
    );
  }
});
```

如果你想问：“为什么 JavaScript 代码里面出现了 HTML 的语法”，那么恭喜你已经初步体会到 React 的奥妙了。这种语法被称为 JSX，它是一种 JavaScript 语法拓展。JSX 允许我们写 HTML 标签或 React 标签，它们终将被转换成原生的 JavaScript 并创建 DOM。

在 React 框架中，除了可以用 JavaScript 写 HTML 以外，我们甚至可以写 CSS，这在后面的例子中可以看到。

#### 理解React

前端界总是喜欢创造新的概念，仿佛谁说的名词更晦涩，谁的水平就越高。如果你和当时的我一样，听到 React 这个概念一脸懵逼的话，只要记住以下定义即可：

> React 是一套可以用简洁的语法高效绘制 DOM 的框架

上文已经解释过了何谓“简洁的语法”，因为我们可以暂时放下 HTML 和 CSS，只关心如何用 JavaScript 构造页面。

所谓的“高效”，是因为 React 独创了 Virtual DOM 机制。Virtual DOM 是一个存在于内存中的 JavaScript 对象，它与 DOM 是一一对应的关系，也就是说只要有 Virtual DOM，我们就能渲染出 DOM。

当界面发生变化时，得益于高效的 DOM Diff 算法，我们能够知道 Virtual DOM 的变化，从而高效的改动 DOM，避免了重新绘制 DOM。

当然，React **并不是前端开发的全部**。从之前的描述也能看出，它**专注于 UI 部分**，对应到 MVC 结构中就是 View 层。要想实现完整的 MVC 架构，还需要 Model 和 Controller 的结构。在前端开发时，我们可以采用 Flux 和 Redux 架构，它们并非框架(Library)，而是和 MVC 一样都是一种架构设计(Architecture)。

如果不从事前端开发，就不用深入的掌握 Flux 和 Redux 架构，但理解这一套体系结构对于后面理解 React Native **非常重要**。

### React Native

#### 融合

前面我们介绍了移动端通过 JSON 文件传递信息的不足之处：只能传递配置信息，无法表达逻辑。从本质上讲，这是因为 JSON 毕竟只是纯文本，它缺乏像编程语言那样的运行能力。

而 React 在前端取得突破性成功以后，JavaScript 布道者们开始试图一统三端。他们利用了移动平台能够运行 JavaScript 代码的能力，并且发挥了 JavaScript 不仅仅可以传递配置信息，还可以表达逻辑信息的优点。

当痛点遇上特点，两者一拍即合，于是乎：

> 一个基于 JavaScript，具备动态配置能力，面向前端开发者的移动端开发框架，React Native，诞生了！

看到了么，这是一个面向前端开发者的框架。它的宗旨是让前端开发者像用 React 写网页那样，用 React Native 写移动端应用。这就是为什么 React Native 自称：

> Learn once，Write anywhere!

而非很多跨平台语言，项目所说的：

> Write once, Run anywhere!

React Native 希望前端开发者学习完 React 后，能够用同样的语法、工具等，分别开发安卓和 iOS 平台的应用并且不用一行原生代码。

如果用一个词概括 React Native，那就是：**Native 版本的 React**。

#### 原理概述

React Native 不是黑科技，我们写的代码总是以一种非常合理，可以解释的方式的运行着，只是绝大多数人没有理解而已。接下来我以 iOS 平台为例，简单的解释一下 React Native 的原理。

首先要明白的一点是，**即使使用了 React Native，我们依然需要 UIKit 等框架，调用的是 Objective-C 代码。总之，JavaScript 只是辅助，它只是提供了配置信息和逻辑的处理结果。React Native 与 Hybrid 完全没有关系，它只不过是以 JavaScript 的形式告诉 Objective-C 该执行什么代码。**

其次，React Native 能够运行起来，全靠 Objective-C 和 JavaScript 的交互。对于没有接触过 JavaScript 的人来说，非常有必要理解 JavaScript 代码如何被执行。

我们知道 C 系列的语言，经过编译，链接等操作后，会得到一个二进制格式的可执行文，所谓的运行程序，其实是运行这个二进制程序。

而 JavaScript 是一种脚本语言，它不会经过编译、链接等操作，而是在运行时才动态的进行词法、语法分析，生成抽象语法树(AST)和字节码，然后由解释器负责执行或者使用 JIT 将字节码转化为机器码再执行。整个流程由 JavaScript 引擎负责完成。

**苹果提供了一个叫做 JavaScript Core 的框架，这是一个 JavaScript 引擎**。通过下面这段代码可以简单的感受一下 Objective-C 如何调用 JavaScript 代码：

```objective-c
JSContext *context = [[JSContext alloc] init];
JSValue *jsVal = [context evaluateScript:@"21+7"];
int iVal = [jsVal toInt32];
```

这里的 `JSContext` 指的是 JavaScript 代码的运行环境，通过 `evaluateScript` 即可执行 JavaScript 代码并获取返回结果。

JavaScript 是一种单线程的语言，它不具备自运行的能力，因此总是被动调用。很多介绍 React Native 的文章都会提到 “JavaScript 线程” 的概念，实际上，它表示的是 Objective-C 创建了一个单独的线程，这个线程只用于执行 JavaScript 代码，而且 JavaScript 代码只会在这个线程中执行。

### Objective-C 与 JavaScript 交互

提到 Objective-C 与 JavaScript 的交互，不得不推荐 bang神的这篇文章：[React Native通信机制详解 ](http://blog.cnbang.net/tech/2698/)。虽然其中不少细节都已经过时，但是整体的思路值得学习。

本节主要分析 Objective-C 与 JavaScript 交互时的整理逻辑与流程，下一节将通过源码来分析具体原理。

#### JavaScript 调用 Objective-C

由于 JavaScript Core 是一个面向 Objective-C 的框架，在 Objective-C 这一端，我们对 JavaScript 上下文知根知底，可以很容易的获取到对象，方法等各种信息，当然也包括调用 JavaScript 函数。

真正复杂的问题在于，JavaScript 不知道 Objective-C 有哪些方法可以调用。

**React Native 解决这个问题的方案是在 Objective-C 和 JavaScript 两端都保存了一份配置表，里面标记了所有 Objective-C 暴露给 JavaScript 的模块和方法。这样，无论是哪一方调用另一方的方法，实际上传递的数据只有 `ModuleId`、`MethodId` 和 `Arguments` 这三个元素，它们分别表示类、方法和方法参数，当 Objective-C 接收到这三个值后，就可以通过 runtime 唯一确定要调用的是哪个函数，然后调用这个函数。**

再次重申，上述解决方案只是一个抽象概念，可能与实际的解决方案有微小差异，比如实际上 Objective-C 这一端，并没有直接保存这个模块配置表。具体实现将在下一节中随着源码一起分析。

#### 闭包与回调

既然说到函数互调，那么就不得不提到回调了。对于 Objective-C 来说，执行完 JavaScript 代码再执行 Objective-C 回调毫无难度，难点依然在于 JavaScript 代码调用 Objective-C 之后，如何在 Objective-C 的代码中，回调执行 JavaScript 代码。

目前 React Native 的做法是：在 JavaScript 调用 Objective-C 代码时，注册要回调的 Block，并且把 `BlockId` 作为参数发送给 Objective-C，Objective-C 收到参数时会创建 Block，调用完 Objective-C 函数后就会执行这个刚刚创建的 Block。

Objective-C 会向 Block 中传入参数和 `BlockId`，然后在 Block 内部调用 JavaScript 的方法，随后 JavaScript 查找到当时注册的 Block 并执行。

#### 图解

好吧，如果你是新手，并且坚持读到了这里，估计已经懵逼了。不要担心，与 JavaScript 的交互确实不是一下子能够完全理清楚的，你可以先参考这个示意图：

![1171077-75412d65af198cf5](http://jbcdn2.b0.upaiyun.com/2016/06/3f6f42f026c8af2b9f5648c038cb5254.png)

注：

1. 本图由 bang 的文章中的图片修改而来
2. 本图只是一个简单的示意图，不建议当做时序图使用，请参考下一节源码分析。
3. Objective-C 和 JavaScript 的交互总是由前者发起，本图为了简化，省略了这一步骤。

### RN源码分析

分析的部分直接看原博主的[博客](http://ios.jobbole.com/85788/)

### React Native 优缺点分析

经过一长篇的讨论，其实 React Native 的优缺点已经不难分析了，这里简单总结一下：

#### 优点

1. 复用了 React 的思想，有利于前端开发者涉足移动端。
2. 能够利用 JavaScript 动态更新的特性，快速迭代。
3. 相比于原生平台，开发速度更快，相比于 Hybrid 框架，性能更好。

#### 缺点

1. 做不到 `Write once, Run everywhere`，也就是说开发者依然需要为 iOS 和 Android 平台提供两套不同的代码，比如参考[官方文档](https://facebook.github.io/react-native/docs/getting-started.html)可以发现不少组件和API都区分了 Android 和 iOS 版本。即使是共用组件，也会有平台独享的函数。
2. 不能做到完全屏蔽 iOS 端或 Android 的细节，前端开发者必须对原生平台有所了解。加重了学习成本。对于移动端开发者来说，**完全不具备**用 React Native 开发的能力。
3. 由于 Objective-C 与 JavaScript 之间切换存在固定的时间开销，所以性能必定不及原生。比如目前的官方版本无法做到 UItableview(ListView) 的视图重用，因为滑动过程中，视图重用需要在异步线程中执行，速度太慢。这也就导致随着 Cell 数量的增加，占用的内存也线性增加。

综上，我对 React Native 的定位是：

> 利用脚本语言进行原生平台开发的一次成功尝试，降低了前端开发者入门移动端的门槛，一定业务场景下具有独特的优势，几乎不可能取代原生平台开发。

### 大牛说

唐巧曾这样评论RN的出现：

> **人才稀缺的问题**
>
> 首先的问题是：移动开发人才的稀缺。看看那些培训班出来的人吧，经过3个月的培训就可以拿到8K甚至上万的工作。在北京稍微有点工作经验的iOS开发，就要求2万一个月的工资。这说明当前移动互联网和创业的火热，已经让业界没有足够的开发人才了，所以大家都用涨工资来抢人才。而由于跨平台的框架（例如PhoneGap、RubyMotion）都还是不太靠谱，所以对于稍微大一些的公司，都会选择针对iOS和Android平台分别做不同的定制开发。而JavaScript显然是一个群众基础更广的语言，这将使得相关人才更容易获得，同时由于后面提到的代码复用问题得到解决，也能节省一部分开发人员。
>
> **代码复用的问题**
>
> React Native虽然强调自己不是“Write once, run anywhere”的框架，但是它至少能像Google的[j2objc](https://github.com/google/j2objc)那样，在Model层实现复用。那些底层的、与界面无关的逻辑，相信React Native也可以实现复用。这样，虽然UI层的工作还是需要做iOS和Android两个平台，但如果抽象得好，Logic和Model层的复用不但可以让代码复用，更可能实现底层的逻辑的单元测试。这样移动端的代码质量将更加可靠。
>
> 其实React Native宣传的“Learning once, write anywhere”本身也是一种复用的思想。大家厌烦了各种各样的编程语言，如果有一种语言真的能够统一移动开发领域，对于所有人都是好事。
>
> **UI排版的问题**
>
> 我自己一直不喜欢苹果新推出的AutoLayout那套解决方案，其实HTML和CSS在界面布局和呈现上深耕多年，Android也是借鉴的HTML的那套方案，苹果完全可以也走这套方案的。但是苹果选择发明了一个Constraint的东西来实现排版。在企业的开发中，其实大家很少使用Xib的，而手写Constraint其实是非常痛苦的。所以出现了[Masonry](https://github.com/Masonry/Masonry)一类的开源框架来解决这类同行的痛苦。
>
> 我一直在寻找使用类似HTML + CSS的排版，但是使用原生控件渲染的框架。其实之前[BeeFramework](https://github.com/gavinkwoe/BeeFramework)就做了这方面的事情。所以我还专门代表InfoQ对他进行过采访。BeeFramework虽然开源多年，而且有2000多的star数，但是受限于它自身的影响力以及框架的复杂性，一直没有很大的成功。至少我不知道有什么大的公司采用。
>
> 这次Facebook的React Native做的事情相比BeeFramework更加激进。它不但采用了类似HTML + CSS的排版，还把语言也换成了JavaScript，这下子改变可以称作巨大了。但是Facebook有它作为全球互联网企业的光环，相信会有不少开发者跟进采用React Native。
>
> 不过也说回来，Facebook开源的也不一定都好，比如[three20](https://github.com/facebookarchive/three20)就被Facebook放弃了，但是不可否认three20作为一个框架，在那个时期的特定价值。所以React Native即使没有成功，它也将人们关注的焦点放在了移动开发的效率上了。很可能会有越来越多相关的框架因此涌现出来。
>
> **MVVM**
>
> MVVM在Web开发领域相当火热，而iOS领域的[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)虽然很火，但是还是非常小众。究其原因，一方面是ReactiveCocoa带来的编程习惯上的改变实在太大，ReactiveCocoa和MVVM的学习成本还是很高。另一方面是ReactiveCocoa在代码可读性、可维护性和协作上不太友好。
>
> 而Web开发领域对MVVM编程模式的接受程度就大不相同了，在Web开发中有相当多的被广泛使用的MVVM的框架，例如[AngularJS](http://en.wikipedia.org/wiki/AngularJS)。相信React Native会推动MVVM应用在移动端的开发。
>
> **动态更新**
>
> 终于说到最 “鸡冻人心” 的部分了。你受够了每次发新版本都要审核一个星期吗？苹果的审核团队在效率上的低下，使得我们这一群狠不得每天迭代更新一版的敏捷开发团队被迫每 2 周或 1 个月更新一次版本。很多团队上一个版本还没审核结束，下一个版本就做好了。
>
> React Native的语言是基于JavaScript，这必然会使得代码可以从服务器端动态更新成为可能。到时候，每天更新不再是梦想。当然，代码的安全性将更一步受到挑战，如何有效保护核心代码的安全将是一个难题。
>
> **总结**
>
> 不管怎么样，这确确实实是一个移动互联网的时代，我相信随着几年的发展，移动互联网的开发生态也会积累出越来越多宝贵的框架，以支撑出更加伟大的App出现。作为一个移动开发者，我很高兴能够成为这个时代的主角，用移动开发技术改变人们的生活。
>
> 愿大家珍惜这样的机会，玩得开心～

### 感谢

[React Native 从入门到原理  --bestswift](http://ios.jobbole.com/85788/)
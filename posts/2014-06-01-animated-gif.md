# Flipboard 在 iOS 上怎样播放 GIF 动画

本文翻译自：[http://engineering.flipboard.com/2014/05/animated-gif/](http://engineering.flipboard.com/2014/05/animated-gif/)

原作者：[Raphael Schaad](https://twitter.com/raphaelschaad)

译者：[@nixzhu](https://twitter.com/nixzhu)

---

Flipboard 一直谋求的是“烹饪原始Web”并将其转化为如杂志般优雅的东西。我们考虑到许多细节——从文章的排版到相片的布局——以尽可能忠实地展现内容的本质。

而对于 GIF 来说，我们想让它们在我们的应用里自动播放。自动播放是动态 GIF 的主要诉求。然而，iOS 上的许多应用仅仅只是渲染 GIF 的静态帧——一个由在真机上实时地流畅播放多个 GIF 所导致的复杂性而带来的不幸结果。

某些人可能会想，这么[古老的图片格式][1]对于在现代的 iOS 设备上工作的开发者来说应该是开箱即用的吧。但甚至是 Apple 自家的一些应用都不支持播放它们。当用移动浏览器查看过它们后，系统经常会变慢到像在爬行。在定时重放并保证保真度的同时，保持较小的内存占用和 CPU 使用率实在是一个挑战。

我们关于支持动态 GIF 的需求如下：

* 以堪比桌面浏览器的播放速度同时播放多个 GIF 
* 保证可变帧延迟
* 在内存压力下依然表现优雅
* 在第一次播放时消除延迟或卡顿
* 如同现代浏览器那样解译 fast GIF 的帧延迟（注解1）

因为没有内置的方法或开源库能够满足所有这些需求，所以我们创建了一个引擎并从[去年发布][2]开始起就一直磨练它。我们认为与社区分享它是我们目前最好的选择。

如果你想增加对动态 GIF 的支持以增强你的 iOS 应用，请前往 GitHub [按照简单的指示来使用我们的开源组件][3]。

![][4]

## iOS 为动态图像提供了什么

iOS 上显示图片的典型方式是用图像数据创建一个 [`UIImage`][5] 并将其展现在屏幕上的某个 [`UIImageView`][6] 中。然而，虽然有超过一打的初始化方法，但没有一个可以从单个的多帧图像（注解2）里创建一个动态图像。Image View 仅仅显示第一个静态帧，所以屏幕上没有动画。程序员必须用 Apple 的 [`ImageIO`][7] 框架将每一帧加载为一个单独的图像并使用 `UIImageView` 的 API： `animatedImages` 、`animationDuration` 以及 `animationRepeatCount` 来完成动画。

这种方法的缺点是无法保证 GIF 的可变帧延迟。

![][8]

具有可变帧延迟的五个帧。

让我们假设帧和元数据的加载代码在 `UIImage+animatedImage`  类别里并考虑如下代码：

```Objective-C
imageView.animationDuration = image.delayTimes[0] * animatedImage.frameCount;
```

![][9]

```Objective-C
imageView.animationDuration = image.totalDelayTimes;
```

![][10]

很明显代码使用第一帧的延迟作为每一帧的延迟或预先假设全部帧延迟的长度，但这两种方式都不会带来很好的效果。

另一种选择是使用 `UIWebView` ，这样 GIF 就由 WebKit 负责解码和显示，就如同桌面浏览器那样。

然而，Web View 没有为在移动设备上播放 GIF 而做优化，经常会降低播放速度。同时也不能很好地控制回放过程或内存占用。

## 实现你的自定义播放

一个用 `UIImageView` 显示并支持可变帧延迟的方式是找到所有帧延迟的[最大公约数][11]并将延迟稍长的帧多放入几个到对应的 `animatedImages` 数组里。

>译者注：就是说，尽管每一帧的延迟都一样，但因为原本延迟长的那些帧因为相应的多放了几个进去，这样播放时，整个动画的感觉就不会变。

![][12]  
槽的持续时间（Slot Duration）由 GCD 决定，是 1 秒。

某个帧第一次显示在屏幕上时，压缩的图像数据会被解码为未压缩的位图形式。这是一个较为消耗 CPU 的操作，因此第一次播放时会变慢。

更棘手的是内存占用；一旦图像被解码，位图就附加到图像对象上并会一直在它的整个生命周期里存在。这将使得我们的 Image View 缓存着所有这些巨大的（注解3）位图数据——这几乎是最糟糕的消耗内存的方式（注解4）。

当 Apple 给 `UIImage` 和 `UIImageView` 添加动画属性时，他们实际上是将其设计为用于小型的 UI 动画，例如转动的加载指示器，而不是用于大型的动态图像。也许某些应用可以摆脱这种方法，但在我们的情况里，这样做会使得我们要为每一个 GIF 添加一个播放按钮而且一次只允许播放一个。这就将 GIF 的乐趣剥除了，对于我们的用户体验来说完全不可接受。

## 帧的按需产出和消费

在内存有限的情况下，若不能存储问题的结果那就只能每次都重新计算。在我们的情况里，我需要在帧被显示之前加载并解码它，并将不在屏幕上的那些帧清除。这就是所谓的[生产者-消费者问题][13]；一个线程生产数据，另一个线程消耗它。我们需要一个生产者产出一个用于视图的帧流，以按需消费它们。生产者在可用内存变少时会节流，而消费者将相应改变帧定时。译者注：大概是说，内存紧张时，动画就会变慢。

![][14]  
重要组件的概述已在 UML 中高亮显示

### FLAnimatedImage

[`FLAnimatedImage`][15] 用 GIF 数据初始化，之后它的工作就是在被通过 `- (UIImage *)imageLazilyCachedAtIndex:(NSUInteger)index` 请求时尽可能快地递发那一帧，而且仅使用小部分内存。

它试着智能地根据图像尺寸和内存情况选择帧缓存的大小：如果是小的 GIF ，我们会试着将所有帧都放在内存里，减轻 CPU 的负担。如果是很大的 GIF，我们会试着通过只缓存足够用于实时回放的帧来降低内存占用。

当系统发出内存警告时，所有的动态图像实例都丢弃所有位于离屏缓存（off-screen buffer）内的帧并倒回到按需解码。过一段时间，它们会再次建立起缓存。如果发生了多次内存警告，它们将会保持在每一帧都按需解码。设计这种行为时，避免最糟糕的用户体验（即应用崩溃）是非常重要的，为此我们宁愿降低播放速度。

### FLAnimatedImageView

[`FLAnimatedImageView`][16] 可接受一个 `FLAnimatedImage` 并在内部使用一个 `CADisplayLink` 实时播放它。

为了避免处理复杂的锁获取（Lock Acquisition），我们将在没有准备好消费者所需数据的情况下让生产者返回 `nil`，并保持显示前一帧。这简化了多线程代码并稍慢一点得到所期望的结果（译者注：大概指返回nil的那一帧），而且能正确地播放动画。

### 设计一个封装良好的插入式组件

`FLAnimatedImage` 直接继承自 `NSObject` ，因为它若继承于 `UIImage` 能获得收益其实很少。另一方面说来，`FLAnimatedImageView` 完全兼容 `UIImageView` 子类并可立即投入使用以取代其位置；设置一个 `UIImage` 或一个 `FLAnimatedImage` 在其上都可以以我们所期望的方式正常工作。所以无论何处我们要显示一个图像，我们就使用 `FLAnimatedImageView` ，然后它就自动正确地处理剩下的事情。

当考虑架构时，创建一个自包含、可重用的组件是非常重要的。多个 `FLAnimatedImage`-`FLAnimatedImageView` 对依然可在没有中央缓存时被使用。它们都尊重系统并且独立地尝试做到最好以成为里面的伟大公民（They're all aware of the system and independently try to do the best to be great citizens.）。

这个模块有大约1000行的代码并试图坚持“做一件事并做好”的[Unix哲学][17]。目前，它已经经过良好测试，可用于生产环境。它依然还有[改进空间][3]，我们欢迎社区的贡献者。

## .gif

Flipboard 的读者已创造了许多动态杂志，例如“[GIF Me A Break][18]”、“[Goals goals goals!][19]” 或 “[Cat GIFs 😹][20]” 。GIF 不只是一个文件格式，它是一种文化，可以说是[ Internet 的原生艺术形式][21]。通过分享此工程挑战以及[源代码][3]，我们希望能贡献它更长久的生命。

==============

1. 这里有个[很长的历史](https://bugzilla.mozilla.org/show_bug.cgi?id=440882)介绍，关于如何限制太快的 GIF。
2. iOS 5 为 `UIImage` 添加了新的初始化方法以便创建一个“动态图像”（由私有类 `_UIAnimatedImage` 支持），但它依然期望其图像还是独立的图像。
3. 一个 1MB 的 GIF 可变成 55MB 的未压缩数据！（800x600 像素 * RGBA 4 字节每像素 * 30 帧）
4. WWDC 2011 session “iOS Performance In Depth” 描述了堆对象只是冰山的一角。这可以用虚拟机跟踪仪进行测量。“脏内存” 是需要监视的数据，它的内存没有映射到文件，因此无法被清除。我们预绘制的图像将在此类别里。注意到“Memory Tag 70”同样来自图像（ImageIO）。


特别感谢 Ryan、Evan、Troy、Eugene、Josh、Charles 以及 Chris 提供想法和改进建议。

=====================

有许多事情我们都没有亲眼所见，但并不能表示我们可以有意或无意地忽略它们。请耐心了解一段历史：[六四事件](http://zh.wikipedia.org/wiki/%E5%85%AD%E5%9B%9B%E4%BA%8B%E4%BB%B6)。

---

欢迎转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！

[1]: http://en.wikipedia.org/wiki/Graphics_Interchange_Format#Animated_GIF "Wikipedia: Animated GIF format description"
[2]: http://inside.flipboard.com/2013/08/14/new-flipboard-update-is-out-with-gifs-for-all-and-top-stories/ "Inside Flipboard: New Flipboard Update Is Out, With GIFs For All and Top Stories"
[3]: https://github.com/Flipboard/FLAnimatedImage "FLAnimatedImage open source on GitHub"
[4]: http://engineering.flipboard.com/assets/animatedgif/flanimatedimage-flipboard.gif
[5]: https://developer.apple.com/library/ios/documentation/UIKit/Reference/UIImage_Class/Reference/Reference.html "Apple Doc: UIImage"
[6]: https://developer.apple.com/library/ios/documentation/uikit/reference/UIImageView_Class/Reference/Reference.html "Apple Doc: UIImageView"
[7]: https://developer.apple.com/library/ios/documentation/graphicsimaging/conceptual/ImageIOGuide/imageio_intro/ikpg_intro.html "Apple Doc: ImageIO"
[8]: http://engineering.flipboard.com/assets/animatedgif/frame-delays1.png
[9]: http://engineering.flipboard.com/assets/animatedgif/frame-delays2.png
[10]: http://engineering.flipboard.com/assets/animatedgif/frame-delays3.png
[11]: http://en.wikipedia.org/wiki/Greatest_common_divisor "Wikipedia: Greatest common divisor"
[12]: http://engineering.flipboard.com/assets/animatedgif/frame-delays4.png
[13]: https://en.wikipedia.org/wiki/Producer–consumer_problem "Wikipedia: Producer-consumer problem"
[14]: http://engineering.flipboard.com/assets/animatedgif/uml.png
[15]: https://github.com/Flipboard/FLAnimatedImage/blob/master/FLAnimatedImageDemo/FLAnimatedImage/FLAnimatedImage.h "Source code: FLAnimatedImage class"
[16]: https://github.com/Flipboard/FLAnimatedImage/blob/master/FLAnimatedImageDemo/FLAnimatedImage/FLAnimatedImageView.h "Source code: FLAnimatedImageView class"
[17]: http://en.wikipedia.org/wiki/Unix_philosophy#McIlroy:_A_Quarter_Century_of_Unix "Wikipedia: Unix philosophy"
[18]: https://flipboard.com/section/gif-me-a-break-bcG3Lj "Flipboard magazine: GIF Me A Break"
[19]: https://flipboard.com/section/goals-goals-goals!-b7DfXm "Flipboard magazine: Goals goals goals!"
[20]: https://flipboard.com/section/cat-gifs-%F0%9F%98%B9-b49Y6v "Flipboard magazine: Cat GIFs"
[21]: https://medium.com/message/af8673796c44 "JIF is the format, GIF is the culture"
[22]: https://flipboard.com
  

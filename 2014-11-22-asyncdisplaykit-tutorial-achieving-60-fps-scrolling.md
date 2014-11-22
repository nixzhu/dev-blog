本文翻译自 [http://www.raywenderlich.com/86365/asyncdisplaykit-tutorial-achieving-60-fps-scrolling](http://www.raywenderlich.com/86365/asyncdisplaykit-tutorial-achieving-60-fps-scrolling) 

原作者：[René Cacheaux](http://www.twitter.com/rcachATX)

译者：[@nixzhu](https://twitter.com/nixzhu)

=================================

# AsyncDisplayKit 教程：达到 60 FPS 的滚动帧率

Facebook 的 [Paper](https://www.facebook.com/paper) 团队给我们带来另一个很棒的库：[AsyncDisplayKit](https://github.com/facebook/AsyncDisplayKit)。这个库能让你通过将图像解码、布局以及渲染操作放在后台线程，从而带来超级响应的用户界面，也就是说不再会因界面卡顿而阻断用户交互。既然这么厉害，那就在本教程里学一下它吧。

例如，对于非常复杂的界面，你可以使用 AsyncDisplayKit 构建它而得到一种如丝般顺滑的，60帧每秒的滑动体验。而平常的 UIKit 优化就不太可能克服这样的性能挑战。

在本教程中，你将从一个初始项目开始，它主要有一个 `UICollectionView` 的滑动问题，而使用 AsyncDisplayKit 将大大提高其滑动性能。一路上，你将学会如何在旧项目中使用 AsyncDisplayKit。

>**注意：**在开始本教程之前，你应该已熟悉 Swift、Core Animation 以及 Core Graphics。

>查看本站的 Swift 教程[Swift Tutorial: A Quick Start](http://www.raywenderlich.com/74438/swift-tutorial-a-quick-start)、[Swift Tutorial Part 2: A Simple iOS App](http://www.raywenderlich.com/74904/swift-tutorial-part-2-simple-ios-app)、[Swift Tutorial Part 3: Tuples, Protocols, Delegates, and Table Views](http://www.raywenderlich.com/75289/swift-tutorial-part-3-tuples-protocols-delegates-table-views)。若要深入，看看 [Core Animation Programming Guide](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html) 以及 [Quartz 2D Programming Guide](https://developer.apple.com/library/mac/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html) 。WWDC 2012 的 session [“iOS App Performance: Graphics and Animations”](https://developer.apple.com/videos/wwdc/2012/#238) 是另外一个很棒的资源，我强烈推荐。你可以多次观看，每次都能学到新东西！

## 开始

开始之前，先看看 [AsyncDisplayKit 的介绍](https://code.facebook.com/posts/721586784561674/introducing-asyncdisplaykit-for-smooth-and-responsive-apps-on-ios/ "here")。以对它有个简要的概念，知道它是要解决什么问题。

准备好了后就[下载初始项目](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/Layers-Start.zip)吧。你需要使用 Xcode 6.1 和 iOS 8.1 SDK 来编译它。

>**注意：**本教程的代码使用 AsyncDisplayKit 1.0 来编写。这个版本已经被包含在初始项目中了。

你要研究的项目是由 `UICollectionView` 制作的卡片式界面来描述不同的雨林动物。每张信息卡包括一个图片、名字以及一个对雨林动物的描述。卡片的背景图是主图片的模糊版。视觉设计的细节保证了文字的清晰易读。

[![Layers app card view](http://cdn5.raywenderlich.com/wp-content/uploads/2014/11/iOS-Simulator-Screen-Shot-7-Nov-2014-21.10.03-281x500.png)](http://cdn4.raywenderlich.com/wp-content/uploads/2014/11/iOS-Simulator-Screen-Shot-7-Nov-2014-21.10.03.png)

在 Xcode 中，打开初始项目里的 _Layers.xcworkspace_ 。

在本教程里，请遵循以下原则以体会 AsyncDisplayKit 的那些十分吸引人的好处。

* 将应用运行在真机上。在模拟器里运行很难看出性能改善。
* 应用是通用的，但在 iPad 上看起来最好。
* 最后，要真正感激这个库能为你所做的事情，请尽量在最旧的能运行 iOS 8.1 的设备上运行本应用。第三代的 iPad 最好，因为它虽有视网膜屏幕，但运行得不是很快。

一旦你选定了设备，那就编译并运行本项目。你会看到如下界面：

[![IMG_0002](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/IMG_0002-375x500.png)](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/IMG_0002.png)

试着滑动 Collection View 并注意那可怜的帧率。在第三代 iPad 上，帧率大概只有 15-20 FPS，实在丢掉太多帧了。在本教程的最后，你能在 60 FPS （或非常接近）的帧率上滑动它。

>**注意：**你所看到的图像都在 App 的 asset 目录里，并不是从网络上获取的。

## 测量响应速度

在一个旧项目中使用 AsyncDisplayKit 前，你应该通过 Instruments 测量你的 UI 的性能，这样才有一个基准线以便对比改动的效果。

最重要的是，你要知道是 **CPU-绑定** 还是 **GPU-绑定**。也就是说，是 CPU 还是 GPU 拉低了应用的帧率。这个信息会告诉你该充分利用 AsyncDisplayKit 的哪个特性以优化应用的性能。

如果你有时间，看看之前提到的 WWDC 2012 session 和/或在真实设备上使用 Instruments 来评估初始项目的时间曲线。滑动性能是 **CPU-绑定** 的。你能猜到是什么原因导致了 Collection View 丢掉这么多帧吗？

>丢帧是因为模糊 cell 的背景图像时阻塞了主线程。


## 为项目准备好使用 AsyncDisplayKit

在旧项目里使用 AsyncDisplayKit，归结起来就是使用 Display Node 层次结构替换视图层次结构和/或 Layer 树。各种 Display Node 是 AsyncDisplayKit 的关键所在。它们位于视图之上，而且是线程安全的，也就是说之前在主线程才能执行的任务现在也可以在非主线程执行。这就能减轻主线程的工作量以执行其他操作，例如处理触摸事件，或如在本应用的情况里，处理 Collection View 的滑动。

这就意味着在本教程里，你的第一步是移除视图层次结构。

### 移除视图层次结构

打开 _RainforestCardCell.swift_ 并删除 `awakeFromNib()` 中所有的 `addSubview(...)` 调用，然后得到如下：

```Swift
override func awakeFromNib() {
  super.awakeFromNib()
  contentView.layer.borderColor =
    UIColor(hue: 0, saturation: 0, brightness: 0.85, alpha: 0.2).CGColor
  contentView.layer.borderWidth = 1
}
```

接下来，替换 `layoutSubviews()` 的内容如下：

```Swift
override func layoutSubviews() {
  super.layoutSubviews()
}
```

再将 `configureCellDisplayWithCardInfo(cardInfo:)` 的内容替换如下：

```Swift
func configureCellDisplayWithCardInfo(cardInfo: RainforestCardInfo) {
  //MARK: Image Size Section
  let image = UIImage(named: cardInfo.imageName)!
  featureImageSizeOptional = image.size
}
```

删除 `RainforestCardCell` 的所有视图属性，只留一个如下：

```Swift
class RainforestCardCell: UICollectionViewCell {
  var featureImageSizeOptional: CGSize?
  ...
}
```

最后，编译并运行，你看到的就全是空空如也的卡片：

[![IMG_0001](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/IMG_0001-375x500.jpg)](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/IMG_0001.jpg)

现在所有的 cell 都空了，滑动起来超级顺滑。你的目标是保证之后添加回取代各视图的 node 后，滑动依然顺滑。

你可用 Instruments 的  Core Animation 模版在真机上检测应用的性能，看看你的改动如何影响帧率。

### 添加一个占位图

打开 _RainforestCardCell.swift_ ，给 `RainforestCardCell` 添加一个可选的 `CALayer` 变量，名为 `placeholderLayer`：

```Swift
class RainforestCardCell: UICollectionViewCell {
  var featureImageSizeOptional: CGSize?
  var placeholderLayer: CALayer!
  ...
}
```

你之所以需要一个占位图是因为显示会异步完成，如果这个过程需要些时间，那用户就会看到空的 cell —— 这并不愉快。就如同如果你要从网络上获取图像，那么就需要用占位图来填充 cell，这能让你的用户知道内容还没有准备好。虽然在我们这种情况里，你是在后台线程绘制而不是从网络下载。

在 `awakeFromNib()` 里，删除 `contentView` 的 border 设置再创建并配置一个 `placeholderLayer`。将其添加到 cell 的 `contentView` 的 Layer 上。现在这个方法如下：

```Swift
override func awakeFromNib() {
  super.awakeFromNib()
 
  placeholderLayer = CALayer()
  placeholderLayer.contents = UIImage(named: "cardPlaceholder")!.CGImage
  placeholderLayer.contentsGravity = kCAGravityCenter
  placeholderLayer.contentsScale = UIScreen.mainScreen().scale
  placeholderLayer.backgroundColor = UIColor(hue: 0, saturation: 0, brightness: 0.85, alpha: 1).CGColor
  contentView.layer.addSublayer(placeholderLayer)
}
```

在 `layoutSubviews()` 里，你需要布局 `placeholderLayer`。替换这个方法为：

```Swift
override func layoutSubviews() {
  super.layoutSubviews()
 
  placeholderLayer?.frame = bounds
}
```

编译并运行，你从虚无的边缘回来了：

[![IMG_0003](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/IMG_0003-375x500.png)](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/IMG_0003.png)

朴素的 CALayer 不是由 UIView 支持的，当它们改变 frame 时，默认会有隐式动画。这就是为何你看到 layer 在布局时放大。要修复这个问题，改动 `layoutSubviews` 如下：

```Swift
override func layoutSubviews() {
  super.layoutSubviews()
 
  CATransaction.begin()
  CATransaction.setValue(kCFBooleanTrue, forKey: kCATransactionDisableActions)
  placeholderLayer?.frame = bounds
  CATransaction.commit()
}
```

编译并运行，问题解决了。

现在占位图不会乱动，不再动画它们的 frame 了。

## 第一个 Node

重建 App 的第一步是给每一个 `UICollectionView` cell 添加一个背景图片 Node，步骤如下：

*   创建、布局并添加一个图像 Node 到 `UICollectionView` cell；
*   处理 cell 重用 Node 和它们的 layer；以及
*   模糊图像 Node

但在做之前，打开 _Layers-Bridging-Header.h_ 并导入 AsyncDisplayKit ：

```Objective-C
#import <AsyncDisplayKit/AsyncDisplayKit.h>
```

这会让所有的 Swift 文件都能访问 AsyncDisplayKit 的各种类。

编译一下，确保没有错误。

### 方向：雨林 Collection View 结构

现在，我们来看看 Collection View 的组成：

*   _View Controller_ ：`RainforestViewController` 没有什么花哨的东西。它只是为所有的雨林卡片获取一个数据数组，并为 `UICollectionView` 实现 Data Source。事实上，你不需要花太多时间到 View Controller 上。
*   _Data Source_ ：大部分时间都将花在 cell 类 `RainforestCardCell` 上。View Controller 出队每个 cell 并将雨林卡片的数据用 `configureCellDisplayWithCardInfo(cardInfo:)` 传给它。cell 就使用这个数据来配置自身。
*   _Cell_ ：在 `configureCellDisplayWithCardInfo(cardInfo:)` 里，cell 创建、配置、布局以及添加 Node 到它自己身上。这就意味着每次 View Controller 出队一个 cell，这个 cell 就会创建并添加给它自己一个新的 Node 层次结构。

如果你使用 View 而不是 Node，那么这样做对于性能来说就不是最佳策略。但因为你可以异步地创建、配置以及布局，而且 Node 也是异步地绘制的，所以这不会是一个问题。真正的难点是在 cell 准备重用时取消任何在进行的异步操作并移除旧 Node 。

> **注意** ：本教程的这个策略来添加 Node 到 cell 还算 OK。对于精通 AsyncDisplayKit 来说，这是很好的第一步。

> 然而，在实际生产中，你最好使用 `ASRangeController` 来缓存你的 Node，这样你就不用每次在 cell 重用时重建它的 Node 层次结构。`ASRangeController` 超出了本教程的范围，但若你想了解更多的信息，看看头文件 _ASRangeController.h_ 的注释吧。

再注意一下：1.1 版的 AsyncDisplayKit （本教程编写时还未放出，但会在此后不久放出）包含有 `ASCollectionView`。使用 `ASCollectionView` 会让本 App 的整个 Collection View 都由 Display Node 控制。而在本教程中，每个 cell 会包含一个 Display Node 层次结构。如上面所解释的，这能工作，但如果使用 `ASCollectionView` 可能会更好。给力的 `ASCollectionView`!

OK，该动手了。

### 添加背景图片 Node

现在你要走一遍用 Node 配置 cell 的过程，一次一步：

打开 _RainforestCardCell.swift_ 并替换 `configureCellDisplayWithCardInfo(cardInfo:)` 为：

```Swift
func configureCellDisplayWithCardInfo(cardInfo: RainforestCardInfo) {
  //MARK: Image Size Section
  let image = UIImage(named: cardInfo.imageName)!
  featureImageSizeOptional = image.size
 
  //MARK: Node Creation Section
  let backgroundImageNode = ASImageNode()
  backgroundImageNode.image = image
  backgroundImageNode.contentMode = .ScaleAspectFill
}
```

这就创建并配置了一个 `ASImageNode` 常量，叫做 `backgroundImageNode`。

>**注意**：确保包含 `//MARK:` 注释，这样更容易看清代码位置。

AsyncDisplayKit 带有好几种 Node 类型，包括 `ASImageNode`，用于显示图片。它相当于 `UIImageView`，除了 `ASImageNode` 是默认异步地解码图片。

添加如下代码到 _configureCellDisplayWithCardInfo(cardInfo:)_ 底部：

```Swift
backgroundImageNode.layerBacked = true
```

这让 `backgroundImageNode` 变为 Layer 支持的 Node。

Node 可由 `UIView` 支持或 `CALayer` 支持。当 Node 需要处理事件时（例如触摸事件），你就要使用  `UIView` 支持的 Node。如果你不需要处理事件，只需要显示一下内容，那使用 Layer 支持的 Node 会更加轻量，因此可以获得一个小的性能提升。

因为本教程的 App 不需要处理事件，所以你可让所有的 Node 都设置为 Layer 支持的。在上面的代码中，由于 `backgroundImageNode` 为 Layer 支持的，AsyncDisplayKit 会创建一个 `CALayer` 用于雨林动物图像内容的显示。

继续 `configureCellDisplayWithCardInfo(cardInfo:)` 并添加如下代码：

```Swift
//MARK: Node Layout Section
backgroundImageNode.frame = FrameCalculator.frameForContainer(featureImageSize: image.size)
```

这里使用 `FrameCalculator` 为 `backgroundImageNode` 布局。

`FrameCalculator`  是一个帮助类，它包装了cell 的布局，为每个 Node 返回 frame。注意所有的东西都是手动布局的， _没有使用 Auto Layout 约束_ 。如果你需要构建自适应布局或者本地化驱动的布局，那就要注意，因为你不能给 Node 添加约束。

接下来，添加如下代码到 `configureCellDisplayWithCardInfo(cardInfo:)` 底部：

```Swift
//MARK: Node Layer and Wrap Up Section
self.contentView.layer.addSublayer(backgroundImageNode.layer)
```

这句将 `backgroundImageNode` 的 Layer 添加到 cell `contentView` 的 Layer 上。

注意，AsyncDisplayKit 会为 `backgroundImageNode` 创建一个 Layer。然而，你必须要将 Node 放到某个 Layer 树中才能在屏幕上显示。这个 Node 会异步地绘制，所以直到绘制完成，它的内容都不会显示，尽管它的 Layer 已经在一个 Layer 树中。

从技术角度来说， Layer 一直都存在。但渲染图像是异步进行的。Layer 初始化时没有内容（例如是透明的）。一旦渲染完成，Layer 的 contents 就会更新为包含图像内容。

在这个点，cell 的 contentView 的 Layer 将会包含两个 Sublayer：一个占位图和 Node 的 Layer。在 Node 完成绘制前，只有占位图会显示。

注意到 `configureCellDisplayWithCardInfo(cardInfo:)` 会在每次 cell 出队时被调用。每次 cell 被回收，这个逻辑会添加一个新的 Sublayer 到 cell 的 `contentView` Layer 上。不要担心，你很快会解决这个问题。

回到 _RainforestCardCell.swift_ 开头，给 `RainforestCardCell` 添加一个 `ASImageNode` 变量存为属性 `backgroundImageNode`，如下：

```Swift
class RainforestCardCell: UICollectionViewCell {
  var featureImageSizeOptional: CGSize?
  var placeholderLayer: CALayer!
  var backgroundImageNode: ASImageNode? ///< ADD THIS LINE
  ...
}
```

你之所以需要这个属性是因为必须要有某个东西将 `backgroundImageNode` 的引用保留住，否则 ARC 就会将其释放，也就不会有任何东西显示出来——即使 Node 的 Layer 在一个 Layer 树中，你依然需要保留 Node。

在 `configureCellDisplayWithCardInfo(cardInfo:)` 底部的 _Node Layer and Wrap Up Section_ ，设置 cell 新的 `backgroundImageNode`  为之前的 `backgroundImageNode`：

```Swift
self.backgroundImageNode = backgroundImageNode
```

下面是完整的 `configureCellDisplayWithCardInfo(cardInfo:)` 方法：

```Swift
func configureCellDisplayWithCardInfo(cardInfo: RainforestCardInfo) {
  //MARK: Image Size Section
  let image = UIImage(named: cardInfo.imageName)!
  featureImageSizeOptional = image.size
 
  //MARK: Node Creation Section
  let backgroundImageNode = ASImageNode()
  backgroundImageNode.image = image
  backgroundImageNode.contentMode = .ScaleAspectFill
  backgroundImageNode.layerBacked = true
 
  //MARK: Node Layout Section
  backgroundImageNode.frame = FrameCalculator.frameForContainer(featureImageSize: image.size)
 
  //MARK: Node Layer and Wrap Up Section
  self.contentView.layer.addSublayer(backgroundImageNode.layer)
  self.backgroundImageNode = backgroundImageNode
}
```

编译并运行，观察 AsyncDisplayKit 是如何异步地使用图像设置 Layer 的 contents 的。这能让你在 CPU 还在绘制 Layer 的内容的同时上下滑动界面。

[![IMG_0006](http://cdn5.raywenderlich.com/wp-content/uploads/2014/10/IMG_0006-375x500.png)](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/IMG_0006.png)

如果你运行在旧设备上，注意图像是如何弹出到位置——这是爆米花特效，但不总是让人喜欢！本教程的最后一节会搞定这个不令人愉快的弹出效果，给你展示如何让图像自然地淡入，如同摇滚巨星。

如之前所讨论的，新的 Node 会在每次 cell 被重用时创建。这并不很理想，因为这意味着新的 Layer 会在每次 cell 被重用时加入。

如果你想看看 Sublayer 堆积太多的影响，那就不停的滑上滑下多次，然后加断点打印出 cell 的 contentView 的 Layer 的 `sublayers` 属性。你会看到很多 Layer，这并不好。

### 处理 Cell 重用

继续 _RainforestCardCell.swift_ ，给 `RainforestCardCell` 添加一个叫做 `contentLayer` 的 `CALayer` 属性。这个属性也是一个可选类型：

```Swift
class RainforestCardCell: UICollectionViewCell {
  var featureImageSizeOptional: CGSize?
  var placeholderLayer: CALayer!
  var backgroundImageNode: ASImageNode?
  var contentLayer: CALayer? ///< ADD THIS LINE
  ...
}
```

你将使用此属性去移除 cell 的 contentView 的 Layer 树中旧的 Node Layer。虽然你可以简单地保留 Node 并访问其 Layer 属性，但上面的写法更加明确。

添加如下代码到 `configureCellDisplayWithCardInfo(cardInfo:)` 结尾：

```Swift
self.contentLayer = backgroundImageNode.layer
```

这句让  `backgroundImageNode` 的 Layer 保留到 `contentLayer` 属性。

替换 `prepareForReuse()` 的实现如下：

```Swift
override func prepareForReuse() {
  super.prepareForReuse()
  backgroundImageNode?.preventOrCancelDisplay = true
}
```

因为 AsyncDisplayKit 能够异步地绘制 Node，所以 Node 让你能预防从头绘制或取消任何在进行的绘制。无论是你需要预防或取消绘制，都可将 `preventOrCancelDisplay` 设置为 `true`，如上面代码所示。在本例中，你要在 cell 被重用前取消任何正在进行的绘制活动。

接下来，添加如下代码到 `prepareForReuse()` 尾部：

```Swift
contentLayer?.removeFromSuperlayer()
```

这将  `contentLayer` 从其 Superlayer （也就是 `contentView` 的 Layer）中移除。

每次一个 cell 被回收时，这个代码就移除 Node 的旧 Layer ，因而解决了堆积问题。所以在任何时间，你的 Node 最多只有两个 Sublayer：占位图和 Node 的 Layer。

接下来添加如下代码到 `prepareForReuse()` 尾部：

```Swift
contentLayer = nil
backgroundImageNode = nil
```

这确保 cell 释放它们的引用，这样如有必要，ARC 才好做清理工作。

编译并运行。这次，没有 Sublayer 会堆积的问题，且所有不必要的绘制都会被取消。

[![IMG_0006](http://cdn5.raywenderlich.com/wp-content/uploads/2014/10/IMG_0006-375x500.png)](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/IMG_0006.png)

是时候来点儿模糊效果了，Baby，模糊哦。

[![asyncscroll](http://cdn5.raywenderlich.com/wp-content/uploads/2014/11/asyncscroll.png)](http://cdn5.raywenderlich.com/wp-content/uploads/2014/11/asyncscroll.png)

### 模糊图像

要模糊图像，你要添加一个额外的步骤到图像 Node 的显示过程里。

继续 _RainforestCardCell.swift_ ，在 `configureCellDisplayWithCardInfo(cardInfo:)` 的设置  `backgroundImageNode.layerBacked` 的后面，添加如下代码：

```Swift
backgroundImageNode.imageModificationBlock = { input in
  if input == nil {
    return input
  }
  if let blurredImage = input.applyBlurWithRadius(
    30,
    tintColor: UIColor(white: 0.5, alpha: 0.3),
    saturationDeltaFactor: 1.8,
    maskImage: nil, 
    didCancel:{ return false }) {
      return blurredImage
  } else {
    return image
  }
}
```

`ASImageNode` 的 `imageModificationBlock` 给你一个机会在显示之前去处理底层的图像。这是非常实用的功能，它让你能对图像 Node 做一些操作，例如添加滤镜等。

在上面的代码里，你使用 `imageModificationBlock` 来为 cell 的背景图像应用模糊效果。关键点就是图像 Node 将会绘制它的内容并在后台执行这个闭包，而主线程依然顺滑流畅。这个闭包接受原始的  `UIImage` 并返回一个修改过的  `UIImage`。

上面的代码使用了  `UIImage` 的模糊 category，它由 Apple 在 WWDC 2013 提供，使用了 Accelerate framework 在 CPU 上模糊图像。因为模糊会消耗很多时间和内存，这个版本的 category 被修改为包含了取消机制。这个模糊方法将定期调用 `didCancel` 闭包来决定是否应该要停止模糊。

现在，上面的代码给  `didCancel` 简单地返回 `false`。之后你会重写 `didCancel` 闭包。

>**注意**：还记得第一次运行 App 时 Collection View 那可怜的滑动效果吗？模糊方法阻塞了主线程。通过使用 AsyncDisplayKit 将模糊放入后台，你就大幅度地提高了 Collection View 的滑动性能。简直天壤之别。

编译并运行，观察模糊效果：

[![IMG_0009](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/IMG_0009-375x500.png)](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/IMG_0009.png)

注意你可以如何非常流畅地滑动 Collection View。

当 Collection View 出队一个 cell 时，一个模糊操作将开始于后台线程。当用户快速滑动时，Collection View 会重用每个 cell 多次，并开始许多模糊操作。我们的目标是在 cell 准备被重用时取消正在进行中的模糊操作。

你已经在 `prepareForReuse()` 里取消了 Node 的绘制操作 ，但一旦控制被移交给处理你图像修改的闭包，那就是你的责任来处理 Node 的 `preventOrCancelDisplay` 设置，你现在就要做。

### 取消模糊操作

要取消进行中的模糊操作，你需要实现模糊方法的 `didCancel` 闭包。

添加一个捕捉列表到 `imageModificationBlock` 以捕捉一个 `backgroundImageNode` 的 weak 引用：

```Swift
backgroundImageNode.imageModificationBlock = { [weak backgroundImageNode] input in
   ...
}
```

你需要 weak 引用来避免闭包和图像 Node 之间的保留环问题。你将使用这个 weak  `backgroundImageNode` 来确定是否要取消模糊操作。

是时候构建模糊取消闭包了。添加下面代码到 `imageModificationBlock`：

```Swift
backgroundImageNode.imageModificationBlock = { [weak backgroundImageNode] input in
  if input == nil {
    return input
  }
 
  // ADD FROM HERE...
  let didCancelBlur: () -> Bool = {
    var isCancelled = true
    // 1
    if let strongBackgroundImageNode = backgroundImageNode {
      // 2
      let isCancelledClosure = {
        isCancelled = strongBackgroundImageNode.preventOrCancelDisplay
      }
 
      // 3
      if NSThread.isMainThread() {
        isCancelledClosure()
      } else {
        dispatch_sync(dispatch_get_main_queue(), isCancelledClosure)
      }
    }
    return isCancelled
  }
  // ...TO HERE
 
  ...
}
```

下面解释一下这些代码：

1. 得到 `backgroundImageNode` 的 strong 引用，准备用其干活。如果 `backgroundImageNode` 在本次运行时消失，那么 `isCancelled` 将保持为 true，然后模糊操作会被取消。如果没有 Node 需要显示，自然没有必要继续模糊操作。
2. 在此你将操作取消检查包在闭包里，因为一旦 Node 创建它的 Layer 或 View，那就只能在主线程访问 Node 的属性。由于你需要访问 `preventOrCancelDisplay`，所以你必须在主线程检查。
3. 最后，确保  `isCancelledClosure` 是在主线程运行，无论是已在主线程而直接运行，还是不在主线程而通过  `dispatch_sync` 来调度。它必须是一个同步的调度，因为我们需要闭包完成，并在 `didCancelBlur` 闭包返回之前设置 `isCancelled`。

在调用 `applyBlurWithRadius(...)` 中，修改传递给 `didCancel` 的参数，替换一直返回  `false` 的闭包为你刚才定义并保留在 `didCancelBlur` 的闭包。

```Swift
if let blurredImage = input.applyBlurWithRadius(
  30,
  tintColor: UIColor(white: 0.5, alpha: 0.3),
  saturationDeltaFactor: 1.8,
  maskImage: nil,
  didCancel: didCancelBlur) {
  ...
}
```

编译并运行。你看你不会注意到太多差别，但现在任何在 cell 离开屏幕时还未完成的模糊都会被取消了。这就意味着设备比之前做得更少。你可能观察到轻微的性能提升，特别是在较慢的设备如第三代 iPad 上运行时。

[![IMG_0009](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/IMG_0009-375x500.png)](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/IMG_0009.png)

当然，若没有东西在前面，背景就不是真正的背景！你的卡片需要内容。通过下面四个小节，你将学会：

*   创建一个容器 Node，它将所有的 Subnode 绘制到一个单独的 `CALayer` 里；
*   构建一个 Node 层次结构；
*   创建一个自定义的 `ASDisplayNode`  子类；并
*   在后台构建并布局 Node 层次结构；

做完这些，你就会得到一个看起来和添加 AsyncDisplayKit 之前一样的 App，但有着黄油般顺滑的滑动体验。

## 栅格化的容器 Node

直到现在，你一直在操作 cell 内的一个单独的 Node。接下来，你将创建一个容器 Node，它会包含所有的卡片内容。

### 添加一个容器 Node

继续 _RainforestCardCell.swift_ ，在 `configureCellDisplayWithCardInfo(cardInfo:)` 的  `backgroundImageNode.imageModificationBlock` 后面以及 _Node Layout Section_ 前面添加如下代码：

```Swift
//MARK: Container Node Creation Section
let containerNode = ASDisplayNode()
containerNode.layerBacked = true
containerNode.shouldRasterizeDescendants = true
containerNode.borderColor = UIColor(hue: 0, saturation: 0, brightness: 0.85, alpha: 0.2).CGColor
containerNode.borderWidth = 1
```

这就创建并配置了一个叫做 `containerNode` 的 `ASDisplayNode` 常量。注意这个容器的 `shouldRasterizeDescendants`，这是一个关于节点如何工作的提示以及一个如何让它们工作得更好地机会。

如单词 “descendants（子孙）” 所暗示的，你可以创建 AsyncDisplayKit Node 的层次结构或树，就如你可以创建 Core Animation Layer 的层次结构一样。例如，如果你有一个都是 Layer 支持的 Node 层次结构，那么 AsyncDisplayKit 将会为每个 Node 创建一个分离的 `CALayer`，Layer 层次结构将会和 Node 层次结构一样，如同镜像。

这听起来很熟悉：它类似于当你使用普通的 UIKit 时，Layer 层次结构镜像于 View 层次结构。然而，这个 Layer 的栈有一些不同的效果：

*   首先，因为是异步渲染，你就不会看到每个 Layer 一个接一个地显示。当 AsyncDisplayKit 绘制完成每个 Layer，它马上制作 Layer 的显示内容。所以如果你有一个 Layer 的绘制比其他 Layer 耗时更长，那么它将会在它们之后显示。用户会看到零碎的 Layer 组件，这个过程通常是不可见的，因为 Core Animation 会在显示任何东西之前重绘所有必须的 Layer 。
*   第二，有许多 Layer 能够引起性能问题。每个 `CALayer` 都需要一个支持存储来保存它的像素位图和内容。同样，Core Animation 必须将每个 Layer 通过 XPC 发给渲染服务器。最后，渲染服务器可能需要重绘一些 Layer 以复合它们，例如在混合 Layer 时。总的来说，更多的 Layer 意味着 Core Animation 更多的工作。所以限制 Layer 使用的数量有许多不同的好处。

为了解决这个问题，AsyncDisplayKit 有一个方便的特性：它允许你绘制一个 Node 层次结构到一个单独的 Layer 容器里。这就是 `shouldRasterizeDescendants` 所做的。当你设置它，那在完成所有的 Subnode 的绘制之前，`ASDisplayNode` 将不会设置 Layer 的 contents。

所以在之前的步骤里，设置容器 Node 的 `shouldRasterizeDescendants` 为 `true` 有两个好处：

*   它确保卡片一次显示所有的 Node，如同旧的同步绘制；
*   而且它通过栅格化 Layer 栈为单个 Layer 并较少未来的合成而提高了效率。

不足之处是，由于你将所有的 Layer 放入一个位图，你就不能在之后单独动画某个 Node 了。

要获得更多信息，请看 `shouldRasterizeDescendants` 在头文件 _ASDisplayNode.h_ 里的注释。

接下来，在 _Container Node Creation Section_ 后，添加 `backgroundImageNode` 为 `containerNode` 的 Subnode：

```Swift
//MARK: Node Hierarchy Section
containerNode.addSubnode(backgroundImageNode)
```

>**注意**：添加 Node 的顺序很重要，就如同 subview 和 sublayer。最先添加的 Node 会被之后添加的阻挡显示。

替换 _Node Layout Section_ 的第一行为：

```Swift
//MARK: Node Layout Section
containerNode.frame = FrameCalculator.frameForContainer(featureImageSize: image.size)
```

最后，使用 `FrameCalculator` 布局 `backgroundImageNode`：

```Swift
backgroundImageNode.frame = FrameCalculator.frameForBackgroundImage(
  containerBounds: containerNode.bounds)
```

这设置 `backgroundImageNode` 填满整个 `containerNode`。

你几乎完成了新的 Node 层次结构，但首先你需要正确地设置 Layer 层次结构，因为容器 Node 现在是根。

### 管理容器 Node 的 Layer

在  _Node Layer and Wrap Up Section_ ，将 `backgroundImageNode` 的 Layer 添加到 `containerNode` 的 Layer 上而不是 `contentView` 的 Layer 上：

```Swift
// Replace the following line...
// self.contentView.layer.addSublayer(backgroundImageNode.layer)
// ...with this line:
self.contentView.layer.addSublayer(containerNode.layer)
```

删除下面的  `backgroundImageNode` 保留：

```Swift
self.backgroundImageNode = backgroundImageNode
```

因为 cell 只需要单独保留容器 Node ，所以你要移除 `backgroundImageNode` 属性。

不再设置 cell 的 `contentLayer` 属性为  `backgroundImageNode` 的 Layer，现在将其设置为 `containerNode` 的 Layer：

```Swift
// Replace the following line...
// self.contentLayer = backgroundImageNode.layer
// ...with this line:
self.contentLayer = containerNode.layer
```

给 `RainforestCardCell` 添加一个可选的 `ASDisplayNode` 实例存储为属性 `containerNode`：

```Swift
class RainforestCardCell: UICollectionViewCell {
  var featureImageSizeOptional: CGSize?
  var placeholderLayer: CALayer!
  var backgroundImageNode: ASImageNode?
  var contentLayer: CALayer?
  var containerNode: ASDisplayNode? ///< ADD THIS LINE
  ...
}
```

记住你需要保留你自己的 Node ，如果你不这么做它们就会被立即释放。

回到 `configureCellDisplayWithCardInfo(cardInfo:)`，在 _Node Layer and Wrap Up Section_ 最后，设置 `containerNode` 属性为 `containerNode`  常量：

```Swift
self.containerNode = containerNode
```

编译并运行。模糊的图像将会再此显示！但还有最后一件事要去改变，因为现在有了新的 Node 层次结构。回忆之前 cell 重用时你将图像停止显示。现在你需要让整个 Node 层次结构停止显示。

### 在新的 Node 层次结构上处理 Cell 重用

继续 _RainforestCardCell.swift_ ，在 `prepareForReuse()` 里，替换设置 `backgroundImageNode.preventOrCancelDisplay` 为在 `containerNode` 上调用 `recursiveSetPreventOrCancelDisplay(...)` 并传递 `true`：

```Swift
override func prepareForReuse() {
  super.prepareForReuse()
 
  // Replace this line...
  // backgroundImageNode?.preventOrCancelDisplay = true
  // ...with this line:
  containerNode?.recursiveSetPreventOrCancelDisplay(true)
 
  contentLayer?.removeFromSuperlayer()
  ...
}
```

当你要取消整个 Node 层次结构的绘制，就使用 `recursiveSetPreventOrCancelDisplay()`。这个方法将会设置这个 Node 以及其所有子 Node 的 `preventOrCancelDisplay` 属性，无论 `true` 或 `false`。

接下来，依然在 `prepareForReuse()`，用设置  `containerNode` 为 `nil` 替换设置 `backgroundImageNode` 为 `nil`：

```Swift
override func prepareForReuse() {
  ...
  contentLayer = nil
 
  // Replace this line...
  // backgroundImageNode = nil
  // ...with this line:
  containerNode = nil
}
```

移除 `RainforestCardCell` 的 `backgroundImageNode` 属性：

```Swift
class RainforestCardCell: UICollectionViewCell {
  var featureImageSizeOptional: CGSize?
  var placeholderLayer: CALayer!
  // var backgroundImageNode: ASImageNode? ///< REMOVE THIS LINE
  var contentLayer: CALayer?
  var containerNode: ASDisplayNode?
  ...
}
```

编译并运行。这个 App 就如之前一样，但现在你的图像 Node 在容器 Node 内，而重用依然和它应有的方式一样。

[![IMG_0009](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/IMG_0009-375x500.png)](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/IMG_0009.png)

## Cell 内容

目前为止你有了一个 Node 层次结构，但容器内还只有一个 Node——图像 Node。现在是时候设置 Node 层次结构去复制在添加 AsyncDisplayKit 之前时应用的视图层次结构了。这意味着添加 text 和一个未模糊的特征图像。

### 添加特征图像

我们要添加特征图像了，它是一个未模糊的图像，显示在卡片的顶部。

打开 _RainforestCardCell.swift_  并找到 `configureCellDisplayWithCardInfo(cardInfo:)`。在 _Node Creation Section_ 的底部，添加如下代码：

```Swift
let featureImageNode = ASImageNode()
featureImageNode.layerBacked = true
featureImageNode.contentMode = .ScaleAspectFit
featureImageNode.image = image
```

这会创建并配置一个叫做 `featureImageNode` 的 `ASImageNode` 常量。它被设置为 Layer 支持的，放大以适用，并设置显示图像，这次不需要模糊。

在 _Node Hierarchy Section_ 的最后，添加 `featureImageNode` 为 `containerNode` 的 Subnode：

```Swift
containerNode.addSubnode(featureImageNode)
```

你正在用更多 Node 填充容器哦！

在 _Node Layout Section_ ，使用 `FrameCalculator` 布局  `featureImageNode`：

```Swift
featureImageNode.frame = FrameCalculator.frameForFeatureImage(
  featureImageSize: image.size,
  containerFrameWidth: containerNode.frame.size.width)
```

编译并运行。你就会看到特征图像在卡片的顶部出现，位于模糊图像的上方。注意特征图像和模糊图像是如何在同一时间跳出。这是你之前添加的 `shouldRasterizeDescendants` 在起作用。

[![IMG_0015](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/IMG_0015-375x500.png)](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/IMG_0015.png)

### 添加 Title 文本

接下来添加文字 Label，以显示动物的名字和描述。首先来动物名字吧。

继续 `configureCellDisplayWithCardInfo(cardInfo:)`，找到 _Node Creation Section_ 。添加下列代码到这节尾部，就在创建 `featureImageNode` 之后：

```Swift
let titleTextNode = ASTextNode()
titleTextNode.layerBacked = true
titleTextNode.backgroundColor = UIColor.clearColor()
titleTextNode.attributedString = NSAttributedString.attributedStringForTitleText(cardInfo.name)
```

这就创建了一个叫做 `titleTextNode` 的 `ASTextNode` 常量。

`ASTextNode` 是另一个 AsyncDisplayKit 提供的 Node 子类，其用于显示文本。它是一个具有 `UILabel` 效果的 Node。它接受一个 attributedString，由 TextKit 支持，有许多特性如文本链接。要学到更多关于这个 Node 的功能，去看 _ASTextNode.h_ 吧。

初始项目包含有一个 `NSAttributedString` 的扩展，它提供了一个工厂方法去生成一个属性字符串用于 Title 和 Description 文本以显示在雨林卡片上。上面的代码使用了这个扩展的 `attributedStringForTitleText(...)` 方法。

现在，在 _Node Hierarchy Section_ 底部，添加如下代码：

```Swift
containerNode.addSubnode(titleTextNode)
```

这就添加了 `titleTextNode` 到 Node 层次结构里。它将位于特征图像和背景图像之上，因为它在它们之后添加。

在 _Node Layout Section_ 底部添加如下代码：

```Swift
titleTextNode.frame = FrameCalculator.frameForTitleText(
  containerBounds: containerNode.bounds,
  featureImageFrame: featureImageNode.frame)
```

一样使用 `FrameCalculator` 布局 `titleTextNode`，就像 `backgroundImageNode` 和 `featureImageNode` 那样。

编译并运行。你就有了一个 Title 显示在特征图像的顶部。再次说明， Label 只会在整个 cell 准备好渲染时才渲染。

[![IMG_0017](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/IMG_0017-375x500.png)](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/IMG_0017.png)

### 添加 Description 文本

添加一个有着 Description 文本的 Node 和添加 Title 文本的 Node 类似。

回到 `configureCellDisplayWithCardInfo(cardInfo:)` ，在 _Node Creation Section_ 最后，添加如下代码。就在之前创建 `titleTextNode` 的语句之后：

```Swift
let descriptionTextNode = ASTextNode()
descriptionTextNode.layerBacked = true
descriptionTextNode.backgroundColor = UIColor.clearColor()
descriptionTextNode.attributedString = 
  NSAttributedString.attributedStringForDescriptionText(cardInfo.description)
```

这就创建并配置了一个叫做 `descriptionTextNode` 的 `ASTextNode` 实例。

在  _Node Hierarchy Section_ 最后，添加 `descriptionTextNode` 到 `containerNode`：

```Swift
containerNode.addSubnode(descriptionTextNode)
```

在 _Node Layout Section_ ，一样使用 `FrameCalculator` 布局 `descriptionTextNode`：

```Swift
descriptionTextNode.frame = FrameCalculator.frameForDescriptionText(
  containerBounds: containerNode.bounds,
  featureImageFrame: featureImageNode.frame)
```

编译并运行。现在你能看到 Description 文本了。

[![IMG_0018](http://cdn1.raywenderlich.com/wp-content/uploads/2014/10/IMG_0018-375x500.png)](http://cdn3.raywenderlich.com/wp-content/uploads/2014/10/IMG_0018.png)

## Custom Node Subclasses 自定义 Node 子类

目前为止，你使用了 `ASImageNode` 和 `ASTextNode`。这会带你走很远，但有些时候你需要你自己的 Node，就如同某些时候在传统的 UIKit 编程里你需要自己的 View 一样。

### 创建梯度 Node 类

接下来，你将给 _GradientView.swift_ 添加 Core Graphics 代码来构建一个自定义的梯度 Display Node。这会被用于创建一个绘制梯度的自定义 Node 。梯度图会显示在特征图像的底部以便让 Title 看起来更加明显。

打开 _Layers-Bridging-Header.h_ 并添加如下代码：

```Objective-C
#import <AsyncDisplayKit/_ASDisplayLayer.h>
```

需这一步是因为这个类没有包含在库的主头文件里。你在子类化任何 `ASDisplayNode` 或 `_ASDisplayLayer` 时都需要访问这个类。


菜单 _File\New\File…_ 。选择 _iOS\Source\Cocoa Touch Class_ 。命名类为 _GradientNode_ 并使其作为 _ASDisplayNode_ 的子类。选择 _Swift_ 语言并点击 _Next_ 。保存文件再打开 _GradientNode.swift_ 。

添加如下方法到这个类：

```Swift
class func drawRect(bounds: CGRect, withParameters parameters: NSObjectProtocol!,
    isCancelled isCancelledBlock: asdisplaynode_iscancelled_block_t!, isRasterizing: Bool) {
 
}
```

如同 `UIView` 或 `CALayer`，你可以子类化 `ASDisplayNode` 去做自定义绘制。你可以使用如同用于 `UIView` 的 Layer 或单独的 `CALayer` 的绘制代码，这取决于客户 Node 如何配置 Node。查看 _ASDisplayNode+Subclasses.h_ 获取更多关于子类化 `ASDisplayNode` 的信息。

进一步，`ASDisplayNode` 的绘制方法比在 `UIView` 和 `CALayer` 里的接受更多参数，给你提供方法少做工作，并更有效率。

要为你的自定义 Display Node 填充内容，你需要实现来自 `_ASDisplayLayerDelegate` 协议的 `drawRect(...)` 或 `displayWithParameters(...)`。在继续之前，看看 _`_ASDisplayLayer.h`_ 得到这个方法和它们参数的信息。搜索 `_ASDisplayLayerDelegate`。重点看看头文件注释里关于 `drawRect(...)` 的描述。

因为梯度图位于特征图的上方，使用 Core Graphics 绘制，所以你需要使用 `drawRect(...)` 。

打开 _GradientView.swift_ 并拷贝 `drawRect(...)` 的内容到 _GradientNode.swift_ 的 `drawRect(...)`，如下：

```Swift
class func drawRect(bounds: CGRect, withParameters parameters: NSObjectProtocol!,
    isCancelled isCancelledBlock: asdisplaynode_iscancelled_block_t!, isRasterizing: Bool) {
  let myContext = UIGraphicsGetCurrentContext()
  CGContextSaveGState(myContext)
  CGContextClipToRect(myContext, bounds)
 
  let componentCount: UInt = 2
  let locations: [CGFloat] = [0.0, 1.0]
  let components: [CGFloat] = [0.0, 0.0, 0.0, 1.0,
    0.0, 0.0, 0.0, 0.0]
  let myColorSpace = CGColorSpaceCreateDeviceRGB()
  let myGradient = CGGradientCreateWithColorComponents(myColorSpace, components,
    locations, componentCount)
 
  let myStartPoint = CGPoint(x: bounds.midX, y: bounds.maxY)
  let myEndPoint = CGPoint(x: bounds.midX, y: bounds.midY)
  CGContextDrawLinearGradient(myContext, myGradient, myStartPoint,
    myEndPoint, UInt32(kCGGradientDrawsAfterEndLocation))
 
  CGContextRestoreGState(myContext)
}
```

然后删除 _GradientView.swift_，编译并确保没有错误。

### 添加梯度 Node

打开 _RainforestCardCell.swift_ 并找到 `configureCellDisplayWithCardInfo(cardInfo:)`。在 _Node Creation Section_ 底部，添加如下代码，就在创建 `descriptionTextNode` 的代码之后：

```Swift
let gradientNode = GradientNode()
gradientNode.opaque = false
gradientNode.layerBacked = true
```

这就创建了一个叫做 `gradientNode` 的 `GradientNode` 常量。

在 _Node Hierarchy Section_，在添加 `featureImageNode` 那样下面，添加 `gradientNode` 到 `containerNode`：

```Swift
//MARK: Node Hierarchy Section
containerNode.addSubnode(backgroundImageNode)
containerNode.addSubnode(featureImageNode)
containerNode.addSubnode(gradientNode) ///< ADD THIS LINE
containerNode.addSubnode(titleTextNode)
containerNode.addSubnode(descriptionTextNode)
```

梯度 Node 需要这个位置才能在特征图之上，Title 之下。

然后添加如下代码到 _Node Layout Section_ 底部：

```Swift
gradientNode.frame = FrameCalculator.frameForGradient(
  featureImageFrame: featureImageNode.frame)
```

编译并运行。你将看到梯度在特征图的底部。Title 确实看得更清楚了！

[![IMG_0019](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/IMG_0019-375x500.png)](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/IMG_0019.png)

### 爆米花特效

如之前提到的，cell 的 Node 内容会在完成绘制时“弹出”。这不是很理想。所以让我们继续，以修复这个问题。但首先，更加深入 AsyncDisplayKit 以看看它是如何工作的。

在 `configureCellDisplayWithCardInfo(cardInfo:)` 的 _Container Node Creation Section_ ，关闭容器 Node 的 `shouldRasterizeDescendants`： 

```Swift
containerNode.shouldRasterizeDescendants = false
```

编译并运行。你会注意到现在容器层次结构里不同的 Node 一个接一个的弹出。你会看到文字弹出，然后是特征图，然后是模糊背景图。

当 `shouldRasterizeDescendants` 关闭后，AsyncDisplayKit 就不是绘制一个容器 Layer 了，它会创建一个镜像卡片 Node 层次结构的 Layer 树。记得爆米花特效存在是因为每个 Layer 都在它绘制结束后立即出现，而某些 Layer 比另外一个花费更多时间在绘制上。

这不是我们所需要的，但它描述了 AsyncDisplayKit 的工作方式。我们不想要这个行为，所以还是将 `shouldRasterizeDescendants` 打开：

```Swift
containerNode.shouldRasterizeDescendants = true
```

编译并运行。又回到整个 cell 在其渲染结束后弹出了。

该重新思考如何摆脱爆米花特效了。但首先，让我们看看 Node 在后台如何构造。

## 在后台构造 Node

除了异步地绘制，使用 AsyncDisplayKit，你同样可以异步地创建、配置以及布局。深呼吸一下，因为这就是你接下来要做的事情。

### 创建一个 Node 构造操作（Operation）

你要将 Node 层次结构的构造包装到一个 NSOperation 中。这样做很棒，因为这个操作能很容易的在不同的操作队列上执行，包括后台队列。

打开  _RainforestCardCell.swift_ 。然后添加如下方法：

```Swift
func nodeConstructionOperationWithCardInfo(cardInfo: RainforestCardInfo, image: UIImage) -> NSOperation {
  let nodeConstructionOperation = NSBlockOperation()
  nodeConstructionOperation.addExecutionBlock { 
    // TODO: Add node hierarchy construction
  }
  return nodeConstructionOperation
}
```

绘制并不是唯一会拖慢主线程的操作。对于复杂的屏幕，布局计算也有可能变的昂贵。目前为止，本教程当前状态的项目，一个缓慢的 Node 布局会引起  Collection View 丢帧。

60 FPS 意味着你有大约 17ms 的时间让你的 cell 准备好显示，否则一个或多个帧就会被丢掉。这在 Table View 和 Collection View 有很复杂的 cell 时是非常常见的，滑动时丢帧就是这个原因。

AsyncDisplayKit 前来救援！

你将使用上面的 `nodeConstructionOperation` 将所有 Node 层次结构构造以及布局从主线程剥离并放入后台 `NSOperationQueue`，进一步确保 Collection View 能尽量以接近 60 FPS 的帧率滑动。

>**警告：**你可以在后台访问并设置 Node 的属性，但只能在 Node 的 Layer 或 View 被创建之前，也就是当你第一次访问 Node 的 Layer 或 View 属性时。

>一旦 Node 的 Layer 或 View 被创建，你必须在主线程才能访问和设置 Node 的属性，因为 Node 将会转发这些调用到它的 Layer 或 View。如果你得到一个崩溃 log 说“Incorrect display node thread affinity”，那就意味着在创建 Node 的 Layer 或 View 之后，你依然尝试在后台访问或设置 Node 的属性。

修改 `nodeConstructionOperation` 操作 Block 的内容如下：

```Swift
nodeConstructionOperation.addExecutionBlock {
  [weak self, unowned nodeConstructionOperation] in
  if nodeConstructionOperation.cancelled {
    return
  }
  if let strongSelf = self {
    // TODO: Add node hierarchy construction
  }
}
```

在这个操作运行时，cell 可能已经被释放了。在那种情况下，你不需要做任何工作。类似的，如果操作被取消了，那一样也没有工作要做了。

之所以对 nodeConstructionOperation` 使用 _unowned_  引用是为了避免在操作和执行闭包之间产生保留环。

现在找到 `configureCellDisplayWithCardInfo(cardInfo:)`。将任何在 _Image Size Section_ 之后的代码移动到 `nodeConstructionOperation` 的执行闭包里。将代码放在 `strongSelf` 的条件语句里，即TODO的位置。之后 `configureCellDisplayWithCardInfo(cardInfo:)` 将看起来如下：

```Swift
func configureCellDisplayWithCardInfo(cardInfo: RainforestCardInfo) {
  //MARK: Image Size Section
  let image = UIImage(named: cardInfo.imageName)!
  featureImageSizeOptional = image.size
}
```

目前，你会有一些编译错误。这是因为操作 Block 里的 `self` 是 weak 引用，因此是可选的。但你有一个 self 的 strong 引用，因为代码在可选绑定语句内。所以替换错误的几行成下面的样子：

```Swift
strongSelf.contentView.layer.addSublayer(containerNode.layer)
strongSelf.contentLayer = containerNode.layer
strongSelf.containerNode = containerNode
```

最后，添加如下代码到你刚改动的三行之下：

```Swift
containerNode.setNeedsDisplay()
```

编译确保没有错误。如果你现在运行，那么只有占位图会显示，因为 Node 的创建操作还没有实际使用。让我们来添加它。

### 使用 Node 创建操作

打开 _RainforestCardCell.swift_ 并添加如下属性：

```Swift
class RainforestCardCell: UICollectionViewCell {
  var featureImageSizeOptional: CGSize?
  var placeholderLayer: CALayer!
  var backgroundImageNode: ASImageNode?
  var contentLayer: CALayer?
  var containerNode: ASDisplayNode?
  var nodeConstructionOperation: NSOperation? ///< ADD THIS LINE
  ...
}
```

这就添加了一个叫做 `nodeConstructionOperation` 的可选属性

当 cell 准备回收时，你会使用这个属性去取消 Node 的构造。这会在用户非常快速地滑动 Collection View 时发生，特别是如果布局还需要一些计算时间的话。

在 `prepareForReuse()` 添加如下指示的代码：

```Swift
override func prepareForReuse() {
  super.prepareForReuse()
 
  // ADD FROM HERE...
  if let operation = nodeConstructionOperation {
    operation.cancel()
  }
  // ...TO HERE
 
  containerNode?.recursiveSetPreventOrCancelDisplay(true)
  contentLayer?.removeFromSuperlayer()
  contentLayer = nil
  containerNode = nil
}
```

这就在 cell 重用时取消了操作，所以如果 Node 创建还没完成，它也不会完成。

现在找到 `configureCellDisplayWithCardInfo(cardInfo:)` 并添加如下指示的代码：

```Swift
func configureCellDisplayWithCardInfo(cardInfo: RainforestCardInfo) {
  // ADD FROM HERE...
  if let oldNodeConstructionOperation = nodeConstructionOperation {
    oldNodeConstructionOperation.cancel()
  }
  // ...TO HERE
 
  //MARK: Image Size Section
  let image = UIImage(named: cardInfo.imageName)!
  featureImageSizeOptional = image.size
}
```

这个 cell 现在会在它准备重用并开始配置时，取消任何进行中的 Node 构造操作。这确保了操作被取消，即使 cell 在准备好重用前就被重新配置。

编译并确保没有错误。

### 在主线程运行

AsyncDisplayKit 允许你在非主线程做许多工作。但当它要面对 UIKit 和 CoreAnimation 时，你还是需要在主线程做。目前为止，你从主线程移走了所有的 Node 创建。但还有一件事需要被放在主线程——即设置 CoreAnimation 的 Layer 层次结构。

在 _RainforestCardCell.swift_ 里，找到 `nodeConstructionOperationWithCardInfo(cardInfo:image:)` 并替换 _Node Layer and Wrap Up Section_ 为如下代码： 

```Swift
// 1
dispatch_async(dispatch_get_main_queue()) { [weak nodeConstructionOperation] in
  if let strongNodeConstructionOperation = nodeConstructionOperation {
    // 2
    if strongNodeConstructionOperation.cancelled {
      return
    }
 
    // 3
    if strongSelf.nodeConstructionOperation !== strongNodeConstructionOperation {
      return
    }
 
    // 4
    if containerNode.preventOrCancelDisplay {
      return
    }
 
    // 5
    //MARK: Node Layer and Wrap Up Section
    strongSelf.contentView.layer.addSublayer(containerNode.layer)
    containerNode.setNeedsDisplay()
    strongSelf.contentLayer = containerNode.layer
    strongSelf.containerNode = containerNode
  }
}
```

下面描述一下：

1. 回忆到当 Node 的 Layer 属性被第一个访问时，所有的 Layer 会被创建。这就是为何你必须运行 Node Layer 并在主线程包装小节，因此代码访问 Node 的 Layer。
2. 操作被检查以确定是否在添加 Layer 之前就已经取消了。在操作完成前，cell 被重用或者重新配置，就很可能会出现这样的情况，那你就不应该添加 Layer 了。
3. 作为一个保险，确保 Node 当前的 `nodeConstructionOperation` 和调度此闭包的操作是同一个 `NSOperation` 。
4. 如果 `containerNode` 的 `preventOrCancel` 是 `true` 就立即返回。如果构造操作完成，但 Node 的绘制还没有被取消，你依然不想 Node 的 Layer 显示在 cell 里。
5. 最后，添加 Node 的 Layer 到层次结构中，如果必要，这将创建 Layer。

编译确保没有错误。

### 开始 Node 创建操作

你依然没有 _实际_ 创建和开始操作。让我们现在来来吧。

继续在 _RainforestCardCell.swift_ 里，改变 `configureCellDisplayWithCardInfo(cardInfo:)` 的方法签名为： 

```Swift
func configureCellDisplayWithCardInfo(
  cardInfo: RainforestCardInfo,
  nodeConstructionQueue: NSOperationQueue)
```

这里添加了一个新的参数 `nodeConstructionQueue`。它就是一个用于 Node 创建操作的入队的 `NSOperationQueue` 。 

在 `configureCellDisplayWithCardInfo(cardInfo:nodeConstructionQueue:)` 底部，添加如下代码：

```Swift
let newNodeConstructionOperation = nodeConstructionOperationWithCardInfo(cardInfo, image: image)
nodeConstructionOperation = newNodeConstructionOperation
nodeConstructionQueue.addOperation(newNodeConstructionOperation)
```

这就创建了一个 Node 构造操作，将其保留在 `nodeConstructionOperation` 属性，并将其添加到传入的队列。

最后，打开 _RainforestViewController.swift_ 。给 `RainforestViewController` 添加一个叫做 `nodeConstructionQueue` 的初始化为常量的属性，如下：

```Swift
class RainforestViewController: UICollectionViewController {
  let rainforestCardsInfo = getAllCardInfo()
  let nodeConstructionQueue = NSOperationQueue() ///< ADD THIS LINE
  ...
}
```

接下来，在 `collectionView(collectionView:cellForItemAtIndexPath indexPath:)` 里，传递 View Controller 的 `nodeConstructionQueue` 到 `configureCellDisplayWithCardInfo(cardInfo:nodeConstructionQueue:)` ：

```Swift
cell.configureCellDisplayWithCardInfo(cardInfo, nodeConstructionQueue: nodeConstructionQueue)
```

cell 将会创建一个新的 Node 构造操作并将其添加到 View Controller 的操作队列里并发运行。记住在 cell 出队时就会创建一个新 Node 层次结构。这并不理想，但足够好。如果你要缓存 Node 的重用，看看 `ASRangeController` 吧。

哦呼，OK，现在编译并运行！你将看到和之前一样的效果，但现在布局和渲染都没在主线程执行了。牛！我打赌里你重来没有想过你会看到这一天你所做的事情。这就是 AsyncDisplayKit 的威力。你可以将更多更多不需要在主线程的操作从主线程移除，这将给主线程更多机会处理用户交互，让你的 App 摸起来如黄油般顺滑。

[![IMG_0019](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/IMG_0019-375x500.png)](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/IMG_0019.png)

## 淡入 Cell

现在是有趣的部分。在这个简短的小节，你将学到：

*   用自定义 Display Layer 子类来支持 Node；
*   触发 Node Layer 的隐式动画。

这将会确保你移除爆米花特效并最终带来良好的淡入动画。

### 创建一个新的 Layer 子类。

菜单 _File\New\File…_ ，选择 _iOS\Source\Cocoa Touch Class_ 并单击 _Next_ 。命名类为 _AnimatedContentsDisplayLayer_ 并使其作为 _`_ASDisplayLayer`_ 的子类。选择 _Swift_ 语言并单击 _Next_。最后保存并打开 _AnimatedContentsDisplayLayer.swift_ 。

现在添加如下方法到类：

```Swift
override func actionForKey(event: String!) -> CAAction! {
  if let action = super.actionForKey(event) {
    return action
  }
 
  if event == "contents" && contents == nil {
    let transition = CATransition()
    transition.duration = 0.6
    transition.type = kCATransitionFade
    return transition
  }
 
  return nil
}
```

Layer 有一个 contents 属性，它告诉系统为这个 Layer 绘制什么。AsyncDisplayKit 通过在后台渲染 contents 并最后在主线程设置 contents。

这个代码将会添加一个过渡动画，这样 contents 就会淡如到 View 中。你可以在 Apple 的 [Core Animation Programming Guide](https://developer.apple.com/library/mac/documentation/cocoa/conceptual/CoreAnimation_guide/ReactingtoLayerChanges/ReactingtoLayerChanges.html) 找到更多关于隐式 Layer 动画以及 `CAAction` 的信息.。

编译并确保没有错误。

### 淡入容器 Node

你已经设置好一个 Layer 会在其 contents 被设置时淡入，你现在就要使用这个 Layer。

打开 _RainforestCardCell.swift_ 。在 `nodeConstructionOperationWithCardInfo(cardInfo:image:)` 里，在 _Container Node Creation Section_ 开头，改动如下行：

```Swift
// REPLACE THIS LINE...
// let containerNode = ASDisplayNode()
// ...WITH THIS LINE:
let containerNode = ASDisplayNode(layerClass: AnimatedContentsDisplayLayer.self)
```

这会告诉容器 Node 使用 `AnimatedContentsDisplayLayer` 实例作为其支持 Layer，因此自动带来淡入的效果。

>**注意：**只有 `_ASDisplayLayer` 的子类才能被异步地绘制。

编译并运行。你将看到容器 Node 会在其绘制好之后淡入。

[![IMG_0023](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/IMG_0023-375x500.png)](http://cdn2.raywenderlich.com/wp-content/uploads/2014/10/IMG_0023.png)

## 又往何处去？

恭喜！在你需要高性能地滑动你的用户界面的时候，你有了另外一个工具在手。

在本教程里，你通过替换视图层次结构为一个栅格化的 AsyncDisplayKit Node 层次结构，显著改善了一个性能很差的 Collection View 的滑动性能。多么令人激动！

这只是一个例子而已。AsyncDisplayKit 保有提高 UI 性能到一定水平的承诺，这通过平常的 UIKit 优化往往难以达到。

实际说来，要充分利用 AsyncDisplayKit，你需要对标准 UIKit 的真正性能瓶颈的所在有足够的了解。AsyncDisplayKit 很棒的一点是它引发我们探讨这些问题并思考我们的 App 能如何在物理的极限上更快以及更具响应性。

AsyncDisplayKit 是探讨此性能前沿的一个非常强大的工具。明智地使用它，并步步逼近超级响应UI的极限。

这仅仅是 AsyncDisplayKit 的一个开始！它作者和贡献者每天都在构建新的特性。请关注 1.1 版的 `ASCollectionView` 以及 `ASMultiplexImageNode`。从头文件中可看到“ASMultiplexImageNode 是一个图像 Node，它能加载并显示一个图像的多个版本。例如，它可以在高分辨率的图像还在渲染时先显示一个低分辨率的图像。” 非常酷，对吧 :]

你可以[在此](http://cdn4.raywenderlich.com/wp-content/uploads/2014/10/Layers-Finished.zip)下载最终的 Xcode 项目。

AsyncDisplayKit 的指导在[这里](http://asyncdisplaykit.org/guide/ "here")，AsyncDisplayKit 的 Github 仓库在[这里](https://github.com/facebook/AsyncDisplayKit "here")。

这个库的作者在收集 API 设计的反馈。你可以在 Facebook 上 的 Paper Engineering Community group 分享你的想法，或者直接参与到 AsyncDisplayKit 的开发中，通过 [GitHub](https://github.com/facebook/AsyncDisplayKit) 贡献你的 pull request。

===============

译者注：最近我所在公司开发的[秒视 CatchChat](http://catchchat.me/) 推出了1.6版，它是一个可以发送与接收图片以及限定时长视频的社交应用，具有设计精美，操作快速的特点，欢迎使用！我的秒视ID是 nixzhu，也欢迎来扰！

译者注：欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条微博 [http://weibo.com/2076580237/Bxu0rq1D0](http://weibo.com/2076580237/Bxu0rq1D0) 或 Tweet [https://twitter.com/nixzhu/status/536130283070685184](https://twitter.com/nixzhu/status/536130283070685184) 以分享给更多人！

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝扫描下方二维码随便捐助一点，以慰劳译者的辛苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)
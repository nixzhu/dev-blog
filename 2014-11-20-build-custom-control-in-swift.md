
本文翻译自 [http://www.thinkandbuild.it/building-a-custom-and-designabl-control-in-swift/](http://www.thinkandbuild.it/building-a-custom-and-designabl-control-in-swift/ "Permalink to How to build a custom (and “designable”) control in Swift") 

原作者：[Yari D'areglia](https://twitter.com/bitwaker)

译者：[@nixzhu](https://twitter.com/nixzhu)

=================================

# 使用 Swift 构建自定义（且“可设计”的）控件

大约两年前，我写了一篇关于在 iOS 上构件自定义控件的教程。那篇教程获得了社区的高度赞赏，所以我决定将其用 Swift 更新并加入 Designable/Inspectable 属性以支持直接在 Interface Builder 里调整控件的外观。

在开始之前，你可以先稍微看看[最终的效果](http://vimeo.com/111882383)。

### 那就开始吧！

无论你是自己设计自定义用户界面，还是有专职的设计师帮你做，UIKit 的标准控件实在不能完全满足你的需求。

例如，如若你想制作一个控件来帮助用户从 0 到 360 度之间选择一个角度？一个解决办法是创建一个圆形的滑块，并让用户拖动把手来选择一个角度值。你很可能在许多其他界面里看过这样的设计，但 UIKit 里并没有直接提供类似的东西。

那么这就是一个绝佳的例子，能让我们暂时搁置 UIKit，并构建出特别的东西来。

### 子类化 UIControl

**UIControl** 类是 UIView 的子类，而且它是所有 UIKit 控件（例如 UIButton，UISlider，UISwitch 等）的父类。UIControl 实例的主要作用是创建一个逻辑以便将操作（action）引导到它们的目标（target）上，对于一个特别的用户界面，大部分时候都是根据它状态来画出不同的控件形态（例如高亮，选中，使不能……）

使用 UIControl 时我们要管理三个重要的任务：

* 画出用户界面
* 追踪用户的交互
* 管理 目标－操作（Target-Action） 模式

在这个圆形滑块（Circular Slider）中，我们将要定义一个**用户界面**（就是圆形滑块本身），然后用户可以与之**交互**（移动把手）。用户的决定会被转化成**操作（action）**对应到控件的**目标（target）**

We are ready to open XCode. I suggest you to download the full project at the end of this article and follow this tutorial reading my code.

我们准备好打开 Xcode。我建议你下载本文的[完整项目][8]，以便跟着本教程来阅读代码。

我们将走一遍上面列出的三个步骤。这些步骤是完全模块化的，也就是说，如果你对我如何画出组件不感兴趣，那可以直接跳到第二步和第三步。

打开 **BWCircluarSlider.swift** 跟着下面的小结走：

### 1) 绘制用户界面

我爱 Core Graphics 而且我想要创建的东西你还可以在你自己试验时再修改。我唯一想用 UIKit 绘制的部分是 textfield，它用于呈现滑块的值。

警告：需要一些 **Core Graphics** 知识，至少你要能看懂这些代码，一路上我会尽量解释它们。

让我们先分析一下这个控件的不同部分，以便对我们要绘制的东西有更好的体会。

首先是一个**黑色圆圈**，它用作滑块的背景。

![][1]

然后是**操作区域**，用一个从蓝色到紫色的梯度值填充。

![][2]

再之后是**把手**，用户会拖动它以选择不同的角度值。

![][3]

最后，用一个 **TextField** 来显示选择的角度值。在下一个版本里，它同样会接受从键盘输入的值。

![][4]

我们主要使用 `drawRect` 函数绘制界面，那第一步就是得到当前的绘图上下文。

```Swift
let ctx = UIGraphicsGetCurrentContext()
```

#### 绘制背景

背景被定义为一个360°的圆弧。这很容易通过 `CGContextAddArc` 来定义出合适的路径再将其画出。

下面的代码即可完成这个简单的任务：


```Swift
// 构建圆圈
CGContextAddArc(ctx, CGFloat(self.frame.size.width / 2.0), CGFloat(self.frame.size.height / 2.0), radius, 0, CGFloat(M_PI * 2), 0)

// 设置填充/笔画颜色
UIColor(red: 0.0, green: 0.0, blue: 0.0, alpha: 1.0).set()

// 设置线段信息
CGContextSetLineWidth(ctx, 72)
CGContextSetLineCap(ctx, kCGLineCapButt)

// 绘制！
CGContextDrawPath(ctx, kCGPathStroke)
```

函数 **CGContextArc** 接受圆弧的中心坐标以及半径（一个私有的整型变量）。然后它需要以弧度表示的开始角度和结束角度（你可以在 BWCircularSlider.swift 文件的顶部找到一个数学帮助方法的列表），最后一个参数是绘制方向，0表示逆时针方向。

其他代码是一些设置，例如颜色和线宽。最后我们使用 **CGContextDrawPath** 画出我们的圆弧路径。

#### 绘制操作区域

这个部分就稍微要些技巧。我们要绘制一个被图像遮盖的线性梯度图。来看看这是怎样的过程。

![][5]

掩模图像就像一个洞，通过它我们只能看见原始梯度方块的一部分。

稍微有趣的不同是这一次的圆弧将有一个阴影，以创建一个有着虚化效果的掩模图像。

#### 创建掩模图像


```Swift
UIGraphicsBeginImageContext(CGSizeMake(self.bounds.size.width,self.bounds.size.height));
let imageCtx = UIGraphicsGetCurrentContext()
CGContextAddArc(imageCtx, CGFloat(self.frame.size.width/2)  , CGFloat(self.frame.size.height/2), radius, 0, CGFloat(DegreesToRadians(Double(angle))) , 0);
UIColor.redColor().set()

// 使用阴影创建模糊效果
CGContextSetShadowWithColor(imageCtx, CGSizeMake(0, 0), CGFloat(self.angle/15), UIColor.blackColor().CGColor);

// 定义路径
CGContextSetLineWidth(imageCtx, Config.TB_LINE_WIDTH)
CGContextDrawPath(imageCtx, kCGPathStroke)

// 保存上下文内容为 mask
var mask:CGImageRef = CGBitmapContextCreateImage(UIGraphicsGetCurrentContext());
UIGraphicsEndImageContext();
```

首先，我们创建一个图像上下文，然后激活阴影。函数 **CGContextSetShadowWithColor** 帮助我们选择：

* 上下文
* 偏移值（这里并不需要）
* 虚化值（我们通过当前角度处以15来参数化这个值以便当用户操作时在模糊区域获得一个简单的动画）
* 颜色

我们再次绘制一个圆弧，这一次取决于**当前角度**。

例如，角度是 360° 时我们绘制一个满圆，如果是 90° 我们就绘制它的一部分。最后我们使用函数 **CGBitmapContextCreateImage** 从当前的绘制上得到一个图像。这个图像就是我们的掩模图像。

#### 裁减上下文

现在我们有了一个掩模图像，我们就能定义一个“洞”以通过它来看到梯度图像。

我们使用函数 **CGContextClipToMask** 来裁剪上下文，并传递我们刚才创建的掩模图像。


```Swift
CGContextClipToMask(ctx, self.bounds, mask);
```

最后我们绘制梯度图像：

```Swift
// 将颜色分成不同部分(rgba)
let startColorComps:UnsafePointer = CGColorGetComponents(startColor.CGColor);
let endColorComps:UnsafePointer = CGColorGetComponents(endColor.CGColor);

let components : [CGFloat] = [
    startColorComps[0], startColorComps[1], startColorComps[2], 1.0,     // 起始颜色 color
    endColorComps[0], endColorComps[1], endColorComps[2], 1.0      // 结束颜色
]

// 设置梯度
let baseSpace = CGColorSpaceCreateDeviceRGB()
let gradient = CGGradientCreateWithColorComponents(baseSpace, components, nil, 2)

// 梯度方向
let startPoint = CGPointMake(CGRectGetMidX(rect), CGRectGetMinY(rect))
let endPoint = CGPointMake(CGRectGetMidX(rect), CGRectGetMaxY(rect))

// 绘制梯度图
CGContextDrawLinearGradient(ctx, gradient, startPoint, endPoint, 0);
CGContextRestoreGState(ctx);
```

绘制梯度的过程比较长，但基本上分成4个部分（如代码中的注释所示）：

* 定义颜色
* 定义梯度方向
* 选择颜色空间
* 创建并绘制梯度

感谢掩模图像，梯度矩形只有一部分是可见的了。

#### 绘制把手

现在我们要根据当前角度在正确的位置绘制把手。这一步在绘制方面很简单（就是画出一个白色圆盘），但它需要一些计算才能得到把手的正确位置。

我们使用三角学转换一个**标量数字**为一个 **CGPoint**。别害怕，只不过是使用预定义的 Sin 和 Cos 函数罢了。

```Swift
func pointFromAngle(angleInt: Int) -> CGPoint {
    // 圆形中心
    let centerPoint = CGPointMake(self.frame.size.width/2.0 - Config.TB_LINE_WIDTH/2.0, self.frame.size.height/2.0 - Config.TB_LINE_WIDTH/2.0);

    // 圆圈上点的位置
    var result:CGPoint = CGPointZero
    let y = round(Double(radius) * sin(DegreesToRadians(Double(-angleInt)))) + Double(centerPoint.y)
    let x = round(Double(radius) * cos(DegreesToRadians(Double(-angleInt)))) + Double(centerPoint.x)
    result.y = CGFloat(y)
    result.x = CGFloat(x)

    return result;
}
```

给定一个角度，找到圆周上的合适位置，我们同时也需要圆周的中心和半径。

使用 sin 函数我们得到 Y 轴的值，使用 cos 函数我们得到 X 轴的值。

记住每个函数的返回值都以1为假想半径。我们只需将结果乘以我们的半径然后相对于圆周中心移动即可。

我希望下面的公式会帮助你更好地理解：

    point.y = center.y + (radius * sin(angle));
    point.x = center.x + (radius * cos(angle));

现在我们知道如何得到把手的位置，那就使用我们刚创建的函数来绘制它：

```Swift
func drawTheHandle(ctx: CGContextRef) {
    CGContextSaveGState(ctx);

    // 我爱阴影
    CGContextSetShadowWithColor(ctx, CGSizeMake(0, 0), 3, UIColor.blackColor().CGColor);

    // 获取把手位置
    var handleCenter = pointFromAngle(angle)

    // 绘制！
    UIColor(white:1.0, alpha:0.7).set();
    CGContextFillEllipseInRect(ctx, CGRectMake(handleCenter.x, handleCenter.y, Config.TB_LINE_WIDTH, Config.TB_LINE_WIDTH));

    CGContextRestoreGState(ctx);
}
```

步骤是：

* 保存当前上下文（当你要在不同的函数做绘制时，保存当前上下文状态是一个最佳实践）
* 为把手设置阴影
* 定义把手颜色并使用函数 **CGContextFillEllipseInRect** 绘制

我们可以在 drawRect 的最后调用这个函数。

```Swift
drawTheHandle(ctx)
```

现在我们做完了绘制的部分。

###  2) 跟踪用户的交互

通过子类化 UIControl，我们能覆写三个特殊方法来提供自定义的跟踪行为。

#### 开始跟踪

当一个触摸事件发生于控件范围时，方法 **beginTrackingWithTouch** 就会被控件调用。

来看看如何覆写它：

```Swift
override func beginTrackingWithTouch(touch: UITouch, withEvent event: UIEvent) -> Bool {
    super.beginTrackingWithTouch(touch, withEvent: event)

    return true
}
```

它返回一个 **Bool** 值，这决定了控件是否需要在触摸变为拖动时依然响应。在我们情况里，我们需要跟踪拖动，所以我们返回 **true**。这个方法接受两个参数，触摸对象和事件。

#### 持续跟踪

前一个方法里我们指明要持续跟踪，所以一个特别的方法 **continueTrackingWithTouch** 就会在用户拖动时执行：

```Swift
func continueTrackingWithTouch(touch: UITouch, withEvent event: UIEvent) -> Bool
```

这个方法返回一个 **Bool** 值表示触摸跟踪是否该持续下去。

我们使用这个方法根据触摸的位置来过滤用户的操作。例如，我们可以选择只在触摸位置与把手相交时才激活控件。当然，我们的控件不需要这样做，因为我们想移动把手以响应任意触摸位置。

本教程中，这个方法负责改变把手的位置（以及我们接下来一节要做的发送操作到目标）。

使用如下代码覆写它：


```Swift
override func continueTrackingWithTouch(touch: UITouch, withEvent event: UIEvent) -> Bool {
    super.continueTrackingWithTouch(touch, withEvent: event)

    let lastPoint = touch.locationInView(self)

    self.moveHandle(lastPoint)

    self.sendActionsForControlEvents(UIControlEvents.ValueChanged)

    return true
}
```

首先我们使用 **locationInView** 获取触摸位置。然后将其传递给函数 **moveHandle**，它会将触摸位置转换为把手的合适的位置。

我所说的“合适的位置”是什么意思呢？

把手应该只能在由背景圆弧定义的圆圈的范围内移动。但我们不想要强制用户在这个狭小的空间内移动他的手指才能控制把手，因为这种体验是非常令人沮丧的。所以我们将接受任何触摸位置并将其转换为把手的合适的位置。

函数 **moveHandle** 负责完成这个工作，而且，在这个函数里，我们会执行转换，它会显示我们滑块的角度值。


```Swift
func moveHandle(lastPoint: CGPoint) {
    // 获得中心
    let centerPoint:CGPoint  = CGPointMake(self.frame.size.width/2, self.frame.size.height/2);
    
    // 计算从中心到方向随意位置的方向
    let currentAngle:Double = AngleFromNorth(centerPoint, p2: lastPoint, flipped: false);
    let angleInt = Int(floor(currentAngle))

    // 存储新的角度
    angle = Int(360 - angleInt)

    // 更新 textfiled
    textField!.text = "(angle)"

    // 重绘
    setNeedsDisplay()
}
```

大部分工作由 **AngleFromNorth** 完成。给定 2 个点，它返回一个连接它们的假想线的角度。


```Swift
func AngleFromNorth(p1:CGPoint , p2:CGPoint , flipped:Bool) -> Double {
    var v:CGPoint  = CGPointMake(p2.x - p1.x, p2.y - p1.y)
    let vmag:CGFloat = Square(Square(v.x) + Square(v.y))
    var result:Double = 0.0
    v.x /= vmag;
    v.y /= vmag;
    let radians = Double(atan2(v.y,v.x))
    result = RadiansToDegrees(radians)
    return (result >= 0  ? result : result + 360.0);
}
```

（注意：我不是 AngleFromNorth 的作者，我直接从 Apple 的例子程序 clockControl 里拿来了它）

现在我们有了用角度表示的值，我们将其存储在属性 **angle** 里，并更新 textfield 的值。

函数 setNeedDisplay 确保方法 drawRect 会被尽快调用，以使用新的角度值进行绘制。

#### 结束跟踪

下面的函数会在跟踪结束后调用：


```Swift
override func endTrackingWithTouch(touch: UITouch, withEvent event: UIEvent) {
    super.endTrackingWithTouch(touch, withEvent: event)
}
```

在这里例子里，我们不需要覆写这个函数，但如果你需要在用户与控件完成交互后执行一些额外的操作的话，这个函数就会很有用。

### 3) Target-Action 模式

到了这里后圆形滑块应该可以正常工作了。你现在可以拖动把手并看到 textfield 里值的改变。

#### 为控件事件发送操作

如果我们想要保证 UIControl 的行为的连贯性，那我们就要在值改变时发出通知。我们可以使用函数 **sendActionsForControlEvents** 指定特定的事件类型来做到这一点。此例中，事件类型是 **UIControlEventValueChanged**。

事件类型的可能值相当多（按住cmd并单击 Xcode 里显示的 UIControlEventValueChanged 就能看到这个列表）。例如，如果你的控件是 UITextField 的子类，你可能会对 **UIControlEventEdigitingDidBegin** 感兴趣，或者，如果你想在松开触摸时发出通知，那么可以使用 UIControlTouchUpInside。

如果你回过头看第二节，你会看到我们在函数 continueTrackingWithTouch 返回前调用了 **sendActionsForControlEvents**。

```Swift
self.sendActionsForControlEvents(UIControlEvents.ValueChanged)
```

感谢这一句，当用户移动把手改变滑块值时，每个被注册的对象都会收到这个改变的通知。

### 如何使用控件

我们有了一个自定义控件，那就在我们的应用中使用它吧。

由于我们的控件是 UIControl 的子类，所以它不能直接在 Interface Builder 中使用。但不要担心，我们可以使用一个 UIView 作为桥梁来将 UIControl 作为桥接视图的子视图。

打开文件 **BWCircularSliderView.swift** 查看 **awakeFromNib** 方法：



```Swift
override func awakeFromNib() {
    super.awakeFromNib()

    // 构建滑块
    let slider:BWCircularSlider = BWCircularSlider(startColor:self.startColor, endColor:self.endColor, frame: self.bounds)

    // 关联 Action/Target 到滑块
    slider.addTarget(self, action: "valueChanged:", forControlEvents: UIControlEvents.ValueChanged)

    // 将滑块作为子视图
    self.addSubview(slider)
}
```

我们初始化了一个圆形滑块对象，并通过 **init:startColor:endColor:frame** 为其添加了颜色和 frame 信息。这个自定义的初始化器保存了梯度颜色以及设置控件的 frame 等同于“桥接”视图的边界。这就意味着控件将继承视图的尺寸（你可以使用AutoLayout得到同样的效果）。

我们现在可以定义如何与控件交互，感谢方法 **addTarget:action:forControlEvent:**。

这个方法只是为控件的一个特定的事件设置了一个 target-action 模式。如果你记得，圆形控件会在每次用户移动把手时发送一个 UIControlEventValueChanged。所以我们能够使用如下代码注册一个操作到这个事件上：


```Swift
slider.addTarget(self, action: "valueChanged:", forControlEvents: UIControlEvents.ValueChanged)
```

然后我们编写函数 **valueChanged** ，以便在值改变时做些事情：


```Swift
func valueChanged(slider:BWCircularSlider){
    // 根据新值做些事情……
    println("Value changed (slider.angle)")
}
```

我们使用了 Target-Action 模式，所以收到发送者（这里就是滑块）对应事件时就会调用这个操作。此例中，我们直接访问角度值，并简单地打印它。

Now check the override of the function **willMoveToSuperview**

现在看看覆写方法 **willMoveToSuperview**

```Swift
#if TARGET_INTERFACE_BUILDER
override func willMoveToSuperview(newSuperview: UIView?) {
    let slider:BWCircularSlider = BWCircularSlider(startColor:self.startColor, endColor:self.endColor, frame: self.bounds)
    self.addSubview(slider)
}
#else
```

这个代码，与 **@IBInspectable** 和 **@IBDesignable** 关键字一道，都是直接在 Interface Builder 查看视图的必要条件（查看[这个教程][6]获取更多与 IBDesignable 相关的信息）。

（TARGET_INTERFACE_BUILDER 表示我们只想在 Interface Builder 里执行这个代码，App 运行时就不用了）

现在我们就可以通过 Storyboard 添加一个 BWCircularSliderView 实例，并能在 Interface Builder 里直接修改 startColor 和 endColor 属性，控件的预览将立即更新在屏幕上。

### 总结

照本教程展示的步骤，你可以创建 **任何你想要的东西**。

可能还有其他的方式能够做到类似这样的事情，但我尽量遵守了 Apple 的建议，展示给你的是 100% 的文档推荐方式。

如有任何疑问、建议或者你想要分享的关于自定义控件的好点子，你可以在 [twitter][7] 上找我。

Ciao!

[下载源代码][8] ，[关注 @bitwaker][9]

===============

译者注：最近我所在公司开发的[秒视 CatchChat](http://catchchat.me/) 推出了1.6版，它是一个可以发送与接收图片以及限定时长视频的社交应用，具有设计精美，操作快速的特点，欢迎使用！我的秒视ID是 nixzhu，也欢迎来扰！

译者注：欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝扫描下方二维码随便捐助一点，以慰劳译者的辛苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

[1]: http://www.thinkandbuild.it/wp-content/uploads/2013/02/slider_bg-150x150.png "slider_bg"
[2]: http://www.thinkandbuild.it/wp-content/uploads/2014/11/slider_active-150x150.png "slider_active"
[3]: http://www.thinkandbuild.it/wp-content/uploads/2013/02/slider_handle.png "slider_handle"
[4]: http://www.thinkandbuild.it/wp-content/uploads/2014/11/Screen-Shot-2014-11-14-at-23.09.24.png "Screen Shot 2014-11-14 at 23.09.24"
[5]: http://www.thinkandbuild.it/wp-content/uploads/2014/11/slider_maskaction-300x112.png "slider_maskaction"
[6]: http://www.thinkandbuild.it/building-custom-ui-element-with-ibdesignable/
[7]: https://twitter.com/bitwaker "Twitter"
[8]: https://github.com/ariok/BWCircularSlider
[9]: https://twitter.com/bitwaker

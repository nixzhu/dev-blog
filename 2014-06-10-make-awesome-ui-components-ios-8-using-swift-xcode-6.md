# 使用 Swift 和 Xcode 6 制作超棒的 UI 组件

本文翻译自：[http://www.weheartswift.com/make-awesome-ui-components-ios-8-using-swift-xcode-6/](http://www.weheartswift.com/make-awesome-ui-components-ios-8-using-swift-xcode-6/)

原作者：[Andrei Puni](http://www.weheartswift.com/author/andrei512/)

译者：[@nixzhu](https://twitter.com/nixzhu)

=================================

Apple 在 Xcode 6 里介绍了两个新的 Interface Buidler 声明属性：`IBInspectable` 和 `IBDesignable`；`IBInspectable` 将类的属性暴露在 Interface Buidler 的 Attribute Inspector 里，而用了 `IBDesignable` 后就可以**实时更新视图**了！这简直就像魔法！

我实在等不及看到你会做出怎样炫酷的东西来。

我们也制作了一个简短的[视频（需翻墙）][4]来介绍 `IBInspectable` 和 `IBDesignable`，大概只需要 10 分钟你就可以走完所有的步骤。而[代码在 GitHub 上][5]。

## IBInspectable

我目前找到的，`IBInspectable` 可以处理的类型如下：

* `Int`
* `CGFloat`
* `Double`
* `String`
* `Bool`
* `CGPoint`
* `CGSize`
* `CGRect`
* `UIColor`
* `UIImage`

例子：

```Swift
class OverCustomizableView : UIView {
    @IBInspectable var integer: Int = 0
    @IBInspectable var float: CGFloat = 0
    @IBInspectable var double: Double = 0
    @IBInspectable var point: CGPoint = CGPointZero
    @IBInspectable var size: CGSize = CGSizeZero
    @IBInspectable var customFrame: CGRect = CGRectZero
    @IBInspectable var color: UIColor = UIColor.clearColor()
    @IBInspectable var string: String = "We ❤ Swift"
    @IBInspectable var bool: Bool = false
}
```

在对应 View 的 Attribute Inspector 的顶部，你将看到：

![exposed properties][6]

做完这些就添加了一些用户定义的运行时属性，这样就可以在 IB 里设置的它们的初始值，之后视图加载时就可以使用了。

创建好的运行时属性：

![user define runtime attributes in xcode 6][7]

## IBDesignable

现在到了有趣的的部分了。`IBDesignable` 告诉 Interface Builder 它可以加载视图并渲染视图；而要这个功能正常工作，**视图类必须位于一个框架（framework）内**。但这并不会带来很大的不便，我们马上就会看到。我认为在后面，Interface Builder 将 UIView 代码转换为 NSView 代码，这样它就能动态加载框架并渲染其组件了。

>译者注：估计不久我们就会看到大量的第三方 UI 组件，都可以直接在 IB 里显示和修改，更加直观。

## 创建一个新项目

打开 Xcode 6，创建一个新的“Single Page Application”并选择 Swift 作为编程语言。

## 添加一个新的 target 到项目中

从导航栏选择你的项目文件并通过点击 `+` 添加一个新的 target：

![add a new target in xcode][8]

选择 `Framework & Application Library` 并选中 `Cocoa Touch Framework`：

![creating a cococa touch framework][9]

将其命名为 `MyCustomView`。Xcode 会自动链接 `MyCustomView.framework` 到你的项目中。

## 创建一个自定义个视图类

创建一个新的 Swift 文件并将其添加到 `MyCustomView` 框架。

在框架文件夹上右键单击：

![create new file][10]

选择 Cocoa Touch 文件：

![][11]

将其命名为 `CustomView`，且为 `UIView` 的子类：

![][12]

在上面的视图控制器类里[编写一个类定义][13]。

使用 `@IBDesignable` 关键字来告诉 Xcode 渲染你的视图。

添加三个属性：`borderColor: UIColor`、`borderWidth: CGFloat` 以及 `cornerRadius: CGFloat`。

设置它们的默认值并让它们可视察（Inspectable）：

```Swift
@IBDesignable class CustomView : UIView {
    @IBInspectable var borderColor: UIColor = UIColor.clearColor()
    @IBInspectable var borderWidth: CGFloat = 0
    @IBInspectable var cornerRadius: CGFloat = 0
}
````

## 为 Layer 属性添加逻辑

为每个属性都添加一个[属性观察者（Property Observer）][14]从而更新 `layer`：

```Swift
class CustomView : UIView {
    @IBInspectable var borderColor: UIColor = UIColor.clearColor() {
        didSet {
            layer.borderColor = borderColor.CGColor
        }
    }

    @IBInspectable var borderWidth: CGFloat = 0 {
        didSet {
            layer.borderWidth = borderWidth
        }
    }

    @IBInspectable var cornerRadius: CGFloat = 0 {
        didSet {
            layer.cornerRadius = cornerRadius
        }
    }
}
```

用快捷键 `⌘ Cmd`+`B` 构建框架。

## 测试自定义视图

打开 `Main.storyboard` 并从组件库里添加一个视图。

使用 Identity Inspector 修改其视图类为 `CustomView`：

![chande the class][15]

安排好视图的位置并添加所需的 AutoLayout 约束：

>Tip：按住 `Crtl` 然后点击视图并拖拽鼠标指针以添加其相对于另外一个视图的约束。

![adding autolayout contraints without headaches][16]

玩耍了一阵 `cornerRadius` 之后，我发现在使用较大的值时，它创造了一种有趣的的模式：

![interesting pattern][17]

你可以在 [GitHub][5] 上获取代码。

祝你玩得愉快 😄

一个快速演示（“需翻墙”，我希望在不远的将来我不需要写这一句）：

<a href="https://www.youtube.com/watch?v=9Jb7X0GiRv4" target="_blank"><img src="http://img.youtube.com/vi/9Jb7X0GiRv4/0.jpg" 
alt="IBDesignable - IBInspectable Demo" width="480" height="360" border="1" /></a>

## 挑战

* 制作一个复选框（Checkbox）组件
* 制作一个有 `angle: CGFloat` 属性的组件，可以旋转视图。
* 在 Interface Builder 里渲染一个分形图 😄

如果你发现这篇文章很有用，记得和你的朋友分享哦 😄

=====================

译者注：欢迎非商业转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳译者的幸苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

版权声明：自由转载-非商用-非衍生-保持署名 | [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)


[1]: https://www.github.com/
[2]: http://cocoapods.org/
[3]: https://www.cocoacontrols.com/
[4]: https://www.youtube.com/watch?v=9Jb7X0GiRv4
[5]: https://github.com/WeHeartSwift/IBDesignable-Demo
[6]: https://camo.githubusercontent.com/d9a8cefae7ec146ce2e6fc575bc29dfa26996316/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f6578706f7365642d70726f706572746965732d65313430323037313938313932312e706e67
[7]: https://camo.githubusercontent.com/bc4f397ffbf236a556d418ba00052a27da76c85d/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f776861742d68617070656e732d756e6465722d7468652d686f6f642d65313430323037313933383735332e706e67
[8]: https://camo.githubusercontent.com/0fc74c546afca078156b184c3e7d134647ac24b9/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f6164642d612d6e65772d7461726765742e706e67
[9]: https://camo.githubusercontent.com/d3ef2378a48ebdd93783ef18853563cbb412f918/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f6164642d612d6672616d65776f726b2e706e67
[10]: https://camo.githubusercontent.com/1ada46f333caaf3ccf9f34724529c846f59682eb/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f6e65772d66696c652e706e67
[11]: https://camo.githubusercontent.com/fe228443d227fd537f6ec6236cb7aef7bb30dc6d/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f636f636f612d746f7563682d66696c652e706e67
[12]: https://camo.githubusercontent.com/61a7bd9719427414edc039822d12b6d25e631ca0/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f7365742d7468652d6e616d652d616e642d636c6173732e706e67
[13]: https://github.com/andrei512/writing/blob/master/weheartswift/www.weheartswift.com/swift-classes-part-1
[14]: https://developer.apple.com/library/prerelease/ios/documentation/swift/conceptual/swift_programming_language/Properties.html#//apple_ref/doc/uid/TP40014097-CH14-XID_333
[15]: https://camo.githubusercontent.com/3a125ff58e68321c960096bd2a1c9e9ce63ada5a/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f6368616e67652d7468652d636c6173732d65313430323037353234313133352e706e67
[16]: https://camo.githubusercontent.com/8fbf3ff5ebc871872c7bfd9761d248345b03a384/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f636c69636b2d647261672e706e67
[17]: https://camo.githubusercontent.com/b90513835b05059f64fd4b90e0663f8b95a9f919/687474703a2f2f7777772e7765686561727473776966742e636f6d2f77702d636f6e74656e742f75706c6f6164732f323031342f30362f64657369676e61626c652d766965772d65313430323038373838333435352e706e67
[18]: http://www.weheartswift.com/wp-includes/images/smilies/icon_smile.gif
  
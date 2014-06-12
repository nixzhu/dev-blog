# 分析一个有趣的 Swift 项目：LTBouncyPlaceholder

本文尝试分析 [LTBouncyPlaceholder](https://github.com/lexrus/LTBouncyPlaceholder) 项目的实现

项目作者：[lexrus](http://lextang.com/)

分析作者：[@nixzhu](https://twitter.com/nixzhu)

=================================

我希望你已经下载了 LTBouncyPlaceholder 的 [Demo](https://github.com/lexrus/LTBouncyPlaceholder) ，用 Xcode 6 打开并编译、运行，然后在界面中显示的几个 UITextField 里输入一些文字来体验这个扩展。看到 Placeholder 的动画了吗？PS：iOS 8 的键盘也挺带感。

<img src="https://cloud.githubusercontent.com/assets/219689/3242824/e3d3f00e-f14f-11e3-8028-08327d011499.gif" width="322" height="581" alt="Demo" style="max-width:100%;">

## 开始

我首先观察到扩展里重载了 `willMoveToSuperview:`，对于这个方法，Xcode 的文档里写有：

>The default implementation of this method does nothing. Subclasses can override it to perform additional actions whenever the superview changes.

也就是说，这个方法默认不做事情，但子类可以重载它以便在 superview 改变时执行额外的操作。那么当 UITextField 被加载时，这个方法就会自动调用，所以我们先来看看它做了什么：

```Swift
override func willMoveToSuperview(newSuperview: UIView!) {
    if newSuperview {
        // 1. 首先要求 lt_placeholderLabel 显示它自己
        lt_placeholderLabel.setNeedsDisplay()
        
        // 2. 然后替换了 drawPlaceholderInRect 方法
        struct TokenHolder {
            static var token: dispatch_once_t = 0;
        }
        
        dispatch_once(&TokenHolder.token) {
            var originMethod: Method = class_getInstanceMethod(object_getClass(UITextField()),
                Selector.convertFromStringLiteral("drawPlaceholderInRect:".bridgeToObjectiveC().UTF8String))
            var swizzledMethod: Method = class_getInstanceMethod(object_getClass(UITextField()),
                Selector.convertFromStringLiteral("_drawPlaceholderInRect:".bridgeToObjectiveC().UTF8String))
            method_exchangeImplementations(originMethod, swizzledMethod)

        }
        
        // 3. 最后监听通知，这样用户输入文字或删除文字时，_didChange: 也会执行了
        NSNotificationCenter.defaultCenter().addObserver(self,
            selector: Selector.convertFromStringLiteral("_didChange:"),
            name: UITextFieldTextDidChangeNotification,
            object: nil)
    } else {
        NSNotificationCenter.defaultCenter().removeObserver(self,
            name: UITextFieldTextDidChangeNotification,
            object: nil)
    }
}
```

阅读代码并观察我添加的注释，我们知道了，这个扩展为原类添加了一个新的属性 `lt_placeholderLabel`（因为它是被直接使用的，就像原类的属性一样不需要写 `self.`），而从其名字可以得知它应该是一个用于显示占位符的 UILabel；之后 lexrus 利用 `dispatch_once` 和 `method_exchangeImplementations` 将系统的 `drawPlaceholderInRect:` 实现替换为 `_drawPlaceholderInRect:`，我们稍后会分析它的实现；最后，lexrus 在扩展里监听 `UITextFieldTextDidChangeNotification` 通知，这个通知大家应该比较熟悉，即当 UITextField 里的文字发生改变时，这个通知就会被发出。作者希望在这个通知出现时做一些事情，因而要执行 `_didChange`，我们之后也会分析其实现。

## “虚拟”属性

我们首先看看 `lt_placeholderLabel` ，既然 lexrus 对其调用了 `setNeedsDisplay()` ，那么它肯定要先生成。在 `UITextField+LTBouncyPlaceholder.swift` 里搜索 `lt_placeholderLabel` ，我们就会看到：

```Swift
var lt_placeholderLabel: UILabel {
get {
    var _placeholderLabelObject: AnyObject? = objc_getAssociatedObject(self, kPlaceholderLabelPointer)
    if let _placeholderLabel : AnyObject = _placeholderLabelObject {
        return _placeholderLabel as UILabel
    }
    var _placeholderLabel = UILabel(frame: self.placeholderRectForBounds(self.bounds))
    _placeholderLabel.font = self.font
    _placeholderLabel.text = placeholder
    _placeholderLabel.textColor = UIColor.lightGrayColor()
    self.addSubview(_placeholderLabel)
    objc_setAssociatedObject(self,
        kPlaceholderLabelPointer,
        _placeholderLabel,
        objc_AssociationPolicy(OBJC_ASSOCIATION_RETAIN_NONATOMIC))
    return _placeholderLabel
}
}
```

这是一个实例变量，但要知道我们现在在一个扩展中，并不能直接扩展类的属性。而为了扩展原类的属性，lexrus 使用了被称为“关联对象（Associated Objects）”的技术（请参考mattt编写的文章，[中文翻译](http://nshipster.cn/associated-objects/)或[英文原文](http://nshipster.com/associated-objects/)），利用 `objc_setAssociatedObject` 和 `objc_getAssociatedObject` “虚拟”出一个属性。而这个属性使用起来的感觉和在原类中定义的属性一样。

上面的代码并不复杂，`get` 类似 Objective-C 里的 `getter`，当我们访问这个属性的时候，它就会自动执行。它首先看看是否已有这个“虚拟属性”，有就直接返回。若没有，就利用原类的 `bounds` 和自带方法 `placeholderRectForBounds` 计算一个 `frame` 以便生成了一个新的 `UILabel`，再设置好字体等就作为 `subview` 被添加到 `self`（即 `UITextField`） 上了。最后设置好“虚拟属性”。这样下次再访问此属性时，这个 `UILabel` 就可以直接返回而不会被重复创建了。怎样？一样有 **Lazyload** 的感觉吧？

另外，稍微注意一下 `kPlaceholderLabelPointer` 的使用，它定义在 `UITextField+LTBouncyPlaceholderKeys.swift` 文件里，其实是一个 `CConstVoidPointer` ，相当于 C 的 `const void *`，具体请参考 Apple 提供的文档：[Interacting with C APIs](https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithCAPIs.html#//apple_ref/doc/uid/TP40014216-CH8-XID_13) 一节。

## 方法替换

接下来，我们看看 `_drawPlaceholderInRect:` 的实现：

```Swift
func _drawPlaceholderInRect(rect: CGRect) {
    
}
```

似乎什么都没做，而这正是方法替换的神奇之处。根据 `drawPlaceholderInRect:` 的文档说明：

>You should not call this method directly. If you want to customize the drawing behavior for the placeholder text, you can override this method to do your drawing.

>By the time this method is called, the current graphics context is already configured with the default environment and text color for drawing. In your overridden method, you can configure the current context further and then invoke super to do the actual drawing or do the drawing yourself. If you do render the text yourself, you should not invoke super.

这个方法是用于绘制原生的 Placeholder 的，而我们现在使用了自定义的 Placeholder ，因此原生的对我们来说没有用处了，所以不需要将其绘制出来。


## 处理通知

最后我们再来看看 `_didChange` 做了什么：

```Swift
func _didChange (notification: NSNotification) {
    if notification.object === self {
        if self.text.lengthOfBytesUsingEncoding(NSUTF8StringEncoding) > 0 {
            if alwaysBouncePlaceholder {
                self._animatePlaceholder(toRight: true)
            } else {
                lt_placeholderLabel.hidden = true
            }
        } else {
            if alwaysBouncePlaceholder {
                self._animatePlaceholder(toRight: false)
            } else {
                lt_placeholderLabel.hidden = false
            }
        }
    }
}
```

先确认通知的发送者是自己，然后在 `UITextField` 里输入有文字时，若 `alwaysBouncePlaceholder` 属性的状态为 `true`，就执行 `self._animatePlaceholder(toRight: true)` ，我想大概是将我们刚才讨论的作为 Placeholder 的 `UILabel` 以动画的方式移动到右边。

>注意：如之前提到，UITextField 本身的 Placeholder 因为方法替换，不会被绘制出来，而 `lt_placeholderLabel` 的 `text` 就是设置为 UITextField 自身的 `placeholder` 的，在没有输入文字时，它看起来就是原生的 Placeholder。

## 动画

事实上，只要读者稍微阅读一下 `_animatePlaceholder` 就可以分析出来，lexrus 还使用了一个名为 `lt_rightPlaceholderLabel` 的“虚拟属性”，用于在 `UITextField` 里有字符时，在其最右边显示另外一个 Placeholder，大家运行 Demo 时应该有所体会。

这里的 Core Animation 将 `lt_placeholderLabel` 移动到右边并隐去，与此同时， `lt_rightPlaceholderLabel` 也被移动到右边，但它是渐显，这样就得到我们所体验到的效果。

>作者注：不知道只用一个自定义的 Placeholder 能否实现这个效果，但既然 lexrus 这样写，可能有他的道理。

## 在 IB 里设置运行时属性

最后，在扩展文件的开头，我们还看到 `alwaysBouncePlaceholder` 与 `abbreviatedPlaceholder` 这两个“虚拟属性”，它们被创建所用的技术和 `lt_placeholderLabel` 一致，不再赘述。只需要观察到，这两个属性在 Demo 中的 IB 中对应 UITextField 的 Identity Inspector 里有被使用，这样就等于直接初始化了它们。

另外，`abbreviatedPlaceholder` 意思是“简短的占位符”，它顺便设置了 `lt_rightPlaceholderLabel` 的 `text` 属性，非常合理。注意 `newValue` 应该是 `set` 的默认参数。

## 总结

到此，我们差不多就分析完了这个相当出色的 UITextField 的扩展，它为我们带来了新鲜的使用体验。

而我们学习了一些新技术，特别是以 Swift 语言写成这一点值得大家研究。这些技术是：

* 属性的 set 和 get 的使用
* 在扩展里“创建”属性的方法，即“关联对象（Associated Objects）”
* UITextField 的工作特点，加载、通知等，这些对于自定义控件来说很有用
* 一些 Core Animation 的组合，弹性动画+渐隐渐显
* GCD 的使用，注意到 `dispatch_once` 了吗？它一般用于生成单例。
* 方法替换，作者替换了 `drawPlaceholderInRect:` 以取消原生 Placeholder 的绘制。

## 挑战

1. 如我提到的，是否只用一个自定义的 Placeholder 也能实现这个效果？
2. 用本文分析的这些技术，为系统的其它原生控件编写扩展。你需要的可能仅仅只是一点品味和想像力。
3. 能直接在扩展里使用 IBDesignable 和 IBInspectable 吗？如果不能，尝试使用子类化的方式来实现这个扩展并加上对 IBDesignable 和 IBInspectable 的支持，你可以参考我翻译的[这篇文章](https://github.com/nixzhu/dev-blog/blob/master/2014-06-10-make-awesome-ui-components-ios-8-using-swift-xcode-6.md)。

若本文的分析有任何不当之处，请读者指出，提 Issue 或者直接发送 PR 都可以！

=====================

作者注：欢迎非商业转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！

欢迎转发此条微博 [http://weibo.com/2076580237/B8CSosnAz](http://weibo.com/2076580237/B8CSosnAz)  以分享给更多人！

如果你认为这篇原创文章不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳作者的幸苦（PS：作者刚刚辞职）：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

版权声明：自由转载-非商用-非衍生-保持署名 | [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)

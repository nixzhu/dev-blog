# 区别 iPhone 做布局

现在主流有三种不同的 iPhone 尺寸：4 英寸、4.7 英寸以及 5.5 英寸。从设计上来讲，为特定的屏幕分别优化是很有必要的，当然，这就需要开发者做一些额外的工作。

作者：[@nixzhu](https://twitter.com/nixzhu)

=================================

虽然有所谓的 Adaptive Layout，但依其对屏幕的区分方式，我们并不能区分 iPhone 的不同屏幕尺寸，至少在应用竖屏时不行。

所以，我们依然需要用代码检测不同的屏幕尺寸，然后以其为基准来为界面元素设定如边距、大小之类的参数，以实现不同屏幕下最优的显示效果。

大家一定都写过（或使用过）判断 iPhone 型号的代码，并无甚特别。但重要的是在使用的层面，怎样设计优雅的 API 来完成不同屏幕的适配呢？

首先，我们设计屏幕的尺寸模型，因为和 UIDevice 有关，我们就扩展 UIDevice，有 enum 如下：

```Swift
import UIKit

extension UIDevice {

    enum ScreenModel {
        case Classic
        case Bigger
        case BiggerPlus
    }
    
    // TODO
}
```

注意，这里只从竖屏宽度上区分，将 3.5 英寸和 4 英寸都当作 `Classic`，如果你的应用还要区分 3.5 英寸的设备，那要对应增加 case。

然后，我们实现一个屏幕模型的单例，很明显它只需要初始计算一次即可，因此：

```Swift
    static let screenModel: ScreenModel = {

        let screen = UIScreen.mainScreen()
        let nativeWidth = screen.nativeBounds.size.width

        if nativeWidth == 320 * 2 {
            return .Classic

        } else if nativeWidth == 375 * 2 {
            return .Bigger

        } else if nativeWidth == 414 * 3 {
            return .BiggerPlus
        }

        return .Bigger // Default
        }()
        
    // TODO
```

我们利用 iOS 8 中 UIScreen 的 nativeBounds 来做判断。据其文档描述：

>The bounding rectangle of the physical screen, measured in pixels. (read-only)

>This rectangle is based on the device in a portrait-up orientation. This value does not change as the device rotates.

它表示设备在竖屏时的物理分辨率。有了它，我们就不需要关心两个 Bigger iPhone 的放大模式了。当然，若你的应用需要兼容 iOS 7 或以下版本，那就要换成其他的判断方法。如前所述，这并不是重点。

最后，我们设计一个 API 以对不同的屏幕设置不同的参数。我们需要它使用起来简单，那我们就设定其为 UIDevice 的类方法：

```Swift
    class func matchMarginFrom(classic: CGFloat, _ bigger: CGFloat, _ biggerPlus: CGFloat) -> CGFloat {
        switch screenModel {
        case .Classic:
            return classic
        case .Bigger:
            return bigger
        case .BiggerPlus:
            return biggerPlus
        }
    }
```

注意其参数前的 `_`，这表示在使用时我们不需要写参数名，因此如果我要设定某个元素距离左边的距离，那使用起来的感觉如下：

```Swift
    let leftEdge = UIDevice.matchMarginFrom(15, 30, 40)
```

之后无论你要用其设置 AutoLayout 约束还是计算 CGRect 都可以。

如果没有这个 API 的话，那在每一个需要区别屏幕的地方我们都需要写一个 Switch 语句根据 screenModel 来判断。

另外，如果将来 iPhone 又出了新的型号以至于我们要增加新的屏幕模型，那只需要修改对应增加 ScreenModel 的 case，再进一步修改 matchMarginFrom 的实现。之后所有使用 matchMarginFrom 的地方都会编译失败，这正好给了我们补足新数据的机会，而不用担心匆忙中漏掉某一个。

而如果使用可变参数或者用传递数组的方式来实现 `matchMarginFrom` 就得不到编译器帮我们检查的好处。如果真有增加 ScreenModel case 的一天，那改起所有使用 matchMarginFrom 的地方就不保险了。

因为代码比较简单，就放在 gist [https://gist.github.com/nixzhu/3c8ed0b8f7f24df924ac](https://gist.github.com/nixzhu/3c8ed0b8f7f24df924ac) 里了。


===============

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条 Tweet [https://twitter.com/nixzhu/status/600130762294202368](https://twitter.com/nixzhu/status/600130762294202368) 或微博 [http://weibo.com/2076580237/CinuFffMq](http://weibo.com/2076580237/CinuFffMq)  以分享此文！

如果你认为这篇文章不错，也有闲钱，那你可以用支付宝扫描下方二维码随便捐助一点，以慰劳作者的辛苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)
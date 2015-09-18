# 国产SDK

集成微信分享所累积的愤怒，要请齐天大圣来平复。

作者：[@nixzhu](https://twitter.com/nixzhu)

=================================

听说微信之前是支持系统分享的，后来不知道是何原因去掉了。这给开发者带来了不小的麻烦。一方面，用户对系统分享最熟悉，开发者也省心省力；另一方面，这也是唯一支持 AirDrop 的办法。

所以我猜测微信这么做的原因是为了检测和控制第三方接入，由此开发者必然受制于它的SDK。而且在自己的应用里集成一个不开源的SDK，心里多少都会不愉快，谁知道它会在后面干什么呢？

首先就看看微信的 SDK 有没有 CocoaPods：

```bash
$ pod search wechat
```

结果有不少，我选择了 WeixinSDK 1.4.3，那就将其放到项目的 `podfile` 里安装好。

因为目前我们的项目使用 Swift 1.2，于是在`*-Bridging-Header.h`里

```objective-c
#import "WXApi.h"
```

这个头文件给人一种恶心的感觉！但也没有办法，试试编译。结果run不过去。一看，WeixinSDK 的两个头文件之一的 `WXApiObject.h` 里用了 UIImage，可它居然不引用 `UIKit/UIKit.h`，心中升起莫名的怒火。也许它们是为了打包方便，但这是一种极其不尊重第三方开发者的行为。

那好，我手动在`*-Bridging-Header.h`里多加一句：

```objective-c
#import <UIKit/UIKit.h>
#import "WXApi.h"
```

成功通过编译。

那就先来测试一下，安装其几乎无法目睹的文档：

1. 先添加 CFBundleURLTypes，……，便于回调
2. 再在 appDelegate 的 didFinishLauching 里 `WXApi.registerApp("appID...")`，心中更怒了，为何要在这么早注册它的 appID，我还没打算分享呢。后来百度LBS的李择一说光是这一句，微信SDK就在后面做数据收集了，它会往[http://pingma.qq.com/mstat/report](http://pingma.qq.com/mstat/report)发送RC4加密的数据。鬼才知道它还会干什么！
3. 测试分享链接：

	```swift
    let sendMessageRequest = SendMessageToWXReq()

    sendMessageRequest.scene = 1 // 朋友圈

    let message = WXMediaMessage()
    message.title = "Test"
    message.description = "test share apple.com"

    let webObject = WXWebpageObject()
    webObject.webpageUrl = "http://www.apple.com"
    message.mediaObject = webObject

    sendMessageRequest.message = message

    WXApi.sendReq(sendMessageRequest)
	```
4. 如果要处理微信的回调，还得在 appDelegate 里添加：

	```swift
    func application(application: UIApplication, handleOpenURL url: NSURL) -> Bool {
        return WXApi.handleOpenURL(url, delegate: self)
    }

    func application(application: UIApplication, openURL url: NSURL, sourceApplication: String?, annotation: AnyObject?) -> Bool {
        return WXApi.handleOpenURL(url, delegate: self)
    }
	```
	
	鬼才知道它会不会在 handleOpenURL 时收集数据啊！
	
通常，第 3 步时还要检查 `WXApi.isWXAppInstalled()` 和 `WXApi.isWXAppSupportApi()`，可我发现，在测试剂已安装了微信的前提下，前者可能返回 false，后者返回 true，我已经出离愤怒了。也许微信后台还有激活时间段？后来过了几个小时，这两个方法都返回 true 了，但谁也不知道中间发生了什么。

最后，为了使用 UIActivityController 来做系统分享，我们还必须制作 UIActivity，可微信的分享还分成 Session（聊天） 和 Timeline（朋友圈），它一个破 App 居然要占据两个位置。

还好这并不是很困难的事情：

```swift
class WeChatActivity: UIActivity {

    enum Scene {
        case Session    // 聊天界面
        case Timeline   // 朋友圈

        var value: Int32 {
            switch self {
            case Session:
                return 0
            case Timeline:
                return 1
            }
        }

        var activityType: String {
            switch self {
            case Session:
                return "xxx.shareToWeChatSession"
            case Timeline:
                return "xxx.shareToWeChatTimeline"
            }
        }

        var activityTitle: String {
            switch self {
            case Session:
                return NSLocalizedString("WeChat Session", comment: "")
            case Timeline:
                return NSLocalizedString("WeChat Timeline", comment: "")
            }
        }

        var activityImage: UIImage? {
            switch self {
            case Session:
                return UIImage(named: "wechat_session")
            case Timeline:
                return UIImage(named: "wechat_timeline")
            }
        }
    }

    struct Message {
        let title: String?
        let description: String?
        let thumbnail: UIImage?

        enum Media {
            case URL(NSURL)
            case Image(UIImage)
        }

        let media: Media
    }

    let scene: Scene
    let message: Message

    init(scene: Scene, message: Message) {
        self.scene = scene
        self.message = message

        super.init()
    }

    override class func activityCategory() -> UIActivityCategory {
        return .Share
    }

    override func activityType() -> String? {
        return scene.activityType
    }

    override func activityTitle() -> String? {
        return scene.activityTitle
    }

    override func activityImage() -> UIImage? {
        return scene.activityImage
    }

    override func canPerformWithActivityItems(activityItems: [AnyObject]) -> Bool {

        if WXApi.isWXAppInstalled() && WXApi.isWXAppSupportApi() {
            return true
        }

        return false
    }

    override func performActivity() {

        let request = SendMessageToWXReq()

        request.scene = scene.value

        let message = WXMediaMessage()

        message.title = self.message.title
        message.description = self.message.description
        message.setThumbImage(self.message.thumbnail)

        switch self.message.media {

        case .URL(let URL):
            let webObject = WXWebpageObject()
            webObject.webpageUrl = URL.absoluteString!
            message.mediaObject = webObject

        case .Image(let image):
            let imageObject = WXImageObject()
            imageObject.imageData = UIImageJPEGRepresentation(image, 1)
            message.mediaObject = imageObject
        }

        request.message = message

        WXApi.sendReq(request)

        activityDidFinish(true)
    }
}
```

之后使用时就可以放到 UIActivityViewController 里了，当然，还得请设计师画两个图标。

![系统分享微信](https://raw.githubusercontent.com/nixzhu/MonkeyKing/master/images/system_share.jpg)

事情到这一步好像就结束了，但我心中的愤怒丝毫没有减少。

第二天，[Limon](http://weibo.com/u/1783821582)提到一个开源项目：[openshare][1]，不用那些第三方不开源的SDK即可做到分享或登录等操作，我心中很是激动！

作为一种研究，我决定分析 [openshare][1] 的实现（作者已写了很好的[文章](http://www.gfzj.us/series/openshare/)），然后用 Swift 写一个支持分享到微信的功能即可，这也是目前我们应用只用到的部分。

[openshare][1] 的原理很简单，分析第三方SDK在分享跳转时所传递的数据然后自行构造同样的数据流。虽说原理简单，但想必作者花了不少时间调试和测试。

简单来说，走微信分享时，我们要用微信的 scheme 打开它，这个 URL 里有不少参数，比较大的数据，比如分享图片时的图片要放在系统粘贴板里，因为系统粘贴板是所有应用都可以访问的。

其它细节就不赘述了。不过我发现，相对来说，微信的机制比 QQ 好一点，微信利用的 PropertyList 来做数据交换，比较好格式化；而 QQ 可能比较原始，要在 URL 里编码大量数据。虽说目的都一样，但过程的体验不同。

最后得到一个开源项目：[MonkeyKing][2]，以 Swift 2 编写，但它目前支持只分享链接和图片到微信或 QQ。若你有其它需求，还是请考虑 [openshare][1]。由此，我们就避免了使用微信的SDK带来的愤怒，也避免了用户所要面临的不必要的风险。

不过 [MonkeyKing][2] 的代码比较简短，你若想看实现的细节会比较容易些。

最后我想说，国产SDK的安全性在不开源的情况下几乎没有任何保障。大家能避免使用的时候就尽量避免，能延迟调用的API都尽量延迟（例如`WXApi.registerApp("appID...")`），产品经理也要考虑自己负责的 App 是否需要集成第三方的登录功能（有的 App 真的不需要做一个用户系统，反而受制于人）。

还要加一句，微信的公众号绝对违反万维网的互联精神，任其做大，后患无穷。该独立的时候还请保持独立。

[1]: https://github.com/100apps/openshare
[2]: https://github.com/nixzhu/MonkeyKing

===============

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

如果你认为这篇文章不错，也有闲钱，那你可以用支付宝扫描下方二维码随便捐助一点，以慰劳作者的辛苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

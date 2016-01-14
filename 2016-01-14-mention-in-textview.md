
# 自定义 UITextView 关键字高亮与点击检测

一种很简单的方法，妙手偶得，可比较容易地处理 Mention、Hashtag 等

作者：[@nixzhu](https://twitter.com/nixzhu)

---

我们大概都知道，设置好 UITextView 的 dataDetectorTypes 后，就可自动检测并高亮网络链接、电话号码、地址等功能。而且在 iPhone 6s (Plus) 上，还可以 3D Touch 网络链接，直接就有 pop/peek 的功能。

若我们要增加对 mention 或 hashtag 的检测，例如 `*@nixzhu* hello` 里的 `@nixzhu` 或 `what a great #weather!` 里的 `#weather`，该怎么办呢？

你可能会想着去找一个开源库，例如[RichTextView](https://github.com/kevinzhow/RichTextView)或[SwiftyText](https://github.com/kejinlu/SwiftyText)，它们可能支持很多种标记类型的检测，但也许你就只是需要检测 mention 而已。你当然可以研究其实现来自己改写，看一看 TextKit 相关的文档或 session 视频，弄明白 NSTextStorage、NSLayoutManager、NSTextContainer 等与 UITextView 的关系。

不过今天我发现并实验了一种比较简单的办法，不需要去了解太复杂的东西（当然 TextKit 仍然值得被了解），所以分享在此。

以字符串 `let text = "@nixzhu Do you like Apple? www.apple.com"` 为例，我们希望 `@nixzhu` 高亮且可点击，点击后自然可以执行某种操作。

首先，UITextView 有 `attributedText: NSAttributedString` 属性。我们可以利用正则表达式给上面的字符串的 `@nixzhu` 加上属性，得到一个 NSAttributedString 再设置给 UITextView，这样高亮就很好实现了。代码大概如下：

``` swift
let text = "@nixzhu Do you like Apple? www.apple.com"

let attributedString = NSMutableAttributedString(string: text)

let textRange = NSMakeRange(0, (text as NSString).length)

let mentionPattern = "@[A-Za-z0-9_]+" // 可能有更好的正则模式，或者更适合你 app 的正则模式
let mentionExpression = try! NSRegularExpression(pattern: mentionPattern, options: NSRegularExpressionOptions())

mentionExpression.enumerateMatchesInString(text, options: NSMatchingOptions(), range: textRange, usingBlock: { result, flags, stop in

    if let result = result {
        let subString = (self.text as NSString).substringWithRange(result.range)

        let attributes: [String: AnyObject] = [
            NSLinkAttributeName: subString,
            "CustomDetectionType": "Mention",
        ]

        attributedString.addAttributes(attributes, range: result.range )
    }
})

// TOOD: textView.attributedText = attributedString
```

这里我们给符合模式的子字符串添加上了 NSLinkAttributeName 属性和一个自定义检测属性。如果你将 UITextView 的 URL 检测打开，你可以看到 `@nixzhu` 与 `www.apple.com` 都有了颜色和下划线。也就是说，iOS 会将有 NSLinkAttributeName 属性的字符串当做链接。

![Mention in TextView](https://raw.githubusercontent.com/nixzhu/dev-blog/master/images/mention_textview.png)

此时，我们点击 URL 有反应，点击 mention 也有类似效果，只是没有后续操作，因为我们还没做自定义。

上面提到 iOS 会将有 NSLinkAttributeName 属性的字符串当做链接，因此，我们可以利用 UITextViewDelegate 的一个方法来做处理：

``` swift
extension MentionDetectableTextView: UITextViewDelegate {

    func textView(textView: UITextView, shouldInteractWithURL URL: NSURL, inRange characterRange: NSRange) -> Bool {

        guard let detectionType = self.attributedText.attribute("CustomDetectionType", atIndex: characterRange.location, effectiveRange: nil) as? String where detectionType == "Mention" else {
            return true
        }

        let text = (self.text as NSString).substringWithRange(characterRange)
        let username = text.substringFromIndex(text.startIndex.advancedBy(1))

        if !username.isEmpty {
            tapMentionAction?(username: username)
        }

        return true
    }
}
```

先 guard 看看是否是自定义的类型，不然直接返回 true 让系统处理。若能继续，我们就可以通过参数拿到 username 进而触发一个操作。这里的操作是一个可选闭包，可以在其它地方如 UIViewController 里赋值以做进一步处理。

这样我们就完成了对 mention 的高亮显示和点击检测，其体验和链接一致。而且，原来的链接检测不受影响，默认的 pop／peek 依然有效。

具体代码请看 Demo：[https://github.com/nixzhu/MentionInUITextViewDemo](https://github.com/nixzhu/MentionInUITextViewDemo)

当然，若你还要检测 hashtag 等，稍微修改即可。

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条

* Tweet [https://twitter.com/nixzhu/status/687456414386622464](https://twitter.com/nixzhu/status/687456414386622464) 或
* 微博 [http://weibo.com/2076580237/Dd3N0869i](http://weibo.com/2076580237/Dd3N0869i)  

以分享此文或参与讨论！

虽然我留了一个捐助的二维码，但似乎读者很少给机会让我买早餐：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

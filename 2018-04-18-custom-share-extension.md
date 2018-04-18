

# 自定义Share Extension

作者：[@nixzhu](https://twitter.com/nixzhu)

---

在iOS上，若一个app要接收从其它app过来的数据，通常的做法是实现一个分享扩展。例如，从相册分享图片到你的app中，或者从Safari分享链接到你的app中。

如果你的需求比较简单，那继承`SLComposeServiceViewController`使用系统提供的UI将最方便。但如果设计上有更复杂的要求，你就只能通过自定义`UIViewController`来做了。

通常，分享扩展的起始界面由`MainInterface.storyboard`指定，如果你不想使用Storyboard，也可以修改分享扩展中的Info.plist来指定一个`NSExtensionPrincipalClass` ，它可以直接继承自UIViewController，你可以[在此](https://developer.apple.com/library/content/documentation/General/Conceptual/ExtensibilityPG/ExtensionCreation.html)找到更详细的说明。不过，我们也可以直接修改Storyboard，增加一个View Controller并指定其为Initial View Controller，然后让这个View Controller使用我们自定义的UIViewController。

如果你的自定义UI不是全屏的，我会建议你在之前的Initial View Controller里增加一个Container View，形如：

![Container View](https://github.com/nixzhu/dev-blog/raw/master/images/custom-share-extension-storyboard.png)

这样，基本的UI框架就OK了。如果你要做动画，那在第一个控制器里让Container View动画即可，后续的控制器可以利用delegate来让第一个控制器做事。

UI确定后，接下来考虑获取分享数据。假设从系统相册分享图片。

在App Extension里，UIViewController新增了一个属性`var extensionContext: NSExtensionContext?`，通过它，我们可以让Initial View Controller准备好数据。我们先给`NSExtensionContext`增加一个扩展方法：

``` swift
extension NSExtensionContext {

    func circle_images(in vc: UIViewController, completion: @escaping (_ images: [ShareInfo.Image]) -> Void) {
        let extensionContext = self
        guard let extensionItems = extensionContext.inputItems as? [NSExtensionItem] else {
            return completion([])
        }
        var images: [ShareInfo.Image] = []
        let imageTypeIdentifier = kUTTypeImage as String
        let group = DispatchGroup()
        for extensionItem in extensionItems {
            for attachment in extensionItem.attachments as! [NSItemProvider] {
                if attachment.hasItemConformingToTypeIdentifier(imageTypeIdentifier) {
                    group.enter()
                    var previewImage: UIImage?
                    var fileURL: URL?
                    let loadGroup = DispatchGroup()
                    loadGroup.enter()
                    attachment.loadPreviewImage(options: [:]) { secureCoding, _ in
                        defer {
                            loadGroup.leave()
                        }
                        previewImage = secureCoding as? UIImage
                    }
                    let previewImagePreferredSize = CGSize(width: 300, height: 300)
                    loadGroup.enter()
                    attachment.loadItem(forTypeIdentifier: imageTypeIdentifier, options: nil) { secureCoding, _ in
                        defer {
                            loadGroup.leave()
                        }
                        if let url = secureCoding as? URL {
                            fileURL = url
                        } else if let image = secureCoding as? UIImage {
                            if let data = UIImageJPEGRepresentation(image, 0.9) {
                                let imageName = "\(UUID().uuidString).jpg"
                                let tempImageURL = FileManager.default.temporaryDirectory.appendingPathComponent(imageName)
                                do {
                                    try data.write(to: tempImageURL)
                                    fileURL = tempImageURL
                                    let fixedSize = image.size.circle_fixed(forPreferredSize: previewImagePreferredSize)
                                    previewImage = image.yy_imageByResize(to: fixedSize)
                                } catch {
                                    vc.circle_alert(message: "\(error)")
                                }
                            }
                        }
                    }
                    loadGroup.notify(queue: .main) { [weak self] in
                        defer {
                            group.leave()
                        }
                        guard let fileURL = fileURL else { return }
                        if previewImage == nil {
                            if let image = UIImage(contentsOfFile: fileURL.path) {
                                let fixedSize = image.size.circle_fixed(forPreferredSize: previewImagePreferredSize)
                                previewImage = image.yy_imageByResize(to: fixedSize)
                            }
                        }
                        guard let previewImage = previewImage else { return }
                        let image = ShareInfo.Image(
                            previewImage: previewImage,
                            fileURL: fileURL
                        )
                        images.append(image)
                    }
                }
            }
        }
        group.notify(queue: .main) {
            completion(images)
        }
    }
}
```

除开两层DispatchGroup，这个方法没有难以理解的东西。但要注意的是，loadItem回调里的secureCoding既可能是一个文件URL也可能是一个UIImage（还有其他的可能性，参考[The Struggle with Action Extensions](https://pspdfkit.com/blog/2017/action-extension/)）。而且，loadPreviewImage的回调里不一定能找到UIImage。但通常，若从系统相册分享，Preview Image一般都存在；若从其它地方分享，secureCoding一般都是UIImage或文件URL，Preview Image不一定存在（虽然，按照Apple的建议，数据提供方有责任提供Preview Image）。

其中，`ShareInfo`是一个类似这样的结构：

``` swift
struct ShareInfo {
    struct Image {
        let previewImage: UIImage
        let fileURL: URL
    }
    var images: [Image] = []
    //...
}
```

注意我们用fileURL来指定原图片，而不是直接将其数据拿到生成UIImage，这里有内存占用的考量。因为在Share Extension中，我们可以使用的内存比较有限（iPhone 7上大约70MB），如果用户选择了很多图片，而我们又全部生成UIImage，那内存很可能暴涨，我们的Share Extension进程就会被iOS强制杀掉。此外，用户选择的图片可能是GIF，你可能也需要对它进行特殊的判断，超过一定的大小可能要提示用户或者放弃分享。

对于其它数据，例如Text、Web URL或者File，你可以写出类似的扩展方法。

有了数据之后，就是具体的分享操作了。如果你的app架构合理，例如使用了Framework来封装核心功能，并能在扩展中使用这些Framework，那么你会比较轻松。不然，你要整理一些分享扩展中用到的逻辑，提取代码公用。此外，就是再次关注使用fileURL时的内存占用，你可能需要一些锁机制，一次只处理一个fileURL，让内存能被及时回收。

分享完成或者放弃分享后，正确调用extensionContext的

``` swift
func completeRequest(returningItems items: [Any]?, completionHandler: ((Bool) -> Swift.Void)? = nil)
```

或

``` swift
func cancelRequest(withError error: Error)
```

来确保分享扩展被正确释放。

如果你需要在后台发送，直接使用Background Task可能不行（参考[What we learned building the Tumblr iOS share extension](https://engineering.tumblr.com/post/97658880154/what-we-learned-building-the-tumblr-ios-share)）。我使用的一个hack是先调用completeRequest，但在其completionHandler里等待一个信号量。这样分享扩展并不会立即被释放，让你的上传有时间在后台完成（完成后再发送信号量）。

最后，如果你的app会作为系统分享的数据源，除了数据的质量外，你有责任准备Preview Image，请参考[NSItemProvider](https://developer.apple.com/documentation/foundation/nsitemprovider)相关的API。

广告时间：[「圈子」1.1版](https://itunes.apple.com/cn/app/id1299590483)现已上线，终于可以自由建圈了，欢迎尝试！或者加入我创建的[「可爱的Bug」](https://share.quanziapp.com/circle/sVBgoaB)圈来分享你与Bug的故事。我相信，被说出来的Bug将无处遁形。

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)
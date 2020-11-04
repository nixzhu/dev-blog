# 研究 UIActivityViewController

本文翻译自 [http://nshipster.com/uiactivityviewcontroller/](http://nshipster.com/uiactivityviewcontroller/)

原作者：[Mattt Thompson](http://mattt.me/)

译者：[@nixzhu](https://twitter.com/nixzhu)

---

数据与代码的关系一直都让人好奇。

特定的编程语言，如 [Lisp](http://en.wikipedia.org/wiki/Lisp_programming_language)、[lo](http://en.wikipedia.org/wiki/Io_%28programming_language%29) 和 [Mathematica](http://en.wikipedia.org/wiki/Mathematica) 都是[同像性的（homoiconic）](http://en.wikipedia.org/wiki/Homoiconicity)，意味着它们的代码可作为数据原语呈现，也就是说它们自身就可在代码中被操纵。许多其他语言，包括 Objective-C ，就不同了，它们在这两者之间建立了严格的界限，回避 `eval()` 和其它潜在的动态指示加载方法，以回避风险。

当问题里的数据过大或难以表示为除了字节流之外的其他东西时，那么代码与数据的这种紧张关系就达到了一个新的高度。关于“如何编码、解码以及解释图像、文档和媒体的二进制表示”的问题从最早的操作系统开始就一直存在着。

OS X 的 `Core Services 框架`与 iOS 的`移动 Core Services 框架`都提供函数通过[通用类型标识符（Universal Type Identifiers，即UTI）](http://en.wikipedia.org/wiki/Uniform_Type_Identifier)来根据文件扩展和[MIME类型](http://en.wikipedia.org/wiki/Internet_media_type)识别和分类数据类型。UTI提供了可扩展和可继承的分类系统，它能给予开发人员极大的灵活性，即使是处理最奇特的文件类型。例如，一个 Ruby 源代码文件（.rb）被分类为 Ruby 源代码 > 源代码 > 文本 > 内容 > 数据；一个 QuickTime 电影文件（.mov）被分类为视频 > 电影 > 试听内容 > 内容 > 数据；

在桌面文件系统抽象里，UTI工作得相当好。然而，在一个移动范式里，文件和目录对于用户来说都被隐藏了，于是这很快就失效了。而且，更重要的是，云服务和社交媒体的兴起已经让远程实体比本地文件具有更重要的地位。因此，UTI和URL之间出现了紧张关系。

很明显我们需要其它的某种东西。那 UIActivityViewController 能成为我们拼命追求的解决办法吗？

---

UIActivityViewController ，出现于 iOS 6，在应用里为分享和操作数据提供了一个统一的服务接口。

给出一个可操作数据的集合，那一个 UIActivityViewController 实例就可如下创建：

```Objective-C
NSString *string = ...;
NSURL *URL = ...;

UIActivityViewController *activityViewController =
  [[UIActivityViewController alloc] initWithActivityItems:@[string, URL]
                                    applicationActivities:nil];
[navigationController presentViewController:activityViewController
                                      animated:YES
                                    completion:^{
  // ...
}];
```

这将在屏幕的底部呈现如下所示的东西：

![UIActivityViewController](http://nshipster.s3.amazonaws.com/uiactivityviewcontroller.png)

默认情况下，UIActivityViewController 将显示所有可用于所提供内容的服务，但我们也可以排除特定的 Activity 类型。

```Objective-C
activityViewController.excludedActivityTypes = @[UIActivityTypePostToFacebook];
```

Activity 类型又分为“操作”和“分享”两大类：

#### UIActivityCategoryAction

- UIActivityTypePrint
- UIActivityTypeCopyToPasteboard
- UIActivityTypeAssignToContact
- UIActivityTypeSaveToCameraRoll
- UIActivityTypeAddToReadingList
- UIActivityTypeAirDrop


#### UIActivityCategoryShare

- UIActivityTypeMessage
- UIActivityTypeMail
- UIActivityTypePostToFacebook
- UIActivityTypePostToTwitter
- UIActivityTypePostToFlickr
- UIActivityTypePostToVimeo
- UIActivityTypePostToTencentWeibo
- UIActivityTypePostToWeibo

每个 Activity 类型都支持好多种不同的数据类型。例如，一条 Tweet 可能由 NSString 以及一个附加的图像 和/或 URL 所组成。

### 不同的 Activity 类型所支持的数据类型

<table>
  <thead>
    <tr>
      <th>
        Activity 类型
      </th>
      <th>
        字符串
      </th>
      <th>
        属性字符串
      </th>
      <th>
        URL
      </th>
      <th>
        Data
      </th>
      <th>
        图像
      </th>
      <th>
        Asset
      </th>
      <th>
        其它
      </th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>
        发布到 Facebook
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        ✓
      </td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>
        发布到 Twitter
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        ✓
      </td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>
        发布到 Weibo
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
    </tr>
    <tr>
      <td>
        信息
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓*
      </td>
      <td>
        ✓*
      </td>
      <td></td>
      <td>
        ✓*
      </td>
      <td>
        <tt>sms://</tt> <tt>NSURL</tt>
      </td>
    </tr>
    <tr>
      <td>
        邮件
      </td>
      <td>
        ✓+
      </td>
      <td>
        ✓+
      </td>
      <td>
        ✓+
      </td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>
        打印
      </td>
      <td></td>
      <td></td>
      <td></td>
      <td>
        ✓+
      </td>
      <td>
        ✓+
      </td>
      <td></td>
      <td>
        <tt>UIPrintPageRenderer</tt>,
        <tt>UIPrintFormatter</tt>, &amp;
        <tt>UIPrintInfo</tt>
      </td>
    </tr>
    <tr>
      <td>
        拷贝到剪贴板
      </td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        <tt>UIColor</tt>, <tt>NSDictionary</tt>
      </td>
    </tr>
    <tr>
      <td>
        添加到联系人
      </td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
      <td>
        ✓
      </td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>
        保存到相机胶卷
      </td>
      <td></td>
      <td></td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        ✓
      </td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>
        添加到阅读列表
      </td>
      <td></td>
      <td></td>
      <td>
        ✓
      </td>
      <td></td>
      <td></td>
      <td></td>
      <td></td>
    </tr>
    <tr>
      <td>
        发布到 Flickr
      </td>
      <td></td>
      <td></td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
    </tr>
    <tr>
      <td>
        发布到 Vimeo
      </td>
      <td></td>
      <td></td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        ✓
      </td>
      <td></td>
    </tr>
    <tr>
      <td>
        发布到腾讯微博
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
    </tr>
    <tr>
      <td>
        AirDrop
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
      <td>
        ✓
      </td>
      <td>
        ✓
      </td>
      <td></td>
    </tr>
  </tbody>
</table>

## &lt;UIActivityItemSource> & UIActivityProvider

类似于一个[剪贴板条目](https://developer.apple.com/library/mac/documentation/cocoa/reference/NSPasteboardItem_Class/Reference/Reference.html)只在必要时才提供数据，为了避免过多的内存分配或处理时间， Activity 条目可以是自定义类型。

### &lt;UIActivityItemSource>

获取数据项

- activityViewControllerPlaceholderItem:
- activityViewController:itemForActivityType:

提供数据项信息

- activityViewController:subjectForActivityType:
- activityViewController:dataTypeIdentifierForActivityType:
- activityViewController:thumbnailImageForActivityType:suggestedSize:

一个关于这些方法如何使用的例子是自定义一个消息，其根据是否要分享到 Facebook 或 Twitter 分别定义。

```Objective-C
- (id)activityViewController:(UIActivityViewController *)activityViewController
         itemForActivityType:(NSString *)activityType
{
    if ([activityType isEqualToString:UIActivityTypePostToFacebook]) {
        return NSLocalizedString(@"Like this!");
    } else if ([activityType isEqualToString:UIActivityTypePostToTwitter]) {
        return NSLocalizedString(@"Retweet this!");
    } else {
        return nil;
    }
}
```

## 创建一个自定义 UIActivity

除了上述系统提供的 Activity ，你也可以自己创建 Activity。

作为例子，让我们创建一个自定义 Activity 类型，它能接受一个图片 URL 并使用 [mustache.me](http://mustache.me/) 为其安上一瞥胡子。

![Jony Ive Before](http://nshipster.s3.amazonaws.com/jony-ive-unstache.png)之前

![Jony Ive After](http://nshipster.s3.amazonaws.com/jony-ive-mustache.png)之后

首先，我们为 Activity 类型定义一个[反向DNS标识符（reverse-DNS identifier）](http://en.wikipedia.org/wiki/Reverse_domain_name_notation)，指定类别为 UIActivityCategoryAction ，然后提供一个本地化的标题和一个合适于iOS版本的图像：

```Objective-C
static NSString * const HIPMustachifyActivityType = @"com.nshipster.activity.Mustachify";
```

```Objective-C
#pragma mark - UIActivity

+ (UIActivityCategory)activityCategory {
    return UIActivityCategoryAction;
}

- (NSString *)activityType {
    return HIPMustachifyActivityType;
}

- (NSString *)activityTitle {
    return NSLocalizedString(@"Mustachify", nil);
}

- (UIImage *)activityImage {
    if (NSFoundationVersionNumber > NSFoundationVersionNumber_iOS_6_1) {
        return [UIImage imageNamed:@"MustachifyUIActivity7"];
    } else {
        return [UIImage imageNamed:@"MustachifyUIActivity"];
    }
}
```

接下来，我们创建一个帮助函数，HIPMatchingURLsInActivityItems，它返回一个由任何所支持类型的图像 URL 组成的数组。

```Objective-C
static NSArray * HIPMatchingURLsInActivityItems(NSArray *activityItems) {
    return [activityItems filteredArrayUsingPredicate:[NSPredicate predicateWithBlock:
    ^BOOL(id item, __unused NSDictionary *bindings) {
        if ([item isKindOfClass:[NSURL class]] &&
            ![(NSURL *)item isFileURL]) {
            return [[(NSURL *)item pathExtension] caseInsensitiveCompare:@"jpg"] == NSOrderedSame ||
            [[(NSURL *)item pathExtension] caseInsensitiveCompare:@"png"] == NSOrderedSame;
        }

        return NO;
    }]];
}
```

这个函数用于 `-canPerformWithActivityItems:` 和 `prepareWithActivityItems:` 以取得第一个 PNG 或 JPEG 的加了胡子的图像 URL，如果有的话。

```Objective-C
- (BOOL)canPerformWithActivityItems:(NSArray *)activityItems {
    return [HIPMatchingURLsInActivityItems(activityItems) count] > 0;
}

- (void)prepareWithActivityItems:(NSArray *)activityItems {
    static NSString * const HIPMustachifyMeURLFormatString = @"http://mustachify.me/%d?src=%@";

    self.imageURL = [NSURL URLWithString:[NSString stringWithFormat:HIPMustachifyMeURLFormatString, self.mustacheType, [HIPMatchingURLsInActivityItems(activityItems) firstObject]]];
}
```

我们的网络服务提供了好几种胡子选项，它们定义在一个 NS_ENUM 中：

```Objective-C
typedef NS_ENUM(NSInteger, HIPMustacheType) {
    HIPMustacheTypeEnglish,
    HIPMustacheTypeHorseshoe,
    HIPMustacheTypeImperial,
    HIPMustacheTypeChevron,
    HIPMustacheTypeNatural,
    HIPMustacheTypeHandlebar,
};
```

最终，我们提供一个 UIViewController 来显示图像。在这个例子里，一个简单的 UIWebView 控制器就够了。

```Objective-C
@interface HIPMustachifyWebViewController : UIViewController <UIWebViewDelegate>
@property (readonly, nonatomic, strong) UIWebView *webView;
@end
```

```Objective-C
- (UIViewController *)activityViewController {
    HIPMustachifyWebViewController *webViewController = [[HIPMustachifyWebViewController alloc] init];

    NSURLRequest *request = [NSURLRequest requestWithURL:self.imageURL];
    [webViewController.webView loadRequest:request];

    return webViewController;
}
```

要使用我们全新的胡子 Activity，我们简单地将其传递给一个 UIActivityViewController 的初始化函数即可：


```Objective-C
HIPMustachifyActivity *mustacheActivity = [[HIPMustachifyActivity alloc] init];
UIActivityViewController *activityViewController =
  [[UIActivityViewController alloc] initWithActivityItems:@[imageURL]
                                    applicationActivities:@[mustacheActivity];
```

## 手动调用操作

现在正是回忆起 “UIActivityViewController 允许用户执行它们选择的操作” 的好时机，但当情况需要时，分享依然可以手动调用。

为了完整性，下面就来介绍手动执行这些操作的步骤：

### 打开 URL

```Objective-C
NSURL *URL = [NSURL URLWithString:@"http://nshipster.com"];
[[UIApplication sharedApplication] openURL:URL];
```

系统支持的 URL scheme 包括：mailto://、tel://、sms://、and maps://。

### 添加到 Safari 阅读列表

```Objective-C
@import SafariServices;

NSURL *URL = [NSURL URLWithString:@"http://nshipster.com/uiactivityviewcontroller"];
[[SSReadingList defaultReadingList] addReadingListItemWithURL:URL
                                                        title:@"NSHipster"
                                                  previewText:@"..."
                                                        error:nil];
```

### 保存到相册

```Objective-C
UIImage *image = ...;
id completionTarget = self;
SEL completionSelector = @selector(didWriteToSavedPhotosAlbum);
void *contextInfo = NULL;
UIImageWriteToSavedPhotosAlbum(image, completionTarget, completionSelector, contextInfo);
```

### 发送短信

```Objective-C
@import MessageUI;

MFMessageComposeViewController *messageComposeViewController = [[MFMessageComposeViewController alloc] init];
messageComposeViewController.delegate = self;
messageComposeViewController.recipients = @[@"mattt@nshipster•com"];
messageComposeViewController.body = @"Lorem ipsum dolor sit amet";
[navigationController presentViewController:messageComposeViewController animated:YES completion:^{
    // ...
}];
```

### 发送邮件

```Objective-C
@import MessageUI;

MFMailComposeViewController *mailComposeViewController = [[MFMailComposeViewController alloc] init];
[mailComposeViewController setToRecipients:@[@"mattt@nshipster•com"]];
[mailComposeViewController setSubject:@"Hello"];
[mailComposeViewController setMessageBody:@"Lorem ipsum dolor sit amet"
                                   isHTML:NO];
[navigationController presentViewController:mailComposeViewController animated:YES completion:^{
    // ...
}];
```

### 发送推文

```Objective-C
@import Twitter;

TWTweetComposeViewController *tweetComposeViewController =
    [[TWTweetComposeViewController alloc] init];
[tweetComposeViewController setInitialText:@"Lorem ipsum dolor sit amet."];
[self.navigationController presentViewController:tweetComposeViewController
                                        animated:YES
                                      completion:^{
    //...
}];
```

## IntentKit

虽然所有这些都让人影响深刻也很有用，但在与 Android 上的丰富的 Intent 模型对比之下，iOS 的 Activity 范式中还是有一些特殊的缺失。

在 Android 上，应用可以注册不同的 Intent，以表面它们可用于地图或作为浏览器，而且能被选择为相关 Activity 的默认应用，例如导航或将某个URL加入书签。

虽然 iOS 缺少可扩展的基础架构来支持这些，但一个第三方的库，叫做 IntentKit，由 [@lazerwalker ](https://github.com/lazerwalker)(有着 [f*ingblocksyntax.com](http://goshdarnblocksyntax.com/) 的声誉) 编写，它是一个有趣的关于我们如何缩小差距的例子。

![IntentKit](https://raw.github.com/intentkit/IntentKit/master/example.gif)

正常情况下，在一开始一个开发者就要做许多工作，例如查询某个特定的应用是否已被安装，以及构造一个 URL 以支持某个特定的 Activity 等。

IntentKit 合并了连接到这些最流行的服务（如 Web、地图、邮件、Twitter、Facebook以及Google+这些客户端）的逻辑，而且它的 UI 非常类似 UIActivityViewController。

任何想将其应用的分享体验提高一个层次的人都应该研究一下它。

还有一个要发出的有力争论是，关于 iOS 平台的长期生存能力取决于如 UIActivityViewController 这样的分享机制。俗话说，“信息需要自由”。而任何阻挡人民联盟的事物终将输给那些不阻挡联盟的事物。

公开的 [远程视图控制器（remote view controller）](http://oleb.net/blog/2012/10/remote-view-controllers-in-ios-6/) API 的未来前景给了我关于iOS上分享的未来希望。虽然对现在来说，我们当然可以做得比 UIActivityViewController 更差。（译者注：是这个意思吗？ For now, though, we could certainly do much worse than UIActivityViewController.）

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

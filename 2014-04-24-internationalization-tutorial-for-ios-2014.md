# iOS 应用的国际化（2014）

本文翻译自 [http://www.raywenderlich.com/64401/internationalization-tutorial-for-ios-2014](http://www.raywenderlich.com/64401/internationalization-tutorial-for-ios-2014)

原作者：[Ali Hafizji](http://www.raywenderlich.com/u/ali.hafizji)

译者：[@nixzhu](https://twitter.com/nixzhu)

==========================================

2014 年 4 月 23 日更新：本文最初由 [Sean Berry](http://twitter.com/regularberry) 编写， 因为 iOS 7，现在由 [Ali Hafizji](http://www.raywenderlich.com/u/ali.hafizji) 将此文全面更新。

创造出色应用算得上是不小的壮举，除开伟大的代码、华丽的设计以及直观的交互外，还有更多要做的事。要爬上 App Store 的排行榜需要适时的产品营销、跟随着用户群不断扩大的能力，以及利用工具和技术尽可能广泛地达到越多的受众越好。

国际市场对许多开发者来说是事后考虑，但感谢由 App Store 所提供的无痛的全球分发系统，任何 iOS 开发者要将他们的应用在超过 150 个国家的市场发布，只需要一个简单的点击。亚洲和欧洲分别代表着一个不断增长的潜在客户群，他们中的大多数人都不以英语为母语，但为了充分发掘你的应用在全球市场的潜力，你至少要将应用的国际化语言做好。

本教程将对一个叫做 `iLikeIt` 的简单应用添加国际化支持来引导你学一遍国际化的基本内容。这个简单的应用有一个 Label 和一个 `You Like?` Button。无论用户何时点击 `You Like?` ，一些乐观的销售数据和对应的图片就在按钮下以淡入的方式出现。

但目前，这个应用还只支持英文—— os vamos a traducir!

>Note：国际化的另一个重要方面是使用 Auto Layout 去处理文字大小的变化。然而，为了保持本教程的简单，我们不会专注于 Auto Layout ，但我们为其准备有[另外的教程](http://www.raywenderlich.com/64392/video-tutorial-beginning-auto-layout)。

## 国际化 vs 本地化

在你开始本教程之前，明白国际化与本地化的区别非常重要，而这些内容常常让人困惑。

简单地说，国际化是为国际化能力而设计你的应用的过程。举例来说：

- 以用户的母语处理文字地输入、输出。
- 处理不同的日期、时间以及数字格式。
- 利用适当的日历和时区来处理数据。

国际化对于你（开发者）来说是一个小活动，即利用系统提供的 API 来增添或修改你的代码以使得你的应用在中文或阿拉伯语下同在英文下一样好。

对比之下，本地化仅仅是将应用的用户界面和资源翻译至不同的语言，这是一些你能够且应该交给别人的事情，除非碰巧你精通你的应该所支持的每一种语言。:]

## 开始

第一步就来[下载](http://cdn2.raywenderlich.com/wp-content/uploads/2014/03/StarterKit.zip) iLikeIt 启动项目，整篇教程里都会用到它。

用 Xcode 5 打开项目，并在模拟器中运行。在按下 `You like?` 之后你会看到如下界面：

![Starter product screenshot](http://cdn2.raywenderlich.com/wp-content/uploads/2014/03/starter_project_screenshot.png)

如你在截图中所见的，你需要本地化 4 个元素：

- UI 元素：`Hello` Label
- UI 元素：`You like?` Button
- 销售数据文本：`Yesterday you sold 1000000 apps`
- 图像文本：`I LIKE IT`

花点时间浏览一下文件和目录，使自己熟悉项目结构。 `Main.storyboard` 包含一个单视图屏幕，它是 `ViewController` 类的实例。

## 从代码中分离文本

目前，所有应用显示的文本都以硬编码的方式存放在 `Main.storyboard` 和 `ViewController` 里。为了本地化这些字符串，你就要将它们放在单独的文件中。为了避免将它们硬编码到你的方法里，你只需简单地从该文件引用这些字符串即可。

Xcode 使用以 ".strings" 为扩展名的多个文件来存储和检索所有能用于应用的字符串，以支持多种语言。你的代码中的一个简单方法将会根据当前 iOS 设备的语言去查找并返回所需要的字符串。

那就让我们来试试看。打开菜单 `File > New > File`，选择 Resource 下 `Strings Fils` ，如下所示：

![Choose strings file](http://cdn3.raywenderlich.com/wp-content/uploads/2014/03/Screen-Shot-2014-03-07-at-4.30.35-pm-700x471.png)

点击 `Next` ，将文件命名为 `Localizable.strings` ，然后点击 `Save`。

>注意：`Localizable.strings` 是 iOS 用于本地化文本的默认文件名。请抗拒将其改名的冲动，否则你将在每次引用本地化字符串时都要去输入一遍你的 .strings 文件名。

现在，你已经创建了 `Localizable.strings` 文件，你需要添加所有目前硬编码在应用里的文本。你将按照如下简单的特定规则来做：

```
"KEY" = "CONTENT";
```

这些 键/内容 对（key/content pairs） 就像一个 NSDictionary ，而惯例是使用默认语言的翻译作为内容的键值，例如对于 `You Like?` ，你就这样写：

```
"You like?" = "You like?";
```

键/内容 对 同样可以包含格式化字符串：

```
"Yesterday you sold %@ apps" = "Yesterday you sold %@ apps";
```

现在切换到 `ViewController.m` ，找到 `viewDidLoad` 方法。目前应用如下设置 likeButton 和 salesCountLabel ：

```Objective-C
_salesCountLabel.text = [NSString stringWithFormat:@"Yesterday you sold %@ apps", @(1000000)];
[_likeButton setTitle:@"You like?" forState:UIControlStateNormal];
```

作为替代，你需要从 `Localizable.strings` 文件里读取字符串。那把这两行都用一个叫做 的 `NSLocalizedString` 宏进行修改，如下所示：

```Objective-C
_salesCountLabel.text = [NSString stringWithFormat:NSLocalizedString(@"Yesterday you sold %@ apps", nil), @(1000000)];
[_likeButton setTitle:NSLocalizedString(@"You like?", nil) forState:UIControlStateNormal];
```

所谓宏，即它们包裹一个稍长的代码片段为更易管理的长度，并使用 `#define` 指令创建。

如果你好奇 `NSLocalizedString` 宏在后面做了什么，按着 Control 在 `NSLocalizedString` 上点击，它的定义就会显示出来，如下：

```Objective-C
#define NSLocalizedString(key, comment) 
    [[NSBundle mainBundle] localizedStringForKey:(key) value:@"" table:nil]
```

`NSLocalizedString` 宏根据当前的语言设置使用 `localizedStringForKey` 方法去查找给定键值的字符串。它传递 nil 给 table name，所以它使用默认的 strings 文件名（即 `Localizable.strings`）。对于完整细节，请看 Apple 的 [NSBundle Class Reference](http://developer.apple.com/library/mac/#documentation/Cocoa/Reference/Foundation/Classes/NSBundle_Class/Reference/Reference.html)。

>Note：这个宏接受一个注释作为参数，但似乎没什么用。这是因为除了亲手在 `Localizable.strings` 中键入每个 key/value pair 之外，你还可以使用 iOS SDK 提供的一个叫做 `genstrings` 的工具来自动做到这一点（这对于大型项目相当方便）。  
如果你使用这个方法，你可以在每个字符串上放一个注释，它们会显示在默认字符串的旁边以作为翻译者的辅助。例如，你可以添加一个注释指明字符串被使用在怎样的上下文里。

好，已经有了足够的背景信息——让我们开搞！

编译并运行你的项目，它应该和之前一样在主屏幕上显示同样的文本，但西班牙语在哪里？现在你的应用已经设置好本地化，添加翻译不在话下。

## 添加西班牙语本地化

要支持另外一种语言，点击左边窗格里蓝色的 iLikeIt 项目文件夹，在右边的窗格里选择 `Project`（注意不是 Target），然后在 info 标签下你将看到一个 `Localizations` 分段。点击 `+` 然后选择 `Spanish (es)` 。

![Adding Spanish](http://cdn1.raywenderlich.com/wp-content/uploads/2014/03/localization_steps-700x457.png)

之后出现的屏幕会询问你哪些文件需要做本地化。让它们全部保持选中状态，然后单击 `Finish` 。注意：`Localizable.strings` 没有显示在这个列表里，先不要慌张！

![Select files to localize](http://cdn4.raywenderlich.com/wp-content/uploads/2014/03/select_files_to_localize-700x470.png)

在这个点上，Xcode 已经在后面设置好一些目录，它们包含有不同版本的 `InfoPlist.strings` 和 `Main.storyboard` 以适合你所选择的语言。你可以打开项目文件夹自己看看，大概如下所示：

![New project structure](http://cdn4.raywenderlich.com/wp-content/uploads/2014/03/project_structure.png)

看到 `en.lproj` 和 `es.lproj` 了吗？它们包含有特定语言版本的文件。

`en` 是 English 的本地化代号， `es` 就是 Spanish 的本地化代号。请看一个这个[完整的语言代号列表](http://www.loc.gov/standards/iso639-2/php/English_list.php)，以获取其他语言的代号。

从现在开始，当你的应用想要得到英语版的某个文件，它就会去 `en.lproj` 找，而当它想要西班牙语版的某个文件，它就会去 `es.lproj` 找。

就是这么简单！将你的资源文件放在合适的文件夹里，iOS 就会负责剩下的事情。

但先等一下，`Localizable.strings` 呢？要让 Xcode 知道你想让它本地化，在左窗格里选中这个文件，然后在右边窗格里打开 `File Inspector`。你会看到一个叫做 `Localize` 的按钮，点击它，选择英语（因为它目前就只有英语），最后点击 `Localize`。

![Localize button](http://cdn1.raywenderlich.com/wp-content/uploads/2014/03/Localize_button-700x457.png)

现在 `File Inspector` 窗格会显示这个文件属于哪些语言。目前，如你所见，这个文件只有英文的本地化。点击 `Spanish` 左边的 box 就可以添加西班牙语的本地化了。

![Select spanish button](http://cdn5.raywenderlich.com/wp-content/uploads/2014/03/select_spanish_button.png)

回到左边窗格并点击 `Localizable.strings` 前面的小箭头，就会显示出子元素。你会看到有两个版本的文件：一个是为英语准备的，另一个是为西班牙语准备的：

![Localization files](http://cdn4.raywenderlich.com/wp-content/uploads/2014/03/localization_files.png)

要修改西班牙语的文本，选择 `Localizable.strings (Spanish)` 并用下面显示的文体替换它的内容：

```
"Yesterday you sold %@ apps" = "Ayer le vendi&oacute; %@ aplicaciones";
"You like?" = "~Es bueno?~";
```

恭喜恭喜！你的应用现在支持双语了！

为了测试一切工作正常，更改你的模拟器/设备的显示语言为西班牙语，打开 `设置(Settings)` 应用：通用->多语言环境->语言->Espanol

如果你还在运行 Xcode debugger ，先 `Stop` ，再重新编译并运行应用，你将看到：

![Spanish version](http://cdn3.raywenderlich.com/wp-content/uploads/2014/03/espanol_version-308x500.png)

## 语言环境 vs 语言

1 百万是个相当不错的销售数字；让我们为其添加一点格式，使之看起来更棒！

打开 `ViewController.m` 并将设置 `_salesCountLabel` 的那些行用下列语句替换：

```Objective-C
NSNumberFormatter *numberFormatter = [[NSNumberFormatter alloc] init];
[numberFormatter setNumberStyle:NSNumberFormatterDecimalStyle];
NSString *numberString = [numberFormatter stringFromNumber:@(1000000)];
_salesCountLabel.text = [NSString stringWithFormat:NSLocalizedString(@"Yesterday you sold %@ apps", nil), numberString];
```

编译并运行应用，现在数字看起来更加易读了：

![Number formatted](http://cdn5.raywenderlich.com/wp-content/uploads/2014/03/number_formatted-308x500.png)

这在美国人看了很棒，但在西班牙 1 百万写作 “1.000.000″ 而不是 “1,000,000″ 。在西班牙语下运行应用，你就会看到分隔0的还是逗号。因为在 iOS 中，数字的格式化基于地区/国家，而不是语言，所以为了观察西班牙的某个人会看到怎样的销售数字，打开 `设置（Settings.app）` 通过导航到 `通用General -> 多语言环境International ->区域格式Region Format -> Spanish -> Spain` 来修改语言环境：

![Spanish region format](http://cdn3.raywenderlich.com/wp-content/uploads/2014/03/spanish_region_format-308x500.png)

再次编译并运行应用，你就会看到格式正确的数字：

![Spain number formatting](http://cdn1.raywenderlich.com/wp-content/uploads/2014/03/spain_number_formatting-308x500.png)

额外的说明，`NSNumberFormatter` 自动地以合适的区域格式化你的数字。只要有可能，请抗拒重新发明轮子的冲动，因为在 iOS 上，按着 Apple 的方式做事才有回报。

## 国际化 Storyboard

你的 Storyboard 中的 UI 元素，例如 Label、Button 以及图片可以用代码设置，也可以直接在 Storyboard 里设置。你已经学了如何用编程的方式设置文本以支持多种语言，但屏幕顶部的 "Hello" Label 没有 `IBOutlet` 只能在 `Main.storyboard` 中设置他的文本。

当然你可以添加一个 IBOutlet 将其连接到 `Main.storyboard` 中的 Label 上，然后使用 `NSLocalizedString` 设置它的 text 属性，就像 `likeButton` 和 `salesCountLabel` 那样。但这里还有一个更加简单的方式能本地化 Storyboard 元素，而不需要任何代码。

点击  Main.storyboard 左边的小三角形，你就会看到 `Main.storyboard (Base)` 和 `Main.storyboard (Spanish)` 。点击 `Main.storyboard (Spanish)` 打开 编辑器可以看到本地化文本。你已有了一个 Hello Label 的入口，它看起来如下所示：

```Objective-C
/* Class = "IBUILabel"; text = "Hello"; ObjectID = "pUp-yc-27W"; */
"pUp-yc-27W.text" = "Hello";
```

用西班牙语翻译 "Hola" 替换其中的两个 "Hello" ，如下所示：

```Objective-C
/* Class = "IBUILabel"; text = "Hola"; ObjectID = "pUp-yc-27W"; */
"pUp-yc-27W.text" = "Hola";
```

>Note：永远不要直接修改自动生成的 ObjectID。同样，不要直接复制粘贴上面的代码，因为你的 Label 的 ObjectID 很可能跟上面的不一样。

## 图片的国际化

因为应用使用的一个图像包含有英文字符，所以你还需要本地化它。如果西班牙语应用里有一些英文片段不止是看起来很业余，同时也有损于它的整体可用性和市场潜力。

要本地化图片，首先下载这个西班牙语的图片（在大多数浏览器上都是：右键－>存储图像为…）

![Me Gusta](http://cdn2.raywenderlich.com/wp-content/uploads/2014/03/megusta.png)

打开 `Images.xcassets` 并通过拖动将刚下载的这个 `megusta.png` 到左边的图片列表以添加到资产目录（asset catalog）。资产目录不能被国际化，所以你需要使用一个简单的解决办法来本地化这个图像。

打开 `Localizable.strings (English)` 并添加下面一行：

```
"imageName" = "ilike";
```

类似地，添加下面一行到 `Localizable.strings (Spanish)` ：

```
"imageName" = "megusta";
```

从现在开始，你将使用 `imageName` 作为 key 来检索本地化版本的图像。打开 ViewController.m 并添加如下一行代码到 `viewDidLoad` 方法中：

```Objective-C
[_imageView setImage:[UIImage imageNamed:NSLocalizedString(@"imageName", nil)]];
```

译者注：这样使用资源目录真的好吗？

如果有必要，将 模拟器/设备 切换到西班牙语，然后编译并运行，你就会看到本地化版本的图像显示出来了。

![Spanish image](http://cdn4.raywenderlich.com/wp-content/uploads/2014/03/spanish_image-308x500.png)

恭喜！你已经拥有了能本地化应用到多种不同语言的全部工具。

>Note：这种方法，适用于每个语言都有不同的文件名。一个更好的方法可能是本地化一个资源文件夹，如[此文](http://stackoverflow.com/questions/13921833/how-can-i-localize-a-folder-of-images-for-ios)所描述的。

译者注：果然有更好的方法！

## 无偿奖励

作为最后的奖励，让我们本地化应用的名字吧。你的 ` Info.plist` 有一个特殊的文件（`InfoPlist.strings`），为了适用不同的语言，你可以在它里面以设置一个字符串去覆盖原本的名字设定。要在西班牙语下给应用一个不同的名字，打开 `Supporting Files > InfoPlist.strings (Spanish)` 并插入下面一行：

```
"CFBundleDisplayName" = "Me Gusta";
```

## 练习：国际化音频文件

如果你已经走到这么远，那么你应该对国际化的基本知识比较熟悉了。这是个简单的练习，通过处理这两个不同的音频文件，你可以测试一下自己新掌握的知识。这两个音频文件一个是英语的，另一个是西班牙语的。根据用户选择的语言播放合适的文件即可。

下面是必要步骤的简短描述：

1. 下载音频文件。
2. 先拷贝 `box-en.wav` 到项目中。
3. 打开音频文件的 file inspector 并点击 localize 按钮，确保你选择了英语和西班牙语作为支持的语言。
4. 重命名第二个音频文件（box-es.wav），使其和第一个的名字（box-en.wav）一样，然后将其拷贝到 `es.Iproj` 文件夹中。
5. 确保在 Finder 提示时选择 “替换文件”

## 下一步该怎么走？

这里是[最终的项目](http://cdn2.raywenderlich.com/wp-content/uploads/2014/03/Final-Project.zip)，包含有上面的教程中你所编写的所有代码。

现在你知道了国际化一个 iPhone 应用的基本技术，那就为你的某个旧应用或在设计下一个应用时添加一门外语吧。正如你所看到的，这几乎不花时间，而且能将应用推给更广泛、更多样化的受众，那些不会英语的受众会因此而感激你！

对于具体的翻译，你也许可以使用 Google 在 [http://www.google.com/translate](http://www.google.com/translate) 提供的免费翻译服务，但它的结果可能有错误。如果你能花上一点钱，那有好几个 [列在苹果公司的国际化和本地化页面底部的第三方供应商](https://developer.apple.com/internationalization/) 可以选择。不同的供应商的价格稍有差异，但基本上都少于每个单词 10 美分。

如果你有任何问题，或有关于国际化的建议，请加入下面的讨论！

===============

译者注：欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条微博 [http://weibo.com/2076580237/B1fcVd0rF](http://weibo.com/2076580237/B1fcVd0rF) 以分享给更多人！

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳译者的幸苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/blob/master/images/nixzhu_alipay.png)

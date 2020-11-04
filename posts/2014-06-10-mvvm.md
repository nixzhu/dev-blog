# 介绍 MVVM

本文翻译自：[http://www.objc.io/issue-13/mvvm.html](http://www.objc.io/issue-13/mvvm.html)

原作者：[Ash Furrow](https://twitter.com/ashfurrow)

译者：[@nixzhu](https://twitter.com/nixzhu)

---

我于 2011 年在 500px 找到自己的第一份 iOS 开发工作。虽然我已经在大学里做了好几年 iOS 外包开发，但这才是我的一个真正的 iOS 开发工作。我被招聘为去实现拥有漂亮设计的 iPad 应用，而且是其唯一的 iOS 开发者。在短短七周里，我们就发布了 1.0 并持续迭代，添加了更多特性，但从本质上，代码库也变得更加复杂了。

有时我感觉就像我不知道在做什么。我知道自己的设计模式——就像任何好的编程人员那样——但我太接近我在做的产品以至于不能客观地衡量我的架构决策的有效性。它给我带来了另外一位开发者，我意识到我们陷入困境了。

从没听过 MVC ？有人称之为 Massive View Controller（重量级视图控制器）。这就是我们那时候的感觉。我不打算介绍令人汗颜的细节，但它足以说明，如果我不得不再次重来一次，我绝对会做出不同的决策。

我会修改一个关键架构，并将其带入我从那时起就在开发的各种应用，即使用一种叫做 Model-View-ViewModel 的架构替换 Model-View-Controller。

所以，到底 MVVM 是什么？与其专注于说明 MVVM 的来历，不如让我们看一个典型的 iOS 是如何构建的，并从那里了解  MVVM：

![Typical Model-View-Controller setup](http://www.objc.io/images/issue-13/mvvm1.png)

我们看到的是一个典型的 MVC 设置。Model 呈现数据，View 呈现用户界面，而 View Controller 调节它两者之间的交互。Cool！

稍微考虑一下，虽然 View 和 View Controller 是技术上不同的组件，但它们几乎总是手牵手在一起，成对的。你什么时候看到一个 View 能够与不同 View Controller 配对？或者反过来？所以，为什么不正规化它们的连接呢？

![Intermediate](http://www.objc.io/images/issue-13/intermediate.png)

这更准确地描述了你可能已经编写的 MVC 代码。但它并没有做太多事情来解决 iOS 应用中日益增长的重量级视图控制器。在典型的 MVC 应用里，*许多*逻辑被放在 View Controller 里。它们中的一些确实属于 View Controller，但更多的是所谓的“表示逻辑（presentation logic）”，以 MVVM 属术语来说——就是那些从 Model 转换数据为 View 可以呈现的东西的事情，例如将一个 `NSDate` 转换为一个格式化过的 `NSString`。

我们的图解里缺少某些东西。某些使我们可以放置所有表示逻辑的东西。我们打算将其称为“View Model”——它位于 View/Controller 与 Model 之间：

![Model-View-ViewModel](http://www.objc.io/images/issue-13/mvvm.png)

看起好多了！这个图解准确地描述了什么是 MVVM：一个 MVC 的增强版，我们正式连接了视图和控制器，并将表示逻辑从 Controller 移出放到一个新的对象里，即 View Model。MVVM 听起来很复杂，但它本质上就是一个精心优化的 MVC 架构，而 MVC 你早已熟悉。

现在我们知道了*什么*是 MVVM，但*为什么*某个人会想要去使用它呢？在 iOS 上使用 MVVM 的动机，对我来说，无论如何，就是它能减少 View Controller 的复杂性并使得表示逻辑更易于测试。通过一些例子，我们将看到它如何达到这些目标。

此处有三个重点是我希望你看完本文能带走的：

- MVVM 兼容你当下使用的 MVC 架构。
- MVVM 让你的应用更加可测试。
- MVVM 配合一个绑定机制效果最好。

如我们之前所见，MVVM 基本上就是 MVC 的改进版，所以很容易就能看到它如何被整合到现有使用典型 MVC 架构的应用中。让我们看一个简单的 `Person` Model 以及相应的 View Controller：

```Objective-C
@interface Person : NSObject

- (instancetype)initwithSalutation:(NSString *)salutation firstName:(NSString *)firstName lastName:(NSString *)lastName birthdate:(NSDate *)birthdate;

@property (nonatomic, readonly) NSString *salutation;
@property (nonatomic, readonly) NSString *firstName;
@property (nonatomic, readonly) NSString *lastName;
@property (nonatomic, readonly) NSDate *birthdate;

@end
```

Cool！现在我们假设有了一个 `PersonViewController` ，在 `viewDidLoad` 里，只需要基于它的 `model` 属性设置一些 Label 即可。

```Objective-C
- (void)viewDidLoad {
    [super viewDidLoad];
    
    if (self.model.salutation.length > 0) {
        self.nameLabel.text = [NSString stringWithFormat:@"%@ %@ %@", self.model.salutation, self.model.firstName, self.model.lastName];
    } else {
        self.nameLabel.text = [NSString stringWithFormat:@"%@ %@", self.model.firstName, self.model.lastName];
        }
    
    NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
    [dateFormatter setDateFormat:@"EEEE MMMM d, yyyy"];
    self.birthdateLabel.text = [dateFormatter stringFromDate:model.birthdate];
}
```

这全都直截了当，vanilla MVC。现在来看看我们如何用一个 View Model 来增强它。

```Objective-C
@interface PersonViewModel : NSObject

- (instancetype)initWithPerson:(Person *)person;

@property (nonatomic, readonly) Person *person;

@property (nonatomic, readonly) NSString *nameText;
@property (nonatomic, readonly) NSString *birthdateText;

@end
```

我们的 View Model 的实现大概如下：

```Objective-C
@implementation PersonViewModel

- (instancetype)initWithPerson:(Person *)person {
    self = [super init];
    if (!self) return nil;
    
    _person = person;
    if (person.salutation.length > 0) {
        _nameText = [NSString stringWithFormat:@"%@ %@ %@", self.person.salutation, self.person.firstName, self.person.lastName];
    } else {
        _nameText = [NSString stringWithFormat:@"%@ %@", self.person.firstName, self.person.lastName];
    }
    
    NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
    [dateFormatter setDateFormat:@"EEEE MMMM d, yyyy"];
    _birthdateText = [dateFormatter stringFromDate:person.birthdate];
    
    return self;
}

@end
```

Cool！我们已经将 `viewDidLoad` 中的表示逻辑放入我们的 View Model 里了。此时，我们新的 `viewDidLoad` 就会非常轻量：

```Objective-C
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.nameLabel.text = self.viewModel.nameText;
    self.birthdateLabel.text = self.viewModel.birthdateText;
}
```

所以，如你所见，并没有对我们的 MVC 架构做太多改变。还是同样的代码，只不过移动了位置。它与 MVC 兼容，带来[更轻量的 View Controllers](http://objccn.io/issue-1/)。

可测试，嗯？是怎样？好吧，View Controller 是出了名的难以测试，因为它们做了太多事情。在 MVVM 里，我们试着尽可能多的将代码移入 View Model 里。测试 View Controller 就变得容易多了，因为它们不再做一大堆事情，而 View Model 非常易于测试。让我们来看看：

```Objective-C
SpecBegin(Person)
    NSString *salutation = @"Dr.";
    NSString *firstName = @"first";
    NSString *lastName = @"last";
    NSDate *birthdate = [NSDate dateWithTimeIntervalSince1970:0];

    it (@"should use the salutation available. ", ^{
        Person *person = [[Person alloc] initWithSalutation:salutation firstName:firstName lastName:lastName birthdate:birthdate];
        PersonViewModel *viewModel = [[PersonViewModel alloc] initWithPerson:person];
        expect(viewModel.nameText).to.equal(@"Dr. first last");
        });

    it (@"should not use an unavailable salutation. ", ^{
        Person *person = [[Person alloc] initWithSalutation:nil firstName:firstName lastName:lastName birthdate:birthdate];
        PersonViewModel *viewModel = [[PersonViewModel alloc] initWithPerson:person];
        expect(viewModel.nameText).to.equal(@"first last");
        });

    it (@"should use the correct date format. ", ^{
        Person *person = [[Person alloc] initWithSalutation:nil firstName:firstName lastName:lastName birthdate:birthdate];
        PersonViewModel *viewModel = [[PersonViewModel alloc] initWithPerson:person];
        expect(viewModel.birthdateText).to.equal(@"Thursday January 1, 1970");
        });
SpecEnd
```

如果我们没有将这个逻辑移入 View Model，我们将不得不实例化一个完整的 View Controller 并伴随 View，再比较我们 View 中 Lable 的值。这样做不只是会变成一个麻烦的间接层，而且它同样代表了一个十分脆弱的测试。现在，我们可以按意愿自由地修改视图层级而不必担心破坏我们的单元测试。使用 MVVM 带来的对于测试的好处是清晰，甚至对于这个简单的例子来说也一样，而在有更复杂的表示逻辑的情况下，这个好处会更加明显。

注意到在这个简单的例子中， Model 是不可变的，所以我们可以只在初始化的时候指定我们 View Model 的属性。对于可变 Model，我们还需要使用一些绑定机制，这样 View Model 就能在背后的 Model 改变时更新自身的属性。此外，一旦 View Model 上的 Model 发生改变，那 View 的属性也需要更新。Model 的改变应该级联向下通过 View Model 进入 View。

在 OS X 上，我们可以使用 Cocoa 绑定，但在 iOS 上我们并没有这样好的配置可用。我们想到了 KVO（Key-Value Observation），而且它确实做了很伟大的工作。然而，对于一个简单的绑定都需要很大的样板，更不用说有许多属性需要绑定了。作为替代，我个人喜欢使用 ReactiveCocoa，但 MVVM 并未强制我们使用 ReactiveCocoa。MVVM 是一个伟大的典范，它自身独立，只是在有一个良好的绑定框架时做得更好。

我们覆盖了不少内容：从普通的 MVC 派生出 MVVM，看它们是如何相兼容的范式，从一个可测试的例子观察 MVVM，并看到 MVVM 在有一个配对的绑定机制时工作得更好。如果你有兴趣学习更多关于 MVVM 的知识，你可以看看[这篇博客](http://www.teehanlax.com/blog/model-view-viewmodel-for-ios/)，它用更多细节解释了 MVVM 的好处，或者 [这一篇](http://www.teehanlax.com/blog/krush-ios-architecture/)关于我们如何在最近的项目里使用 MVVM 获得巨大的成功。我同样还有一个经过完整测试，基于 MVVM 的应用，叫做 [C-41](https://github.com/AshFurrow/C-41) ，它是开源的。去看看吧，如果你有任何疑问，请[告诉我](https://twitter.com/ashfurrow)。


---

欢迎转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！

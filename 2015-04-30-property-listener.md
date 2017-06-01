# 用 Swift 实现轻量的属性监听系统

本文的主要目的是解决客户端开发中对“模型的一处修改，UI 要多处更新”的问题。当然，我们要知晓解决方案的细节和思考过程，以及看到其能达到的效果。我们会用到函数式编程的思想，以及伟大的“泛型”。请相信我，我们并非为了使用新技术而使用新技术。如果一个问题有更好的方法去解决，那为何不替换掉旧方法呢？

作者：[@nixzhu](https://twitter.com/nixzhu)

---

假如你正在写的 App 是有用户系统的，也就是用户需要管理自己的信息，如修改名字、头发颜色之类的。

单独拿名字来说，除开在修改界面，可能在系统的其他界面也会使用到它，这就涉及到在更新名字后再更新其他界面的问题。

你的第一直觉是什么呢？多半是使用通知，也就是 NSNotification。这是一种很好的办法，虽然逻辑松散，写起来有些麻烦。比如要定义一个通知名，发送通知，各界面都监听通知再处理，等等。

例如，对于如下 3 个界面，都有显示名字。通过 push，用户可以在第 3 个界面里修改名字，这就需要更新这 3 个界面的名字，不然用户 pop 返回时就会觉得奇怪。

![UI](https://github.com/nixzhu/dev-blog/raw/master/images/property_listener_ui.png)

假如我们的名字放在一个叫做 UserInfo 的类里（访问和修改都使用单例），如下：

```Swift
class UserInfo {

    static let sharedInstance = UserInfo()

    struct Notification {
        static let NameChanged = "UserInfo.Notification.NameChanged"
    }

    var name: String = "NIX" {
        didSet {
            NSNotificationCenter.defaultCenter().postNotificationName(Notification.NameChanged, object: name)
        }
    }
}
```

同时我们定义了一个通知。在 name 被改变后就发出这个通知，并把 name 传出去。

三个界面分别为 FirstViewController、SecondViewController、ThirdViewController，都有一个 button 在正中间。其中前两个负责 push，最后一个点击后可以改名字。因此，对于 FirstViewController 来说：

```Swift
class FirstViewController: UIViewController {

    @IBOutlet weak var nameButton: UIButton!

    override func viewDidLoad() {
        super.viewDidLoad()

        title = "First"

        nameButton.setTitle(UserInfo.sharedInstance.name, forState: .Normal)

        NSNotificationCenter.defaultCenter().addObserver(self, selector: "updateUI:", name: UserInfo.Notification.NameChanged, object: nil)
    }

    func updateUI(notification: NSNotification) {
        if let name = notification.object as? String {
            nameButton.setTitle(name, forState: .Normal)
        }
    }
}
```

除了加载时设置 button 之外，我们还要监听通知，并在 name 被改变时更新 button 的 title。

SecondViewController 的代码类似 FirstViewController，不赘述。

对于 ThirdViewController，除了设置和通知外，还有一个 button 的 target-action 方法用于修改名字，也很简单：

```Swift
@IBAction func changeName(sender: UIButton) {

    let alertController = UIAlertController(title: "Change name", message: nil, preferredStyle: .Alert)

    alertController.addTextFieldWithConfigurationHandler { (textField) -> Void in
        textField.placeholder = self.nameButton.titleLabel?.text
    }

    let action: UIAlertAction = UIAlertAction(title: "OK", style: .Default) { action -> Void in
        if let textField = alertController.textFields?.first as? UITextField {
            UserInfo.sharedInstance.name = textField.text // 更新名字
        }
    }
    alertController.addAction(action)

    self.presentViewController(alertController, animated: true, completion: nil)
}
```

似乎并不麻烦，看起来也算合理，那上面这样写有什么问题？我想答案是太重复。为了减少重复，我们来增加自己的知识，让脑神经稍微痛苦一点，好形成一些新的联结或破坏一些旧的联结。

我们可以传递闭包给 UserInfo，它将闭包存储起来，并在 name 被改变时调用这些闭包，这样闭包里的操作就会被执行了。自然，我们要在闭包里更新 UI。

这样，新的 UserInfo 如下：

```Swift
class UserInfo {

    static let sharedInstance = UserInfo()

    typealias NameListener = String -> Void

    var nameListeners = [NameListener]()

    class func bindNameListener(nameListener: NameListener) {
        self.sharedInstance.nameListeners.append(nameListener)
    }

    class func bindAndFireNameListener(nameListener: NameListener) {
        bindNameListener(nameListener)

        nameListener(self.sharedInstance.name)
    }

    var name: String = "NIX" {
        didSet {
            nameListeners.map { $0(self.name) }
        }
    }
}
```

我们删除了通知相关的代码，定义了 NameListener，增加了一个 nameListeners 用于保存监听者闭包，并实现两个类方法 `bindNameListener` 和 `bindAndFireNameListener` 来保存（并触发）监听者闭包。而在 name 的 didSet 里，我们只需要调用每个闭包即可，这里用了 map，也很直观。

那么 FirstViewController 的代码就简化为：

```Swift
class FirstViewController: UIViewController {

    @IBOutlet weak var nameButton: UIButton!

    override func viewDidLoad() {
        super.viewDidLoad()

        title = "First"

        UserInfo.bindAndFireNameListener { name in
            self.nameButton.setTitle(name, forState: .Normal)
        }
    }
}
```

我们删除了通知相关的代码和 `updateUI` 方法，只需要将我们更新 UI 的闭包绑定到 UserInfo 即可。因为我们也需要初始设置 button，所以用了 `bindAndFireNameListener`。

SecondViewController 和  ThirdViewController 的修改类似 FirstViewController，不赘述。

这样一来，设置 UI 的操作和更新 UI 的操作就被很好地“融合”到一起了。代码比第一版的的逻辑性更强，VC 也更简单。

但是还有一个问题， UserInfo 里的 nameListeners 数组可能会越来越长，比如用户不断地 push/pop。虽然在有限的时间里，nameListeners 的数量不会变的非常大，程序的性能可以接受，但这毕竟是一种浪费（内存和CPU时间）。我们再来解决这个问题。

问题关键是我们的闭包并没有名字，我们无法将其找出并删除。例如对于 SecondViewController 来说，第一次进入它时，bindAndFireNameListener 执行了一次，如果 pop 再 push，它又执行了一次。那么，第一次被绑定的闭包其实没有任何用处了，因为第二次看到的 VC 是新生成的。如果我们能为闭包取名字，我们就能在第二次进入时用新的闭包替换旧的闭包，从而保证 nameListeners 的数量不会无限制的增长，也就不会浪费内存和 CPU 了。

为了限制 nameListeners 的无限制增长，我们可以将 nameListeners 改成 nameListenerSet，类型从 Array 改成 Set，这样绑定时就能保证其中“同一个地方添加的闭包”最多只有一个。但很不幸，我们无法将闭包 NameListener 放入 Set，因为闭包无法实现 Hashable 协议，而这正是使用 Set 所需要的。 

似乎陷入困境了！

不要恐慌。虽然一个单纯的闭包无法实现 Hashable，但我们可以将其再封装一次，例如放入一个 struct 里，我们再让 struct 实现 Hashable 协议。前面刚提到过，闭包无法实现 Hashable，那么我们必然要在 struct 放入另外一个可以 Hashable 的属性来帮助我们的 struct 实现 Hashable。也就是：为闭包取一个名字。因此，我们新的 UserInfo 如下：

```Swift
func ==(lhs: UserInfo.NameListener, rhs: UserInfo.NameListener) -> Bool {
    return lhs.name == rhs.name
}

class UserInfo {

    static let sharedInstance = UserInfo()

    struct NameListener: Hashable {
        let name: String

        typealias Action = String -> Void
        let action: Action

        var hashValue: Int {
            return name.hashValue
        }
    }

    var nameListenerSet = Set<NameListener>()

    class func bindNameListener(name: String, action: NameListener.Action) {
        let nameListener = NameListener(name: name, action: action)

        self.sharedInstance.nameListenerSet.insert(nameListener) // TODO：需要处理同名替换
    }

    class func bindAndFireNameListener(name: String, action: NameListener.Action) {
        bindNameListener(name, action: action)

        action(self.sharedInstance.name)
    }

    var name: String = "NIX" {
        didSet {
            for nameListener in nameListenerSet {
                nameListener.action(name)
            }
        }
    }
}
```

我们设计了一个新的 struct：NameListener，它有一个 name 表明它是谁，原来的闭包就变成了 action，也很合理。为了满足 Hashable 协议，我们用 name.hashValue 来作为 struct 的 hashValue。另外，因为 Hashable 继承于 Equatable，我们也要实现一个 `func ==`。

另外，为了 API 更好使用，我们将 `bindNameListener` 与 `bindAndFireNameListener` 改造为接受一个 name 和一个 action 作为参数，在方法内部才“合成”一个 nameListener，这样 API 在使用时看起来会更合理，如下：

```Swift
UserInfo.bindAndFireNameListener("FirstViewController.nameButton") { name in
    self.nameButton.setTitle(name, forState: .Normal)
}
```

我们只在闭包前面增加了一个闭包的“名字”而已。

最后，UserInfo 的 name 的 didSet 里要稍微修改，因为是 Set，没法 map 了，那就改成最传统的循环吧。

# 小结

我们面临一个“一处修改，多处更新”的问题，起初时我们用通知来实现，并无不可。之后我们想要更合理（或者更酷）一些，于是利用 Swift 的闭包特性实现了一个监听者模式。最后，我们使用包装的办法，解决了监听者可能会无限制增长的问题。

而这一切的目的，都是为了让代码更有逻辑性，并减少 VC 的代码量。

最后的最后，UserInfo 里可能会包含其他类型的属性，例如 `var hairColor: UIColor`，如果它也面临“一处修改，多处更新”的问题，那么我们也需要实现一个 HairColorListener 吗？

也许我们该利用 Swift 的泛型编写一个更加合理的 Listener，你说对吧？

非最终的效果请查看并运行 Demo 代码：[https://github.com/nixzhu/PropertyListenerDemo](https://github.com/nixzhu/PropertyListenerDemo)。如果你愿意的话，可以查看 git 的各个 commit 以得到整个过程。

（最终的）更好的泛型实现在分支 [generic](https://github.com/nixzhu/PropertyListenerDemo/tree/generic) 里，它的关键就是利用泛型实现一个 `class Listenable<T>` 以对应任何类型的属性，它内部再实现监听系统即可。当然，我们也让监听者支持泛型（`struct Listener<T>`）以便执行 action 时可以传递任意类型的参数。还有少许细节不同，例如 UserInfo 里直接使用 static 变量更方便，不需要用一个单独的单例再访问其属性。

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

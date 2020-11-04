# 处理键盘通知

自从 iOS 8 引入了第三方键盘扩展后（或更早），键盘通知就不太正常了。例如，若用户使用中文拼音键盘，弹出时`UIKeyboardWillShowNotification`可能发送不只一次（有可能两次，甚至三次）。

作者：[@nixzhu](https://twitter.com/nixzhu)

---

近日在 iOS 9 beta4 上，我更观察到 `UIKeyboardWillShowNotification` 可能会少发，导致之前根据其发送次数做的算法不能正常工作，结果就是在使用中文拼音键盘时，本该处于键盘上方的输入框会被键盘挡住大部分。

为了解决这个问题，同时也对键盘通知相关的代码做整理并重构（毕竟这些代码分散在 ViewController 里也不好维护，更难以重用），我想写一个单独的库是最好的选择。至于库的名字，“键盘侠”就很不错。虽然在中文里它不算个好词汇，不过英文念着还不错：KeyboardMan。

先说一下之前对键盘通知`UIKeyboardWillShowNotification`发送多次的处理。

在做键盘跟随动画时，我们需要根据键盘的高度来调整某些 View 的位置，或者要更新 UIScrollView（UITableView、UICollectionView）的 `contentOffset` 和 `contentInset`，以使某些内容不被键盘挡住。

既然是键盘跟随动画，那必然要监听`UIKeyboardWillShowNotification`以获取键盘高度以及动画参数（时长和曲线类型）。因为当用户使用某些键盘时，`UIKeyboardWillShowNotification`并不止发送一次，第一次的高度并不是最后完整键盘的高度。如果我们简单地独立对待每一次通知，但由于调整`contentOffset`应该用增量的方式，将导致我们要在处理键盘通知前纪录当前的`contentOffset`并利用它实现增量的效果。很明显，“键盘弹出前的`contentOffset`”需要我们的小心维护，自然，这并不有趣。

然后，上面的方式可能失效，例如当`UIKeyboardWillShowNotification`本该发送两次时却只发送了一次，那我们就不能获取到正确的键盘高度，以此，即不能正确设置`contentOffset`，也会导致键盘上的输入框会被键盘挡住（输入框的位置调整不需要考虑增量的问题，只需要正确的键盘高度）。

虽说`UIKeyboardWillShowNotification`的发送次数不够很可能是 iOS 9 beta4 的 bug，但我们很难保证这样的 bug 不会在之后的正式版中出现。因此，我们还需要更好的办法。

键盘通知除了我们常见的四个：

```swift
let UIKeyboardWillShowNotification: String
let UIKeyboardDidShowNotification: String
let UIKeyboardWillHideNotification: String
let UIKeyboardDidHideNotification: String
```

之外，还有两个 iOS 5 才引入的：

```swift
let UIKeyboardWillChangeFrameNotification: String
let UIKeyboardDidChangeFrameNotification: String
```

经我测试，在`UIKeyboardWillShowNotification`发送次数不正确时，`UIKeyboardWillChangeFrameNotification`和`UIKeyboardDidChangeFrameNotification`都能正确发送。这自然会成为解决问题的关键。

因为我们要做的是键盘跟随动画，因此不考虑`UIKeyboardDidChangeFrameNotification`，因为`Did`表明它“滞后”了。那么`UIKeyboardWillChangeFrameNotification`就成为了我们唯一的希望。

通过监听它，我们可以观察到它会在`UIKeyboardWillShowNotification`之前或者在`UIKeyboardDidHideNotification`之后发出。因为键盘隐藏的通知并没有不正常，所以我们不需要关心其在`UIKeyboardDidHideNotification`的发送。也就是说，我们要把`UIKeyboardWillChangeFrameNotification`当作`UIKeyboardWillShowNotification`来用，以保证获取到正确的键盘高度。但这样以来，键盘出现通知的“次数”就多了，我们还要想办法缩减到正确的次数。

我们先把键盘通知分成两类：Show 和 Hide。因为`UIKeyboardWillChangeFrameNotification`会被当作 Show 来来使用，需要避免它在 Hide 时生效。

于是我们定义一个结构 KeyboardInfo：

```swift
public struct KeyboardInfo {

    public let animationDuration: NSTimeInterval
    public let animationCurve: UInt

    public let frameBegin: CGRect
    public let frameEnd: CGRect
    public var height: CGFloat {
        return frameEnd.height
    }
    public let heightIncrement: CGFloat

    public enum Action {
        case Show
        case Hide
    }
    public let action: Action
    let isSameAction: Bool
}
```

并定义一个变量：

```swift
var keyboardInfo: KeyboardInfo?
```

每次收到键盘通知时，我们就更新此变量，其中`action`能表示当前是 Show 还是 Hide，而`isSameAction`需要计算，表示当前的`action`是否与之前的一样，可用于区别键盘通知类型的转换。

那么我们的通知处理逻辑如下：

```swift
func keyboardWillShow(notification: NSNotification) {

    handleKeyboard(notification, .Show)
}

func keyboardWillChangeFrame(notification: NSNotification) {

    if let keyboardInfo = keyboardInfo {

        if keyboardInfo.action == .Show {
            handleKeyboard(notification, .Show)
        }
    }
}

func keyboardWillHide(notification: NSNotification) {

    handleKeyboard(notification, .Hide)
}

func keyboardDidHide(notification: NSNotification) {

    keyboardInfo = nil
}
```

其中，私有函数`handleKeyboard(_, _)` 将通知里的信息取出生成 KeyboardInfo 并赋值给`keyboardInfo`。

然后注意`keyboardWillChangeFrame`函数，它处理`UIKeyboardWillChangeFrameNotification`。因为此通知会在`UIKeyboardWillShowNotification`之前发送，要将它当作`UIKeyboardWillShowNotification`来用的前提是：

1. `keyboardInfo` 不存在，表示键盘还未弹出过，（因为 `UIKeyboardWillShowNotification` 至少会发送一次，故不处理 `UIKeyboardWillChangeFrameNotification`）
2. `keyboardInfo`已存在，只要保证前一次是 Show 再处理即可。

最后，键盘通知的次数处理，在设置 `keyboardInfo` 时，我们增加一个 willSet

```swift
var keyboardInfo: KeyboardInfo? {
    willSet {
        if let info = newValue {
            if !info.isSameAction || info.heightIncrement != 0 {
                //TODO
            }
        }
    }
}
```

可以看出，我们只会在键盘Action改变时，或键盘高度增量不等于 0 时才进行真正的处理。由此，就可以避免因为将`UIKeyboardWillChangeFrameNotification`当作`UIKeyboardWillShowNotification`用而导致“次数”反而增加了。

不过还有一个新情况：当键盘出现后，若用户按下 Home 进入后台，然后回到本应用，那么 iOS 还会再发送`UIKeyboardWillShowNotification`和`UIKeyboardWillChangeFrameNotification`，而我们并不需要它们。好在这样的情况很好处理，只需在 willSet 的顶部先判断一下应用的状态即可：

```swift
var keyboardInfo: KeyboardInfo? {
    willSet {
        if UIApplication.sharedApplication().applicationState != .Active {
            return
        }

        if let info = newValue {
            if !info.isSameAction || info.heightIncrement != 0 {
                //TODO
            }
        }
    }
}

```

此外，iOS 的键盘在某些设备上还可以拆分（Split）和浮动（Undock），这时系统会发送两个 Hide 通知，若之后再 dismiss 时，系统会发送 `UIKeyboardWillChangeFrameNotification`，不过这个时候就不能将其当做 Show 来处理了。好在上面的`keyboardWillChangeFrame`函数已经避免了这样的情况。

有了这些代码和考量后，我们就可以暴露“闭包”给外部，闭包的执行就放在上面代码 TODO 的位置。

出于方便的考虑，KeyboardMan 共暴露三个闭包：

```swift
public var animateWhenKeyboardAppear: ((appearPostIndex: Int, keyboardHeight: CGFloat, keyboardHeightIncrement: CGFloat) -> Void)? 

public var animateWhenKeyboardDisappear: ((keyboardHeight: CGFloat) -> Void)?

public var postKeyboardInfo: ((keyboardMan: KeyboardMan, keyboardInfo: KeyboardInfo) -> Void)?
```

其中前两个闭包比较方便，放在其中的代码会被自动“动画”，易于使用。第三个将每次刷新的 KeyboardInfo 发送出去，使用的逻辑就交给程序员了。另外，稍微注意一下 animateWhenKeyboardAppear 闭包的`appearPostIndex`参数，它表示“本次”键盘出现时，通知发送到第几次了（每次都从0开始，有可能你的代码里用得到）。如果你用 postKeyboardInfo 闭包那么可用`keyboardMan`参数取到它。

还有一些细节，包括通知监听开启或关闭的实现（注意 deinit 里设置属性并不会触发对应的 willSet 或 didSet），通知内容解析的实现，具体请看 KeyboardMan 的代码。另外，Demo 里有三个闭包的基本用法。

项目地址：[https://github.com/nixzhu/KeyboardMan](https://github.com/nixzhu/KeyboardMan)

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

# 使用状态机的好处


所谓[（有限）状态机](http://zh.wikipedia.org/wiki/%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E6%9C%BA)？就是一个有着不同状态的黑盒子。我们可以看到它的状态，也可以改变它的状态，无论是从内部还是从外部。当然，我们希望能根据它不同的状态来做一些设置或操作。根据时机的需要，我们可能也希望能在它转换状态之前或之后做一些操作。

作者：[@nixzhu](https://twitter.com/nixzhu)

---

下图是一个火箭（你可能会觉得不像），上面有两个按钮，Fire 就是“点火”，Abort 就是“中止”。在点火之后，会倒计时 5 秒，如果在这 5 秒之内没有被中止，那么火箭就会发射。

![Rocket](https://github.com/nixzhu/dev-blog/raw/master/images/state_machine_rocket.png)

这个火箭是一个 RocketView，除了背景图，上面只有两个 Button 和 一个 Label，非常简单。这个 RocketView 被放置在一个 VC 的 view 里，除了它的高宽之外，一并用 AutoLayout 约束其横向居中，以及 bottom 的位置。

初始时，只有 Fire 可以点击，当它被点击后，它自己就不能再被点击了，同时 Abort 变为可以点击。当倒计时结束，火箭真正发射时，要设置这两个按钮都不能点击，因为它们已没有作用了。这一点很容易理解：因为状态不一样了，外观（UI）自然该不一样。

假如我们现在要编写一些逻辑代码，为两个按钮增加 target-action 以便完成一些操作。我们可以在 VC 里访问到这个 RocketView，再拿到它的 Button，然后 addTarget... 就可以了。

大概类似如下：

```Swift
// in VC

let rocketView = RocketView()

rocketView.fireButton.addTarget(self, action: "fire", forControlEvents: .TouchUpInside)


func fire() {
	// 改变 rocketView 的状态（设置按钮等）
	// 再做其它事情
}

```

但这样代码就会很散乱。一种更好的办法是在 RocketView 里就绑定好 target-action，然后用 delegate 或“闭包”来触发外部代码。

```Swift
// in RocketView

var fireAction: (() -> Void)?

fireButton.addTarget(self, action: "fire", forControlEvents: .TouchUpInside)


func fire() {
	// 改变 rocketView 的状态（设置按钮等）
	// 再……
	if let action = fireAction {
		action()
	}
}

// in VC

let rocketView = RocketView()
rocketView.fireAction = {
	// 做其它事情
}

```

这样写的好处是类似“改变 rocketView 的状态”这样的操作就不需要暴露在 VC 中。而且，如果“外部”的 VC 认为不需要在 fire 时“做其它事情”，那它完全可以不去设置 fireAction，非常自然。对于另外一个 abortButton，我们照样写一个内部的 target-action，以及一个 abortAction 给外部就行了。

但这样做仍然不够完美。现在有两个按钮，那就有两个“改变 rocketView 的状态”这样的操作。还因为有倒计时，它也会触发一些操作，并“改变 rocketView 的状态”，代码就更分散了。如果有更多状态，状态转换时的逻辑更加不好理清。

例如对于我们 RocketView 的状态：

![States](https://github.com/nixzhu/dev-blog/raw/master/images/state_machine_states.png)

当按下 fireButton 时，火箭从 Standby 状态进入 CountDown 状态，也就是发射倒计时。当倒计时结束时，火箭就会进入 Launch 状态，也就发射了。若在倒计时结束前按下 abortButton，那么火箭就会停止倒计时，回到 Standby 状态。当然了，火箭上了太空完成任务后，我们可以遥控它着陆，于是它会从 Launch 状态返回 Standby 状态，由此可以重复利用。

我们希望在代码层面做到什么样呢？制造一个集中管理状态的地方，这样分散的几个“改变内部状态”就可以集中起来管理了。同时，我们也应该为 RocketView 暴露出一个闭包，它提供“之前的状态”和“现在的状态”给外部操作，这样外面的 VC 就好利用状态信息做一些操作。比如当火箭进入 Launch 状态时，我们希望 RocketView 向上飞行，那只需用动画改变其 AutoLayout 的 bottom 约束即可。

合理利用 Swift 的 enum、属性的 willSet 和 didSet 以及闭包，我们可以写出这样的代码：

```Swift
// 所有必要的状态
enum State: Printable {
    case Standby
    case CountDown
    case Launch

    var description: String {
        switch self {
        case .Standby:
            return "Standby"
        case .CountDown:
            return "CountDown"
        case .Launch:
            return "Launch"
        }
    }
}

// 前一个状态：根据具体需求，有可能不需要考虑前一个状态，就可以省去
var previousState: State = .Standby

// 当前状态
var currentState: State = .Standby {
    willSet {
        // 先更新“之前”的状态（因为是 will，所以 currentState 还未改变）

        previousState = currentState

        // 再做一些状态改变之前要做的事情（可以 switch newValue 或 previousState 来做不同的操作)
        // ...
    }

    didSet {

        // 执行状态转换后操作

        if let stateTransitionAction = stateTransitionAction {
            stateTransitionAction(previousState: previousState, currentState: currentState)
        }

        // 还可以再做一些“内部”才关心的事情
        // ...

        switch currentState {
        case .Standby:
            fireTimer?.invalidate()

            countDown = 5

            countDownLabel.text = "\(currentState)"

            fireButton.enabled = true
            abortButton.enabled = false

        case .CountDown:
            countDown = 5

            fireTimer = makeNewFireTimer()

            fireButton.enabled = false
            abortButton.enabled = true

        case .Launch:
            fireTimer?.invalidate()

            countDownLabel.text = "\(currentState)"

            fireButton.enabled = false
            abortButton.enabled = false

        default:
            break
        }

    }
}

// 用闭包来将状态转换操作暴露给”外部“，外部可以利用状态的不同执行不同的操作
var stateTransitionAction: ((previousState: State, currentState: State) -> Void)?
```

我们成功的将不同状态时“改变内部状态”的代码都集中在 currentState 的 didSet 里。

而按钮以及定时器的 Action 只需要简单地改变状态即可，看起来格外清爽：

```Swift
func fire() {
   currentState = .CountDown
}

func abort() {
    currentState = .Standby
}

func countDownToFire(timer: NSTimer) {

    countDown--

    if (countDown == 0) {
        currentState = .Launch
    }
}
```

而 VC 里只需要关心 RocketView 的飞行和着陆：

```Swift
rocketView.stateTransitionAction = { (previousState, currentState) in

    println("state from \(previousState) to \(currentState)")

    switch (previousState, currentState) {

    case (.CountDown, .Launch):
        UIView.animateWithDuration(3.0, delay: 0, options: .CurveEaseIn, animations: { () -> Void in
            self.rocketViewBottomConstraint.constant = CGRectGetHeight(self.view.bounds)
            self.view.layoutIfNeeded()

        }, completion: { (finished) -> Void in
            self.landingButton.enabled = true
        })

    default:
        if currentState == .Standby {
            self.landingButton.enabled = false

            UIView.animateWithDuration(1.0, delay: 0, options: .CurveEaseOut, animations: { () -> Void in
                self.rocketViewBottomConstraint.constant = 20
                self.view.layoutIfNeeded()

            }, completion: { (finished) -> Void in
            })
        }
    }
}
```

由此，在 VC 里设置一个按钮做返回遥控器也就一句话的事情：

```Swift
@IBAction func landing(sender: UIBarButtonItem) {
    rocketView.currentState = .Standby
}
```

最后的效果请查看并运行 Demo 代码：[https://github.com/nixzhu/StateMachineDemo](https://github.com/nixzhu/StateMachineDemo) 

# 小结

这是一个放在 View 内部的很简单的状态机（实际上状态机的原理就是这么简单），通过它我们能将状态转换时的各种操作集中管理，并且 VC 会变得更轻量。不同的状态机可能有不同的要求，比如有的可能需要在状态转换之前做一些操作，本例中的状态机也可以扩展。

状态机是一种对可变模型的抽象，实际上几乎没有不变的模型。从状态机的视角，我们可以站在更高层面观察问题。而且很可能你在无意中就已经在使用状态机的视角，只不过没有太明确而已。

虽然很多时候你不会面临很复杂的情况，但懂一些状态机的知识可以让你写出更易读的代码，维护起来更加轻松。而且很酷，对吧？

# 补记

一个迷你的状态机：[Redstone](https://github.com/nixzhu/Redstone)，实现简单，使用方便，可满足绝大部分需求。

# 补记 2

另一个更简单的使用转移函数的实现：[StateMachine-TransitionFunction.swift](https://gist.github.com/nixzhu/3f6dfd063f784269a29320b8d38e1719)

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

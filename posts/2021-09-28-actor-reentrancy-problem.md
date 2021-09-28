# Swift 中 Actor 的重入问题

本文翻译自：[https://swiftsenpai.com/swift/actor-reentrancy-problem/](https://swiftsenpai.com/swift/actor-reentrancy-problem/)

原作者：[Lee Kah Seng](https://swiftsenpai.com/author/kahseng-lee123/)

译者：[@nixzhu](https://twitter.com/nixzhu)

---

当我第一次看到 WWDC 关于 actor 的介绍时，我对它的能力以及它将如何在不久的将来改变我们编写异步代码的方式感到兴奋。通过使用 actor，编写[无数据竞赛](https://swiftsenpai.com/swift/actor-prevent-data-race/)和死锁的异步代码从未如此简单。

但这并不意味着 actor 就不存在线程问题。如果我们不够小心，我们还是可能在使用 actor 时引入**重入问题**（Reentrancy Problem）。

在本文中，我将带领你了解什么是重入问题，为什么它会是个问题，以及如何防止它的发生。如果这是你第一次听说重入问题，一定要继续读下去，这样在你下次使用 actor 时就不会手忙脚乱。

* * *

## 真实的例子

展示重入问题的最好方法是使用一个真实的例子。考虑下面这个 `BankAccount` actor，它有一个 `balance` 变量。

```swift
actor BankAccount {
    
    private var balance = 1000
    
    // ...
    // ...
}
```

我们将在稍后给这个 `BankAccount` actor 增加一个提款功能，但在此之前，让我们给它增加一个私有方法，以检查账户是否有足够的余额用于提款：

```swift
private func canWithdraw(_ amount: Int) -> Bool {
    return amount <= balance
}
```

在此基础上，我们再定义另一个模拟授权过程的私有方法：

```swift
private func authorizeTransaction() async -> Bool {
    
    // 等 1 秒
    try? await Task.sleep(nanoseconds: 1 * 1000000000)
    
    return true
}
```

现实中的授权过程应该相当缓慢，因此我们将其声明为异步方法。我们不打算实现实际的授权工作流程，而是等待 1 秒钟，然后返回 true 来模拟交易已被授权的情况。

有了这些，我们现在可以这样实现提款功能：

```swift
func withdraw(_ amount: Int) async {
    
    guard canWithdraw(amount) else {
        return
    }
    
    guard await authorizeTransaction() else {
        return
    }
    
    balance -= amount
}
```

实现相当简明。我们首先检查账户余额。如果余额足够，我们将继续授权交易。一旦我们成功授权交易，我们将从余额中扣除提款金额，表明钱已经从账户中提取。

之后，让我们添加一些 `print` 语句来帮助我们监控整个提款过程：

```swift
func withdraw(_ amount: Int) async {
    
    print("🤓 检查余额以提取: \(amount)")
    
    guard canWithdraw(amount) else {
        print("🚫 余额不够提取: \(amount)")
        return
    }
    
    guard await authorizeTransaction() else {
        return
    }
    print("✅ 交易已授权: \(amount)")
    
    balance -= amount
    
    print("💰 账户余额: \(balance)")
}
```

下面是 `BankAccount` actor 的完整实现：

```swift
actor BankAccount {
    
    private var balance = 1000
    
    func withdraw(_ amount: Int) async {
        
        print("🤓 检查余额以提取: \(amount)")
        
        guard canWithdraw(amount) else {
            print("🚫 余额不够提取: \(amount)")
            return
        }
        
        guard await authorizeTransaction() else {
            return
        }
        
        print("✅ 交易已授权: \(amount)")
        
        balance -= amount
        
        print("💰 账户余额: \(balance)")
    }
    
    private func canWithdraw(_ amount: Int) -> Bool {
        return amount <= balance
    }
    
    private func authorizeTransaction() async -> Bool {
        
        // 等 1 秒
        try? await Task.sleep(nanoseconds: 1 * 1000000000)
        
        return true
    }
}
```

### 模拟重入问题

现在，让我们考虑 2 次提款同时发生的情况。 我们可以通过在 2 个独立的异步任务中触发  `withdraw(_:)`  方法来模拟。

```swift
let account = BankAccount()

Task {
    await account.withdraw(800)
}

Task {
    await account.withdraw(500)
}
```

你认为结果会是什么？

乍一看，您可能认为第一次提款（800）应该通过，而第二次提款（500）会因余额不足而被拒绝。 很不幸，事实并非如此。 这是我们从 Xcode 控制台得到的结果：

```swift
🤓 检查余额以提取: 800
🤓 检查余额以提取: 500
✅ 交易已授权: 800
💰 账户余额: 200
✅ 交易已授权: 500
💰 账户余额: -300
```

如你所见，这两笔交易都通过了，用户能够提取超过账户余额的资金。如果你是银行的老板，你肯定不希望这样的情况发生。

让我们仔细看看 `withdraw(_:)` 的实现，你会发现，我们遇到的问题实际上是由以下 3 个原因引起的：

1.  在 `withdraw(_:)` 中存在一个暂停点，即 `await authorizeTransaction()`。
2.  在暂停点之前和之后，第二笔交易的 `BankAccount` 的状态（`balance` 的值）是不同的。
3.  `withdraw(_:)` 在其之前的执行完成之前就被再次调用。

由于 `withdraw(_:)` 中的暂停点，第二笔交易的余额检查发生在第一笔交易完成之前。在那段时间里，账户仍然有足够的余额来进行第 2 笔交易，这就是为什么第 2 笔交易的余额检查会通过。

这是一个非常典型的重入问题，而且当它产生时，Swift actor 似乎不会给我们任何编译错误。如果是这样，我们要怎么做才能防止这种情况的发生呢？

* * *

## 为重入设计 actor

根据苹果公司的说法，actor 重入可防止死锁，并保证向前处理。然而，它并不保证 actor 的可变状态在每次 await 前后保持不变。

因此，作为开发者，我们必须要时刻注意，每次 await 都是一个潜在的暂停点，actor 的可变状态在每次 await 后都可能发生变化。换句话说，我们有责任防止重入问题的发生。

如果是这样，有什么预防手段吗？

### 在同步代码中执行状态变更

苹果工程师建议的第一种方法是始终在**同步**代码中修改 actor 状态。正如你在我们的例子中所看到的，当余额扣除发生时，就是我们修改 actor 状态的时机，而当我们检查账户余额时，则是我们读取 actor 状态的时机。这 2 个时机被一个暂停点分开。

![The root cause of the actor reentrancy problem](https://i2.wp.com/swiftsenpai.com/wp-content/uploads/2021/09/actor-reentrancy-suspension.png?resize=1024%2C355&ssl=1)

因此，为了确保余额检查和余额扣除同步运行，我们需要做的就是在执行余额检查之前对交易进行授权。

```swift
func withdraw(_ amount: Int) async {
    
    // 检查余额前执行授权
    guard await authorizeTransaction() else {
        return
    }
    print("✅ 交易已授权: \(amount)")
    
    print("🤓 检查余额以提取: \(amount)")
    guard canWithdraw(amount) else {
        print("🚫 余额不够提取: \(amount)")
        return
    }
    
    balance -= amount
    
    print("💰 账户余额: \(balance)")
}
```

再次运行代码，我们会得到如下输出：

    ✅ 交易已授权: 800
    🤓 检查余额以提取: 800
    💰 账户余额: 200
    ✅ 交易已授权: 500
    🤓 检查余额以提取: 500
    🚫 余额不够提取: 500

很棒！我们的代码现在成功地拒绝了第二笔交易。然而，取款流程并不完全合理。如果账户余额不足，授权交易的意义何在？

如果是这样的话，我们还有什么其他的选择，可以在解决重入问题的同时保持原来的提款流程？

### 在暂停点之后检查 actor 的状态

苹果工程师建议的另一种预防方法是：在暂停点之后对 actor 的状态再进行一次检查。这可以确保我们对 actor 的可变状态所做的任何假设在暂停点前后保持一致。

对于我们的例子，我们假设在授权过程后账户余额是足够的。因此，为了防止重入问题，我们必须在交易被授权后再次检查账户余额。

```swift
func withdraw(_ amount: Int) async {
    
    print("🤓 检查余额以提取: \(amount)")
    guard canWithdraw(amount) else {
        print("🚫 余额不够提取: \(amount)")
        return
    }
    
    guard await authorizeTransaction() else {
        return
    }
    print("✅ 交易已授权: \(amount)")
    
    // Check balance again after the authorization process
    guard canWithdraw(amount) else {
        print("⛔️ 余额不够提取: \(amount) (已授权)")
        return
    }

    balance -= amount
    
    print("💰 账户余额: \(balance)")
    
}
```

下面是我们从上述 `withdraw(_:)` 方法中得到的结果：

    🤓 检查余额以提取: 800
    🤓 检查余额以提取: 500
    ✅ 交易已授权: 800
    💰 账户余额: 200
    ✅ 交易已授权: 500
    ⛔️ 余额不够提取: 500 (已授权)

就这样，我们成功地防止了重入问题的发生，同时保持了原有的提款流程。

正如你所看到的，没有银弹可以防止所有种类的重入问题。我们要根据实际需求来调整我们所采取的方法。

如果你想看一个更复杂的现实生活中的重入问题，我强烈推荐 [Donny Wals](https://twitter.com/DonnyWals) 的[这篇文章](https://github.com/nixzhu/dev-blog/blob/main/posts/2021-09-28-using-swifts-async-await-to-build-an-image-loader.md)。在这篇文章中，你会看到他是如何使用一个字典来防止他的图片下载器 actor 由于重入问题而下载两次图片的。

* * *

## 线程安全 vs 重入

现在你已了解是什么原因导致了重入问题，以及我们如何防止它。现在让我们转换焦点，谈谈线程安全和重入的区别。

我们都知道，一个 actor 会在自己的上下文中保证线程安全，那为什么我们还会出现重入问题呢？

根据苹果公司的说法，一个 actor 将保证对其可变状态的互斥访问。这就是为什么在刚才的例子中，`amount = 800` 的事务总是在第一时间发生。如果一个 actor 不是线程安全的，我们将无法得到这样一个一致的结果。我们有时可能会得到这样的结果：`amount = 500` 的事务首先被触发。

尽管重入问题发生在多线程环境中，但这并不意味着这是一个线程安全问题。重入问题的发生是因为我们假设 actor 的状态不会在暂停点前后发生变化，而不是因为我们试图同时改变 actor 的可变状态。因此，重入问题不等同于线程安全问题。

* * *

## 收尾

在本文中，你已了解到，actor 可以保证线程安全，但它并不能防止重入问题。因此，我们必须时刻注意 actor 的状态可能会在暂停点前后发生变化，我们有责任确保我们的代码在 actor 的状态发生变化后仍能正确运行。

如果你想尝试一下本文中的示例代码，你可以[从这里](https://github.com/LeeKahSeng/SwiftSenpai-Swift-Concurrency)获取。

* * *

你觉得这篇文章有帮助吗？如果你觉得有帮助，请随时查看我其他与 Swift 并发相关的文章。

* [Preventing Data Races Using Actors in Swift](https://swiftsenpai.com/swift/actor-prevent-data-race/)
* [Making Network Requests with Async/await in Swift](https://swiftsenpai.com/swift/async-await-network-requests/)
* [Getting Started with Swift Concurrency](https://swiftsenpai.com/swift/swift-concurrency-get-started/)

更多与iOS开发和Swift相关的文章，请务必在 [Twitter](https://twitter.com/Lee_Kah_Seng) 上关注我，并订阅我的每月通讯。

谢谢你的阅读。 👨🏻💻

* * *

欢迎转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！
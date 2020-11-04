# 防止点击 Cell 时 ViewController 被重复 Push

寻找疑难问题的解决办法，再做合理分析以便确定可使用

作者：[@nixzhu](https://twitter.com/nixzhu)

---

不少 iOS 开发者都遇到过：在 `tableView(:didSelectRowAtIndexPath:)` 里做 push 到其它界面的操作，但某些用户点击一个 cell 后，可能会触发两次 push。这种情况出现的概率较小，而检查代码也很难发现问题，实在让人烦恼！

因为毕竟做了两次 push，那么 `tableView(:didSelectRowAtIndexPath:)` 必然被调用了两次。但又因为这是一个 delegate 方法，是由 iOS 来调用的，所以怎样才算被“选中”实在难以猜想，何况不同的 iOS 版本还可能有差异。不过我们仍然有办法做处理，毕竟它被调用了两次，就算是用计数再判断的办法，我们也能防止同一次点击出现重复的 push。真实的问题是这种情况难以调试，因为问题很不容易出现，你怎么知道修改后的代码是工作的呢？

没有关系，路都是人走出来的。[搜索一下](https://www.google.com/search?client=safari&rls=en&q=ios+didselectrowatindexpath+push+twice&ie=UTF-8&oe=UTF-8)就能看到其他开发者也遇到过此问题，我将常用的解决办法整理如下：

``` swift
func tableView(tableView: UITableView, didSelectRowAtIndexPath indexPath: NSIndexPath) {
    defer {
        tableView.deselectRowAtIndexPath(indexPath, animated: true)
    }

    if let navigationController = navigationController {
        guard navigationController.topViewController == self else {
            return
        }
    }

    // TODO: performSegueWithIdentifier...
}
```

上面代码利用的是 push 后会改变 navigationController 的 viewControllers stack，那么其 topViewController 自然就改变了。所以在 push 之前，我们先保证此时的 topViewController 是当前 ViewController，这样就可避免重复 push。

我在这里还用了 defer 关键字，也就是无论如何，我们都可以取消 cell 的选中。当然，我们还应该判断 navigationController 的存在性，因为当前的 ViewController 不一定内嵌在某个 UINavigationController 里，segue 也并非只有 push(show) 一种。

这样写真的有效吗？好在我能稍微比较容易地重现此问题。

在我自己的实验里，若长按 cell，然后稍微减少手指的压力再立即增加压力，触发此 bug 的可能性就比较大。（这也许和具体的 app UI 逻辑的实现有关，也许你遇到的问题不是这样。另外，如果你没有观察到两次 push 间 viewControllers stack 的改变，那说明你遇到的问题更加诡异，此方法也可能不适用。）

我通过打印的办法确认，在出现重复 push 的情况下，iOS 9.2 会调用两次 `tableView(:didSelectRowAtIndexPath:)`，但在此之间，当第一次 performSegue 时，UINavigationController 的 viewControllers stack 就已经被改变了，也就是说，第二次就不会再 push 了。

既然只要 performSegue 就能改变 UINavigationController 的 viewControllers stack，那我们也不需要在 `tableView(:didSelectRowAtIndexPath:)` 里做处理，直接重载 ViewController 的 `performSegueWithIdentifier(:sender:)` 即可：

``` swift
override func performSegueWithIdentifier(identifier: String, sender: AnyObject?) {
    if let navigationController = navigationController {
        guard navigationController.topViewController == self else {
            return
        }
    }

    super.performSegueWithIdentifier(identifier, sender: sender)
}
```

这样代码逻辑会更好一点。进一步，若你的 app 里有许多 ViewController，也许它们都有可能出现重复 push。那为了避免重复代码，可以写一个基类让其他 ViewController 继承。

注意，你可能会想说，怎么不在 `shouldPerformSegueWithIdentifier(:sender:)` 里做处理呢？不是更合理吗？

这个方法看起来挺像这么回事，但可惜，它只会作用于直接在 IB 里连线的 segue，如果你手动 performSegue，它并不会被调用。这样的逻辑也说得通，毕竟你都手动调用了，哪还有该不该的问题呢？

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

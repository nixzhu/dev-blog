# 让preferredStatusBarStyle真的工作（iOS 10以后）

作者：[@nixzhu](https://twitter.com/nixzhu)

---

当你想将Status Bar上的文字改成白色，先（打开还没被GFW屏蔽的VPN）用Google搜索一下，然后开心地在某个`UIViewController`的子类里写下：

``` swift
class ViewController: UIViewController {

    override var preferredStatusBarStyle: UIStatusBarStyle {
        return .lightContent
    }
}
```

你满心欢喜地编译、运行，却发现上面的代码不工作。

原因也很简单，你的VC很可能是被嵌入一个`UINavigationController`或者`UITabBarController中`。

系统会从`window`的`rootViewController`（通常是某个容器VC）开始确认`preferredStatusBarStyle`。

为了让这个确认可以传递下去，你需要子类化`UINavigationController`和`UITabBarController`，并重载：

``` swift
class NavigationController: UINavigationController {

    override var childViewControllerForStatusBarStyle: UIViewController? {
        return visibleViewController
    }
}

class TabBarController: UITabBarController {

    override var childViewControllerForStatusBarStyle: UIViewController? {
        return selectedViewController
    }
}
```

这样系统会依据返回的VC的`preferredStatusBarStyle`来决定Status Bar的Style。当然，你要先使用子类化的`UINavigationController`和`UITabBarController`来做此VC的容器。

通常，这样就不会有问题了。不过，总是有一个例外。

当你present一个VC的时候，被present的VC的`preferredStatusBarStyle`不会工作（尽管它会是`visibleViewController`）。你必须在present前设置：

``` swift
    vc.modalPresentationCapturesStatusBarAppearance = true
```

这样“确认工作”才会继续传递下去。

---

参考：[UIViewController.preferredStatusBarStyle](https://developer.apple.com/documentation/uikit/uiviewcontroller/1621416-preferredstatusbarstyle)

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

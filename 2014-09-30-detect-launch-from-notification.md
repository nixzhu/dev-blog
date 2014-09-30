# 检测是否通过点击通知来启动App

作者：[@nixzhu](https://twitter.com/nixzhu)

=================================

最近做的一个需求：App 会收到不同类型的远程推送通知，在某些推送到达时，若用户是通过点击通知（从通知栏、通知中心或锁屏通知）进入应用，那需要做一些额外的操作（例如执行一些后台请求、显示某个界面等）。当然，若是从 SpringBoard 启动应用，就不做这些额外的操作。

在[这篇文章](http://www.abdus.me/ios-programming-tips/handle-push-notifications-when-arrived-ios/)里，作者比较详细的说明了一般方法。我现在简要介绍如下：

首先是应用状态：

* 关闭状态（应用被用户手动杀掉、被系统杀掉或者手机刚刚重启）
* 运行状态（应用位于前台，当前用户很可能在使用它）
* 暂停状态（用户按下 Home 键，锁屏键，或者通过多任务切换到其他应用，那我们的应用就暂停了）

对于关闭状态，我们能很好地确定是通过点击通知而不是从SpringBoard点击应用图标来进入应用的。只需要检查 appDelegate 的`didFinishLaunchingWithOptions:` 方法的参数 launchOptions 是否存在即可。当然，`willFinishLaunchingWithOptions:` 也可以。

更进一步，我们可以通过 `[launchOptions objectForKey:UIApplicationLaunchOptionsRemoteNotificationKey]` 来取出这个通知，以便再对通知的类型等做一些判断，以便后续操作。

但上面这篇文章的介绍并不全面。

因为，若应用处于暂停状态，无论是从点击通知还是从 SpringBoard 进入应用，`willFinishLaunchingWithOptions:` 和 `didFinishLaunchingWithOptions:` 都是不会执行的。也就是说，上面的方式无效了。

但天无绝人之路，Apple 早已考虑好这种情况，你可以在你所使用的接收通知的 appDelegate 方法，例如 `didReceiveRemoteNotification:` 里，来判断通知类型，进而做出其他操作。

需要注意的是，这些方法在运行状态和暂停状态都会执行（不同的是是否会听到通知声以及看到通知）。所以该如何区别呢？

答案是，在你做特别的操作之前，先判断 `application.applicationState == UIApplicationStateInactive` ，这样就能保证，当应用是在暂停状态下点击通知进入的，就会执行你的特别操作，而从 SpringBoard 进入时就不会。当然，应用在运行状态时也不会（不然当用户正在使用时，而突然进入某个特别的界面不是很奇怪吗？）。

这也是我新学的一招，因为以前做应用很少处理通知。特别感谢网易“老汉”的指导，他还特别指明：

>在实际的代码中，需要注意当应用在前台时，若用户下拉通知中心（或上拉控制中心），app 也是处于 inactive 状态的。要和真正的 background -> inactive -> active 区分，不然就可能会判断不准确。


=====================

作者注：欢迎非商业转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！

如果你认为这篇原创文章不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳作者的辛苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

版权声明：自由转载-非商用-非衍生-保持署名 | [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)

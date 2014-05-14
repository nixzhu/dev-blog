# GCD 深入理解：第二部分

本文翻译自 [http://www.raywenderlich.com/63338/grand-central-dispatch-in-depth-part-2](http://www.raywenderlich.com/63338/grand-central-dispatch-in-depth-part-2)

原作者：[Derek Selander](http://www.raywenderlich.com/u/Lolgrep)

译者：[Riven](http://weibo.com/riven0951)、[@nixzhu](https://twitter.com/nixzhu)

前半部分由 Riven 翻译，但他由于太忙而搁置，后由 NIX 整理校对并翻译后半部分。

==============================

欢迎来到GCD深入理解系列教程的第二部分（也是最后一部分）。

在本系列的[第一部分](4)中，你已经学到超过你想像的关于并发、线程以及GCD 如何工作的知识。通过在初始化时利用 `dispatch_once`，你创建了一个线程安全的 `PhotoManager` 单例，而且你通过使用 `dispatch_barrier_async` 和 `dispatch_sync` 的组合使得对 `Photos` 数组的读取和写入都变得线程安全了。

除了上面这些，你还通过利用 `dispatch_after` 来延迟显示提示信息，以及利用 `dispatch_async` 将 CPU 密集型任务从 ViewController 的初始化过程中剥离出来异步执行，达到了增强应用的用户体验的目的。

如果你一直跟着第一部分的教程在写代码，那你可以继续你的工程。但如果你没有完成第一部分的工作，或者不想重用你的工程，你可以[下载第一部分最终的代码](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/GooglyPuff_End_1.zip)

那就让我们来更深入地探索 GCD 吧！

## 纠正过早弹出的提示

你可能已经注意到当你尝试用 Le Internet 选项来添加图片时，一个 `UIAlertView` 会在图片下载完成之前就弹出，如下如所示：

![Premature Completion Block][6]

问题的症结在 PhotoManagers 的 `downloadPhotoWithCompletionBlock:` 里，它目前的实现如下：

    - (void)downloadPhotosWithCompletionBlock:(BatchPhotoDownloadingCompletionBlock)completionBlock
    {
        __block [NSError][7] *error;
    &nbsp;
        for (NSInteger i = 0; i &lt; 3; i++) {
            [NSURL][8] *url;
            switch (i) {
                case 0:
                    url = [[NSURL][8] URLWithString:kOverlyAttachedGirlfriendURLString];
                    break;
                case 1:
                    url = [[NSURL][8] URLWithString:kSuccessKidURLString];
                    break;
                case 2:
                    url = [[NSURL][8] URLWithString:kLotsOfFacesURLString];
                    break;
                default:
                    break;
            }
    &nbsp;
            Photo *photo = [[Photo alloc] initwithURL:url
                                  withCompletionBlock:^(UIImage *image, [NSError][7] *_error) {
                                      if (_error) {
                                          error = _error;
                                      }
                                  }];
    &nbsp;
            [[PhotoManager sharedManager] addPhoto:photo];
        }
    &nbsp;
        if (completionBlock) {
            completionBlock(error);
        }
    }

在方法的最后你调用了 `completionBlock` ——因为此时你假设所有的照片都已下载完成。但很不幸，此时并不能保证所有的下载都已完成。

`Photo` 类的实例方法用某个 URL 开始下载某个文件并立即返回，但此时下载并未完成。换句话说，当 `downloadPhotoWithCompletionBlock:` 在其末尾调用 `completionBlock` 时，它就假设了它自己所使用的方法全都是同步的，而且每个方法都完成了它们的工作。

然而，`-[Photo initWithURL:withCompletionBlock:]` 是异步执行的，会立即返回——所以这种方式行不通。

因此，只有在所有的图像下载任务都调用了它们自己的 Completion Block 之后，`downloadPhotoWithCompletionBlock:` 才能调用它自己的 `completionBlock` 。问题是：你该如何监控并发的异步事件？你不知道它们何时完成，而且它们完成的顺序完全是不确定的。

或许你可以写一些比较 Hacky 的代码，用多个布尔值来记录每个下载的完成情况，但这样做就缺失了扩展性，而且说实话，代码会很难看。

幸运的是， 解决这种对多个异步任务的完成进行监控的问题，恰好就是设计 dispatch_group 的目的。

### Dispatch Groups（调度组）

Dispatch Group 会在整个组的任务都完成时通知你。这些任务可以是同步的，也可以是异步的，即便在不同的队列也行。而且在整个组的任务都完成时，Dispatch Group 可以用同步的或者异步的方式通知你。因为要监控的任务在不同队列，那就用一个 `dispatch_group_t` 的实例来记下这些不同的任务。

当组中所有的事件都完成时，GCD 的 API 提供了两种通知方式。

第一种是 `dispatch_group_wait` ，它会阻塞当前线程，直到组里面所有的任务都完成或者等到某个超时发生。这恰好是你目前所需要的。

打开 PhotoManager.m，用下列实现替换 `downloadPhotosWithCompletionBlock:`：

```Objective-C
- (void)downloadPhotosWithCompletionBlock:(BatchPhotoDownloadingCompletionBlock)completionBlock
{
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{ // 1
 
        __block NSError *error;
        dispatch_group_t downloadGroup = dispatch_group_create(); // 2
 
        for (NSInteger i = 0; i < 3; i++) {
            NSURL *url;
            switch (i) {
                case 0:
                    url = [NSURL URLWithString:kOverlyAttachedGirlfriendURLString];
                    break;
                case 1:
                    url = [NSURL URLWithString:kSuccessKidURLString];
                    break;
                case 2:
                    url = [NSURL URLWithString:kLotsOfFacesURLString];
                    break;
                default:
                    break;
            }
 
            dispatch_group_enter(downloadGroup); // 3
            Photo *photo = [[Photo alloc] initwithURL:url
                                  withCompletionBlock:^(UIImage *image, NSError *_error) {
                                      if (_error) {
                                          error = _error;
                                      }
                                      dispatch_group_leave(downloadGroup); // 4
                                  }];
 
            [[PhotoManager sharedManager] addPhoto:photo];
        }
        dispatch_group_wait(downloadGroup, DISPATCH_TIME_FOREVER); // 5
        dispatch_async(dispatch_get_main_queue(), ^{ // 6
            if (completionBlock) { // 7
                completionBlock(error);
            }
        });
    });
}
```

按照注释的顺序，你会看到：

1. 因为你在使用的是同步的 `dispatch_group_wait` ，它会阻塞当前线程，所以你要用 `dispatch_async` 将整个方法放入后台队列以避免阻塞主线程。
2. 创建一个新的 Dispatch Group，它的作用就像一个用于未完成任务的计数器。
3. `dispatch_group_enter` 手动通知 Dispatch Group 任务已经开始。你必须保证 `dispatch_group_enter` 和 `dispatch_group_leave` 成对出现，否则你可能会遇到诡异的崩溃问题。
4. 手动通知 Group 它的工作已经完成。再次说明，你必须要确保进入 Group 的次数和离开 Group 的次数相等。
5. `dispatch_group_wait` 会一直等待，直到任务全部完成或者超时。如果在所有任务完成前超时了，该函数会返回一个非零值。你可以对此返回值做条件判断以确定是否超出等待周期；然而，你在这里用 `DISPATCH_TIME_FOREVER` 让它永远等待。它的意思，勿庸置疑就是，永－远－等－待！这样很好，因为图片的创建工作总是会完成的。
6. 此时此刻，你已经确保了，要么所有的图片任务都已完成，要么发生了超时。然后，你在主线程上运行 `completionBlock` 回调。这会将工作放到主线程上，并在稍后执行。
7. 最后，检查 `completionBlock` 是否为 nil，如果不是，那就运行它。

编译并运行你的应用，尝试下载多个图片，观察你的应用是在何时运行 completionBlock 的。

>注意：如果你是在真机上运行应用，而且网络活动发生得太快以致难以观察 completionBlock 被调用的时刻，那么你可以在 Settings 应用里的开发者相关部分里打开一些网络设置，以确保代码按照我们所期望的那样工作。只需去往 Network Link Conditioner 区，开启它，再选择一个 Profile，“Very Bad Network” 就不错。

如果你是在模拟器里运行应用，你可以使用 [来自 GitHub 的 Network Link Conditioner][9] 来改变网络速度。它会成为你工具箱中的一个好工具，因为它强制你研究你的应用在连接速度并非最佳的情况下会变成什么样。

目前为止的解决方案还不错，但是总体来说，如果可能，最好还是要避免阻塞线程。你的下一个任务是重写一些方法，以便当所有下载任务完成时能异步通知你。

在我们转向另外一种使用 Dispatch Group 的方式之前，先看一个简要的概述，关于何时以及怎样使用有着不同的队列类型的 Dispatch Group ：

* 自定义串行队列：它很适合当一组任务完成时发出通知。
* 主队列（串行）：它也很适合这样的情况。但如果你要同步地等待所有工作地完成，那你就不应该使用它，因为你不能阻塞主线程。然而，异步模型是一个很有吸引力的能用于在几个较长任务（例如网络调用）完成后更新 UI 的方式。
* 并发队列：它也很适合 Dispatch Group 和完成时通知。

### Dispatch Group，第二种方式

上面的一切都很好，但在另一个队列上异步调度然后使用 dispatch_group_wait 来阻塞实在显得有些笨拙。是的，还有另一种方式……

在 PhotoManager.m 中找到 `downloadPhotosWithCompletionBlock:` 方法，用下面的实现替换它：


```Objective-C
- (void)downloadPhotosWithCompletionBlock:(BatchPhotoDownloadingCompletionBlock)completionBlock
{
    // 1
    __block NSError *error;
    dispatch_group_t downloadGroup = dispatch_group_create(); 
 
    for (NSInteger i = 0; i < 3; i++) {
        NSURL *url;
        switch (i) {
            case 0:
                url = [NSURL URLWithString:kOverlyAttachedGirlfriendURLString];
                break;
            case 1:
                url = [NSURL URLWithString:kSuccessKidURLString];
                break;
            case 2:
                url = [NSURL URLWithString:kLotsOfFacesURLString];
                break;
            default:
                break;
        }
 
        dispatch_group_enter(downloadGroup); // 2
        Photo *photo = [[Photo alloc] initwithURL:url
                              withCompletionBlock:^(UIImage *image, NSError *_error) {
                                  if (_error) {
                                      error = _error;
                                  }
                                  dispatch_group_leave(downloadGroup); // 3
                              }];
 
        [[PhotoManager sharedManager] addPhoto:photo];
    }
 
    dispatch_group_notify(downloadGroup, dispatch_get_main_queue(), ^{ // 4
        if (completionBlock) {
            completionBlock(error);
        }
    });
}
```

下面解释新的异步方法如何工作：

1. 在新的实现里，因为你没有阻塞主线程，所以你并不需要将方法包裹在 `async` 调用中。
2. 同样的 `enter` 方法，没做任何修改。
3. 同样的 `leave` 方法，也没做任何修改。
4. `dispatch_group_notify` 以异步的方式工作。当 Dispatch Group 中没有任何任务时，它就会执行其代码，那么 `completionBlock` 便会运行。你还指定了运行 `completionBlock` 的队列，此处，主队列就是你所需要的。

对于这个特定的工作，上面的处理明显更清晰，而且也不会阻塞任何线程。


## The Perils of Too Much Concurrency 太多并发带来的危险


With all of these new tools at your disposal, you should probably thread everything, right!?

既然你的工具箱里有了这些新工具，你大概做任何事情都想使用它们，对吧？

![Thread_All_The_Code_Meme][10]

看看 PhotoManager 中的 `downloadPhotosWithCompletionBlock` 方法。你可能已经注意到这里的 `for` 循环，它迭代三次，下载三个不同的图片。你的任务是尝试让 `for` 循环并发运行，以提高其速度。

`dispatch_apply` 刚好可用于这个任务。

`dispatch_apply` 表现得就像一个 `for` 循环，但它能并发地执行不同的迭代。这个函数是同步的，所以和普通的 `for` 循环一样，它只会在所有工作都完成后才会返回。

当在 Block 内计算任何给定数量的工作的最佳迭代数量时，必须要小心，因为过多的迭代和每个迭代只有少量的工作会导致大量开销以致它能抵消任何因并发带来的收益。而被称为`跨越式（striding）`的技术可以在此帮到你，即通过在每个迭代里多做几个不同的工作。

>译者注：大概就能减少并发数量吧，作者是提醒大家注意并发的开销，记在心里！

那何时才适合用 `dispatch_apply` 呢？

* 自定义串行队列：串行队列会完全抵消 `dispatch_apply` 的功能；你还不如直接使用普通的 `for` 循环。
* 主队列（串行）：与上面一样，在串行队列上不适合使用 `dispatch_apply` 。还是用普通的 `for` 循环吧。
* 并发队列：对于并发循环来说是很好选择，特别是当你需要追踪任务的进度时。

回到 `downloadPhotosWithCompletionBlock:` 并用下列实现替换它：


```Objective-C
- (void)downloadPhotosWithCompletionBlock:(BatchPhotoDownloadingCompletionBlock)completionBlock
{
    __block NSError *error;
    dispatch_group_t downloadGroup = dispatch_group_create();
 
    dispatch_apply(3, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^(size_t i) {
 
        NSURL *url;
        switch (i) {
            case 0:
                url = [NSURL URLWithString:kOverlyAttachedGirlfriendURLString];
                break;
            case 1:
                url = [NSURL URLWithString:kSuccessKidURLString];
                break;
            case 2:
                url = [NSURL URLWithString:kLotsOfFacesURLString];
                break;
            default:
                break;
        }
 
        dispatch_group_enter(downloadGroup);
        Photo *photo = [[Photo alloc] initwithURL:url
                              withCompletionBlock:^(UIImage *image, NSError *_error) {
                                  if (_error) {
                                      error = _error;
                                  }
                                  dispatch_group_leave(downloadGroup);
                              }];
 
        [[PhotoManager sharedManager] addPhoto:photo];
    });
 
    dispatch_group_notify(downloadGroup, dispatch_get_main_queue(), ^{
        if (completionBlock) {
            completionBlock(error);
        }
    });
}
```

你的循环现在是并行运行的了；在上面的代码中，在调用 `dispatch_apply` 时，你用第一次参数指明了迭代的次数，用第二个参数指定了任务运行的队列，而第三个参数是一个 Block。

要知道虽然你有代码保证添加相片时线程安全，但图片的顺序却可能不同，这取决于线程完成的顺序。

编译并运行，然后从 “Le Internet” 添加一些照片。注意到区别了吗？

在真机上运行新代码会稍微更快的得到结果。但我们所做的这些提速工作真的值得吗？

实际上，在这个例子里并不值得。下面是原因：

* 你创建并行运行线程而付出的开销，很可能比直接使用  `for` 循环要多。若你要以合适的步长迭代非常大的集合，那才应该考虑使用 `dispatch_apply`。
* 你用于创建应用的时间是有限的——除非实在太糟糕否则不要浪费时间去提前优化代码。如果你要优化什么，那去优化那些明显值得你付出时间的部分。你可以通过在 Instruments 里分析你的应用，找出最长运行时间的方法。看看 [如何在 Xcode 中使用 Instruments][11] 可以学到更多相关知识。
* 通常情况下，优化代码会让你的代码更加复杂，不利于你自己和其他开发者阅读。请确保添加的复杂性能换来足够多的好处。

记住，不要在优化上太疯狂。你只会让你自己和后来者更难以读懂你的代码。

## GCD 的其他趣味

等一下！还有更多！有一些额外的函数在不同的道路上走得更远。虽然你不会太频繁地使用这些工具，但在对的情况下，它们可以提供极大的帮助。

### 阻塞——正确的方式

这可能听起来像是个疯狂的想法，但你知道 Xcode 已有了测试功能吗？:] 我知道，虽然有时候我喜欢假装它不存在，但在代码里构建复杂关系时编写和运行测试非常重要。

Xcode 里的测试在 `XCTestCase` 的子类上执行，并运行任何方法签名以 `test` 开头的方法。测试在主线程运行，所以你可以假设所有测试都是串行发生的。

当一个给定的测试方法运行完成，XCTest 方法将考虑此测试已结束，并进入下一个测试。这意味着任何来自前一个测试的异步代码会在下一个测试运行时继续运行。

网络代码通常是异步的，因此你不能在执行网络获取时阻塞主线程。也就是说，整个测试会在测试方法完成之后结束，这会让对网络代码的测试变得很困难。也就是，除非你在测试方法内部阻塞主线程直到网络代码完成。

>注意：有一些人会说，这种类型的测试不属于集成测试的首选集（Preferred Set）。一些人会赞同，一些人不会。但如果你想做，那就去做。

![Gandalf_Semaphore][12]

导航到 GooglyPuffTests.m 并查看 `downloadImageURLWithString:`，如下：

```Objective-C
- (void)downloadImageURLWithString:(NSString *)URLString
{
    NSURL *url = [NSURL URLWithString:URLString];
    __block BOOL isFinishedDownloading = NO;
    __unused Photo *photo = [[Photo alloc]
                             initwithURL:url
                             withCompletionBlock:^(UIImage *image, NSError *error) {
                                 if (error) {
                                     XCTFail(@"%@ failed. %@", URLString, error);
                                 }
                                 isFinishedDownloading = YES;
                             }];
 
    while (!isFinishedDownloading) {}
}
```

这是一种测试异步网络代码的幼稚方式。 While 循环在函数的最后一直等待，直到 `isFinishedDownloading` 布尔值变成 True，它只会在 Completion Block 里发生。让我们看看这样做有什么影响。

通过在 Xcode 中点击  Product / Test 运行你的测试，如果你使用默认的键绑定，也可以使用快捷键 ⌘+U 来运行你的测试。

在测试运行时，注意 Xcode debug 导航栏里的 CPU 使用率。这个设计不当的实现就是一个基本的 [自旋锁][14] 。它很不实用，因为你在 While 循环里浪费了珍贵的 CPU 周期；而且它也几乎没有扩展性。

>译者注：所谓自旋锁，就是某个线程一直抢占着 CPU 不断检查以等到它需要的情况出现。因为现代操作系统都是可以并发运行多个线程的，所以它所等待的那个线程也有机会被调度执行，这样它所需要的情况早晚会出现。

你可能需要使用前面提到的 Network Link Conditioner ，已便清楚地看到这个问题。如果你的网络太快，那么自旋只会在很短的时间里发生，难以观察。

>译者注：作者反复提到网速太快，而我们还需要对付 GFW，简直泪流满面！

你需要一个更优雅、可扩展的解决方案来阻塞线程直到资源可用。欢迎来到信号量。

### Semaphores 信号量

信号量是一种老式的线程概念，由非常谦卑的 Edsger W. Dijkstra 介绍给世界。信号量之所以比较复杂是因为它建立在操作系统的复杂性之上。

如果你想学到更多关于信号量的知识，看看这个链接[它更细致地讨论了信号量理论][15]。如果你是学术型，那可以看一个软件开发中经典的[哲学家进餐问题][15]，它需要使用信号量来解决。

信号量让你控制多个消费者对有限数量资源的访问。举例来说，如果你创建了一个有着两个资源的信号量，那同时最多只能有两个线程可以访问临界区。其他想使用资源的线程必须在一个…你猜到了吗？…FIFO队列里等待。

让我们来使用信号量吧！

打开 GooglyPuffTests.m 并用下列实现替换 `downloadImageURLWithString:`：


```Objective-C
- (void)downloadImageURLWithString:(NSString *)URLString
{
    // 1
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
 
    NSURL *url = [NSURL URLWithString:URLString];
    __unused Photo *photo = [[Photo alloc]
                             initwithURL:url
                             withCompletionBlock:^(UIImage *image, NSError *error) {
                                 if (error) {
                                     XCTFail(@"%@ failed. %@", URLString, error);
                                 }
 
                                 // 2
                                 dispatch_semaphore_signal(semaphore);
                             }];
 
    // 3
    dispatch_time_t timeoutTime = dispatch_time(DISPATCH_TIME_NOW, kDefaultTimeoutLengthInNanoSeconds);
    if (dispatch_semaphore_wait(semaphore, timeoutTime)) {
        XCTFail(@"%@ timed out", URLString);
    }
}
```

下面来说明你代码中的信号量是如何工作的：

1. 创建一个信号量。参数指定信号量的起始值。这个数字是你可以访问的信号量，不需要有人先去增加它的数量。（注意到增加信号量也被叫做发射信号量）。译者注：这里初始化为0，也就是说，有人想使用信号量必然会被阻塞，直到有人增加信号量。
2. 在 Completion Block 里你告诉信号量你不再需要资源了。这就会增加信号量的计数并告知其他想使用此资源的线程。
3. 这会在超时之前等待信号量。这个调用阻塞了当前线程直到信号量被发射。这个函数的一个非零返回值表示到达超时了。在这个例子里，测试将会失败因为它以为网络请求不会超过 10 秒钟就会返回——一个平衡点！

再次运行测试。只要你有一个正常工作的网络连接，这个测试就会马上成功。请特别注意 CPU 的使用率，与之前使用自旋锁的实现作个对比。

关闭你的网络链接再运行测试；如果你在真机上运行，就打开飞行模式。如果你的在模拟器里运行，你可以直接断开 Mac 的网络链接。测试会在 10 秒后失败。这很棒，它真的能按照预想的那样工作！

还有一些琐碎的测试，但如果你与一个服务器组协同工作，那么这些基本的测试能够防止其他人就最新的网络问题对你说三道四。

### 使用 Dispatch Source

GCD 的一个特别有趣的特性是 Dispatch Source，它基本上就是一个低级函数的 grab-bag ，能帮助你去响应或监测 Unix 信号、文件描述符、Mach 端口、VFS 节点，以及其它晦涩的东西。所有这些都超出了本教程讨论的范围，但你可以通过实现一个 Dispatch Source 对象并以一个相当奇特的方式来使用它来品尝那些晦涩的东西。

第一次使用 Dispatch Source 可能会迷失在如何使用一个源，所以你需要知晓的第一件事是 `dispatch_source_create` 如何工作。下面是创建一个源的函数原型：

```Objective-C
dispatch_source_t dispatch_source_create(
   dispatch_source_type_t type,
   uintptr_t handle,
   unsigned long mask,
   dispatch_queue_t queue);
```

第一个参数是 `dispatch_source_type_t` 。这是最重要的参数，因为它决定了 handle 和 mask 参数将会是什么。你可以查看 [Xcode 文档][17] 得到哪些选项可用于每个 `dispatch_source_type_t` 参数。

下面你将监控 `DISPATCH_SOURCE_TYPE_SIGNAL` 。如[文档所显示的][18]：

一个监控当前进程信号的 Dispatch Source。 handle 是信号编号，mask 未使用（传 0 即可）。

这些 Unix 信号组成的列表可在头文件  [signal.h][19] 中找到。在其顶部有一堆 `#define` 语句。你将监控此信号列表中的 `SIGSTOP` 信号。这个信号将会在进程接收到一个无法回避的暂停指令时被发出。在你用 LLDB 调试器调试应用时你使用的也是这个信号。

去往 PhotoCollectionViewController.m 并添加如下代码到 `viewDidLoad` 的顶部，就在 `[super viewDidLoad]` 下面：


```Objective-C
- (void)viewDidLoad
{
  [super viewDidLoad];
 
  // 1
  #if DEBUG
      // 2
      dispatch_queue_t queue = dispatch_get_main_queue();
 
      // 3
      static dispatch_source_t source = nil;
 
      // 4
      __typeof(self) __weak weakSelf = self;
 
      // 5
      static dispatch_once_t onceToken;
      dispatch_once(&onceToken, ^{
          // 6
          source = dispatch_source_create(DISPATCH_SOURCE_TYPE_SIGNAL, SIGSTOP, 0, queue);
 
          // 7
          if (source)
          {
              // 8
              dispatch_source_set_event_handler(source, ^{
                  // 9
                  NSLog(@"Hi, I am: %@", weakSelf);
              });
              dispatch_resume(source); // 10
          }
      });
  #endif
 
  // The other stuff
}
```

这些代码有点儿复杂，所以跟着注释一步步走，看看到底发生了什么：

1. 最好是在 DEBUG 模式下编译这些代码，因为这会给“有关方面（Interested Parties）”很多关于你应用的洞察。 :] 
2. Just to mix things up，你创建了一个 `dispatch_queue_t` 实例变量而不是在参数上直接使用函数。当代码变长，分拆有助于可读性。
3. 你需要 `source` 在方法范围之外也可被访问，所以你使用了一个 static 变量。
4. 使用 `weakSelf` 以确保不会出现保留环（Retain Cycle）。这对 `PhotoCollectionViewController` 来说不是完全必要的，因为它会在应用的整个生命期里保持活跃。然而，如果你有任何其它会消失的类，这就能确保不会出现保留环而造成内存泄漏。
5. 使用 `dispatch_once` 确保只会执行一次 Dispatch Source 的设置。
6. 初始化 `source` 变量。你指明了你对信号监控感兴趣并提供了 `SIGSTOP` 信号作为第二个参数。进一步，你使用主队列处理接收到的事件——很快你就好发现为何要这样做。
7. 如果你提供的参数不合格，那么 Dispatch Source 对象不会被创建。也就是说，在你开始在其上工作之前，你需要确保已有了一个有效的 Dispatch Source 。
8. 当你收到你所监控的信号时，`dispatch_source_set_event_handler` 就会执行。之后你可以在其 Block 里设置合适的逻辑处理器（Logic Handler）。
9. 一个基本的 `NSLog` 语句，它将对象打印到控制台。
10. 默认的，所有源都初始为暂停状态。如果你要开始监控事件，你必须告诉源对象恢复活跃状态。

编译并运行应用；在调试器里暂停并立即恢复应用，查看控制台，你会看到这个来自黑暗艺术的函数确实可以工作。你看到的大概如下：

```Text
2014-03-29 17:41:30.610 GooglyPuff[8181:60b] Hi, I am:
```

你的应用现在具有调试感知了！这真是超级棒，但在真实世界里该如何使用它呢？

你可以用它去调试一个对象并在任何你想恢复应用的时候显示数据；你同样能给你的应用加上自定义的安全逻辑以便在恶意攻击者将一个调试器连接到你的应用上时保护它自己（或用户的数据）。

>译者注：好像挺有用！

一个有趣的主意是，使用此方式的作为一个堆栈追踪工具去找到你想在调试器里操纵的对象。

![What_Meme][20]

稍微想想这个情况。当你意外地停止调试器，你几乎从来都不会在所需的栈帧上。现在你可以在任何时候停止调试器并在你所需的地方执行代码。如果你想在你的应用的某一点执行的代码非常难以从调试器访问的话，这会非常有用。有机会试试吧！

![I_See_What_You_Did_Meme][21]

将一个断点放在你刚添加在 viewDidLoad 里的事件处理器的 `NSLog` 语句上。在调试器里暂停，然后再次开始；应用会到达你添加的断点。现在你深入到你的 PhotoCollectionViewController 方法深处。你可以访问 PhotoCollectionViewController 的实例得到你关心的内容。非常方便！

>注意：如果你还没有注意到在调试器里的是哪个线程，那现在就看看它们。主线程总是第一个被 libdispatch 跟随，它是 GCD 的坐标，作为第二个线程。之后，线程计数和剩余线程取决于硬件在应用到达断点时正在做的事情。

在调试器里，键入命令：`po [[weakSelf navigationItem] setPrompt:@"WOOT!"]`

然后恢复应用的执行。你会看到如下内容：

![Dispatch_Sources_Xcode_Breakpoint_Console][22]

![Dispatch_Sources_Debugger_Updating_UI][23]

With this method, you can make updates to the UI, inquire about the properties of a class, and even execute methods — all while not having to restart the app to get into that special workflow state. Pretty neat.

使用这个方法，你可以更新 UI、查询类的属性，甚至是执行方法——所有这一切都不需要重启应用并到达某个特定的工作状态。相当优美吧！

>译者注：发挥这一点，是可以做出一些调试库的吧？


## 之后又该往何处去？


你可以[在此下载最终的项目][24]。

我讨厌再次提及此主题，但你真的要看看 [如何使用 Instruments][11] 教程。如果你计划优化你的应用，那你一定要学会使用它。请注意 Instruments 擅长于分析相对执行：比较哪些区域的代码相对于其它区域的代码花费了更长的时间。如果你尝试计算出某个方法实际的执行时间，那你可能需要拿出更多的自酿的解决方案（Home-brewed Solution）。

同样请看看 [如何使用 NSOperations 和 NSOperationQueues][25] 吧，它们是建立在 GCD 之上的并发技术。大体来说，如果你在写简单的用过就忘的任务，那它们就是使用 GCD 的最佳实践，。NSOperations 提供更好的控制、处理大量并发操作的实现，以及一个以速度为代价的更加面向对象的范例。

记住，除非你有特别的原因要往下流走（译者的玩笑：即使用低级别 API），否则永远应尝试并坚持使用高级的 API。如果你想学到更多或想做某些非常非常“有趣”的事情，那你就应该冒险进入 Apple 的黑暗艺术。

祝你好运，玩得开心！有任何问题或反馈请在下方的讨论区贴出！

=====================

译者注：欢迎非商业转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！

欢迎转发此条微博 [http://weibo.com/2076580237/B4eCrBWrI](http://weibo.com/2076580237/B4eCrBWrI)  以分享给更多人！

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳译者的幸苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

版权声明：自由转载-非商用-非衍生-保持署名 | [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)

[1]: http://www.raywenderlich.com/feed/
[2]: http://twitter.com/rwenderlich
[3]: http://cdn3.raywenderlich.com/wp-content/uploads/2014/03/Concurrency_vs_Parallelism.png "Learn about concurrency in this Grand Central Dispatch in-depth tutorial series."
[4]: https://github.com/nixzhu/dev-blog/blob/master/2014-04-19-grand-central-dispatch-in-depth-part-1.md
[5]: http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/GooglyPuff_End_1.zip
[6]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2014-01-17-at-5.49.51-PM-308x500.png
[7]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSError_Class/
[8]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSURL_Class/
[9]: http://nshipster.com/network-link-conditioner/
[10]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/Thread_All_The_Code_Meme.jpg
[11]: http://www.raywenderlich.com/23037/how-to-use-instruments-in-xcode
[12]: http://cdn1.raywenderlich.com/wp-content/uploads/2014/01/Gandalf_Semaphore.png
[13]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSString_Class/
[14]: http://en.wikipedia.org/wiki/Spinlock
[15]: http://greenteapress.com/semaphores/
[16]: http://en.wikipedia.org/wiki/Dining_philosophers_problem
[17]: https://developer.apple.com/library/mac/documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html#//apple_ref/doc/constant_group/Dispatch_Source_Type_Constants
[18]: https://developer.apple.com/library/mac/documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html#//apple_ref/c/macro/DISPATCH_SOURCE_TYPE_SIGNAL"
[19]: http://www.opensource.apple.com/source/xnu/xnu-1456.1.26/bsd/sys/signal.h
[20]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/What_Meme.jpg
[21]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/I_See_What_You_Did_Meme.png
[22]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/Dispatch_Sources_Xcode_Breakpoint_Console-650x500.png
[23]: http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/Dispatch_Sources_Debugger_Updating_UI-308x500.png
[24]: http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/GooglyPuff-Final.zip
[25]: http://www.raywenderlich.com/19788/how-to-use-nsoperations-and-nsoperationqueues

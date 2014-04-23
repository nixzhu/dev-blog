# 开发使用 iBeacon 的 iOS 7 应用

本文翻译至 [http://www.raywenderlich.com/66584/ios7-ibeacons-tutorial](http://www.raywenderlich.com/66584/ios7-ibeacons-tutorial)

原作者：[Chris Wagner](http://www.raywenderlich.com/u/cwagdev)

译者：[@nixzhu](https://twitter.com/nixzhu)

===========================================

你是否希望过你的手机能够显示你在一栋大楼里的位置？例如[商场](http://techcrunch.com/2014/01/06/inmarket-rolls-out-ibeacons-to-200-safeway-giant-eagle-grocery-stores-to-reach-shoppers-when-it-matters/)或[棒球场](http://www.macrumors.com/2014/01/30/mlb-ibeacon-rollout/)。

诚然，GPS 能够告诉你你在建筑的哪一边。但要在这些钢筋混凝土做成的棺材里得到 GPS 信号就实在需要好运气了。你所需要的那种东西要能在建筑内部让你的设备确定其物理位置。

这就是…… [iBeacon](http://en.wikipedia.org/wiki/IBeacon)！

这这篇 iBeacon 教程中你将创建一个应用，它能让你注册已知的 iBeacon 发射器并在你的手机离开其区域时告知你。如果你还不明白这能有什么用，只需想想有多少次你早早地来到工作场所或学校，却不得不回头去拿你忘在家里电脑包。

这个方便的小应用和一个 iBeacon 能够节省你的汽油钱、时间甚至是面子——当你星期一早上第一件事是冲出门去找回你的电脑包，那你周围同事就会议论纷纷！（译者注：不知如何翻译，选择意译（when you throw out some choice language around your colleagues first thing Monday morning as you storm out the door to retrieve your laptop bag!））

这个应用的使用场景是将某个 iBeacon 发射器安装在你的电脑包、钱包甚至是你的猫的项圈上——即任何你不想丢失的重要物件。一旦你的设备离开发射器的范围，你的应用就会检测到并通知你。

这篇 iBeacon 教程完成后，你就会懂得如何监控 iBeacon 并在你遇到一个时做出合适的反应。

## iBeacon 硬件

当 Apple 在 iOS 7 中介绍 iBeacon 时，他们也宣布任何一个兼容的 iOS 设备都能作为一个 iBeacon 。然而，他们也表示硬件制造商同样也能制造单独的、低功耗的 iBeacon 。在本文发表前，距离 iOS 7 的推出已过去大约 6 个月，现在已有许多家公司宣布和推出了独立的硬件 iBeacon 发射器。

iBeacon 使用 [Bluetooth LE](http://en.wikipedia.org/wiki/Bluetooth_low_energy) 技术，所以你必须要有一个内置有低功耗蓝牙的 iOS 设备以便与 iBeacon 协同工作。目前这个列表里包含如下一些设备：

- iPhone 4s 或更新的
- 第三代 iPad 或更新的
- iPad mini 或更新的
- 第五代iPod touch 或更新的

我足够幸运，得到一些在 [KS Technologies](http://www.kstechnologies.com/products/particle) 由天才（且友好）的团队创造的测试设备（evaluation units）。他们做的 iBeacon 硬件，叫做 _Particle_ ，已预编程为能广播特定 UUID 、主要值和次要值的组合——你马上就会学到这些是什么东西。它还装有一颗纽扣电池，足够让你的 iBeacon 持续运行 6 个月。

![KSTechnology-Particles](http://cdn2.raywenderlich.com/wp-content/uploads/2014/03/DSC_9569_grande-250x250.jpg)

KS Technologies 在 App Store 商已有两款应用：[Particle Detector](https://itunes.apple.com/us/app/particle-detector/id724226138?mt=8) 和 [Particle Accelerator](https://itunes.apple.com/us/app/particle-accelerator/id727105504?mt=8)。Particle Detector 让你可以不需要编写一行代码就能轻松监控 iBeacon ，而 Particle Accelerator 让你可以无线地重配置设备，也就是更新它们地固件。

还有其它的 iBeacon 提供商，一个快速的 [Google 搜索](https://www.google.com/search?client=safari&rls=en&q=ibeacon+hardware&ie=UTF-8&oe=UTF-8) 可以将它们透露给你。为了达成本文的目的，你将专注于 KS Technologies 提供的 iBeacon，虽然其基本内容能应用于大部分其它的 iBeacon 硬件。

>注意：如果你没有一个独立的 iBeacon 发射器，但你有另外一个支持低功耗蓝牙技术的 iOS 设备，你可以通过创建一个应用来模拟一个 iBeacon ，这在 [iOS 7 by Tutorials](http://www.raywenderlich.com/store/ios-7-by-tutorials) 的第 22 章—— Core Location 里的新东西——中有所描述。

## UUID、主要、次要标识符

如果你不熟悉 iBeacon，你可能也不熟悉术语 `UUID`、`主要值（major value）` 和 `次要值（minor value）`。

一个 iBeacon 除了是一个低功耗蓝牙设备之外什么也不是，它们以特定结构发布信息。这些特定的东西超出本教程的范围，但要明白的一件重要事情是 iOS 之所以能够监控这些 iBeacon 就是基于 `UUID`、`主要值` 和 `次要值`。

UUDID 是 Universally Unique Identifier（通用唯一标识符）的缩写，它实际上是一个随机字符串；`B558CBDA-4472-4211-A350-FF1196FFE8C8` 就是一个例子。在 iBeacon 的讨论范围里，一个 UUID 通常用于表示你的顶层标识。作为开发者如果你生成一个 UUID 并将其分配给你的 iBeacon 设备，那么当一个设备检测到你的 iBeacon 时，它就知道它是在和哪个 iBeacon 通信。

主要值与次要值在 UUID 之上提供了稍多的粒度。这些值只是 16 位无符号整数，能够标识每个单独的 iBeacon ，甚至是具有同样 UUID 的哪些。

举个例子，如果你有多间百货公司，那么你所有的 iBeacon 发射器都可有同一个 UUID ，但每个店都有它自己的主要值，而里面的每个部门就会有它自己的次要值。你的应用能够对一个位于你在迈阿密、佛罗里达店的鞋类部们里的 iBeacon 做出响应。

你可以在这篇 [维基百科文章](http://en.wikipedia.org/wiki/Universally_unique_identifier) 中学到更多关于 UUID 的知识。

## 入门

从[这里](http://cdn1.raywenderlich.com/wp-content/uploads/2014/03/ForgetMeNot-starter-project.zip)下载启动项目——它包含一个简单的界面，能添加和移除其表格视图中的条目。表格中的每个条目标识一个单独的 iBeacon 发射器，它在真实世界里就代表那些你不想落在身后的东西。

编译并运行应用；你会看到一个空列表，缺少条目。按下右上角的 `+` 按钮以添加一个新条目，如下图所示：

![First Launch](http://cdn4.raywenderlich.com/wp-content/uploads/2014/03/iOS-Simulator-Screen-shot-Mar-1-2014-12.18.27-PM-281x500.png)  
首次运行

要添加一个条目，你就简单地为这个条目输入一个名字以及对应其 iBeacon 的三个值。试着用 `8AEFB031-6C32-486F-825B-E26FA193487D` 作为 UUID ， `1` 作为主要值， `2` 作为次要值，它们暂时作为占位符，如下所示：

![Adding an Item](http://cdn3.raywenderlich.com/wp-content/uploads/2014/03/Screenshot-2014.02.28-01.04.10-281x500.png)  
添加一个条目

>注意：你可以通过伴侣应用如 [Particle Detector](https://itunes.apple.com/us/app/particle-detector/id724226138?mt=8) 来找到你的 iBeacon 的 UUID ，或者翻看一下你的 iBeacon 的文档。

按下 `Save` 回到条目列表；你会看到你的条目的位置为 _Unknown_ ，如下所示：

![ForgotMeNot-item-location-unknown](http://cdn2.raywenderlich.com/wp-content/uploads/2014/03/iOS-Simulator-Screen-shot-Mar-1-2014-12.32.02-PM-281x500.png)

如果你希望的话，还可以添加更多条目，或者滑动删除已存在的条目。 `NSUserDefaults` 将这些条目放在一个列表中持久保存，所以它们在应用重启后依然可用。

从表面上看，它似乎没有做什么事情；但更多有趣东西还在下面呢。这个项目的大部分是为经验丰富的 iOS 开发者准备的。这个应用的独特之处是 `RWTItem` 模型，它表示列表中的条目。

打开 `RWTItem.h` 在 Xcode 中看一看。模型类反映用户请求的接口，而且它实现了 `NSCoding` ，所以它能被序列化和反序列化到磁盘上以持久存储。

现在看看 `RWTAddItemViewController.m` 。这是添加新条目的控制器。它是一个简单的 `UITableViewController` ，它在用户的输入上做一些验证以保证用户输入合法的名字和 UUID，名字字段只要求至少有一个字符，而 UUID 字段，根据 UUID 规格，有更多严格要求。

仔细看看 `viewDidLoad:` 中的下列一行：

```Objective-C
NSString *uuidPatternString = @"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$";
```

这是一个[正则表达式](http://en.wikipedia.org/wiki/Regular_expression)模式，它将输入的 UUID 与标准的 UUID 格式对比。如果字符串的格式符合 ` XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX` ，它就返回 TRUE ，这里的 X 是一个十六进制的值，从 0 到 F。所有的 iBeacon 必须广播一个 UUID 而且它必须符合上面的规格。

右上角的 `Save` 按钮在 `nameTextField` 和 `uuidTextField` 都验证合法之后才会变成可点击的。

现在你熟悉了启动项目，你可以继续实现 iBeacon 元素到你自己的项目！

## 监听你的 iBeacon

你的设备不会自动监听你的 iBeacon ——首先你必须告诉它这样去做。 `CLBeaconRegion` 类表示一个 iBeacon；`CL` 类前缀表示它是 Core Location 框架的一部分。

可能你会有些奇怪 iBeacon 会与 Core Location 相关，毕竟它是蓝牙设备，但考虑到 iBeacon 提供微定位信息对应 GPS 提供宏定位信息，也就不奇怪了。在将一个 iOS 设备当作一个iBeacon 而编程时，你就要利用 Core Bluetooth 框架，而在监控 iBeacon 时，你只需同 Core Location 打交道。

你要做的首要工作是将 `RWTItem` 适配到 `CLBeaconRegion` 。

打开 `RWTItem.h` 并用如下代码取代其内容：

```Objective-C
#import <Foundation/Foundation.h>
 
@import CoreLocation;
 
@interface RWTItem : NSObject <NSCoding>
 
@property (strong, nonatomic, readonly) NSString *name;
@property (strong, nonatomic, readonly) NSUUID *uuid;
@property (assign, nonatomic, readonly) CLBeaconMajorValue majorValue;
@property (assign, nonatomic, readonly) CLBeaconMinorValue minorValue;
 
- (instancetype)initWithName:(NSString *)name
                        uuid:(NSUUID *)uuid
                       major:(CLBeaconMajorValue)major
                       minor:(CLBeaconMinorValue)minor;
 
 
@end
```

此处唯一的不同是引入了 Core Location 并用 `CLBeaconMajorValue` 和 `CLBeaconMinorValue` 取代了 `uint16_t` 类型以分别表示 `majorValue` 和 `minorValue` 属性。

虽然它们的基本数据类型是一样的，但这样能提高模型的可读性。

打开 `RWTItem.m` 并更新 `initWithName:uuid:major:minor:` 方法签名中的类型，如下所示：

```Objective-C
- (instancetype)initWithName:(NSString *)name
                        uuid:(NSUUID *)uuid
                       major:(CLBeaconMajorValue)major
                       minor:(CLBeaconMinorValue)minor
```

这样就抑制了编译器警告。

打开 `RWTItemsViewController.m` ，在其它引入之下添加一个 CoreLocation 引入语句，再为这个类添加 `CLLocationManagerDelegate` 协议，最后以 `CLLocationManager` 创建一个新的 `locationManager` 属性，如下所示：

```Objective-C
#import "RWTItemCell.h"
 
@import CoreLocation;
 
static NSString * const kRWTStoredItemsKey = @"storedItems";
 
@interface RWTItemsViewController () <UITableViewDataSource, UITableViewDelegate, CLLocationManagerDelegate>
 
@property (weak, nonatomic) IBOutlet UITableView *itemsTableView;
@property (strong, nonatomic) NSMutableArray *items;
@property (strong, nonatomic) CLLocationManager *locationManager;
 
@end
```

接下来，在 `viewDidLoad:` 中添加一个调用去初始化 `self.locationManager` 属性，将其 delegate 指定为 `self` ，如下所示：

```Objective-C
- (void)viewDidLoad {
    [super viewDidLoad];
 
    self.locationManager = [[CLLocationManager alloc] init];
    self.locationManager.delegate = self;
 
    [self loadItems];
}
```

这就为 locationManager 指明了本类希望接收它的 delegate 方法呼叫。

现在你有一个 `CLLocationManager` 实例，你可以使用 `CLBeaconRegion` 指导你的应用开始监控特定区域。当你注册一个要监视的区域时，这些区域就会在你的应用的两次加载之间做持久化存储。这在稍后看来会很重要，当你对跨过区域的边界做出回应时，你的应用很可能没有在运行。

你列表中的 iBeacon 条目由 `RWTItem` 模型通过 `items` 属性表示。然而，为了开始监控一个区域， `CLLocationManager` 期望你提供一个 `CLBeaconRegion` 实例。

打开 `RWTItemsViewController.m` 并创建如下帮助方法：

```Objective-C
- (CLBeaconRegion *)beaconRegionWithItem:(RWTItem *)item {
    CLBeaconRegion *beaconRegion = [[CLBeaconRegion alloc] initWithProximityUUID:item.uuid
                                                                           major:item.majorValue
                                                                           minor:item.minorValue
                                                                      identifier:item.name];
    return beaconRegion;
}
```

这会返回一个源自所提供 `RWTItem` 的新的 `CLBeaconRegion` 实例。

你能注意到类之间有着相似的结构。所以用 `initWithProximityUUID:major:minor:identifier:` 创建一个 `CLBeaconRegion` 实例非常直接了当。

现在你需要一个方法去监视给定的条目。

继续在 `RWTItemsViewController.m` 中工作，添加如下代码，就在 `beaconRegionWithItem:` 方法下面：

```Objective-C
- (void)startMonitoringItem:(RWTItem *)item {
    CLBeaconRegion *beaconRegion = [self beaconRegionWithItem:item];
    [self.locationManager startMonitoringForRegion:beaconRegion];
    [self.locationManager startRangingBeaconsInRegion:beaconRegion];
}
```

这个方法接受一个 `RWTItem` 实例并使用你之前定义的方法创建一个 `CLBeaconRegion` 实例。然后它告诉位置管理器开始监控给定的区域，并确定它们的距离。一个 iOS 设备收到一个 iBeacon 传输就能大概估计它到 iBeacon 的距离。（正在发射的iBeacon设备与接收设备之间的）距离可分为3个不同范围：

- `即时（Immediate）`，在几厘米内
- `近（Near）`，在几米内
- `远（Far）`，超过 10 米以外

默认情况下，当区域进入和退出时，监控就会通知你，不论你的应用是否在运行。测距，就是另外一回事，只在你的应用运行时才监视附近的区域。

你需要一个在某个条目被删除后停止监控其区域的办法。

添加如下代码到 `RWTItemsViewController.m` 就在 `startMonitoringItem:` 下面：

```Objective-C
- (void)stopMonitoringItem:(RWTItem *)item {
    CLBeaconRegion *beaconRegion = [self beaconRegionWithItem:item];
    [self.locationManager stopMonitoringForRegion:beaconRegion];
    [self.locationManager stopRangingBeaconsInRegion:beaconRegion];
}
```

上面的方法雷同于 `startMonitoringItem:`，除了它调用的是位置管理器的监控和测距的“停止变种（stop variants）”。

现在你有了启动和停止方法，是时候一起使用它们了！开始监控的自然时机是当用户添加一个条目到列表中的时候。

看看 `RWTItemsViewController.m` 中的 `prepareForSegue:` ；你会看到 `RWTAddItemViewController` 有一个回调，它将在你通过 `itemAddedCompletion` Block 属性添加条目时运行。

在 `itemAddedCompletion` Block 中调用 `startMonitoringItem:` ，如下所示：

```Objective-C
[addItemViewController setItemAddedCompletion:^(RWTItem *newItem) {
    [self.items addObject:newItem];
    [self.itemsTableView beginUpdates];
    NSIndexPath *newIndexPath = [NSIndexPath indexPathForRow:self.items.count-1 inSection:0];
    [self.itemsTableView insertRowsAtIndexPaths:@[newIndexPath]
                               withRowAnimation:UITableViewRowAnimationAutomatic];
    [self.itemsTableView endUpdates];
    [self startMonitoringItem:newItem]; // Add this statement
    [self persistItems];
}];
```

现在当你添加一个新条目时就会立即调用 `startMonitoringItem:` 。对后续调用 `persistItems` 的说明：这会接受所有条目并将它们持久化存储在 `NSUserDefaults` 中，因此用户不用在每次应用加载时重新输入它们的条目。

在 `RWTItemsViewController.m` 中的 `viewDidLoad:` 中调用 `loadItems` ，它从 `NSUserDefaults` 中读取持久化存储的条目再将它们保存在 `items` 数组中。

继续在 `RWTItemsViewController.m` 中工作，更新 `loadItems` 以确保每个条目都被监控，如下所示：

```Objective-C
- (void)loadItems {
    NSArray *storedItems = [[NSUserDefaults standardUserDefaults] arrayForKey:kRWTStoredItemsKey];
    self.items = [NSMutableArray array];
 
    if (storedItems) {
        for (NSData *itemData in storedItems) {
            RWTItem *item = [NSKeyedUnarchiver unarchiveObjectWithData:itemData];
            [self.items addObject:item];
            [self startMonitoringItem:item]; // Add this statement
        }
    }
}
```

现在每次你从持久化存储里加载条目时，你就指示位置管理器开始监控它们。

你现在必须处理从列表中移出条目的情况。

用下面的代码替换 `tableView:commitEditingStyle:forRowAtIndexPath:` 的内容：

```Objective-C
- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        RWTItem *itemToRemove = [self.items objectAtIndex:indexPath.row];
        [self stopMonitoringItem:itemToRemove];
        [tableView beginUpdates];
        [self.items removeObjectAtIndex:indexPath.row];
        [tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationAutomatic];
        [tableView endUpdates];
        [self persistItems];
    }
}
```

在上面的代码中你检查 editing style 是否是 `UITableViewCellEditingStyleDelete`；如果是，你就从 `items` 中移除条目并使用它去执行 `stopMonitoringItem:` 。

你应该震惊于很多工作实现起来都_非常_简单。Apple 已经为了使 iBeacon 对于开发者来说非常易于使用而做了大量复杂的工作。

到这里，你的进步就非常大了！现在你的应用在适当的时候会自动开始和停止监听特定的 iBeacon 。

你可以编译并运行你的应用；但即使你已经注册的 iBeacon 中某一个出现在你的应用的范围内，它也不知道如何应对。

这就来修正它！

## 找到 iBeacon 时做出响应

现在你的位置管理器正在监听 iBeacon，是时候响应它们了！

在你之前初始化 `CLLocationManager` 的时候，你同时设置其 delegate 为 `self` 。那就来现实这些 delegate 方法吧。

首先是添加一些错误处理，因为你正在同非常具体的硬件特性打交道，你需要知道任何原因导致的监控和测距失败。

添加如下两个方法到 `RWTItemsViewController.m` 中：

```Objective-C
- (void)locationManager:(CLLocationManager *)manager monitoringDidFailForRegion:(CLRegion *)region withError:(NSError *)error {
    NSLog(@"Failed monitoring region: %@", error);
}
 
- (void)locationManager:(CLLocationManager *)manager didFailWithError:(NSError *)error {
    NSLog(@"Location manager failed: %@", error);
}
```

这些方法简单地输出它们接收到的错误信息，之后你可将其用于分析发生的问题。

如果你的应用运行一切正常，那么你永远不会看到这个调用的输出。然而，如果真的发生了某些异常情况，那这些log消息所提供的可能就是非常有价值的信息。这比出错后什么也没有要好的多。简单的 `NSLog` 语句对于一个教程来说足够了，但在你开发的应用中，你还是应该用更加完善的方式处理这些错误情况。

下一步是实时显示你的注册 iBeacon 的感知接近值（perceived proximity）。

添加如下 stubbed-out 方法到 `RWTItemsViewController.m` 中：

```Objective-C
- (void)locationManager:(CLLocationManager *)manager
        didRangeBeacons:(NSArray *)beacons
               inRegion:(CLBeaconRegion *)region
{
 
}
```

上面的 delegate 方法将在 iBeacon 到达范围内、离开范围或某个 iBeacon 的范围改变时被调用。

你应用的目标是使用传入的有范围的 iBeacon 去更新条目列表并监视它们的感知接近值。你将从遍历 beacons 数组去匹配你的列表中有范围的 iBeacon 开始。

更新 `locationManager:` 的实现，如下所示：

```Objective-C
- (void)locationManager:(CLLocationManager *)manager
        didRangeBeacons:(NSArray *)beacons
               inRegion:(CLBeaconRegion *)region
{
    for (CLBeacon *beacon in beacons) {
        for (RWTItem *item in self.items) {
            // Determine if item is equal to ranged beacon
        }
    }
}
```

现在每次你测量 iBeacon，你就遍历 beacons 和条目列表。

注意第二个 `for` 循环里的注释；在这里你需要实现一个逻辑以检测是否这是代表当前条目的 iBeacon 。

打开 `RWTItem.h` 并添加如下方法申明：

```Objective-C
- (BOOL)isEqualToCLBeacon:(CLBeacon *)beacon;
```

这个新方法用一个 `RWTItem` 实例去比较一个 `CLBeacon` 实例，看看它们是否相等——即，是否所有的标识符都匹配。

添加如下属性到 `RWTItem.h` ：

```Objective-C
@property (strong, nonatomic) CLBeacon *lastSeenBeacon;
```

这个属性存储最新的 _见过_ 特定条目的 `CLBeacon` 实例；这将用于显示接近信息（proximity information）。

添加如下方法到 `RWTItem.m` ：

```Objective-C
- (BOOL)isEqualToCLBeacon:(CLBeacon *)beacon {
    if ([[beacon.proximityUUID UUIDString] isEqualToString:[self.uuid UUIDString]] &&
        [beacon.major isEqual: @(self.majorValue)] &&
        [beacon.minor isEqual: @(self.minorValue)])
    {
        return YES;
    } else {
        return NO;
    }
}
```

一个 `CLBeacon` 实例将等价于一个 `RWTItem.`，如果它们的 UUID、主要值以及次要值都一样的话。

现在你还需要完成测距 delegate 方法，将会用到上面的帮助方法。

更新 `RWTItemsViewController.m` 中的 `locationManager:didRangeBeacons:inRegion` 以调用你的新方法，如下所示：

```Objective-C
- (void)locationManager:(CLLocationManager *)manager
        didRangeBeacons:(NSArray *)beacons
               inRegion:(CLBeaconRegion *)region
{
    for (CLBeacon *beacon in beacons) {
        for (RWTItem *item in self.items) {
            if ([item isEqualToCLBeacon:beacon]) {
                item.lastSeenBeacon = beacon;
            }
        }
    }
}
```

在上面的代码中，当你找到匹配的条目和 iBeacon 时，你就设置 `lastSeenBeacon` 。

现在可以使用这个属性来显示被定位的 iBeacon （the ranged iBeacon） 的接近感知值（perceived proximity）。

打开 `RWTItemCell.m` 并如下所示代码更新 `setItem:` ：

```Objective-C
- (void)setItem:(RWTItem *)item {
    if (_item) {
        [_item removeObserver:self forKeyPath:@"lastSeenBeacon"];
    }
 
    _item = item;
    [_item addObserver:self 
            forKeyPath:@"lastSeenBeacon"
               options:NSKeyValueObservingOptionNew
               context:NULL];
 
    self.textLabel.text = _item.name;
}
```

当你为 Cell 设置条目时你还添加了一个观察者（observer）去关注 `lastSeenBeacon` 属性。另外，如果你的 Cell 已经有了一个条目，你先移除旧的观察者，以便符合键值观察的要求。

在 Cell 被 dealloc 时，你也需要移除观察者。就在 `RWTItemCell.m` 中添加如下方法：

```Objective-C
- (void)dealloc {
    [_item removeObserver:self forKeyPath:@"lastSeenBeacon"];
}
```

现在你就在观察 `lastSeenBeacon` 的值了，你可以在响应 iBeacon 的接近值（iBeacon’s proximity） 的任何改变时放入一些逻辑。

每个 `CLBeacon` 实例都有一个 `proximity` 属性，它是一个 `enum` ，所以你必须将它的整数值转换为更有意义的东西。

添加如下方法到 `RWTItemCell.m` ：

```Objective-C
- (NSString *)nameForProximity:(CLProximity)proximity {
    switch (proximity) {
        case CLProximityUnknown:
            return @"Unknown";
            break;
        case CLProximityImmediate:
            return @"Immediate";
            break;
        case CLProximityNear:
            return @"Near";
            break;
        case CLProximityFar:
            return @"Far";
            break;
    }
}
```

这就利用 `proximity` 返回了人类可读的接近值，然后用于下面的方法。

现在添加 `observeValueForKeyPath:ofObject:change:context:` 方法，如下：

```Objective-C
- (void)observeValueForKeyPath:(NSString *)keyPath 
                      ofObject:(id)object 
                        change:(NSDictionary *)change 
                       context:(void *)context {
    if ([object isEqual:self.item] && [keyPath isEqualToString:@"lastSeenBeacon"]) {
        NSString *proximity = [self nameForProximity:self.item.lastSeenBeacon.proximity]
        self.detailTextLabel.text = [NSString stringWithFormat:@"Location: %@", proximity];
    }
}
```

在每次 `lastSeenBeacon` 值改变时，都会调用上面的方法，它会用接近值设置 Cell 的 `detailTextLabel.text` 属性。

编译并运行你的应用；保证你的 iBeacon 已被注册，然后移动你的设备接近或远离你的 iBeacon。你会见到 Label 随着你的移动而更新，如下所示：

![ForgetMeNot-item](http://cdn4.raywenderlich.com/wp-content/uploads/2014/03/Screenshot-2014.02.28-01.04.18-281x500.png)

现在你可能已发现接近感知受到你的 iBeacon 的物理位置的严重影响；如果它被放在在某个东西的内部，例如一个盒子或者一个包里，信号就可能被屏蔽，因为 iBeacon 是低功耗设备，它的信号非常容易衰减。

在你设计你自己的应用，以及决定你的 iBeacon 硬件的最佳摆放位置时，请将这一点记在脑子里。

## 通知

到这里事情快要完成了；你有一个你的 iBeacon 的列表，而且能实时监控它们的接近值。但这还没达到你的应用的最终目标。你依然需要在应用没有运行的情况下通知用户他们忘记了他们的电脑包或他们的猫跑掉了——甚至更糟糕的情况，比如他们的猫带着电脑包跑掉了！ :]

![zorro-ibeacon](http://cdn4.raywenderlich.com/wp-content/uploads/2014/03/zorro-ibeacon.jpg)

它们看起来都天真无辜，对吧？

如果到这里你学会了什么，那一定是了解到不需要太多代码就能添加许多的 iBeacon 功能——而这最后的几个方法也不会有太大差别。

打开 `RWTAppDelegate.m` 引入 Core Location 模块：

```Objective-C
@import CoreLocation;
```

接下来，更新类扩展以便符合 `CLLocationManagerDelegate` 协议，再添加一个 `CLLocationManager` 属性，如下所示：

```Objective-C
@interface RWTAppDelegate () <CLLocationManagerDelegate>
 
@property (strong, nonatomic) CLLocationManager *locationManager;
 
@end
```

与之前一样，你需要初始化位置管理器并设置它们的 delegate 。

在 `application:didFinishLaunchingWithOptions:` 的顶部添加如下语句：

```Objective-C
self.locationManager = [[CLLocationManager alloc] init];
self.locationManager.delegate = self;
```

乍一看，这好像有些太简单了。当你的应用加载时，新分配的  `CLLocationManager` 实例有什么用？它又如何知道要监视的区域？

回想到在你的应用中任何你使用 `startMonitoringForRegion:` 添加的监视区域都被所有的位置管理器共享。所以你免费得到的一点持久化，实在是极有帮助。

若没有这个功能，就将由你来找出哪些区域正在被监视并在应用加载时重新开始监视它们。但就算这样也不够，因为你的应用还不知道在遇到某个区域时就醒过来。

感谢 Apple 在 Core Location 中已经为你做了许多繁重的工作。这里的最后一步只是简单地在 Core Location 遇到某个区域并唤醒你的应用时做出响应即可。

添加如下方法到 ` RWTAppDelegate.m` 地底部：

```Objective-C
- (void)locationManager:(CLLocationManager *)manager didExitRegion:(CLRegion *)region {
    if ([region isKindOfClass:[CLBeaconRegion class]]) {
        UILocalNotification *notification = [[UILocalNotification alloc] init];
        notification.alertBody = @"Are you forgetting something?";
        notification.soundName = @"Default";
        [[UIApplication sharedApplication] presentLocalNotificationNow:notification];
    }
}
```

你的位置管理器将在你离开某个区域时调用上面的方法，这就是这个应用有用的时刻。你不需要在你接近你的电脑包时被告知，只需在你离开它太远时通知你。

此处你检查区域是否是一个 `CLBeaconRegion` ，因为如果你同时也在执行地理定位区域监视的话，它还可能是一个 `CLCircularRegion` 。然后你就发送一个本地通知，附带一个消息“Are you forgetting something?” 。

编译并运行你的应用；离开某个你的注册的 iBeacon，然后一旦你离开得足够远，你就会看到通知弹出来。

![ForgetMeNot-notification](http://cdn1.raywenderlich.com/wp-content/uploads/2014/03/Screenshot-2014.02.28-01.05.36-281x500.png)

如果你实际上不可能离开你的 iBeacon 太远（译者注：房间太小？），那就将它的电源关掉或者移除它的电池以便测试这个功能。

>注意：最后关于 iBeacon 和 iOS 行为的记录：  
iOS 7.1 添加了当它遇到被监视的 iBeacon 时从后台唤醒应用的能力。之前，用户需要打开应用以响应通知，但现在全都免费工作了！  
Apple 以未文档化的方式推迟“退出通知（exit notifications）”。这可能是特别设计的，以便你的应用不会过早收到通知，比如你在某个范围的边缘游荡或者这个 iBeacon 的信号暂时中断。在我的经验里，“退出通知”通常在某个 iBeacon 离开范围的一到两分钟之后才发生。  
译者注：这是不是也太迟了？

## 下一步怎么走？

现在你有一个有用的应用帮你监视那些你很难找到和监控的事物。

你可以[在此](http://cdn5.raywenderlich.com/wp-content/uploads/2014/04/ForgetMeNot-final-project.zip)下载最终的项目。

用一些想像力和编码能力，你给此应用带来了许多真正有用的特性：

- 当条目离开区域时通知用户
- 重复通知确保用户看到
- 在 iBeacon 回到区域时提示用户
- ……或者其它任何你能想像到的事

这个 iBeacon 教程只是碰到 iBeacon 的可能性的一点皮毛而已。在本教程的开头，我提供了一些文章链接显示职业棒球大联盟和商场正在如何以非常参与的方式（in very engaging ways）使用着 iBeacon 。

iBeacon 并不局限于自定义应用；你也可以将它们与 Passbook 的通行证一起使用。试想一下，当你跑进电影院；你就能用 Passbook 得到电影票。当顾客走近验票人员时，他们的应用将自动在 iPhone 上显示票据。

如果你有关于本教程的任何问题或评论，或着任何使用 iBeacon 的新点子，欢迎加入下面的讨论！


===============

译者注：欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条微博 [http://weibo.com/2076580237/B138xzUwI](http://weibo.com/2076580237/B138xzUwI) 以分享给更多人！
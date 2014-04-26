# 制作一个可以滑动操作的 Table View Cell

本文翻译自 [http://www.raywenderlich.com/62435/make-swipeable-table-view-cell-actions-without-going-nuts-scroll-views](http://www.raywenderlich.com/62435/make-swipeable-table-view-cell-actions-without-going-nuts-scroll-views)

原作者：[Ellen Shapiro](http://www.raywenderlich.com/u/designatednerd)

译者：[@nixzhu](https://twitter.com/nixzhu)

==========================================

Apple 通过 iOS 7 的邮件（Mail）应用介绍了一种新的用户界面方案——向左滑动以显示一个有着多个操作的菜单。本教程将会向你展示如何制作一个这样的 Table View Cell，而不用因嵌套的 Scroll View 陷入困境。如果你还不知道一个可滑动的 Table View Cell 意味着什么，那么看看 Apple 的邮件应用：

![Multiple Options](http://cdn4.raywenderlich.com/wp-content/uploads/2014/01/IMG_3743-180x320.png)

可能你会想，既然 Apple 展示了这种方案，那它应该已将其开放给开发者使用了。毕竟，这能有多难呢？但不幸的是，他们只让开发者使用 Delete 按钮——至少暂时是这样。如果你要添加其他的按钮，或者改变 Delete 按钮上的文字或颜色，那你就必须自己去实现。

译者注：其实文字是可以修改的，但是颜色真的不行！

在本教程中，你将先学习如何实现简单的滑动以删除操作（swipe-to-delete action），之后我们再实现滑动以执行操作（swipe-to-perform-actions）。这会要求你深入研究 iOS 7 `UITableViewCell` 的结构，以便复制出我们需要的行为。你将使用到一些我个人非常喜欢的技术用于检查视图层次结构：为视图上色以及使用 `recursiveDescription` 方法来打印出视图层次结构。

## 开始

打开 Xcode，去往 `File\New\Project…` 并选择 `Master-Detail Application` ，如下所示：

![Master-Detail Application](http://cdn1.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2013-11-28-at-6.42.47-AM-475x320.jpg)

将项目命名为 `SwipeableCell` 并填好你自己的 Organization Name 和  Company Identifier 。选择 `iPhone` 为目标设备并确保 `Use Core Data` 没有被选中，如所示：

![Set Up Project](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/Screen_Shot_2013-11-28_at_1.02.38_PM-472x320.png)

对于这样的概念项目的证明，你最好保证数据模型尽量简单。

打开 `MasterViewController.m` 并找到 `viewDidLoad` 。将默认设置 Navigation Bar  Items 的方法替换为如下实现：

```Objective-C
- (void)viewDidLoad {
  [super viewDidLoad];
 
  //1
  _objects = [NSMutableArray array];
 
  //2
  NSInteger numberOfItems = 30;
  for (NSInteger i = 1; i <= numberOfItems; i++) {
    NSString *item = [NSString stringWithFormat:@"Item #%d", i];
    [_objects addObject:item];
  }
}
```

这个方法做了两件事：

1. 这一行创建并初始化一个 `NSMutableArray` 实例，以后你就可以添加对象到它里面了。如果你的数组没有被初始化，那不论你调用 `addObject:` 多少次，你的那些对象都不会被存储起来。译者注：读者还是尽量用 Lazy Load 来实现吧！
2. 这个循环添加了一些字符串到 `_objects` 数组，应用运行时，这些字符串将用于显示在 Table View 里。你可以修改 numberOfItems 的值，以存储适合你的更多或更少的字符串。

下一步。找到 `tableView:cellForRowAtIndexPath:` 并替换其实现为：

```Objective-C
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];
 
    NSString *item = _objects[indexPath.row];
    cell.textLabel.text = item;
    return cell;
}
```

原本 `tableView:cellForRowAtIndexPath:` 的样板使用日期字符串作为简单数据；而你的实现使用你的数组里的 `NSString` 对象去填充 `UITableViewCell` 的 `textLabel` 。

往下滚动到 `tableView:canEditRowAtIndexPath:` ；你会看到这个方法已经设置为返回 `YES` ，也就是说， Table View 的每一行都支持编辑。

就在这个方法下边，`tableView:commitEditingStyle:forRowAtIndexPath:` 处理对象的删除。然而，因为你还不能添加任何东西到这个应用里，那就先稍微修改它一下以适应你的需求。

用下面的代码替换 `tableView:commitEditingStyle:forRowAtIndexPath:` ：

```Objective-C
- (void)tableView:(UITableView *)tableView commitEditingStyle:(UITableViewCellEditingStyle)editingStyle forRowAtIndexPath:(NSIndexPath *)indexPath {
  if (editingStyle == UITableViewCellEditingStyleDelete) {
    [_objects removeObjectAtIndex:indexPath.row];
    [tableView deleteRowsAtIndexPaths:@[indexPath] withRowAnimation:UITableViewRowAnimationFade];
  } else {
    NSLog(@"Unhandled editing style! %d", editingStyle);
  }
}
```

当用户删除某行时，你就用传入的 Index 将那一行的对象从后面的数组中移除，并告知 Table View 它需要移除同一个 `indexPath` 所表示的那一行 Cell，一确保模型和视图的匹配。

你的应用只允许“delete”这一种编辑方式，但在 else 分支里用 log 记录你没有在处理什么也不错。如果有某个诡异的事情发生，你将会在控制台得到一个提示消息，这比方法静悄悄地返回要好。

最后，还有一些清理要做。依然在 `MasterViewController.m` 里，删除 `insertNewObject` 。这个方法现在不正确，因为插入已经不再被支持了。

编译并运行应用；你会看到一个简单列表，如下所示：

![Closed Easy](http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/iOS-Simulator-Screen-shot-Nov-28-2013-6.55.07-AM-213x320.png)

滑动某一行到左边，你就会看到一个 “Delete” 按钮，如下所示：

![Easy delete button](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/iOS-Simulator-Screen-shot-Nov-28-2013-1.06.41-PM-213x320.png)

喔～——这很简单。但现在是时候弄脏双手，深挖进视图层次结构，看看里面到底发了什么。

## 深入视图层次结构（View Hierarchy）

首先：你要找到 Delete 按钮在视图层次结构里的位置，然后你才能决定是否可以将其用于你自定义的 Cell 。

最容易做到这一点的方式是将 View 的各个部分分别染色，以便清楚地看到它们地位置和范围。

继续在 `MasterViewController.m` 里工作，添加如下两行到 `tableView:cellForRowAtIndexPath:` 里，就在最后的 `return` 语句之上：

```Objective-C
cell.backgroundColor = [UIColor purpleColor];
cell.contentView.backgroundColor = [UIColor blueColor];
```

这些颜色足够让我们看清这些视图在 Cell 中的位置。

再次编译并运行，你会看到着色后的元素，如下面的截图所示：

![Colored Cells](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/iOS-Simulator-Screen-shot-Nov-28-2013-8.08.29-PM-213x320.png)

你会清楚地看到蓝色的 `contentView` 停止在 Accessory Indicator 之前，但整个 Cell 自身以紫色高亮，填满了到 `UITableView` 的边缘。

往左边拖动 Cell ，你会看到类似下面的的界面：

![Start to drag cell](http://cdn1.raywenderlich.com/wp-content/uploads/2014/01/iOS-Simulator-Screen-shot-Nov-28-2013-2.16.46-PM-213x320.png)

看起来 Delete 按钮实际上隐藏在 Cell 的_下面_。唯一能 100% 确保的方式是在视图层次结构中再挖深一点。

为了辅助你的视图考古，你可以用一个只能用于调试的方法，叫做 `recursiveDescription` ，它能打印出任意视图的视图层次结构。注意这是一个私有方法， ｀不应该被包含在任何会被放到 App Store 的代码里｀，但它对与视图层次结构实在非常有用。

>Note：目前有两个付费应用能让你用可视化的方式检查视图层次结构：[Reveal](http://revealapp.com/) 和 [Spark Inspector](http://sites.fastspring.com/foundry376/instant/sparkinspector)。另外，还有一个开源项目也可以很好地做到这件事：[iOS-Hierarchy-Viewer](https://github.com/glock45/iOS-Hierarchy-Viewer) 。  
这些应用的价格和质量各有不同，但它们全都要求在你的项目中添加一个库以便支持它们的产品。但如果你不想在项目里安装任何库的话，那 `recursiveDescription` 绝对是得到这些信息的最好的方式。

添加如下打印语句到 `tableView:cellForRowAtIndexPath:` 中，放在 return 语句之前：

```Objective-C
#ifdef DEBUG
  NSLog(@"Cell recursive description:\n\n%@\n\n", [cell performSelector:@selector(recursiveDescription)]);
#endif
```

一旦添加了这一行代码，你就会得到一个警告，也就是 `recursiveDescription` 未被申明；因为它是一个私有方法，编译器并不知道它的存在，`ifdef / endif` 包装器将会额外确保这行代码不会被编译进最终的 release 版里。

编译并运行；你会看到控制台全都是 log 语句，类似下面这样：

```Objective-C
2014-02-01 09:56:15.587 SwipeableCell[46989:70b] Cell recursive description:
 
<UITableViewCell: 0x8e25350; frame = (0 396; 320 44); text = 'Item #10'; autoresize = W; layer = <CALayer: 0x8e254e0>>
   | <UITableViewCellScrollView: 0x8e636e0; frame = (0 0; 320 44); clipsToBounds = YES; autoresize = W+H; gestureRecognizers = <NSArray: 0x8e1d7d0>; layer = <CALayer: 0x8e1d960>; contentOffset: {0, 0}>
   |    | <UIButton: 0x8e22a70; frame = (302 16; 8 12.5); opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x8e22d10>>
   |    |    | <UIImageView: 0x8e20ac0; frame = (0 0; 8 12.5); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0x8e5efc0>>
   |    | <UITableViewCellContentView: 0x8e23aa0; frame = (0 0; 287 44); opaque = NO; gestureRecognizers = <NSArray: 0x8e29c20>; layer = <CALayer: 0x8e62220>>
   |    |    | <UILabel: 0x8e23d70; frame = (15 0; 270 43); text = 'Item #10'; clipsToBounds = YES; opaque = NO; layer = <CALayer: 0x8e617d0>>
```

又要哇～——信息真不少。你所看到的是递归的描述log语句，在每次 Cell 被创建或回收时都会打印。所以你会看到好几个这种消息，因为初始的屏幕上有好几个 Cell 。`recursiveDescription` 会走遍特定视图的每个子视图，输出子视图的描述，并按照视图层次结构排列。它会递归地做这件事，所以对于每个子视图，它也会再去寻找它们的子视图。

虽然信息很多，但它是根据视图层次结构在每个视图上都调用了 `recursiveDescription` 。因此如果你单独打印每个子视图的描述，你会看到同样的信息，但这个方法在子视图的输出前加了一个 `|` 符号和一些空格，以便反映出视图的结构。

为了更加易读，下面光拿出类名和 Frame 来看：

```Objective-C
<UITableViewCell; frame = (0 396; 320 44);> //1
   | <UITableViewCellScrollView; frame = (0 0; 320 44); > //2
   |    | <UIButton; frame = (302 16; 8 12.5)> //3
   |    |    | <UIImageView; frame = (0 0; 8 12.5);> //4
   |    | <UITableViewCellContentView; frame = (0 0; 287 44);> //5
   |    |    | <UILabel; frame = (15 0; 270 43);> //6
```

目前 Cell 里有六个视图：

1. `UITableViewCell` 这是最高层的视图。 Frame 显示它有 320 点宽和 44 点高——宽度和高度都喝预期的一致，因为它和屏幕一样宽，而高度就是 44 点。
2. `UITableViewCellScrollView` 虽然你不能直接使用这个私有类，但它的名字很好地暗示了它的功能。它的 Size 和 Cell 的一样。据此我们推断它的作用是在 Delete 按钮之上装载滑动出来的内容。
3. `UIButton` 它在 Cell 的最右边，就是 Disclosure Indicator 按钮。注意这不是 Delete 按钮。
4. `UIImageView` 是上面 `UIButton` 的子视图，装载着 Disclosure Indicator 的图像。
5. `UITableViewCellContentView` 另外一个私有类，它包含 Cell 的内容。这个类对于开发者来说就是 `UITableViewCell` 的 `contentView` 属性。但它只作为一个 `UIView` 来暴露在外，这就意味着你只在其上调用使用公开的 `UIView` 方法；而不能使用任何与这个类关联的任何私有方法。
6. `UILabel` 显示  “Item #” 文本。

你会注意到 Delete 按钮并没有显示在上面的视图层次结构排列里。嗯～。可能它只在滑动开始时才被添加到层次结构里。对于优化来说这样做很合理。在不需要 Delete 按钮的时候实在没有必要将其放在那里。要验证这个猜想，就添加如下代码到 `tableView:commitEditingStyle:forRowAtIndexPath:` ，就在处理 delete editing style 的 if 语句中：

```Objective-C
#ifdef DEBUG
    NSLog(@"Cell recursive description:\n\n%@\n\n", [[tableView cellForRowAtIndexPath:indexPath] performSelector:@selector(recursiveDescription)]);
#endif
```

这和之前添加的一样，除了这次我们需要滑动 Cell 以便调用 `tableView:commitEditingStyle:forRowAtIndexPath:` ：

译者注：上面这一段的原文是“This is the same as before, except this time we need to grab the cell from the table view using cellForRowAtIndexPath:.”，按照我的理解，滑动应该调用 `tableView:commitEditingStyle:forRowAtIndexPath:` ，这样才能执行我们新添加的语句。

编译并运行；滑动第一个 Cell，并点击 Delete。然后看看控制台的输出，找到最后一个递归描述，即第一个 Cell 的视图层次结构。你知道它是第一个 Cell ，因为它的 `text` 属性被设置为 `Item #1` 。你应该看到类型下面的打印：

```Objective-C
<UITableViewCell: 0xa816140; frame = (0 0; 320 44); text = 'Item #1'; autoresize = W; gestureRecognizers = <NSArray: 0x8b635d0>; layer = <CALayer: 0xa816310>>
   | <UITableViewCellScrollView: 0xa817070; frame = (0 0; 320 44); clipsToBounds = YES; autoresize = W+H; gestureRecognizers = <NSArray: 0xa8175e0>; layer = <CALayer: 0xa817260>; contentOffset: {82, 0}>
   |    | <UITableViewCellDeleteConfirmationView: 0x8b62d40; frame = (320 0; 82 44); layer = <CALayer: 0x8b62e20>>
   |    |    | <UITableViewCellDeleteConfirmationButton: 0x8b61b60; frame = (0 0; 82 43.5); opaque = NO; autoresize = LM; layer = <CALayer: 0x8b61c90>>
   |    |    |    | <UILabel: 0x8b61e60; frame = (15 11; 52 22); text = 'Delete'; clipsToBounds = YES; userInteractionEnabled = NO; layer = <CALayer: 0x8b61f00>>
   |    | <UITableViewCellContentView: 0xa816500; frame = (0 0; 287 43.5); opaque = NO; gestureRecognizers = <NSArray: 0xa817d40>; layer = <CALayer: 0xa8165b0>>
   |    |    | <UILabel: 0xa8167a0; frame = (15 0; 270 43.5); text = 'Item #1'; clipsToBounds = YES; layer = <CALayer: 0xa816840>>
   |    | <_UITableViewCellSeparatorView: 0x8a2b6e0; frame = (97 43.5; 305 0.5); layer = <CALayer: 0x8a2b790>>
   |    | <UIButton: 0xa8166a0; frame = (297 16; 8 12.5); opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0xa8092b0>>
   |    |    | <UIImageView: 0xa812d50; frame = (0 0; 8 12.5); clipsToBounds = YES; opaque = NO; userInteractionEnabled = NO; layer = <CALayer: 0xa8119c0>>
```

喔～ 看到 Delete 按钮了！在 Content View 下面， 有一个视图的类名为 `UITableViewCellDeleteConfirmationView` 。所那里就是 Delete 按钮被放置的位置。注意到它的 Frame 的 x 值是 320。这就意味着它被放置在 Scroll View 的最远端。但这个 Delete 按钮在你滑动时并没有移动。所以 Apple 必须在每次 Scroll View 滚动的同时移动这个 Delete 按钮。虽然这不是特别重要，但它很有趣！

现在回到 Cell。

你同样已经学了不少关于这个 Cell 如何工作的知识；亦即，那个 `UITableViewCellScrollView` ，它包含 contentView 和 Disclosure Indicator （以及 Delete 按钮，如果它被添加的话），明显是要做_某些事_ 。你可能已经从它的名字以及它是 `UIScrollView` 的子类而猜到了。

你可以通过在 `tableView:cellForRowAtIndexPath:` 下面添加一个简单的 `for` 循环来测试这个假设，就在 `recursiveDescription` 那一行下面：

```Objective-C
for (UIView *view in cell.subviews) {
  if ([view isKindOfClass:[UIScrollView class]]) {
    view.backgroundColor = [UIColor greenColor];
  }
}
```

再次编译并允许应用；绿色高亮确认了这个私有类确实是 `UIScrollView` 的子类，因为它覆盖了 Cell 里所有的紫色。

![Visible Scrollview](http://cdn4.raywenderlich.com/wp-content/uploads/2014/01/iOS-Simulator-Screen-shot-Dec-28-2013-8.53.52-PM-213x320.png)

回想刚才 `recursiveDescription` 输出的 log， `UITableViewCellScrollView` 的 Frame 和 Cell 本身的 Size 是一致的。

但是，这个视图到底有什么用？继续拖动 Cell 到左边，你就会看到 Scroll View 在你拖动 Cell 并 释放时提供了 “弹性（springy）”行为，如下所示：

![swipeable-demo](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/swipeable-demo.gif)

在你创建你自己的自定义 `UITableViewCell` 子类之前，还有一件事要注意，它出至 [UITableViewCell Class Reference](https://developer.apple.com/library/ios/documentation/uikit/reference/UITableViewCell_Class/Reference/Reference.html)：

>如果你想超越预定义样式，你可以添加子视图到 Cell 的 `contentView` 上。在添加子视图时，你自己要负责这些视图的位置以及设置它们的内容。

直白的说，就是，任何对 `UITableViewCell` 的自定义操作只能在 `contentView` 中进行。你不能将自己的视图加在 Cell 下面——而必须将它们加在 Cell 的 `contentView` 上。

这就意味着你将找出你自己的解决方案以便添加自定义按钮。但不要害怕，你可以很容易地复制出 Apple 所使用的方案。

## 可滑动 Table View Cell 的组成列表

这对你来说是什么意思？到了这里，你就有了一个组成列表来制造出一个 `UITableViewCell` 子类，以便放上你自定义的按钮。

我们从 View Stack 的最底部开始列出条目，你的列表如下：

1. `contentView` 是你的基础视图，因为你只能将子视图添加到它上面。
2. 在用户滑动后，任何你想显示的 `UIButon`。
3. 一个位于按钮之上的容器视图来装载你所有的内容。
4. 你可以使用一个 `UIScrollView` 来作为你的容器视图，就像 Apple 使用的，或者使用一个 `UIPanGestureRecognizer` 。这同样能够处理滑动去显示/隐藏按钮。你将在项目中采用后一种方案。
5. 最后，一个装有实际内容的视图。

还有一个可能不那么明显的成分：你必须确保系统提供的 `UIPanGestureRecognizer`  —— 它能让你滑动显示 Delete 按钮 —— 不可用。否则系统手势会和自定义手势冲突。

好消息是设置默认滑动手势不可用的操作相当简单。

打开 `MasterViewController.m` 修改 `tableView:canEditRowAtIndexPath:` 永远返回 `NO`，如下所示：

```Objective-C
- (BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath {
  return NO;
}
```

编译并运行；试着滑动某个 Cell ，你会发现你不能再滑动去删除了。

为了保持简单，你将使用两个按钮来走完这个教程。但同样的技术也可以再一个按钮上工作，或者超过两个按钮的情况——作为提醒，你可能需要执行一些本文没有涉及到的调整，如果你真的添加了多个按钮，你必须将整个 Cell 滑出才能看到所有的按钮。

## 创建一个自定义 Cell

你可以从基本视图和手势识别列表可以看到，在 Table View Cell 中有许多要做的事。你将创建一个自定义的 `UITableViewCell` 子类，以将所有的逻辑放在同一个地方。

去往 ` File\New\ File…` 并选择 `iOS\Cocoa Touch\Objective-C class` ，将新类命名为 `SwipeableCell` ，将它设置为 `UITableViewCell` 的子类 ，如下所示：

![Creating custom cell](http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2013-12-29-at-3.13.23-PM-475x320.jpg)

在 `SwipeableCell.m` 中设置下列类扩展和 `IBOutlet` ，就在 `#import` 语句后，`@implementation` 语句前：

```Objective-C
@interface SwipeableCell()
 
@property (nonatomic, weak) IBOutlet UIButton *button1;
@property (nonatomic, weak) IBOutlet UIButton *button2;
@property (nonatomic, weak) IBOutlet UIView *myContentView;
@property (nonatomic, weak) IBOutlet UILabel *myTextLabel;
 
@end
```

下一步，进入 Storyboard 选中  `UITableViewCell` 原型，如下所示：

![Select Table View Cell](http://cdn4.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2013-12-29-at-3.29.24-PM-367x320.jpg)

打开 Identity Inspector ，然后修改  Custom Class 为 `SwipeableCell` ，如下所示：

![Change Custom Class](http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2013-12-29-at-3.29.11-PM-252x320.jpg)

现在 `UITableViewCell` 原型的名字在左边的 Document Outline 上会显示为 “Swipeable Cell”。右键单击 `Swipeable Cell – Cell` ，你会看到一个你之前设置的 `IBOutlet` 列表：

![New Name and Outlets](http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2013-12-29-at-3.36.48-PM-420x320.jpg)

首先，你要在 Attributes Inspector 里修改两个地方以便自定义视图。设置 Style 为 `Custom`， Selection 为 `None`， Accessory 也为 `None`，截图如下：

![Reset Cell Items](http://cdn1.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2014-01-05-at-5.37.22-PM-282x320.jpg)

然后，拖两个按钮到 Cell 的 Content View 里。在视图的 Attributes Inspector 区设置每个按钮的背景色为比较鲜艳的颜色，并设置每个按钮的文字颜色为比较易读的颜色，这样你就可以清楚地看到按钮。

将第一个按钮放在右边，和 `contentView` 的上下边缘接触。将第二个按钮放在第一个按钮的左边缘处，也和 `contentView` 的上下边缘接触。当你做好后，Cell 看起来如下，可能颜色少有差异：

![Buttons Added to Prototype Cell](http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2013-12-29-at-4.08.29-PM-480x269.jpg)

接下来，将每个按钮和对应的 Outlet 关联起来。右键单击到可滑动Cell上打开它的 Outlets，然后将 button1 拖动到到右边的按钮， button2 拖动到左边的按钮，如下：

![swipeable-button1](http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/swipeable-button1.png)

你需要创建一个方法来处理对每个按钮的点击。

打开 `SwipeableCell.m` 添加如下方法：

```Objective-C
- (IBAction)buttonClicked:(id)sender {
  if (sender == self.button1) {
    NSLog(@"Clicked button 1!");
  } else if (sender == self.button2) {
    NSLog(@"Clicked button 2!");
  } else {
    NSLog(@"Clicked unknown button!");
  }
}
```

这个方法处理对两个按钮的点击，通过在控制台打印记录，你就能确定按钮被点击了。

再次打开 Storyboard ，将两个按钮都连接上 Action 。右键单击 `Swipeable Cell – Cell` 出现 Outlet 和 Action 的列表。从 `buttonClicked:` Action 拖动到你的按钮，如下：

![swipeable-buttonClicked](http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/swipeable-buttonClicked-480x253.png)

从事件列表中选择 `Touch Up Inside` ，如下所示：

 ![swipeable-touchupinside](http://cdn1.raywenderlich.com/wp-content/uploads/2014/01/swipeable-touchupinside.png)

重复上述步骤，用于第二个按钮。现在随便按照任何一个按钮上，都会调用 `buttonClicked:` 。

打开 `SwipeableCell.m` 添加如下属性：

```Objective-C
@property (nonatomic, strong) NSString *itemText;
```

稍后你将更多的和 `itemText` 打交道，但目前，这就是所有你要做的。

打开 `MasterViewController.m` 并在顶部添加如下一行：

```Objective-C
#import "SwipeableCell.h"
```

这将保证这个类知道你自定义的 Cell 子类。

替换 `tableView:cellForRowAtIndexPath:` 的内容为：

```Objective-C
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
  SwipeableCell *cell = [tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];
 
  NSString *item = _objects[indexPath.row];
  cell.itemText = item;
 
  return cell;
}
```

现在该使用你的新 Cell 而不是标准的 `UITableViewCell`。

编译并运行；你会看到如下界面：

![ALL THE BUTTONS!](http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/iOS-Simulator-Screen-shot-Dec-29-2013-4.43.46-PM-213x320.png)

### 添加一个 Delegate

欧耶～ 你的按钮已经出现了！如果你点击任何一个按钮，你都会在控制台看到合适的信息输出。然而，你不能指望 Cell 本身去处理任何直接的 Action 。

比如说，一个 Cell 不能 Present 其他的 View Controller 或直接将其 push 到 Navigation Stack 里。你必须要设置一个 Delegate 来传递按钮的点击事件回到 View Controller 中去处理那个事件。

打开 `SwipeableCell.h` 并在 `@interface` 之上添加如下 Delegate 协议：

```Objective-C
@protocol SwipeableCellDelegate <NSObject>
- (void)buttonOneActionForItemText:(NSString *)itemText;
- (void)buttonTwoActionForItemText:(NSString *)itemText;
@end
```

添加如下 Delegate 属性到 `SwipeableCell.h` ，就在 `itemText` 属性下面：

```Objective-C
@property (nonatomic, weak) id <SwipeableCellDelegate> delegate;
```

更新 `SwipeableCell.m` 中的 `buttonClicked:` 为如下所示：

```Objective-C
- (IBAction)buttonClicked:(id)sender {
  if (sender == self.button1) {
    [self.delegate buttonOneActionForItemText:self.itemText];
  } else if (sender == self.button2) {
    [self.delegate buttonTwoActionForItemText:self.itemText];
  } else {
    NSLog(@"Clicked unknown button!");
  }
}
```

这个更新使得这个方法去调用合适的 Delegate 方法，而不仅仅是打印一句 log。

现在打开 `MasterViewController.m` 并添加如下 delegate 方法：

```Objective-C
#pragma mark - SwipeableCellDelegate
- (void)buttonOneActionForItemText:(NSString *)itemText {
  NSLog(@"In the delegate, Clicked button one for %@", itemText);
}
 
- (void)buttonTwoActionForItemText:(NSString *)itemText {
  NSLog(@"In the delegate, Clicked button two for %@", itemText);
}
```

这个方法目前还是简单的打印到控制台，以确保一切传递都工作正常。

接下来，添加如下协议到 `MasterViewController.m` 顶部的类扩展上以符合协议申明：

```Objective-C
@interface MasterViewController () <SwipeableCellDelegate> {
  NSMutableArray *_objects;
}
@end
```

这只是简单地确认这个类会实现 `SwipeableCellDelegate` 协议。

最后，你要设置这个 View Controller 为 Cell 的 delegate。

添加如下语句到 `tableView:cellForRowAtIndexPath:` ，就在最后的 return 语句之前：

```Objective-C
cell.delegate = self;
```

编译并运行；当你点击按钮时，你就会看到合适的“In the delegate”消息。

### 为按钮添加 Action

如果你看到log消息很很高兴了，也可以跳过下一节。然而，如果你喜欢更加实在的东西，你可以添加一些处理，这样当 delegate 方法被调用时，你就可以显示已经引入的 `DetailViewController` 。

添加如下两个方法到 `MasterViewController.m`：

```Objective-C
- (void)showDetailWithText:(NSString *)detailText
{
  //1
  UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@"Main" bundle:nil];
  DetailViewController *detail = [storyboard instantiateViewControllerWithIdentifier:@"DetailViewController"];
  detail.title = @"In the delegate!";
  detail.detailItem = detailText;
 
  //2
  UINavigationController *navController = [[UINavigationController alloc] initWithRootViewController:detail];
 
  //3
  UIBarButtonItem *done = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemDone target:self action:@selector(closeModal)];
  [detail.navigationItem setRightBarButtonItem:done];
 
  [self presentViewController:navController animated:YES completion:nil];
}
 
//4
- (void)closeModal
{
  [self dismissViewControllerAnimated:YES completion:nil];
}
```

在上面的代码里，你执行了四个操作：

1. 从 Storyboard 里取出 Detail View Controller 并设置其 title 和 detailItem 。
2. 设置一个 `UINavigationController` 作为包含 Detail View Controller 的容器，并给你放置 close 按钮的地方。
3. 添加 close 按钮，关联 `MasterViewController` 里的一个 Action。
4. 设置这个 Action 的响应方法，它将 dismiss 任何以 Modal 方式显示 View Controller

接下来，用下列版本替换你之前添加的两个方法：

```Objective-C
- (void)buttonOneActionForItemText:(NSString *)itemText
{
  [self showDetailWithText:[NSString stringWithFormat:@"Clicked button one for %@", itemText]];
}
 
- (void)buttonTwoActionForItemText:(NSString *)itemText
{
  [self showDetailWithText:[NSString stringWithFormat:@"Clicked button two for %@", itemText]];
}
```

最后，打开 `Main.storyboard` 并选中 `Detail View Controller` 。找到 Identity Inspector 并设置 `Storyboard ID` 为 `DetailViewController` 以匹配类名，如下所示：

![Add Storyboard Identifier](http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2013-12-29-at-5.55.54-PM-450x320.jpg)

如果你忘了这一步， `instantiateViewControllerWithIdentifier` 将会因为不合法的参数而 Crash，其异常表示具有这个标识符的 View Controller 并不存在。

编译并运行；点击某个 Cell 中的按钮，然后看着 Modal View Controller 出现，如下面的截图所示：

![View Launched from Delegate](http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/iOS-Simulator-Screen-shot-Dec-30-2013-10.27.30-PM-213x320.png)

## 添加顶层视图并添加滑动 Action

现在你到了视图工作的后段部分，是时候让顶层部分启动并运行起来了。

打开 `Main.storyboard` 并拖一个 `UIView` 到 `SwipeableTableCell` 上，这个视图将占据整个 Cell 的高和宽，并覆盖按钮，所以在Swipe手势能工作之前，你不会再看到它们了。

如果你要精确地控制，打开 Size Inspector 并设置这个视图地宽和高，分别为 320 和 43：

![swipeable-320-43](http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/swipeable-320-43-480x305.png)

你同样需要一个约束来将视图钉在 contentView 的边缘。选中视图并点击 `Pin` 按钮，选择所有四个间隔约束并设置它们的值为 0 ，如下所示：

![swipeable-constraint](http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/swipeable-constraint-286x320.png)

连接好这个视图的 Outlet，按照之前介绍的步骤：在左边的导航器里右键单击这个可滑动 Cell 并拖动 `myContentView` 到这个新的视图上。

下一步，拖动一个 `UILabel` 到视图里；设置其距离左边 20 点，并设置其垂直剧中。再将其连接到 `myTextLabel` Outlet 上。

编译并运行；你的 Cell 看起来有正常了：

![Back to cells](http://cdn4.raywenderlich.com/wp-content/uploads/2014/01/iOS-Simulator-Screen-shot-Jan-1-2014-3.44.29-PM-213x320.png)

### 添加数据

但为何实际的文本数据没有显示出来？那是因为你只是设置了 `itemText` 属性，而没有做会影响 `myTextLabel` 的事情。

打开 `SwipeableCell.m` 并添加如下方法：

```Objective-C
- (void)setItemText:(NSString *)itemText {
  //Update the instance variable
  _itemText = itemText;
 
  //Set the text to the custom label.
  self.myTextLabel.text = _itemText;
}
```

这个方法覆写了 `itemText` 属性的 setter 方法。除了更新后面的实例变量，它还会更新可见的 Label。

最后，为了让接下来的几步的结果更易看到，你将把 item 的 title 变长一点，以便在 Cell 滑动后依然有一些文本可见。

转到 `MasterViewController.m` 并更新 `viewDidLoad` 中的这一行，这是 item title 生成的地方：

```Objective-C
NSString *item = [NSString stringWithFormat:@"Longer Title Item #%d", i];
```

编译并运行；你就会看到合适的 item title 显示如下：

![Longer Item Titles displayed in custom label](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/iOS-Simulator-Screen-shot-Jan-1-2014-4.12.32-PM-213x320.png)

### 手势识别——GO！

终于到了“有趣的”部分——将数学、约束以及手势识别搅和在一起，以方便地处理滑动操作。

首先，在 `SwipeableCell` 的类扩展里添加如下这些属性：

```Objective-C
@property (nonatomic, strong) UIPanGestureRecognizer *panRecognizer;
@property (nonatomic, assign) CGPoint panStartPoint;
@property (nonatomic, assign) CGFloat startingRightLayoutConstraintConstant;
@property (nonatomic, weak) IBOutlet NSLayoutConstraint *contentViewRightConstraint;
@property (nonatomic, weak) IBOutlet NSLayoutConstraint *contentViewLeftConstraint;
```

关于你所要做的事情，简短版本是这样的：记录一个 Pan 手势并调整你的View的左右约束，根据 a) 用户将 Cell Pan 了多远 b) Cell 在何处以及合适开始移动。

为了做到这一点，你首先要将这个 IBOutlet 连接到 `myContentView` 的左右约束上。这两个约束将视图 钉在 Cell 的 `contentView` 中。

通过打开约束列表，你可以找出这两个约束。通过检查每个约束在 Cell 上的高亮你就能找到那合适的两个。在这个例子中，是 `contentView` 右边和 `contentView` 之间的约束，如下所示：

![Highlighting Constraints](http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2014-01-01-at-5.19.25-PM-480x316.jpg)

一旦你定位到合适的约束，就将其连接到合适的 Outlet 上——在本例中，是 `contentViewRightConstraint` ，如下图所示：

![Hook Up Constraint to IBOutlet](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2014-01-01-at-5.19.12-PM-480x272.jpg)

遵循同样的步骤，连接好 `contentViewLeftConstraint` ，它代表 `contentView` 左边和 `contentView` 之间的约束。

下一步，打开 `SwipeableCell.m` 并修改 `@interface` 语句的类扩展，添加 `UIGestureRecognizerDelegate` 协议：

```Objective-C
@interface SwipeableCell() <UIGestureRecognizerDelegate>
```

然后，依然在  `SwipeableCell.m` 里，添加如下方法：

```Objective-C
- (void)awakeFromNib {
  [super awakeFromNib];
 
  self.panRecognizer = [[UIPanGestureRecognizer alloc] initWithTarget:self action:@selector(panThisCell:)];
  self.panRecognizer.delegate = self;
  [self.myContentView addGestureRecognizer:self.panRecognizer];
}
```

这里设置了 Pan 手势并将其添加到 Cell 上：

再添加如下方法：

```Objective-C
- (void)panThisCell:(UIPanGestureRecognizer *)recognizer {
  switch (recognizer.state) {
    case UIGestureRecognizerStateBegan:
      self.panStartPoint = [recognizer translationInView:self.myContentView];
      NSLog(@"Pan Began at %@", NSStringFromCGPoint(self.panStartPoint));
      break;
    case UIGestureRecognizerStateChanged: {
      CGPoint currentPoint = [recognizer translationInView:self.myContentView];
      CGFloat deltaX = currentPoint.x - self.panStartPoint.x;
      NSLog(@"Pan Moved %f", deltaX);
    }
      break;
    case UIGestureRecognizerStateEnded:
      NSLog(@"Pan Ended");
      break;
    case UIGestureRecognizerStateCancelled:
      NSLog(@"Pan Cancelled");
      break;
    default:
      break;
  }
}
```

这个方法会在 Pan 手势识别器发动时执行，暂时，它只简单地打印 Pan 手势的细节。

编译并运行；用手指拖动 Cell ，你就会看到如下log记录了移动信息：

![Pan Logs](http://cdn1.raywenderlich.com/wp-content/uploads/2014/01/Screen-Shot-2014-01-01-at-7.08.21-PM-195x320.jpg)

如果你往初始点的右边滑动，你会看到正数，往初始点的左边滑动就会看到负数。这些数字将用于调整 `myContentView` 的约束。

### 移动这些约束

从本质上将，你需要通过调整将 Cell 的 `contentView` 钉住的左、右约束来推动 `myContentView` 到左边。右约束将会接受一个正值，而左约束将接受一个绝对值相等的负值。

举例来说，如果 `myContentView` 需要往左移动 5 点，那么 右约束将会接受的值是 5，而左约束将接受的值是 -5 。这将会将整个视图往左边滑动 5 点，而不会改变他的宽度。

听起来蛮容易的——但还有许多移动相关的事情要注意。根据 Cell 是否已经打开和用户 Pan 的方向，你要处理不同的一大把事情。

你同样需要知道 Cell 最远可以滑动多远。你将通过计算被按钮覆盖的区域的宽度来确定这一点。最简单的方法是用视图的整个宽度减去最左边的按钮的最小 X 位置。

为了阐明，下面来个 sneak peek ，以明确的图示表明你所要关注的方面：

![Minimum x of button 2](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/width-min-x-480x254.png)

幸好，感谢 [`CGRect` CGGeometry 函数](https://developer.apple.com/library/mac/documentation/graphicsimaging/reference/CGGeometry/Reference/reference.html) ，这些很容易被转换为代码：

添加如下方法到 `SwipeableCell.m` ：

```Objective-C
- (CGFloat)buttonTotalWidth {
    return CGRectGetWidth(self.frame) - CGRectGetMinX(self.button2.frame);
}
```

添加如下两个骨架方法到 `SwipeableCell.m` ：

```Objective-C
- (void)resetConstraintContstantsToZero:(BOOL)animated notifyDelegateDidClose:(BOOL)endEditing
{
	//TODO: Build.
}
 
- (void)setConstraintsToShowAllButtons:(BOOL)animated notifyDelegateDidOpen:(BOOL)notifyDelegate
{
	//TODO: Build
}
```

这两个骨架方法——一旦你填上血肉——将 snap 打开 Cell 并 snap 关闭 Cell。在你对 pan 手势识别起添加更多处理后，你会回到这两个方法。

替换 `panThisCell:` 中的 `UIGestureRecognizerStateBegan` case 为下列代码：

```Objective-C
case UIGestureRecognizerStateBegan:
  self.panStartPoint = [recognizer translationInView:self.myContentView];	           
  self.startingRightLayoutConstraintConstant = self.contentViewRightConstraint.constant;
  break;
```

你需要存储 Cell 的初始位置（例如，约束值）以确定 Cell 是要打开还是关闭。

下一步你需要添加更多处理以应对 pan 手势识别器的改变。还是在 `panThisCell:` 里，修改 `UIGestureRecognizerStateChanged` case ，如下所示：

```Objective-C
case UIGestureRecognizerStateChanged: { 
  CGPoint currentPoint = [recognizer translationInView:self.myContentView];
  CGFloat deltaX = currentPoint.x - self.panStartPoint.x;
  BOOL panningLeft = NO; 
  if (currentPoint.x < self.panStartPoint.x) {  //1
    panningLeft = YES;
  }
 
  if (self.startingRightLayoutConstraintConstant == 0) { //2
    //The cell was closed and is now opening
    if (!panningLeft) {
      CGFloat constant = MAX(-deltaX, 0); //3
      if (constant == 0) { //4
        [self resetConstraintContstantsToZero:YES notifyDelegateDidClose:NO];
      } else { //5
        self.contentViewRightConstraint.constant = constant;
      }
    } else {
      CGFloat constant = MIN(-deltaX, [self buttonTotalWidth]); //6
      if (constant == [self buttonTotalWidth]) { //7
        [self setConstraintsToShowAllButtons:YES notifyDelegateDidOpen:NO];
      } else { //8
        self.contentViewRightConstraint.constant = constant;
      }
    }
  }
```

上面大部分代码都在 Cell 默认的“关闭”状态下 处理pan手势识别器，下面是细节说明：

1. 判断 pan 手势是往左还是往右。
2. 如果右约束常量为 0 ，意味着 `myContentView` 完全挡住 `contentView` 。因此 Cell 在这里一定已经关闭，而用户准备打开它。
3. 这是处理用户从做到右滑动以关闭 Cell 的 情况。除了说“你不能做那个”之外，你还要处理的情况是，当用户滑动 Cell 只打开一点点，然后他们希望不必抬起他们的手指来结束此手势就可以滑动它关闭。译者注：就是说，打开一点点不会完全显示出后面的按钮，Cell 会自动关闭。
 
 因为一个从左到右的滑动会导致 `deltaX` 为正值，而从右到左的滑动回到导致 `deltaX` 为负值，你必须根据负的 `deltaX` 计算出常量以设置到右约束上。因为是从它与0中找出最大值，所以视图不可能往右边走多远。
4. 如果常量为 0，Cell 就是完全关闭的。调用处理关闭的方法——它（如你回忆起的）在目前还什么也不会做。
5. 如果常量为不为 0，那么你就将其设置到右手边的约束上。
6. 否者，如果是从右往做滑动，那么用户试图打开 Cell 。这在个情况里，常量将会小于负`deltaX`或两个按钮的宽度之和。
7. 如果目标常量是两个按钮的宽度之和，那么 Cell 就被打开至捕捉点（catch point），你应该调用方法来处理这个打开状态。
8. 如果常量不是两个按钮的宽度之和，那就将其设置到右约束上。

哟！处理得真不少… 而这个只是处理了 Cell 已经关闭得情况。你现在还要编写代码处理当手势开始时 Cell 就已经部分开启的情况。

就在刚在添加的代码之下添加如下代码：

```Objective-C
  else {
    //The cell was at least partially open.
    CGFloat adjustment = self.startingRightLayoutConstraintConstant - deltaX; //1
    if (!panningLeft) {
      CGFloat constant = MAX(adjustment, 0); //2
      if (constant == 0) { //3
        [self resetConstraintContstantsToZero:YES notifyDelegateDidClose:NO];
      } else { //4
        self.contentViewRightConstraint.constant = constant;
      }
    } else {
      CGFloat constant = MIN(adjustment, [self buttonTotalWidth]); //5
      if (constant == [self buttonTotalWidth]) { //6
        [self setConstraintsToShowAllButtons:YES notifyDelegateDidOpen:NO];
      } else { //7
        self.contentViewRightConstraint.constant = constant;
      }
    }
  }
 
  self.contentViewLeftConstraint.constant = -self.contentViewRightConstraint.constant; //8
}
    break;
```

这是 if 语句的后半段。因此它用于处理 Cell 原本就打开的情况。

再一次，下面说明你要处理的几个情况：

1. 在这个情况下，你只是接受 `deltaX` ，你就用 rightLayoutConstraint 的原始位置减去 `deltaX` 以便得知要做多少调整。
2. 如果用户从做往右滑动，你必须接受 adjustment 与 0 中的较大值。如果 adjustment 已变成负值，那就说明用户已经把 Cell 滑到边界之外了，Cell 就关闭了，这就让你进入下一个情况。
3. 如果常量为 0，那么 Cell 已经关闭，你就调用处理其关闭的方法。
4. 否则，将常量设置到右约束上。
5. 对于从右到左的滑动，你将接受 adjustment 与 两个按钮宽度之和 中的较小值。如果 adjustment 更大，那就表示用户已经滑出超过捕捉点了。
6. 如果常量刚好等于两个按钮宽度之和，那么 Cell 就打开了，你必须调用处理 Cell 打开的方法。
7. 否则，将常量设置到右约束上。
8. 现在，你已经处理完“Cell关闭”和“Cell部分开启”的情况，在这两个情况里，你都可对左约束做同样的事情：将其设置为右约束常量的负值。这就保证了 `myContentView` 的宽度一直保持不变。

编译并运行；现在你可以来回滑动 Cell ！它不是非常流畅，而且它在你希望的地方之前的一点就停下了。这是因为你还没有真正实现那两个用于处理打开和关闭 Cell 的方法。

>Note：你可以也注意到，Table View 本身已经不会 scroll 了。不要担心，一旦你正确处理好 Cell 的滑动，你就能修复它。

### Snap!

接下来，你要让 Cell Snao 进入合适的位置。你会注意到，如果你放手 Cell 会停到合适的位置。

在你进入方法开始处理之前，你需要一个单独的生成动画的方法。

打开 `SwipeableCell.m` 并添加如下方法：

```Objective-C
- (void)updateConstraintsIfNeeded:(BOOL)animated completion:(void (^)(BOOL finished))completion {
  float duration = 0;
  if (animated) {
    duration = 0.1;
  }
 
  [UIView animateWithDuration:duration delay:0 options:UIViewAnimationOptionCurveEaseOut animations:^{
    [self layoutIfNeeded];
  } completion:completion];
}
```

>Note：0.1 秒的间隔和 ease-out curve  动画都是我从实践和错误中总结出来的。如果你找到其他更让你看着愉悦的速度或动画类型，可以自由修改它们。

接下来，你将填充那两个处理打开和关闭的骨架方法。记得在 Apple 的原始实现里，因为使用了 `UIScrollView` 子类作为最底层的试图，所以会有一点弹性。

要让事情看起来正确，你将在 Cell 撞到边界时给它一点弹性。你同样要确保 `contentView` 和 `myContentView` 有同样的 `backgroundColor` 以造成弹性非常顺滑的错觉。

添加如下常量到 `SwipeableCell.m` 顶部，就在 import 语句之下：

```Objective-C
static CGFloat const kBounceValue = 20.0f;
```

这个常量存储了弹性值，将用于你的弹性动画中。

如下更新 `setConstraintsToShowAllButtons:notifyDelegateDidOpen:` ：

```Objective-C
- (void)setConstraintsToShowAllButtons:(BOOL)animated notifyDelegateDidOpen:(BOOL)notifyDelegate {
  //TODO: Notify delegate.
 
  //1
  if (self.startingRightLayoutConstraintConstant == [self buttonTotalWidth] &&
      self.contentViewRightConstraint.constant == [self buttonTotalWidth]) {
    return;
  }
  //2
  self.contentViewLeftConstraint.constant = -[self buttonTotalWidth] - kBounceValue;
  self.contentViewRightConstraint.constant = [self buttonTotalWidth] + kBounceValue;
 
  [self updateConstraintsIfNeeded:animated completion:^(BOOL finished) {
    //3
    self.contentViewLeftConstraint.constant = -[self buttonTotalWidth];
    self.contentViewRightConstraint.constant = [self buttonTotalWidth];
 
    [self updateConstraintsIfNeeded:animated completion:^(BOOL finished) {
      //4
      self.startingRightLayoutConstraintConstant = self.contentViewRightConstraint.constant;
    }];
  }];
}
```

这个方法在 Cell 完全打开时执行。下面解释发生了什么：

1. 如果 Cell 已经开启，约束已经到达完全开启值，那就返回——否则弹性操作将会一次又一次的发生，就像你继续滑动超过总按钮宽度那样。
2. 你初始设置约束值为按钮总宽度和弹性值的结合值，它将 Cell 拉到左边一点点，这样才好 snap 回来。然后你就调用动画来实现这个设置。
3. 当第一个动画完成，发动第二个动画，它将 Cell 正好打开在从按钮宽度的位置。
4. 当第二个动画完成，重设起始约束否则你会看到多次弹跳。

如下更新 `resetConstraintContstantsToZero:notifyDelegateDidClose:` ：

```Objective-C
- (void)resetConstraintContstantsToZero:(BOOL)animated notifyDelegateDidClose:(BOOL)notifyDelegate {
  //TODO: Notify delegate.
 
  if (self.startingRightLayoutConstraintConstant == 0 &&
      self.contentViewRightConstraint.constant == 0) {
    //Already all the way closed, no bounce necessary
    return;
  }
 
  self.contentViewRightConstraint.constant = -kBounceValue;
  self.contentViewLeftConstraint.constant = kBounceValue;
 
  [self updateConstraintsIfNeeded:animated completion:^(BOOL finished) {
    self.contentViewRightConstraint.constant = 0;
    self.contentViewLeftConstraint.constant = 0;
 
    [self updateConstraintsIfNeeded:animated completion:^(BOOL finished) {
      self.startingRightLayoutConstraintConstant = self.contentViewRightConstraint.constant;
    }];
  }];
}
```

如你所见，这类似于 `setConstraintsToShowAllButtons:notifyDelegateDidOpen:` ，但它的逻辑是关闭 Cell 而不是打开。

编译并运行；随意滑动 Cell 到它的捕捉点，你就会在放手时看到弹性行为。

然而，如果你在 Cell 完全开启或完全关闭之前将释放手指，它将会卡在中间。Whoops! 你还没有处理触摸结束或被取消的情况。

找到 `panThisCell:` 用下列代码替换 `UIGestureRecognizerStateEnded` case ：

```Objective-C
case UIGestureRecognizerStateEnded:
  if (self.startingRightLayoutConstraintConstant == 0) { //1
    //Cell was opening
    CGFloat halfOfButtonOne = CGRectGetWidth(self.button1.frame) / 2; //2
    if (self.contentViewRightConstraint.constant >= halfOfButtonOne) { //3
      //Open all the way
      [self setConstraintsToShowAllButtons:YES notifyDelegateDidOpen:YES];
    } else {
      //Re-close
      [self resetConstraintContstantsToZero:YES notifyDelegateDidClose:YES];
    }
  } else {
    //Cell was closing
    CGFloat buttonOnePlusHalfOfButton2 = CGRectGetWidth(self.button1.frame) + (CGRectGetWidth(self.button2.frame) / 2); //4
    if (self.contentViewRightConstraint.constant >= buttonOnePlusHalfOfButton2) { //5
      //Re-open all the way
      [self setConstraintsToShowAllButtons:YES notifyDelegateDidOpen:YES];
    } else {
      //Close
      [self resetConstraintContstantsToZero:YES notifyDelegateDidClose:YES];
    }
  }
  break;
```

在这里，你根据 Cell 是否已经打开或关闭以及手势结束时 Cell 的位置在执行不同的处理。具体来讲：

1. 通过检查开始右约束值，得知手势开始时 Cell 是否已经打开或关闭。
2. 如果 Cell 是关闭的，那你就正在打开它，你要让 Cell 自动滑动到打开，至少需要先滑动右边按钮(self.button1)一半的宽度。因为你在测量约束的常量，你只需要计算实际的按钮宽度，而不是它在视图中的 X 位置。
3. 接下来，测试约束是否已被打开至超过你希望让 Cell 自动打开的点。如果已经超过，那就自动打开 Cell。如果没有，那就自动关闭 Cell。
4. 此处表示 Cell 从打开的状态开始，你需要那个能让 Cell 自动 snap 关闭的点，至少需要超过最左边按钮的一半。 将不是最左边的按钮的那些按钮的宽度加起来，在这个情况里，只有 self.button1 而已，再加上最左边按钮的一半——也就是 self.button2 —— 以便找到需要的检查点。
5. 测试约束是否以及超过这个点，即你希望 Cell 自动关闭的那个点。如果超过了，关闭 Cell。如果没有，那就重新打开 Cell。

最后，你还要处理一下手势被取消的情况。用如下代码替换 `UIGestureRecognizerStateCancelled` case ：

```Objective-C
case UIGestureRecognizerStateCancelled:
  if (self.startingRightLayoutConstraintConstant == 0) {
    //Cell was closed - reset everything to 0
    [self resetConstraintContstantsToZero:YES notifyDelegateDidClose:YES];
  } else {
    //Cell was open - reset to the open state
    [self setConstraintsToShowAllButtons:YES notifyDelegateDidOpen:YES];
  }
  break;
```

这个处理相当直白；由于用户取消了触摸，表示他们不想改变 Cell 当前的状态，所以你只需要将一切都设置为它们原本的样子即可。

编译并运行；滑动 Cell ，你会看到 Cell Snap 到打开或关闭，而不论你的手指再哪里，如下所示：

![swipeable-bounce](http://cdn4.raywenderlich.com/wp-content/uploads/2014/01/swipeable-bounce.gif)

## 更好地处理 Table View

在最终完成前，只有少数几步了！

首先，你的 `UIPanGestureRecognizer` 有时候会影响 `UITableView` 的 Scroll 操作。由于你已经设置了 Cell 的 Pan 手势识别器 的 `UIGestureRecognizerDelegate` ，你只需要实现一个（有些滑稽且冗长命名的） delegate 方法即可将一切恢复正常。

添加如下方法到 `SwipeableCell.m` ：

```Objective-C
#pragma mark - UIGestureRecognizerDelegate
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
{
   return YES;
}
```

这个方法告知各手势识别器，它们可以同时工作。

编译并运行；打开第一个 Cell 然后你依然可以 Scroll tableView 。

还有一个 Cell 重用引起的小问题：各个行不记得它们的状态，看起来是因为 Cell 重用了它们的视图的 开启/关闭 状态，然后它们的视图就不能正确反应用户的操作了。要查看这一情况，打开一个 Cell ，然后将 Table  Scroll 一点点。你就会注意每次都有一个 Cell 始终保持打开状态，但每次都不同。

要修复这个问题头一半，添加如下方法到 `SwipeableCell.m` ：

```Objective-C
- (void)prepareForReuse {
  [super prepareForReuse];
  [self resetConstraintContstantsToZero:NO notifyDelegateDidClose:NO];
}
```

这个方法确保 Cell 在其回收重利用时再次关闭。

要解决这个问题的后一半，你将添加一个公共方法给 Cell 以促使其打开。然后你会添加一些 delegate 方法以允许 `MasterViewController` 去管理那个 Cell 是打开的。

打开 `SwipeableCell.h` 。在 `SwipeableCellDelegate` 协议的申明里，添加如下两个新的方法，就在已存在的那两个下面：

```Objective-C
- (void)cellDidOpen:(UITableViewCell *)cell;
- (void)cellDidClose:(UITableViewCell *)cell;
```

这些方法将会通知 delegate —— 在你的情况里，就是 Master View Controller —— 某个 Cell 被打开或关闭了。

添加如下公共方法申明到 `SwipeableCell` 的 `@interface` 里：

```Objective-C
- (void)openCell;
```

接下来，打开 `SwipeableCell.m` 并添加 `openCell` 的实现：

```Objective-C
- (void)openCell {
  [self setConstraintsToShowAllButtons:NO notifyDelegateDidOpen:NO];
}
```

这个方法允许 delegate 修改 Cell 的状态。

依然在用一个文件里，找到 `resetConstraintsToZero:notifyDelegateDidOpen:` 并替换其中 `TODO` 为如下代码：

```Objective-C
if (notifyDelegate) {
  [self.delegate cellDidClose:self];
}
```

接下来，找到 `setConstraintsToShowAllButtons:notifyDelegateDidClose:` 并替换其中 `TODO` 为如下代码：

```Objective-C
if (notifyDelegate) {
  [self.delegate cellDidOpen:self];
}
```

这两个修改会在一个 swipe 手势完成时通知 delegate ，无论 Cell 是否以及打开或关闭。

添加如下属性申明到 `MasterViewController.m` 顶部的类扩展里：

```Objective-C
@property (nonatomic, strong) NSMutableSet *cellsCurrentlyEditing;
```

它将存储当前已被打开的 Cell 的列表。

添加如下代码到 `viewDidLoad` 的最后：

```Objective-C
self.cellsCurrentlyEditing = [NSMutableSet new];
```

这个初始化保证了之后你可以正常使用数组。

现在在同一个文件里添加如下方法实现：

```Objective-C
- (void)cellDidOpen:(UITableViewCell *)cell {
  NSIndexPath *currentEditingIndexPath = [self.tableView indexPathForCell:cell];
  [self.cellsCurrentlyEditing addObject:currentEditingIndexPath];
}
 
- (void)cellDidClose:(UITableViewCell *)cell {
  [self.cellsCurrentlyEditing removeObject:[self.tableView indexPathForCell:cell]];
}
```

注意到你添加的时 Index Path 而不是 Cell 本身到列表里。如果你直接添加 Cell 对象，那么之后你就会看到同样的问题，在 Cell 被回收后再次被打开。用了这个方法，你就可以使用合适 的 Index Path 来打开 Cell 了。

最后，添加下面几行到 `tableView:cellForRowAtIndexPath:` ，就在 return 语句之前：

```Objective-C
if ([self.cellsCurrentlyEditing containsObject:indexPath]) {
  [cell openCell];
}
```

如果当前的 Cell 的 Index Path 在列表里，它就会将其设置为打开。

编译并运行；全都搞定了！你现在有了一个能够 Scroll 的 Table View，还能处理 Cell 的打开和关闭状态，并在 Cell 的任意被点击时，使用 delegate 方法来加载任何任务。

## 下一步怎么走？

译者注：吐血，终于翻译到这一句了！

最终的项目可以在[此处](http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/SwipeableTableCell.zip)下载。我还会继续我在此所开发的东西，并组成一个开源项目，以便让事情更有灵活性——在准备好推出时，我会在论坛里贴个链接。

任何时候，如你在不知道他们如何做到的情况下复制出 Apple 所做的某些效果，你都会发现有许多许多的方式去做到这样的效果。所以这里的方案只是这个效果的实现办法之一；然而，它是我所发现的唯一一个不需要处理嵌套 Scroll View 的办法，产生的手势识别冲突也可以非常简单地解决！ :]

写这篇文章时有一些很有用的资源，但文章里最终使用了非常不同的办法。这些资源是 [Ash Furrow 的文章](http://www.teehanlax.com/blog/reproducing-the-ios-7-mail-apps-interface/) 能让一切都工作起来，以及 [Massimiliano Bigatti’s BMXSwipeableCell](https://github.com/mbigatti/BMXSwipableCell) 项目，它现实通过 `UIScrollView` 这条路可以挖到多深。

如果你有任何建议、问题或相关的代码，请在评论区讲出來吧！


===============

译者注：欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条微博 [http://weibo.com/2076580237/B1vqCrWfe](http://weibo.com/2076580237/B1vqCrWfe) 以分享给更多人！

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳译者的幸苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/blob/master/images/nixzhu_alipay.png)

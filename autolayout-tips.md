# AutoLayout Tips

一些值得记录的 AutoLayout 用法。如无意外，作者是：[@nixzhu](https://twitter.com/nixzhu)

## Tip 1：两个不等宽的 View，彼此相邻，并“共同”居中于 Superview

文字可能不好描述，那就来图片：

![AutoLayout Tip 1](https://github.com/nixzhu/dev-blog/raw/master/images/autolayout-tip1.png)

如上图所示，黑色的是 Superview，它有两个 Subview，一个是 imageView，一个是 label。

imageView 和 label 相邻，且“它们的组合”居中于 Superview。label 的宽度是可变的，左右两个示意图非常清楚。

那怎样用 AutoLayout 的约束来描述这样的布局呢？

首先来处理两个 Subview 的“相邻”：

```Objective-C
NSArray *constraintH = [NSLayoutConstraint constraintsWithVisualFormat:@"H:|-(>=0)-[imageView]-[label]-(>=0)-|"
                                                                   options:0
                                                                   metrics:nil
                                                                     views:viewsDictionary];
```

这个约束集描述了 imageView 和 label 相邻，而且，imageView 的左边相对于 Superview 的距离 >=0 ，即是可变的；同样，label 的右边相对于 Superview 的距离也 >= 0，一样可变。但若此时运行代码，它们并不会居中于 Superview，可能都挤在右边或左边。

我们使用一个 helperView 来处理“居中”问题。

```Objective-C
NSLayoutConstraint *constraint3 =[NSLayoutConstraint constraintWithItem:helperView
                                                              attribute:NSLayoutAttributeLeft
                                                              relatedBy:NSLayoutRelationEqual
                                                                 toItem:self.imageView
                                                              attribute:NSLayoutAttributeLeft
                                                             multiplier:1
                                                               constant:0];

NSLayoutConstraint *constraint4 =[NSLayoutConstraint constraintWithItem:helperView
                                                              attribute:NSLayoutAttributeRight
                                                              relatedBy:NSLayoutRelationEqual
                                                                 toItem:self.label
                                                              attribute:NSLayoutAttributeRight
                                                             multiplier:1
                                                               constant:0];

```

helperView 的左边相对于 imageView 的左边对齐，helperView 的右边相对于 label 的右边对齐。最后重点来了：

```Objective-C
NSLayoutConstraint *constraint5 =[NSLayoutConstraint constraintWithItem:helperView
                                                              attribute:NSLayoutAttributeCenterX
                                                              relatedBy:NSLayoutRelationEqual
                                                                 toItem:self.view // Superview
                                                              attribute:NSLayoutAttributeCenterX
                                                             multiplier:1
                                                               constant:0];
```

我们让 helperView 的 X 中心与 Superview 的 X 中心一致。于是，伟大的约束会让 helperView 横向居中于 Superview，虽然我们看不见它，但它会将 imageView 和 label 拉到“共同”居中的位置。

搞定！（注意上面的代码省略了一些竖向居中的约束。）虽然是用代码描述的，但是文字说明很清楚了，直接在 IB 里构建约束也没有问题。关键就是我们的 helperView，它虽不可见，但只要它的约束起作用就可以了。

我也写了一个 [Demo 放在 GitHub](https://github.com/nixzhu/CenterTwoViewsUseAutoLayout)，如有必要，请稍微看看！
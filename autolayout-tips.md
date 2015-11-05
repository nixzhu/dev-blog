# AutoLayout Tips

一些值得记录的 AutoLayout 用法。如无意外，作者是：[@nixzhu](https://twitter.com/nixzhu)

===============================

- [Tip 1：两个不等宽的 View，彼此相邻，并“共同”居中于 Superview](#tip-1)
- [Tip 2：让 AutoLayout 与 UIScrollView 合作无间](#tip-2)

===============================

## Tip 1

**两个不等宽的 View，彼此相邻，并“共同”居中于 Superview**

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


## Tip 2

**让 AutoLayout 与 UIScrollView 合作无间**

只要度过了最开始的不适应期，各位用着 AutoLayout 时应该都是心情愉悦的。虽然手写约束会给人冗长的感觉，但 API 的长度并不会阻碍你对代码的理解。不过大部分时间里，我们都在 Storyboard 里，愉快的链接着约束，日子美好又幸福。

可惜好景不长。有一天，我们遇到的某个需求很可能需要我们用 UIScrollView 来实现。比如一个很长的展示页面，里面有文字、有图片、可能还有一些按钮等等。它们的长度或宽度很可能会超过 iPhone（或 iPad）的高度或宽度。Apple 当然很仔细地研究过小尺寸屏幕，它提供的解决方案就是 UIScrollView。

我们就来试试。

我们拖入一个 UIScrollView 放在 ViewController 的 View 里，并设置其约束，到四边的距离都是0：

![](https://raw.githubusercontent.com/nixzhu/dev-blog/master/images/autolayout_tip2_add_scrollview.png)

更新一下它的 Frame：

![](https://raw.githubusercontent.com/nixzhu/dev-blog/master/images/autolayout_tip2_add_scrollview2.png)

然后我们拖入一个 UILabel 到 UIScrollView 里，这时 Storyboard 就会报告两个错误：

![](https://raw.githubusercontent.com/nixzhu/dev-blog/master/images/autolayout_tip2_add_label.png)

如果英文不太好的话，我们可以用谷歌翻译。ambiguous 的意思是“不明确”，也就是说，UIScrollView 不能确定其内容的宽或高。

由此，我们可以推断，当 UIScrollView 有 SubView 的时候，它就会开始考虑其“内部”的 contentView 的 Size 了。因为 UIScrollView 处于一个 AutoLayout 的环境中，它不能直接得到 SubView 的 Frame，也就不能确定其 contentView 的 Size，于是 Storyboard 就会跑出来告诉我们这件事。

那我们给 UILabel 加上 3 边约束：

![](https://raw.githubusercontent.com/nixzhu/dev-blog/master/images/autolayout_tip2_add_label2.png)

正常来说，只需要上边和左边就能确定 UILabel 的位置，但右边的约束的作用实际是“撑宽”UIScrollView，这时错误就只有一个了：

![](https://raw.githubusercontent.com/nixzhu/dev-blog/master/images/autolayout_tip2_add_label3.png)

很明显，UIScrollView 可以确定其 contentView 的宽度了，因为 UILabel 的宽度固定，它的左边到 UIScrollView 的左边固定，它的右边到 UIScrollView 的右边固定，于是 AutoLayout 系统可以通过这些约束“猜出” UIScrollView 的 contentView 的宽度。

然后，我们再给 UILabel 加上下边的约束，错误就会完全消失了。

![](https://raw.githubusercontent.com/nixzhu/dev-blog/master/images/autolayout_tip2_add_label4.png)

可事情还没完。若我想 UILabel 的右边约束是 20，下边约束也是 20，可行吗？读者可以自行修改约束的值，很明显是可以的。但这时候就“目测”来看，很明显这两个约束的“线段”会长于 20，那为什么 Storyboard 不会报错或警告呢？

如果读者理解了前面的过程，那就会有答案。因为 UILabel 在 UIScrollView 内的 contentView 上，虽然看起来 UIScrollView 很宽很大，但其 contentView 并不是。相反，contentView 的 Size 是由其中的 Subview 的约束所确定的。

这就是你在同时使用 AutoLayout 和 UIScrollView 时所唯一需要明白的地方。

实际上，我们可以更进一步，修改 ViewController 的尺寸。

将 ViewController 的 Size 改为 Freeform：

![](https://raw.githubusercontent.com/nixzhu/dev-blog/master/images/autolayout_tip2_size.png)

并将 View 的 Size 改为 (70, 69)

![](https://raw.githubusercontent.com/nixzhu/dev-blog/master/images/autolayout_tip2_size2.png)

 这时候就看起来“正常”了，对吧？同样的道理，如果你要往 UIScrollView 里放入很多很多的 Subview，那你就先将 View 的 Size 改到一个合适的尺寸，再做 Subview 的布局，注意修改约束的“值”（因为看起来的长度不一定是实际的长度）。
 
最后强调一点，确保约束能让 AutoLayout 确定 UIScrollView 的 contentView 的 Size，此外就是正常的 AutoLayout 用法。当然，有时候我们只需要高度增加，宽度和屏幕一样，那约束好设置吗？

同样有个 [Demo 放在 GitHub](https://github.com/nixzhu/AutoLayoutInUIScrollView)，如有必要，请稍微看看！

===============


欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

如果你认为这些 AutoLayout Tips 不错，也有闲钱，那你可以用支付宝扫描下方二维码随便捐助一点，以慰劳作者的辛苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)
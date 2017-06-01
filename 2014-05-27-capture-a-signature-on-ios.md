# 在 iOS 上捕捉签名

>绘制像 Paper 应用那样的真正平滑的曲线

本文翻译自：[https://www.altamiracorp.com/blog/employee-posts/capture-a-signature-on-ios](https://www.altamiracorp.com/blog/employee-posts/capture-a-signature-on-ios)

原作者：[JASON HARWIG](https://www.altamiracorp.com/profiles/jharwig)

译者：[@nixzhu](https://twitter.com/nixzhu)

---

Square Engineering Blog 上有一篇很棒的文章，为 Android 带来了[更流畅的签名](http://corner.squareup.com/2012/07/smoother-signatures.html)，但我没有找到任何与 iOS 相关的介绍。所以，在 iOS 设备上捕捉用户签名的最好方式是什么呢？

虽然我没有找到关于签名捕捉的文章，但 App Store 上已有一些应用很好地实现了签名捕捉。我的目标用户体验是达到 iPad 应用 [Paper by 53](http://fiftythree.com/paper) 那样的效果（它是一个绘图应用，有着漂亮且极具响应性的笔刷）。

本文所有的代码都可以在 Github 仓库 [SignatureDemo](https://www.github.com/jharwig/SignatureDemo) 里找到。

## 点点相连

![线段签名](https://www.altamiracorp.com/blog/118/files/signature-diagram-lines.png)

最简单的方式是捕捉用户的触摸点并用直线将它们连接在一起。

在 `UIView` 子类的初始化方法中，创建一个 path ，以及用于捕捉触摸事件的手势识别器。

```Objective-C
// 创建一个 path 以连接各点
path = [UIBezierPath bezierPath];

// 捕捉触摸
UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc] initWithTarget:self action:@selector(pan:)];
pan.maximumNumberOfTouches = pan.minimumNumberOfTouches = 1;
[self addGestureRecognizer:pan];
```

捕捉 pan 手势的触摸点，并通过连线将其放入 bézier path 里：

```Objective-C
- (void)pan:(UIPanGestureRecognizer *)pan {
    CGPoint currentPoint = [pan locationInView:self];

    if (pan.state == UIGestureRecognizerStateBegan) {
        [path moveToPoint:currentPoint];
    } else if (pan.state == UIGestureRecognizerStateChanged)
        [path addLineToPoint:currentPoint];

    [self setNeedsDisplay];
}
```

画出 path：

```Objective-C
- (void)drawRect:(CGRect)rect
{
    [[UIColor blackColor] setStroke];
    [path stroke];
}
```

![签名字母J](https://www.altamiracorp.com/blog/118/files/signature-letter-j.png)

上图是用这种技术画出的字母 “J” ，它也揭示了一些问题：在较慢的速度下， iOS 可以捕捉到足够多的触摸点，因此看起来较平滑，但稍快的移动就会导致触摸点之间较大的间距，签名看起来就是一些直线段相连。

2012 年的 Apple 开发者大会有一个 session 叫做 [创建高级手势识别器](https://developer.apple.com/videos/wwdc/2012/?id=233)，它用了一点数学来解决此问题。

## 二次贝塞尔曲线

![二次曲线签名](https://www.altamiracorp.com/blog/118/files/signature-diagram-quadratic.png)

作为用线段连接各触摸点的代替，我们改用二次贝塞尔曲线来连接这些点，使用前述的 WWDC session 上讨论的技术（定位到 42 分 15 秒）。用二次贝塞尔曲线来连接触摸点，使用触摸点作为控制点以及中间点作为起点和终点。

添加二次曲线到之前的代码里要求存储前一个触摸点，所以要添加一个实例变量：

```Objective-C
CGPoint previousPoint;
```

并创建一个函数来计算两点的中间点：

```Objective-C
static CGPoint midpoint(CGPoint p0, CGPoint p1) {
    return (CGPoint) {
        (p0.x + p1.x) / 2.0,
        (p0.y + p1.y) / 2.0
    };
}
```

再更新 pan 手势处理器，用二次曲线替换直线：

```Objective-C
- (void)pan:(UIPanGestureRecognizer *)pan {
    CGPoint currentPoint = [pan locationInView:self];
    CGPoint midPoint = midpoint(previousPoint, currentPoint);

    if (pan.state == UIGestureRecognizerStateBegan) {
        [path moveToPoint:currentPoint];
    } else if (pan.state == UIGestureRecognizerStateChanged) {
        [path addQuadCurveToPoint:midPoint controlPoint:previousPoint];
    }

    previousPoint = currentPoint;

    [self setNeedsDisplay];
}
```

![二次曲线签名字母J](https://www.altamiracorp.com/blog/118/files/signature-letter-j-quadratic.png)

并没有改写太多代码，但我们已经看到了巨大的不同。触摸点不再可见，但对于捕捉签名来说，它依然有些乏味。每个曲线都是同样的宽度，这不符合实际用钢笔签名的感觉。

## 可变的线宽

基于触摸速度，宽度可变，这样就能创建出更加自然的笔画。 `UIPanGestureRecognizer` 已经包含有一个叫做 `velocityInView:` 的方法，它用一个 `CGPoint` 返回当前触摸的速度。

为了渲染出一个可变宽度的笔画，我切换到 OpenGL ES 和一个能转换笔画为三角形（具体说来，是三角带（triangle strip））的叫做镶嵌（tesselation）的技术（OpenGL 支持画线，但 iOS 不支持平滑过渡的可变宽度）。沿曲线的二次分点同样需要被计算，但这超出了本文讨论的范畴。还请看看[ Github 上的源代码](https://www.github.com/jharwig/SignatureDemo)了解细节吧。

给定两个点，计算出一个垂直的向量，它的大小设置为当前厚度的一半。基于 GL_TRIANGLE_STRIP 的自然特性，只需要两个点就能用两个三角形去创建下一个矩形。

![三角带](https://www.altamiracorp.com/blog/118/files/signature-triangle-strip.png)

下面是使用二次贝塞尔曲线，并根据速度调节笔画粗细而创造出的一个具有视觉吸引力且非常自然的签名。

![可变宽度签名字母J](https://www.altamiracorp.com/blog/118/files/signature-letter-j-opengl.png)

>译者注：[原文](https://www.altamiracorp.com/blog/employee-posts/capture-a-signature-on-ios)后的评论也值得一看哦！

---

欢迎转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！

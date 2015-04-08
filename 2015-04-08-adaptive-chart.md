# 生成自适应图表图片的秘密


细节是什么？边界条件就是细节。请做一个会解方程的程序员！

作者：[@nixzhu](https://twitter.com/nixzhu)


=================================

因为一个开发中 WATCH App，我需要一个图表生成工具，即，它可以根据输入的一串形如 `(String: Double)` 的数据（也就是一串 name 和 value，例如 `[("Hello": 50), ("World": 150), ("NIX ZHU": 100)]`），在给定的图片尺寸和 name 字体时，生成一张柱状图。

其中，name 的最大长度很可能超过通过简单计算的 barWidth（即，barWidth = fullWidth / count），所以默认情况下将 name 旋转一个角度放置，使各名字不互相覆盖。如图所示：

![Adaptive Chart Intro](https://github.com/nixzhu/dev-blog/raw/master/images/adaptive_chart_intro.png)

图中的红色是我要求得的变量，以便根据它们“画出”柱状图。让我来解释一下这里遇到的困难。乍一看，`bestNameWidth?` 可以根据 “NIX ZHU” 的宽度 * cos(angle) 来得到，似乎算不上变量。但图中只是示意，我要特别说明一下。例如第二个 name 可能会很长，比如叫做 “WORLD VERY VERY LARGE...”，那么它就会超过 “NIX ZHU” 的右边界。所以 `bestNameWidth?` 的定义如下（带问号的为变量）：

 (1) bestNameWidth? = index in 0..<count: max(nameWidth[index] * cos(angle) - ((count-1) - index)*barWidth?)

什么意思？很简单，就是所有 name 中，根据每个 name 的 index，也就是位置，那 nameWidth * cos(angle) 就是它的实际宽度，再减去它所处位置相关的那么多的 barWidth? ，我们就得到它实际“超出”最后一个 bar 右边界的距离。当然，在本图中，只有最后一个超出而已。但如上面所说，在实际情况里计算下来，很可能中间某些 name 最终“超出”的距离大于最后一个 name 超出的距离，所以我们要取 max 。

我们共有三个变量，所以还需要两个方程，很容易有：

(2) leftMargin + barWidth? * count + rightMargin? = fullWidth

(3) rightMargin? = max(bestNameWidth? - barWidth? * 0.5 + minRightMargin, minRightMargin)

方程 (2) 很好理解。方程 (3) 是说我们至少要保证 rightMargin? 有 minRightMargin 那么大，因为很可能 bestNameWidth? 并没有超过 barWidth? 的一半，例如都是单个字符的 Name，请想象一下。

三个方程解三个未知数，看起来并不难，那就先来约化。

把 (1) 带入 (3)，有：

(4) rightMargin? = max(     index in 0..<count: max(nameWidth[index] * cos(angle) + minRightMargin - ((count-1) - index)*barWidth?)     - barWidth? * 0.5,     minRightMargin)

根据 (2) 有：

(5) barWidth? ＝ (fullWidth - leftMargin - rightMargin?) / count, 再代入 (4)，得到：

(6)  max(    index in 0..<count: max(nameWidth[index] * cos(angle) - ((count-1) - index)  *      (fullWidth - leftMargin - rightMargin?) / count       )     -      (fullWidth - leftMargin - rightMargin?) / count        * 0.5,     minRightMargin)    -    rightMargin?   = 0

在方程 (6) 中，只有一个变量 rightMargin? 了（别被它的外表吓着了），我们只需要解出它即可。但是这个方程因为有循环求最大值等，太繁琐了，反正最后我是不能将 rightMargin? 移项到一边。

似乎陷入困境了，怎么办呢？

在工程中，我们并不需要精确解，我们只需要足够近似的解。假设最好的 rightMargin? 的值为 n，那我们只需要一个 m 使得 abs(m - n) 足够小，例如小于 0.1 ，这比 iPhone 上一个像素要窄得多，足够了。而求近似解，自然该“[牛顿迭代法](http://zh.wikipedia.org/zh/%E7%89%9B%E9%A1%BF%E6%B3%95)”出场了。

我们将方程 (6) 看成是函数 

f(rightMargin?) = max(    index in 0..<count: max(nameWidth[index] * cos(angle) - ((count-1) - index)  *      (fullWidth - leftMargin - rightMargin?) / count       )     -      (fullWidth - leftMargin - rightMargin?) / count        * 0.5,     minRightMargin)    -    rightMargin?

的根，即求取 `f(rightMargin?)  = 0` 时的 `rightMargin?`。

根据[牛顿迭代法](http://zh.wikipedia.org/zh/%E7%89%9B%E9%A1%BF%E6%B3%95)，我们先猜测一个 `rightMargin?`，例如 1，然后由公式：

`x_(n+1) = x_(n) - f(x_(n)) / f'(x_(n))`，其中 `f'` 为 `f` 的导数，`x_(n)` 为 第 n 个 `rightMargin?`

我们可以得出一个更好的 rightMargin?，它更加接近最终解。最后，如果 `x_(n+1)` 与 `x_(n)` 之差的绝对值小于 0.1 ，我们就认为已找到合适的 rightMargin? 了。关键代码如下：

```Swift
var currentTryRightMargin: CGFloat = 1
var nextTryRightMargin: CGFloat = 0

while true {

    var currentBarWidth = (size.width - barLeftMargin - currentTryRightMargin) / CGFloat(nameWidths.count)

    func nMax() -> CGFloat {
        var _max: CGFloat = 0
        for i in 0..<nameWidths.count {
            let newValue = nameWidths[i] * ratio + minBarRightMargin - CGFloat((nameWidths.count - 1) - i) * currentBarWidth
            if newValue > _max {
                _max = newValue
            }
        }

        return _max
    }

    let fN = max(nMax() -  currentBarWidth * 0.5, minBarRightMargin) - currentTryRightMargin
    let fpN = 1 / CGFloat(nameWidths.count) * 0.5 - 1
    
    nextTryRightMargin = currentTryRightMargin - fN / fpN

    if abs(nextTryRightMargin - currentTryRightMargin) < 0.01 {
        break
    }

    currentTryRightMargin = nextTryRightMargin
}

var barRightMargin = currentTryRightMargin
```

其中 `fpN` 是 f 的导数，它是个常量且不为0，那么牛顿迭代法将具有平方收敛的性能，因此可以快速逼近最优解。

注意 `nMax()` 是计算方程 (6) 中里面的 max 的，`currentBarWidth` 其实就是方程 (5)。

最后我们有了一个足够近似的 barRightMargin ，再计算出其他两个变量是分分钟的事情，不再赘述。实际效果如下：

![Adaptive Chart Example](https://github.com/nixzhu/dev-blog/raw/master/images/adaptive_chart_example.png)


我打算根据上面的工作做出一个小巧的图表图片库，专门用于 WATCH。目前已有代码放在 [https://github.com/nixzhu/AdaptiveChartDemo](https://github.com/nixzhu/AdaptiveChartDemo) ，可以画出柱状图和折线图，待增添更多样式。


===============

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条 Tweet [https://twitter.com/nixzhu/status/585622265154428928](https://twitter.com/nixzhu/status/585622265154428928) 或微博 [http://weibo.com/2076580237/Cci3ReTIf](http://weibo.com/2076580237/Cci3ReTIf)  以分享此文！

如果你认为这篇文章不错，也有闲钱，那你可以用支付宝扫描下方二维码随便捐助一点，以慰劳作者的辛苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)
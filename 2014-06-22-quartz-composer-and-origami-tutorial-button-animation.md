# 用 Quartz Composer 和 Origami 制作一个简单的按钮动画

本文翻译自：[http://www.punchkickinteractive.com/blog/2014/04/01/quartz-composer-and-origami-tutorial-button-animation](http://www.punchkickinteractive.com/blog/2014/04/01/quartz-composer-and-origami-tutorial-button-animation)

原作者：[Daniel Cortes](http://www.punchkickinteractive.com/author/daniel-cortes)

译者：[@nixzhu](https://twitter.com/nixzhu)

*******************************************************

本教程所使用的工具要求 OS X 10.8 或更新的系统。[Jay Thrash][5] 制作了一个非常棒的[视频][4]，以四分钟的长度介绍了 Quartz Composer 和 Origami 。

*******************************************************

![Demo of the animation][6]

去年夏天，Punchkick 的一小群人用头脑风暴的方式讨论给feed添加不同项目类型的方式。我有一个添加项目按钮的主意，即，如果按住添加按钮，就会绽放出不同的项目类型按钮，这样用户就可以立即选择其中的一个而不需要某个专门的界面来显示所有的项目类型然后再选择。有些人立马就明白我在说什么，但另外一些人没有——而我也不会用 After Effects 将动画整合在一起以演示我的意图。

六个月后，Facebook 带给我们一个优秀且免费的工具：[Origami][7]。Origami 是 Apple 出品的 Quartz Composer (QC) 的一个库，它让我们制作类似这样的交互原型变得异常容易。QC 以添加节点的方式工作，以 QC 的说法，节点就是 Patch。将它们添加到你的编辑器里然后将它们连线在一起就能创造出可在查看器中查看的组件和交互效果。

结识了 QC 和 Origami 之后，我就能用很短的时间制作出这个动画的原型。我爱上了 QC 和 Origami —— 我希望你在使用它们之后，也会爱上它们。同时，我十二分地感谢 Facebook 创造了 Origami，以及 Apple 创造了 Quartz Composer。

让我们从零开始制作出这个简单的动画吧。

## 1\. 下载所需的东西

如果你打算使用 QC 和 Origami，那你就要先得到它们。Facebook 已经交待了如何开始使用它们，我将其流程借来：

1. [注册成为 Apple 开发者][8]
2. [下载并安装 Quartz Composer][9]
3. [下载并安装 Origami][10]
4. [下载并解压本教程所使用的源文件][11]

## 2\. 打开 Quartz Composer 并创建一个新的构造（Composition）

上面的步骤做完，就打开 QC。你会看到 Template Chooser 界面。对于我们的目标来说，使用 Basic Composition（默认选中）就可以了，所以确保它被选中，然后点击 Choose。一个几乎空白的编辑器窗口和一个空白的查看器窗口将显示出来。

![Quartz Composer's Template Chooser][12]

## 3\. 设置你的编辑器

你会注意到你的编辑器里已经有了一个 Clear。我们之后会需要一个 Clear，但你目前可以暂时删除它（之后你的编辑器里将会空无一物）。

从 Patch Library 里，添加一个 Phone Dimensions、一个 Layer Group、以及一个 Phone。你可以拖动它们进入编辑器，或者双击它们以添加到编辑器。我个人喜欢拖动，但这两种方式都能工作。

一旦你添加了这三个 Patch，就按照下图连接它们。简单说明一下，Phone Dimensions 告诉 Layer Group 所要处理的屏幕尺寸，而 Layer Group 告诉 Phone 要显示什么。每个 Patch 都接收从它们左边端口进来的输入并将数据传出到它们右边的端口。

![Initial editor set-up][13]

连好各个 Patch 后，你的查看器窗口将显示一个 iPhone，但它的屏幕是空白的。

旁注，在 Patch Library 的底部，你可以读到每个 Patch 的介绍。需要了解的信息很多，一点一点慢慢看比较合理。

## 4\. 添加并重命名素材

**双击 Layer Group 进入其内部**，我们将在此构建我们的动画原型。这个 Patch 里的东西将被传递给 Phone 来显示。

添加一个 Clear 以防止任何旧的素材出现在屏幕上。很容易就能注意到 Clear 的右上角有一个数字 1。它代表了此 Patch 的层级。只有那些能影响显示的 Patch 才有数字，所有不要担心在某些 Patch 上看不到数字。

添加了 Clear 之后，就该添加包含在源文件里的图像了。注意到当你拖动一个图像进入具有 Origami 的 QC 时，它会自动创建并连接一个 Image 到一个 Layer 上。按照如下顺序拖动这几个源图像到 QC 的编辑器里：

1. Background.png
2. Button 1.png
3. Button 2.png
4. Button 3.png
5. Plus Button.png

以上述顺序添加它们可以确保层级顺序的正确性。既然我们知道它们的层级顺序正确，而 Clear 位于第 1 层，那我们就可以根据其数字按照其资源名重命名它们。双击在 ‘Layer’ 的左上角，然后按照如下顺序命名它们：

* Layer 2 -> Background
* Layer 3 -> Button 1
* Layer 4 -> Button 2
* Layer 5 -> Button 3
* Layer 6 -> Add Button

无论何时你要制作一个构造，确保你的层级唯一，且可通过名字识别。这样做的好处是，可以在事情变得比较复杂时，让你依然比较轻松。

完成添加素材后，你得到的东西大概如下图。注意到我喜欢将 Image 移动到它对应的 Layer 之下。我认为这样做可以让我的编辑器看起来更加清晰，而且更加易于同时选择两个关联的 Patch。

![Assets added to editor][14]

如果你的查看器窗口还开着，它将显示 Background 和 Add Button。其它的按钮也在，但它们隐藏在 Add Button 后面。

## 5\. 连接第一个交互（Interaction）

让我们做到这一点，即当你点击 Add Button 时，会发生一些事情。我们将让 Add Button 在被点击时轻微地放大。

![First interaction wired][15]

* 首先，点击 Add Button Layer，再点击 Inspector。我们要将此按钮移动到屏幕底部。修改 Y 值为 -480。这样 Add Button 就会在屏幕底部了。（译者注：同时也将 Button 1 2 3 的 Y 值改为 -480 吧，让它们依然藏在 Add Button 后面）
* 接下来，添加一个 Interaction 2，一个 Bouncy Animation，以及一个 Transition。事情将变得有趣起来。
* 连接 Interaction 2 的 Down 输出端口到 Bouncy Animation 的 Number 输入端口。
* 连接 Bouncy Animation 的 Progress 输出端口到 Transition 的 Progress 输入端口。
* 连接 Transition 的 Value 输出端口到 Add Button 的 Width 和 Height 输入端口。注意到，你可以连接一个 Patch 的输出端口到多个输入端口，但一个输入端口只能有一个连线。
* 检查 Transition 并设置其 Start 值 为 138，End 值为 158。
* 检查 Bouncy Animation 并设置其 Friction 为 4，Tension 为 30。
* 最后，我们连接 Interaction 2 和 Add Button Layer。你会注意到 Interaction 2 和 Add Button Layer 有以它们名字命名的端口。这些端口（让我们称之为连接端口（Link Port））让 Patch 之间能互相交流。从 Interaction 2 的连接端口拖动一个线条到 Add Button Layer 的连接端口。

你已经做完了！如果你转到查看器并点击 Add Button，你就会注意到它被点击时会扩张一点，放开时就会恢复正常大小。

恭喜，你构建了一个交互！

## 6\. 让我们屏住呼吸

让我们退后一步看看这是如何工作的。

* Interaction 2 传递 '0′ 给 Bouncy Animation。
* 当  Add Button 被点击，Interaction 2 传递 '1′ 给 Bouncy Animation。
* 当 Bouncy Animation 收到 1，它大概说，“嘿朋友，不要突然从 0 到 1 嘛，让我们渐变吧。” 如果你分流它，你可以真正看到它在做什么。只需要直接连接 Interaction 2 的 Down 输出端口到 Transition 的 Progress 输入端口，你就能看到在没有 Bouncy Animation 的情况下会发生什么。
* Transition 接收 Bouncy Animation 的输出，并使用它的值（从 0 到 1）来修改自己输出值（本例中，从 138 到 158）
* Transition 之后修改 Add Button Layer 的高和宽，那么我们的 Add Button 就会出现动画效果了！

现在我们确实地明白了 Patch 是如何工作以及互相交流的，让我们更进一步吧。

## 7\. 动画其余的按钮

![Final diagram of the animation][16]

要动画这些按钮，我们就要在 Add Button 被点击时调整它们的 X 和 Y 坐标。Button 1 和 Button 3 将需要改变这两个坐标值，而 Button 2 只需要改变它的 Y 坐标值即可，因为它不需要斜向移动。

### 我们从 Button 1 开始

* 添加一个新的 Bouncy Animation 和两个 Transition 到编辑器里。
* 修改新 Bouncy Animation 的 Friction 为 7 以减小反弹效果。
* 将其中一个新 Transition 重命名为 “X Position” ，而另一个就重命名为 “Y Position”。
* 修改 X Position Transition 的 Start 值为 0，End 值为 -180。
* 修改 Y Position Transition 的 Start 值为 -480，End 值为 -400。
* 连接新 Bouncy Animation 的 Progress 输出端口到两个新 Transition 的 Progress 输入端口。
* 连接 Interaction 2 的 Down 输出端口到最新的 Bouncy Animation 的 Number 输入端口。
* 最后，连接两个新 Transition 的 Value 输出端口到 Button 1 Layer 对应的 X Position 和 Y Position 输入端口。

### 到 Button 2

* 复制并粘贴新 Bouncy Animation 和 Y Position Transition 以作为模版。注意到因为 Button 2 只能上下移动，所以不需要调整 X Position。
* 设置 Y Position Translate 的 Start 值为 -480，End 值为 -300。
* 连接 Interaction 2 的 Down 输出端口到这个 Bouncy Animation 的 Number 输入端口。
* 最后，连接 Y Position Transition 的 Value 输出端口到 Button 2 Layer 的 Y Position 输入端口。

### 再到 Button 3

* 复制并粘贴与 Button 1 Layer 关联的 Bouncy Animation、X Position Transition 以及 Y Position Transition 作为模版。我们需要 X 和 Y 两者，因为 Button 3 斜着移动。
* 修改 X Position Transition 的 Start 值为 0，End 值为 180。
* 实际上我们不需要调整 Y Position Translation 的值，它和 Button 1 的值一样。
* 连接 Interaction 2 的 Down 输出端口到最后这个 Bouncy Animation 的 Number 输入端口。
* 最后，连接两个 X Position 和 Y Position Transition 的 Value 输出端口到 Button 3 Layer 对应的 X Position 和 Y Position 输入端口。


## 最后

转到你的查看器并点击 Add Button。你新处理的按钮就会从 Add Button 下弹出来。感觉很棒，对吧？

想要一点家庭作业？在实践中，这个动画不完全匹配我们的需求，即某人从一个按钮滑动他的手指到另一个按钮时，会关闭 Add Button 被按下的必要条件（译者注：也就是说，手指刚移开，Button 1 2 3 就马上隐藏了）。尝试用 Hit Area 来修复此问题吧（提示：检查它然后打开 “Setup Mode” 来找出它如何工作的头绪）。

如果你有任何问题或评论，你可以[在 Twitter 上找到我][17]。对于我们博客的更新，请关注我们[@PunchkickMobile][18]。同时，你可以加入 [Origami Facebook 社区][19]与专家交流 Origami 的问题，并能获取到最新的 Origami 资讯。

>译者注：这一篇[《次时代交互原型神器Origami档案》](http://www.csdn.net/article/2014-06-09/2820131)挺不错，它比较详尽地介绍了 Origami 的各个 Patch，可当作手册阅读。

************************************

译者注：欢迎非商业转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！

欢迎转发此条微博 [http://weibo.com/2076580237/BabP66WY0](http://weibo.com/2076580237/BabP66WY0)  以分享给更多人！

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳译者的幸苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

版权声明：自由转载-非商用-非衍生-保持署名 | [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)


[1]: http://www.punchkickinteractive.com/blog "View all posts in Blog"
[2]: http://www.punchkickinteractive.com/blog/2014/04/01/quartz-composer-and-origami-tutorial-button-animation
[3]: http://www.punchkickinteractive.com/content/uploads/2014/03/OragamiAnimations.jpg
[4]: https://vimeo.com/88468610
[5]: http://jaythrash.com/
[6]: http://www.punchkickinteractive.com/content/uploads/2014/03/Blossom-Animation.gif
[7]: http://facebook.github.io/origami/
[8]: https://developer.apple.com/register/index.action
[9]: http://origami.facebook.com/quartzcomposer/
[10]: http://facebook.github.io/origami/download/
[11]: http://www.punchkickinteractive.com/content/uploads/2014/03/Blossom-Animation-Source-Files.zip
[12]: http://www.punchkickinteractive.com/content/uploads/2014/03/QC-Template-Chooser.png
[13]: http://www.punchkickinteractive.com/content/uploads/2014/03/QC1.png
[14]: http://www.punchkickinteractive.com/content/uploads/2014/03/QC2.png
[15]: http://www.punchkickinteractive.com/content/uploads/2014/03/QC31.png
[16]: http://www.punchkickinteractive.com/content/uploads/2014/03/QC4.png
[17]: http://www.twitter.com/ddggccaa
[18]: http://www.twitter.com/punchkickmobile
[19]: https://www.facebook.com/groups/origami.community/
[20]: http://1.gravatar.com/avatar/31da3e00aaa1510f9378abdfd61ef631?s=150&d=http%3A%2F%2F1.gravatar.com%2Favatar%2Fad516503a11cd5ca435acc9bb6523536%3Fs%3D150&r=G
[21]: http://www.punchkickinteractive.com/author/daniel-cortes





# 在 GitHub 上贡献开源项目的一般步骤

在正确的场合，以正确的方式表达、参与。

作者：[@nixzhu](https://twitter.com/nixzhu)

引用：1. [How to Contribute to an Open Source Project on GitHub](https://egghead.io/series/how-to-contribute-to-an-open-source-project-on-github) 2. [Collaborating on projects using pull requests](https://help.github.com/categories/collaborating-on-projects-using-pull-requests/)

---

我并不是 Git 的专家，也不太会用 GitHub。但对于开源项目，如果我在使用其过程中遇到了问题，我会很乐意去其项目代码托管处（通常是 GitHub）看看有无其他人报告同样的问题，甚至解决办法。如果没有，那我可能会写一个 Issue。同时，我也会尽我所能去研究遇到的问题，寻找其根源。如果确定能修复此问题，我也会提交一个 Pull Request。

开源项目是很好的学习材料，而且一个好的开源项目在很大程度上可以减少使用它的程序员们的工作，让他们更专注于自有的业务逻辑。但我们不该止步于此，若能共同维护一个开源项目，让其越来越好，是一件对所有人都好的事情。

有了参与的主观意愿还不够，我们得学会参与的规则和流程。我们以托管在 GitHub 上的开源项目为例。

## Issue

假如你使用某个开源项目时遇到了问题，那么你首先应该去其 Issues 页面看看，例如 [Yep 的 Issues](https://github.com/CatchChat/Yep/issues)。除了 Open 的之外，还可以看看 Closed，甚至以可能的关键字进行搜索。如果你找到了某个类似的问题，你可以进去留言，说明你所遇问题的情况，或与已有描述的差别。

假如你没有找到描述类似的问题的 Issue，那么你可以点击 New Issue 来新建一个。在 Title 里简明扼要地写下问题的描述，再在 comment 里详细描述所遇问题。除了产生问题的条件，如果可能，加上一些自己对问题的推断甚至可能的解决方案。如果你对问题没有头绪，比如代码在运行中产生了一个死锁，那么你可以将程序的调用栈贴出来。这些信息有利于项目的维护者定位问题的根源。

写 Issue 的重要原则就是不要随意，要将其看成一种参与，一种帮助和自助。因此，词句要清晰，描述要详尽。若发完 Issue 后发现还有需要补充的地方，可以进一步增加 comment。

最后，在提交新的 Issue 之前，应该检查一遍，确保没有明显的错别字和语法错误。

## Pull Request

我想，有能力的程序员不会止步于发 Issue，他们会更想直接以修正代码的方式解决问题，这就需要明了 Pull Request 的流程。

简单来说，Pull Request 是外部开发者参与开源项目的主要方式。因为通常一个开源项目不会允许所有人都去直接修改代码，这会让项目变得混乱，难以维护和测试。

因此，外部开发者需要先 Fork 已有项目，然后在自己 Fork 的项目上进行修改，最后这两个项目就产生了差异，GitHub 可以检测到这样的差异。当外部开发者修改完成，就可以利用 GitHub 将自己的改动以 Pull Request 的方式提交到原项目，原项目维护者再对改动进行检查和测试。在确定没有问题后，将改动合并到项目中。这样外部开发者的这一次参与就完成了。

听起来似乎有些复杂，但正是这样的流程保证了代码的持续稳定。因为写代码和写文章一样，并不是一件随意的事情。

1. Fork。例如在 [Yep](https://github.com/CatchChat/Yep) 的项目上，点击右上角的 Fork 按钮（请大胆地点击，这不会对世界造成任何明显的伤害），然后你就得到了当前 Yep 项目的一个拷贝。
2. Clone。因为我们通常在本地电脑上做修改，因此先将 Fork 的项目 Clone 到本地，大家应该很熟悉吧。例如 `git clone https://github.com/XXX/Yep.git`，其中 XXX 为你的 GitHub 用户名。
3. 保持同步。大多数学会 Fork 的开发者都会自然地产生一个疑问：如果原项目被改动了，那我们自己的拷贝该怎么同步原项目的改动呢？因为之前的拷贝是以那个时候的原项目为原本的。这也不难，在本地项目中，运行 `git remote add upstream https://github.com/CatchChat/Yep.git`，这就将原项目作为了本地项目的上游。你可以在运行此命令之前和之后运行 `git remote -v` 来观察改动。之后，若你想同步原项目的改动，执行 `git fetch upstream` 即可，这会将原项目所有分支的改动都存储在本地，例如原项目 `master` 分支会存为 `upstream/master`，不过这还不会对本地的项目造成影响。将如你想将上游 master 的改动合并到本地，只需先切换到 master 分支 `git checkout master`，再执行合并 `git merge upstream/master` 即可。同理可以按需要处理 develop 分支等。

如果你的改动很小，那么修改所持续的时间不会很长，你可以不做“保持同步”这一步。当你发出 Pull Request 后，若被维护者合并，你就可以从 GitHub 上删除你 Fork 的项目了。若下次你还想再修改，完全可以重新 Fork，这一次同样会以最新的代码为基础进行拷贝。

如果你想长期参与，那在做完第三步后，还可以 `git branch --set-upstream-to=upstream/master master`，这样当上游的 master 有改动后，只需在本地 master 分支 `git pull` 即可。同理，你也可以处理 develop 等分支。

通常，我们都在新分支里做修改，在测试改动没有问题后才合并到主分支。大家经过总结，出现了一套叫做 Git Flow 的流程。简单来说，在 master 外，建立一个 develop 分支用于开发，而且，每一个新特性的开发都会从 develop 分支创建新分支，新特性开发完成后再合并到 develop 分支。

我们也并不需要精通 Git Flow，只需记住两个命令 `git flow feature start XXX` 和 `git flow feature finish XXX`，其中 XXX 是你要开发的特性（或要修复的Bug）的名字。我搜索到一篇很清晰的[关于 Git Flow 的讲解](http://danielkummer.github.io/git-flow-cheatsheet/index.zh_CN.html)，大家一起学习。

Yep 的开发使用了 Git Flow，但作为外部开发者，你也不一定非要使用它，因为本质上 Git Flow 也只是建立分支，并以分支为基础的一套方便工具。

例如 `git flow feature start XXX` 以 develop 为基础，创建一个新的 `feature/XXX` 分支，而 `git flow feature finish XXX` 就将 `feature/XXX` 分支合并到 develop，你完全可以直接用 git 命令来实现。例如 `git checkout -b XXX develop` 就会以 develop 分支新建一个 `XXX` 分支（这里没有自动写上的 feature 前缀），当你完成你的修改后，`git checkout develop` 然后 `git merge --no-ff XXX develop` 就可以将 `XXX` 分支合并进 develop 了。不过，你也可以直接以 `XXX` 分支发 Pull Request，只需先将其 `git push origin XXX` 到 GitHub，然后在你 Fork 的项目主页点击 `Compare & pull request` 按钮即可。如果没有这个按钮，也可以点击 `New pull request` 按钮来发起，注意选择正确的分支即可（你工作的分支到 Yep 的 develop 分支）。

以上是个人对 Git 以及参与 GitHub 项目的粗浅理解，如有错漏，欢迎以 Issue 或 Pull Request 的方式指正！

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

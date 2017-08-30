# Five Factor Testing

# 测试五要素

本文翻译自 [https://www.devmynd.com/blog/five-factor-testing/](https://www.devmynd.com/blog/five-factor-testing/)

原作者：[Sarah Mei](https://www.devmynd.com/blog/author/sarahmei/)

译者：[@nixzhu](https://twitter.com/nixzhu)

---

When I took my first real dev job in the late 90s, it was not common for developers to write their own automated tests. Instead, large companies depended on teams of testers, who tested manually, or were experts in complex (and expensive) automation software. Small companies were more likely to depend on code review, months of "integration" after the "development,"…or most commonly: pure hope.

在90年代后期，我初次做开发工作时，开发者们通常都不写自动测试。那时，大公司依赖测试组做手动测试，或者使用复杂（且昂贵）的自动测试软件。而小公司则更多地依赖代码审查，“开发”后会经过数月的“集成”，或者干脆什么都不做，全凭运气。

But times have changed. Today, on most teams, writing automated tests is a normal part of the software developer's job. Changes to a codebase usually aren't considered complete until there are at least _some_ automated tests exercising them — usually written by the developer who changed the code. This has freed up the dedicated testers, in companies that have them, to focus on more valuable activities such as [exploratory testing][1].

但时代改变了。今时今日，在大多数团队里，编写自动测试已成为开发者工作的一部分。对代码库的改动需要至少搭配一些自动测试（通常由改动代码的开发者编写）来检验，不然改动不会被认为是完成的。这解放了专门的测试人员，（在有他们的公司里）让他们专注于更有价值的活动，例如[探索性测试][1]。

Like most shifts, this one was imperceptible while it was happening, and obvious in hindsight. In the late 90s, when having **any** developer-written tests was weird, it was hard to imagine that twenty years later, having **no** developer-written tests would be weird. But here we are.

与其他转变一样，它发生时悄无声息，完成后就很显眼。在90年代后期，写测试的开发者会显得奇怪。难以想象，20年后，不写测试的开发者反倒显得奇怪。但我们就是走到了今天。

## Welcome to the Future

## 欢迎来到未来

Flying cars! Personal jetpacks! Hoverboards! Writing tests has gone from "novel activity" to "just another thing we do to support our code changes," akin to meetings or email or keeping up with Slack!

飞车！个人喷气背包！悬浮滑板！写测试已从“小说活动”变成“另一件我们所做的支持代码改动的事情”，就像会议、电邮或用Slack交流。

Sigh.

哎～

But like meetings and email and Slack, sometimes we write tests just to do it, whether or not it's actually useful. After that's been going on for a while, you'll hear things like this:

* "The tests are so [slow | flaky | unpredictable]."
* "We don't have the test coverage we need, but we don't have time to update it."
* "Writing tests just doubles the amount of time a story takes me — for no useful reason."
* "That story is finished. I just have to write the tests."

但就像会议、电邮和Slack，有时我们写测试只是机械地去做而已，不管它是否真的有用。这样一段时间后，你会听到如下论述：

* “测试太[慢|片面|不可预料]。”
* “我们的测试没有覆盖我们所需要的，但我们没时间去更新了。”
* “写测试只是花了两倍的时间讲故事，而且是为了无用的理由。”
* “故事结束了。我只是必须写测试。”

To fix these issues and end up with tests (and a testing process) that actually work for us, we need to reconnect with the underlying needs that originally drove us to write tests. Surprisingly, they aren't really written down anywhere. Maybe it's just assumed we know what they are, but I listed them out enough times for people to think _I should just write a blog post and send them that_. So here you go.

要修正这些问题，且让测试（以及一个测试过程）真正服务于我们，那我们就需要重新连接那原初的驱使我们编写测试的潜在需要。令人讶异的是，居然没人写下它们。也许它只是假设我们都知道，但在我为人们列出它们足够多次以后，我想，我应该写一篇博客，以后直接发给他们就好了。那就开始吧。

## The Five Factors

## 五个要素

There are five practical reasons that we write tests. Whether we realize it or not, our personal testing philosophy is based on how we judge the relative importance of these reasons. Many people think about factors 1 and 2 — they are the standard reasons for writing tests, and are often talked about. But the arguments we have about testing, both within our teams and endlessly on the internet, often stem from unarticulated differences in how we think about factors 3, 4, and 5.

我们编写测试有五个原因。不论我们是否意识到，我们的个人测试哲学构都建于我们如何判断这些原因的相对重要性。许多人认为要素1和2是编写测试的标准原因，并经常谈论。但我们关于测试的争论，不论是团队内部还是互联网上，经常来自于我们对要素3、4、5未阐明的理解差异。

First we'll examine each factor in isolation, and then, moving into some concrete examples, we'll consider how they combine when you're deciding how to test your code.

我们先独立检查每个要素，然后看看具体的例子。我们将考虑，在我们决定如何测试代码时，它们会如何结合在一起。

## Good tests can…

## 好的测试能……

> **1\. Verify the code is working correctly**

> **2\. Prevent future regressions**

> **3\. Document the code's behavior**

> **4\. Provide design guidance**

> **5\. Support refactoring**

> **1\. 验证代码的正确性**

> **2\. 防止将来的回归**

> **3\. 记录代码的行为**

> **4\. 提供设计指导**

> **5\. 支持重构**

Let's look at each of these in more detail.

让我们分别看看它们的细节。

## 1\. Verify the code is working correctly

## 1\. 验证代码的正确性

In the most immediate sense, most of us write tests to have confidence that the code we're adding or changing works the way we think it does. In college, back in ye olden dayes, I wrote little shell scripts to exercise my coding assignments. I never turned the scripts in, because at that point, verification was the _only_ goal of my tests. After all, my computer science professors only cared whether the assignment's output was what they wanted.

在最直接的意义上，我们中的大多数写测试的目的就是增强我们增加或改动代码的自信，确保代码的行为与我们预想的一致。在读大学时，我写了一些shell脚本来操练我的编码作业。我从没有将其上交，因为在那时，这些测试脚本的唯一目的就是验证而已。毕竟，我的计算机科学教授只在乎我作业代码的输出是否与他们想要的一致。

## 2\. Prevent future regressions

## 2\. 防止将来的回归

Immediate verification is sufficient for small coding assignments, but most of us work in larger, more complicated codebases, in which other people are also working.

立即验证对小的编码作业来说完全足够，但我们中的大多数都工作于更大更复杂的代码库，而代码库中还有其他人在同时工作。

In this situation, the automated tests you write for your code become part of a "suite," or collection, of tests that all verify different parts of the system. Making a change, and then running the suite and seeing all the tests pass, gives you confidence that your change didn't break anything anywhere else in the application. This prevents "regressions," a fancy word meaning "things that used to work, but don't anymore."

在此状况下，你为你的代码编写的自动测试将成为“套装（测试集）”或集合的一部分，其中的测试将验证系统的不同部分。做出一个改变，然后运行这个测试集并看到所有测试通过，这将给你自信：改动没有破坏程序的其他部分。这能防止“回归”，一个华丽的词汇用于描述那些“过去工作，但现在不再工作的东西”。

Our tests become part of the suite, so that in the future, other developers can have confidence that they didn't accidentally break our stuff.

一旦我们的测试成为套装的一部分，那们在将来，其他开发者也会有这样的信心：他们不会偶然破坏我们的东西。

## 3\. Document the code's behavior

## 3\. 记录代码的行为

> "Programs must be written for people to read, and only incidentally for machines to execute." – [_Hal Abelson_][2]

> “程序是写给人读的，只是顺便让机器执行。”——[_Hal Abelson_][2]

Code is communication — primarily to other developers; secondarily to a computer. Since your automated tests are code, they're also communication, and you can take that further by explicitly designing them to be external documentation for the code they test.

代码即是沟通——主要是和其他开发者，其次才是和计算机。由于你的自动测试也是代码，它们同样也是沟通，并且你可以进一步明确地将它们设计为被测试代码的外部文档。

There are, of course, many ways to document your intent when writing code:

* long form, such as on a wiki or in a README
* comments in your code
* names of program elements like variables, functions, and classes

当然，已有许多方式可以记录你的意图：

* 长文，例如wiki或在README中
* 代码中的注释
* 程序元素（例如变量、函数以及类）的名字

Tests are often overlooked as a form of documentation, but they can be more useful than any of the above to a fellow developer. For one, they're executable — so they don't go out of date. In addition, though, they're usually the easiest way to demonstrate (rather than explain in prose) things like how you're expecting the code to be used, what happens when it encounters an edge case, and why that weird-looking bit is like that.

测试通常被忽视为文档的一种形式，但对将来的开发者来说，它们其实比上述方式更有用。首先，它们是可执行的——因此它们不会过期。再者，比起文档，它们通常更容易演示你希望代码被如何使用，遇到边界情况时会发生什么，以及为何这个奇怪的东西是这样的。

## 4\. Provide design guidance

## 4\. 提供设计指导

Unquestionably, the most controversial claim of testing advocates is that testing leads to better software design. Most of the explanations I've seen for this are grounded in software design theory, which can be difficult to translate into what to do when you sit down at your editor. Other sources don't even really try to explain. Instead, they ask you to take it on faith: "write tests, and over time, your code will be better than if you didn't write tests!"

毫无疑问，测试倡导者的最有争议的宣说是“测试引导出更好的软件设计”。大部分我见过的关于这个观点的解释都基于软件测试理论，但它们很难被翻译为你坐在编辑器前要做的事。其它来源则根本不尝试解释，相反，他们要求你将其作为信仰：“编写测试，经过一段时间，你的代码会变得比你不写测试时要好！”

This lines up with my experience, but I'm not asking you for faith. This idea, which I first heard from a colleague when I was at Pivotal Labs, was a lightbulb moment for me in starting to grasp why this works:

这符合我的经验，但我不会要求你信仰它。在Pivotal Labs时，我第一次从一个同事那里听到这个观点，对我来说，就像一个灯泡亮了，我开始了解它为何有效：

Designing the interface for a piece of code is a delicate tightrope walk between specificity (solving the problem you have right now) and generality (solving a more general class of problems, with a eye to reusing the code elsewhere). Specific code is usually simpler in the now, but harder to evolve later. Generalized code usually adds some complexity now that is not strictly required to solve the current problem, in return for being easier to evolve later.

为一段代码设计接口是一种特殊的走钢索，需要在特定性（解决你当前的问题）和通用性（解决更一般的问题类别，同时注意重用代码到别处）之间做权衡。特定代码通常在当前更简单，但之后很难进化。通用代码通常需要在目前就添加一些复杂性，虽然这并不是解决当前问题所必须的，但回报就是之后更容易地改进。

Learning to pick the right place on that spectrum for the piece of code you're working on is a messy, squishy, difficult-to-acquire skill. However, your tests can actually help you, in a very concrete way.

学会从当前正在处理的代码中挑选正确的位置实在是一个模糊且难以获得的技能。然而，你的测试实际上可以帮到你，而且是以非常具体的方式。

Let's say you're adding a method to a class. Presumably you're doing that because you want to call that method somewhere else. Here's your new method, which enqueues a background job to send an email to the user.

假如你要给一个类添加一个方法。你之所以要这样做是因为你打算在某个地方调用这个方法。下面是你的新方法，它入队一个后台任务以发送电邮给用户。
    
    class User
      # ... other stuff ...
      def send_password_reset_email
        email = UserEmails.password_reset.new(primary_email, full_name)
        BackgroundJobs.enqueue(email, :send)
      end
      # ... more stuff ...
    end

The code where you're using it is the new method's _primary client_. Here's the primary client — a method that is called as the result of an API call, when the user requests a password reset from the client.

使用此方法的代码将是它的**主要客户**。即，当用户从客户端请求一个密码重置操作，作为一个API调用结束后所调用的方法。
    
    class PasswordResetController
      # ... other stuff ...
      def create
        Auditor.record_reset_request(current_user)
        current_user.send_password_reset_email # this line is new
      end
      # ... more stuff ...
    end

When you write a test that calls that method, you're giving it a _secondary client_, in which the method is used in a different context. Here's the unit test for your new method.

当你编写测试调用那个方法，你就给了它**次要客户**，方法将在不同的上下文中被使用。如下单元测试：
    
    describe User do
      # ... other tests ...
      describe("#send_password_reset_email") do
        it("enqueues a job") do
          expect(BackgroundJobs).to_receive(:enqueue)
          User.new.send_password_reset_email
        end
      end
      # ... more tests ...
    end

Using code in two contexts by writing a test for it means you build in a tiny bit of generality beyond what you specifically need in your primary client. It may be almost imperceptible, but importantly, it's _not_ speculative — in other words, you don't risk going too far and building in generality that obscures meaning and that you probably won't use.

通过编写测试，在两个上下文中使用代码，就意味着在主要客户之外获得了一定的通用性。它也许难以察觉，但很重要的是，它**不投机**，换句话说，你不用冒着风险岔开太远，去构建那些你可能还不会用到的通用性。

Over time, this technique loosens your code's ties to the specific problems it solves, and makes it more possible to evolve the codebase in the direction your team needs it to go.

随着时间推移，这个技术避免你的代码太专注特定问题，代码库更容易朝着团队希望的方向进化。

## 5\. Support refactoring

## 5\. 支持重构

The only constant in software is change, so we often want to write code that will be straightforward to evolve once new requirements come in. **Refactoring** is the process of cleaning up and changing the organization of code, without changing its external functionality. When you're refactoring, you need tests to ensure you're not breaking anything by moving code around.

软件里不变的就是改变，所以经常在新需求到来时，我们希望编写的代码能直接进化。**重构**是这样一个过程：清理并改变代码组织，而不改变它内部的功能。当你重构时，你需要测试来确保你移动代码时没有破坏任何其他东西。

A codebase that needs to absorb changes over the long term _must_ have a test suite that supports refactoring, or the rate of development (even as developers are added) will inexorably decrease. So you need automated tests at _different_ levels of your codebase (so you can refactor beneath different interfaces) that you can use to assert that functionality hasn't changed.

一个代码库要长期地吸收变化，那就必须要有一个测试集来支持重构，不然开发效率（即使增加开发者）将不可避免地下降。所以你在不同层面都需要自动测试（这样你可以在不同的接口下重构）来确保功能没有被破坏。

## How To Use This List

## 如何使用此列表

Ok! We've got our list of factors. Now it's just a matter of maximizing them all, right?

好了！我们有了重构要素的列表。现在只要最大化执行它们就行了，对吧？

Well…no. That's impossible. It generally won't even be useful to "pick a favorite" and always optimize for that factor. Which factors are more important will vary between sections of your codebase, and even in the same section over time. This isn't a to-do list; it's a **framework for discussing test strategy**. When you're looking at a test in a pull request or during a code review, think about which of the factors it supports, and which it doesn't. Then you can discuss the test in terms of those factors — are they the right ones? Would it make sense to optimize more for documentation here, rather than future refactoring?

然而…不。这是不可能的。通常“挑选一个最喜欢的”要素并总是为它优化并没有多少作用。越重要的要素越会引起代码库的各个部分变化，甚至是同一个部分在不同时期变化。所以这不是一个to-do list，而是一个**讨论测试策略的框架**。当你观察一个pull request的测试或进行代码审查时，想想它支持哪些要素，不支持哪些。然后就能以这些要素为术语来讨论测试，它们是正确的那个吗？再优化一下文档是不是更好，而不是将来再重构？

Our traditional ways of discussing tests are largely based on morality and shame, and are not very useful. For example, "you need to write an integration test because it's an industry best practice" is a morality argument. The subtext is that everyone else does it, so it must have inherent value, and you're a terrible developer if you don't want to.

我们讨论测试的传统方式主要基于道德和耻辱，但并不太管用。例如，“你需要写一个集成测试，因为这是业界最佳实践”就是一个道德论证。它的潜台词是每个人都这么做，那么一定有其固有价值，所以如果你不这么做，那你一定是个糟糕的开发者。

**No test has inherent value.** A test is _only_ valuable to your project insofar as it supports one or more of the five factors.

**没有那个测试具有固有价值**。一个测试只有在它支持这五个要素的一个或多个时才具有价值。

And keep in mind that an individual test or even a suite, overall, _cannot_ fully support all five factors. They are necessarily somewhat at odds. Here are a few examples of how the factors combine to show what I mean.

记住，单独的测试，甚至是测试集，总的来说，若没有完全支持这五个要素，那它们一定有某些不对劲的地方。下面是一些例子，用一些要素的集合，来说明我的意思。

## Ex 1: Unit Tests and Refactoring

## 例子 1：单元测试和重构

> "The answer to every question in software is 'it depends.'" – [_Sandi Metz_][3]

> “软件中每个问题的答案都是‘依赖’。”——[_Sandi Metz_][3]

Comprehensive unit tests for a class are great for **3\. developer documentation**, but make it hard to **5\. refactor** that interface. To do that, you need a set of tests written one level up (where your class is used) that assert on _outcomes_, so you can make sure the functionality doesn't change, even as you rename methods and move pieces around. **2\. Regression**-oriented unit tests are often less comprehensive and thus easier to work with when refactoring, but they may not sufficiently document.

为一个类做全面的单元测试很符合**要素3.开发者文档**，但它也让**要素5.重构**变得困难。要做到这一点，你需要一个测试集应对每一个类被使用的地方以断言输出，这样你才能确保功能没有被改变，甚至是重命名方法或移动代码片段。面向**要素2.回归**的单元测试通常有更少的全面性，因此易于重构，但它们可能没有足够的文档。

But like Sandi says — it depends. If your class is part of a public API and will change only rarely, comprehensive **3\. developer documentation** with a narrative structure (i.e., meant to be read through) may actually be your primary objective. Since the interface won't change much, the fact that such tests complicate refactoring is not very important.

但就像Sandi说的，看“依赖”。如果你的类是公开API的一部分，很少会改变，那么全面的具有叙事结构（例如从头读到尾）的**要素3.开发者文档**就可能成为你的主要目标。因此接口不怎么改变，那因测试导致重构变复杂也就不那么重要了。

On the other hand, if it's an internal class, you may opt instead for less-comprehensive **2\. regression**-oriented unit tests that hit the happy path plus likely errors, and worry less about narrative organization. These support **5\. refactoring** better than more documentation-oriented tests, and can still serve as **4\. design guidance** and _lightweight_ **3\. developer documentation**, even as the emphasis is placed elsewhere.

另一方面，如果是一个内部类，那可能不太全面的面向**要素2.回归**的单元测试更好，确保主要地方没有错误，对叙事组织更少担忧。这些比面向文档的测试更支持**要素5.重构**，也能作为**要素4.设计指导**和轻量化**要素3.开发者文档**，即使重点在别的地方。

![][4]

You're never picking one factor. **You're always striking a balance between them.** And it can be hard to identify precisely the balance you're striking, particularly if you have some experience writing tests. It's worth doing, though, because you'll find tests that are unconsciously optimizing for a factor that's important somewhere else, but _not right here_. You can often simplify (or even eliminate) those tests as a result.

你不是选择一个要素。**而是在它们之间寻求平衡**。可能很难准确地确定平衡点，特别是你只有一些编写测试的经验。但这值得去做，因为你会发现测试会无意识地优化某个在其他地方来说很重要的要素，而不是这里。你可经常简化（或清除）这些测试以作为结果。

This discussion about unit tests is a perfect example. I've seen systems with versioned public APIs that had (appropriate) documentation-oriented tests. But without thinking about it, they applied that test strategy to similar internal structures, forcing devs to write comprehensive unit tests for everything. This made refactoring really difficult — even internally, where it was supposedly allowed. Once they thought about this in terms of factors, though, they were able to see that they needed to tilt their strategy for the internal structures back towards refactoring support.

这些关于单元测试的讨论是绝佳的例子。我见过一些系统有着定好版本的公开API，有着（适当的）面向文档的测试。但没有经过思考，他们就在内部结构上也执行同样的测试策略，强制开发为所有东西都编写综合单元测试。这让重构变得异常困难，甚至是内部重构，它原本因该是被允许的。若他们曾经考虑过这些要素的术语，那他们就能看到他们需要为内部结构倾斜他们的策略，以取得更好的重构支持。

## Ex 2: Integration Tests, Regressions, and Documentation

## 例子2：集成测试，回归，以及文档

As the running time of a test suite gets longer, its utility for **2\. preventing regressions** goes down, because developers are less likely to run a long test suite. My personal threshold is somewhere around ten minutes — more than that, and I start looking for ways to speed up the suite.

随着测试集的运行时间变长，**要素2.预防回归**的效用就会下降，因为开发者可能不会运行很长的测试集。我的个人阈值是大约10分钟，超过它，我就会开始寻找方法来加速测试集。

Top-level integration tests, which are great for **1\. proving your code works** while you're developing a feature, and **3\. documenting how your app works** afterwards, run quite slowly. They often make up the bulk of time when running a test suite. Once the feature has been written, and the need has shifted from **1\. proving it works** to **2\. preventing regressions**, you can often rewrite slow integration tests in a faster form. For example, an integration test for a web application that uses JavaScript functionality can often be rewritten as a combination of individual backend endpoint tests, and JavaScript unit tests. You have to be careful to be sure you're getting the same coverage, but if test suite running time is important, it's doable.

顶层的集成测试，对**要素1.证明代码工作**在开发过程中来说非常好，而**要素3.记录应用如何工作**则相反，运行起来很慢。它们经常占用运行测试的大部分时间。一旦特性写完，那就需要从**要素1.证明代码工作**切换到**要素2.防止回归**，你可以经常重写缓慢的集成测试为更快的形式。例如，一个使用JavaScript功能的web应用的集成测试，通常都能被重写为独立后端端点测试和JavaScript单元测试的组合。

Even if you have exactly the same coverage, there's still a downside to doing this conversion: if you don't have top-level integration tests, it can be hard to figure out exactly how a feature is supposed to work just by looking at the tests. Your test suite's utility for **3\. documenting functionality** has gone down, because now instead of just looking in one place, you often need to piece it together from several places.

即使是有同样的覆盖率，这样进行转换一样有其缺点：如果你没有一个顶层的集成测试，你很难通过单元测试搞清楚这个特性到底如何工作。你的测试集的工具就失去了**要素3.文档化功能**，因为现在没办法在一个地方看到全部，它们都分散了。

At this point, you could make the deliberate decision to start documenting that top level functionality in another form — screencasts, on a wiki, etc. — if it feels like the value of lowering the suite-wide run time outweighs the value of documenting functionality via integration tests.

在这种情况下，你可决定开始以另一种形式来文档化顶层功能，如截图，或wiki等（如果感觉降低测试集的运行时间要超过集成测试文档化功能的价值）。

## This Is Complicated o_0

## 这很复杂o_0

Yup. Testing, itself, is complicated, because tests are techno-social constructs that support both the code and the team that works on it. As your team's needs change over time — because of business changes, personnel changes, or (most commonly) both — the types of tests you need change alongside. Treat your tests as living documents, rather than the fossilized remnants of past sprints. Consider their actual utility to you, right now, rather than whether a book, or a thought leader, or even your boss says you "must" have them.

是的。测试，它本身，就很复杂，因为测试是一种技术社会结构，用于支持代码和团队的协同工作。若你的团队的需求会随着时间修改（由于商业变化、个人变化、或两种都有），那你的测试也要跟着改变。将你的测试当作活着的文档，而不是过去冲刺所留下的僵死残余物。考虑它们的实际效用，就现在，而不是一本书、一个思想领袖、甚至你的老板说你“必须”要有它们。

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

[1]: https://en.wikipedia.org/wiki/Exploratory_testing
[2]: https://en.wikipedia.org/wiki/Hal_Abelson
[3]: https://en.wikipedia.org/wiki/Sandi_Metz
[4]: https://i2.wp.com/www.devmynd.com/wp-content/uploads/2017/05/stick-and-stones-150x150.jpg?resize=150%2C150&ssl=1
[5]: https://i2.wp.com/www.devmynd.com/wp-content/uploads/2016/04/DevMynd-Profiles-Sarah-e1460565130449.jpg?fit=100%2C100&ssl=1

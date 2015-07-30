# 对函数式编程的一点理解

虽然有不少对“函数式编程”的解释，但我没有遇到让我满意的。我不是说解释要多么的具体、全面，或者让我立马会使用、变得很厉害，我就想知道它是怎么一回事儿。

作者：[@nixzhu](https://twitter.com/nixzhu)

=================================

先考虑一个过程。假设您性别男，爱好女。有位优雅的女士愿意与您做爱，不过这位女士对做爱有些要求：前戏至少要15分钟，性交要求在3到5分钟之间，最后要抱抱30分钟，说些情话。

如果我们用程序来描述上面的过程，大概如下：

``` swift
if forplay(forplayMinutes) {
    if sexualIntercourse(sexualIntercourseMinutes) {
        if hug(hugMinutes, smallTalk) {
            print("Satisfied!")
        } else {
            print("Not satisfied!")
        }
    } else {
        print("Not satisfied!")
    }
} else {
    print("Not satisfied!")
}
```

其中`forplay`，`sexualIntercourse`，`hug`都是函数，相对于具体过程来说，我们已经做了抽象和简化，不然会变得少儿不宜。

它们的实现大概如下：

``` swift
func forplay(minutes: Int) -> Bool {
    return minutes >= 15 ? true : false
}

func sexualIntercourse(minutes: Int) -> Bool {
    return (minutes >= 3 && minutes <= 5) ? true : false
}

func hug(minutes: Int, smallTalk: Bool) -> Bool {
    return ((minutes >= 30) ? true : false) && smallTalk
}
```

因为要按照女士的要求做爱，所以这三个子过程的顺序并不能颠倒，但目前这三个函数并没有体现出顺序。

我们在做的就是所谓的“编程”，那什么是“函数式编程”？

先来分析一下最上面的过程：

1. 首先我们定义了一些输入，如`forplayMinutes`等；
2. 然后我们（主要是您）先执行 forplay，当这个过程符合要求后才进行下一步；
3. 若三个子过程都满足要求，女士才会满足，中间某个过程一旦不满足，那就不可能满足了。

再简化一点说，即：有输入，过程有顺序，过程可能有输出。那什么是“函数式编程”？

别慌，快了。

其实函数式编程的是一种更高层的抽象：我们假设有个“高阶函数”，它能接受任意数据和一个函数（利用范型），并返回经过此函数处理的数据。我们命名此函数为`bind`：

``` swift
func bind<A, B>(a: A?, f: A -> B?) -> B? {
    if let x = a {
        return f(x)
    } else {
        return nil
    }
}
```

因为函数不一定有返回值，所以返回值是可选的。注意这里的`a`参数不一定要是可选的，但是为了`bind`的通用性（后面会看出来），让其可选比较好。

而为了在子过程中体现顺序，我们要为它们增加参数：

``` swift
func forplay(minutes: Int) -> Bool? {
    return minutes >= 15 ? true : false
}

func sexualIntercourse(forplaySatisfied: Bool, minutes: Int) -> Bool? {
    if forplaySatisfied {
        return (minutes >= 3 && minutes <= 5) ? true : false
    } else {
        return nil
    }
}

func hug(sexualIntercourseSatisfied: Bool, minutes: Int, smallTalk: Bool) -> Bool? {
    if sexualIntercourseSatisfied {
        return (minutes >= 30 ? true : false) && smallTalk
    } else {
        return nil
    }
}
```

注意，除了增加判断参数外，为了能适用于 bind，函数的返回值也变为可选了。

由此，该怎么用 bind 来描述上面的过程呢？

``` swift
var satisfied: Bool = false
satisfied = bind(forplayMinutes, forplay) ?? false
satisfied = bind(satisfied, sexualIntercourse) ?? false //错误
satisfied = bind(satisfied, hug) ?? false               //错误

if satisfied {
    print("Satisfied!")
} else {
    print("Not satisfied!")
}
```

可惜我们不能这么写，因为函数`sexualIntercourse`和`hug`都不只接受一个参数，无法适用于`bind`。那怎么办？

我们可以使用[柯里化（Currying）技术](https://zh.wikipedia.org/wiki/%E6%9F%AF%E9%87%8C%E5%8C%96)改造我们的函数。所谓柯里化，就是将接受多个参数的函数变换成接受前几个参数（Swift的实现并不限定为第一个）的函数，而此函数返回一个“接受余下参数并能返回结果的新函数”。虽然柯里化的介绍并不直观，但其实改造很容易。我们只需要将多个参数分割成两个部分即可，我们直接传递参数给第一部分就可生成新函数，如：

``` swift
func add(a: Int, b: Int) -> Int {
	return a + b
}

add(3, 4) // 7

func add(a: Int)(b: Int) -> Int { // 等价于上面的 add 函数
	return a + b
}

let addThree = add(3) // addThree 是一个新的函数，其可接受一个参数

addThree(4) // 7

```

由此，我们改造`sexualIntercourse`和`hug`如下：

``` swift
func sexualIntercourse(minutes: Int)(forplaySatisfied: Bool) -> Bool? {
    //...
}

func hug(minutes: Int, smallTalk: Bool)(sexualIntercourseSatisfied: Bool) -> Bool? {
    //...
}
```

注意`hug`函数，表明了我们对多参数的分割可以在任意位置。事实上，我们也可以写为：

``` swift
func hug(minutes: Int)(smallTalk: Bool)(sexualIntercourseSatisfied: Bool) -> Bool? {
    //...
}
```

当然使用时会更加灵活。

之后，我们就能使用这两个柯里化函数了：

``` swift
//...
satisfied = bind(satisfied, sexualIntercourse(sexualIntercourseMinutes)) ?? false
satisfied = bind(satisfied, hug(hugMinutes, smallTalk)) ?? false

//...
```

此时，我们用`bind`将我们的过程顺序链接起来了。不过`bind`的语法依然让人困惑，我们可以增加一个中缀操作符：

``` swift
infix operator >>> { associativity left precedence 160 }

func >>><A, B>(a: A?, f: A -> B?) -> B? {
    return bind(a, f)
}
```

然后对应的计算就变为：

``` swift
let satisfied = forplayMinutes >>> forplay >>> sexualIntercourse(sexualIntercourseMinutes) >>> hug(hugMinutes, smallTalk) ?? false
```

变成一条链了，顺序非常清楚。当然，我们写成这样可能更好看：

``` swift
let satisfied = forplay(forplayMinutes) >>> sexualIntercourse(sexualIntercourseMinutes) >>> hug(hugMinutes, smallTalk) ?? false
```

三个过程会更清楚。

在函数式编程的思想下，借助柯里化技术与自定义操作符，我们就能做到这样的“高阶操作”。

上面只是将数据和函数bind在一起处理，那若是其它“合成”情况，或更高阶呢？例如有某个函数接受两个函数作为参数并生成一个新的函数又会怎么样（完全不理会数据了）。只要发挥你的想象力，那函数式编程也就不再神秘。

##小结

函数式编程就是将函数当作一种数据类型，或者从更高的层面去看待数据和函数，使其可传递、可生成、可组合、可分解。有了这样的高级抽象，我们对函数的理解会更深刻，我们思考时就能更加自由。

有一点可能很关键，就是“形式”。虽然都是代码，完成的功能也一样，但形式的不同能反映我们思考的不同。语言可以影响思维早就被发现了，这也是类似于“新闻联播”这样的节目可以成为思想控制工具的原因。在生活中，独立思考的阻碍比你想象的要多得多。

如果你已开始用 Swift 写代码，可以考虑在某些地方试用这样的抽象观点，必有益处。

最后是个预告，我最近在写一本关于算法的书（代码用 Swift 2），不会是系统的算法讲解，而是从具体例子实现一些“综合性”的算法，重点在于分析的过程。但只刚开了个头，希望能在 Swift 2 正式版发布前完成，似乎时间不多了，不敢保证。

===============

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

如果你认为这篇文章不错，也有闲钱，那你可以用支付宝扫描下方二维码随便捐助一点，以慰劳作者的辛苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)
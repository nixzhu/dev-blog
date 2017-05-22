# 基于栈的HTML解析器

点渐渐连成了线

作者：[@nixzhu](https://twitter.com/nixzhu)

---

在前一篇[解析器组合子](https://github.com/nixzhu/dev-blog/blob/master/2017-04-12-json-parser.md)之后，我对它算是入了门。最近同事想写一个HTML到Attributed String的转换器，但用第三方库生成的中间数据结构不能满足要求，因此我提议用解析器组合子来做这一步，我以为一个晚上就能搞定。

结果很明显，没有成功。原因是我实现的的解析器组合子还太弱，不能处理左递归（在解析JSON时这刚好不是一个问题）。

处理左递归并不是一件简单的事（至少对我来讲），因此睡了一觉之后，我想到了第二个方案：基于栈的解析。

简单来说，HTML是一些配对标签包裹着字符串或其它配对标签的序列。在解析的过程中，判断当前Token的表意，来决定是否将其压入栈中，或者从栈中取出对应的Token来合成“值”并重新压入栈中。最后，我们将栈中的数据变成一个序列即可。

如果我们只关注特定的一些标签，例如加粗、斜体、段落、链接。那么Token的定义可如下：

``` swift
enum Token {
    case plainText(string: String)
    case beginBoldTag
    case endBoldTag
    case beginItalicTag
    case endItalicTag
    case beginParagraphTag
    case endParagraphTag
    case beginAnchorTag(href: String)
    case endAnchorTag
}
```

解析器组合子并非一无是处，在没有左递归的场合，它依然能工作得很好。比如生成Token串的这一步。

例如解析纯文本：

``` swift
let plainText: Parser<Token> = {
    let letter = satisfy({ $0 != "<" && $0 != ">" })
    let string = map(many1(letter)) { String($0) }
    return map(string) { .plainText(string: $0) }
}()
```

解析加粗开始标签：

``` swift
let beginBoldTag: Parser<Token> = map(word("<b>")) { _ in .beginBoldTag }
```

等等，都很容易写出。

然后利用这些小的解析器，我们就可以做tokenize了：

``` swift
func tokenize(_ htmlString: String) -> [Token] {
    var tokens: [Token] = []
    var remainder = htmlString.characters
    let parsers = [
        plainText,
        beginBoldTag,
        endBoldTag,
        beginItalicTag,
        endItalicTag,
        beginParagraphTag,
        endParagraphTag,
        beginAnchorTag,
        endAnchorTag
    ]
    while true {
        guard !remainder.isEmpty else { break }
        let remainderLength = remainder.count
        for parser in parsers {
            if let (token, newRemainder) = parser(remainder) {
                tokens.append(token)
                remainder = newRemainder
            }
        }
        let newRemainderLength = remainder.count
        guard newRemainderLength < remainderLength else {
            break
        }
    }
    return tokens
}
```

接下来该做解析了，先定义解析的目标，这与我们之前对（简化的）HTML的分析一致：

``` swift
indirect enum Value {
    case plainText(string: String)
    case boldTag(value: Value)
    case italicTag(value: Value)
    case paragraphTag(value: Value)
    case anchorTag(href: String, value: Value)
    case sequence(values: [Value])
}
```

不过需要考虑的是，我们压入栈中的元素可能是Token，也可能是Value，但我们也知道Swift的Array只能装同一种类型的数据。因此我们用enum包装一下：

``` swift
enum Element {
    case token(Token)
    case value(Value)
}
```

这样，Stack就很容易实现了：

``` swift
class Stack {
    var array: [Element] = []

    func push(_ element: Element) {
        array.append(element)
    }

    func pop() -> Element? {
        guard !array.isEmpty else { return nil }
        return array.removeLast()
    }
}
```

最后，我们编写解析函数：

``` swift
func parse(_ tokens: [Token]) -> Value {
    let stack = Stack() // # 1
    var next = 0
    func _parse() -> Bool {
        guard next < tokens.count else { // # 3
            return false
        }
        let token = tokens[next]
        switch token {
        case .plainText(let string): // # 4
            stack.push(.value(.plainText(string: string)))
        case .beginBoldTag: // # 5
            stack.push(.token(.beginBoldTag))
        case .endBoldTag: // # 6
            var elements: [Element] = []
            while let element = stack.pop() {
                if case .token(let value) = element {
                    if case .beginBoldTag = value {
                        break
                    }
                }
                elements.append(element)
            }
            if elements.count == 1 { // # 7
                let element = elements[0]
                if let value = element.value {
                    stack.push(.value(.boldTag(value: value)))
                } else {
                    print("todo: \(elements)")
                }
            } else {
                stack.push(.value(.boldTag(value: .sequence(values: elements.reversed().map({ $0.value }).flatMap({ $0 })))))
            }
        case .beginItalicTag:
            stack.push(.token(.beginItalicTag))
        case .endItalicTag:
            // ...
        case .beginParagraphTag:
            stack.push(.token(.beginParagraphTag))
        case .endParagraphTag:
            // ...
        case .beginAnchorTag(let href):
            stack.push(.token(.beginAnchorTag(href: href)))
        case .endAnchorTag:
            // ...
        }
        return true
    }
    while true { // # 2
        if !_parse() {
            break
        }
        next += 1
    }
    return .sequence(values: stack.array.map({ $0.value }).flatMap({ $0 })) // # 8
}
```

我将类似的部分删除了，简单说明一下：

1. 先创建一个栈，定义next指针，它用来从tokens数组中取下一个token；
2. 然后我们不停地调用`_parse()`函数，当然每次都要增加next，同时利用`_parse()`的返回值来做循环的退出；
3. 在`_parse()`中，确保next不越界（刚好做退出条件），然后判断当前的token；
4. 遇到plainText，直接压入栈中；
5. 遇到开始标签，也压入栈中；
6. 遇到结束标签，就从栈中取出对应到此结束标签的开始标签及之间的所有元素；
7. 通过判断元素的个数，我们决定是将其变成某个特定标签的Value再压入栈中，还是将其变为sequence（注意顺序）作为对应标签的value；
8. 最后将栈中的所有元素作为sequence变成Value返回。

大概的逻辑就这么多。如果要解析更多标签，可对应增加Token和Value的case以及解析判断。

你可在[此处](https://github.com/nixzhu/algorithm-playgrounds/blob/master/stack-based-html-parser.playground/Contents.swift)获取完整的Playground代码。

## 小结

写解析器是一项很好的编程活动，除了能提高代码编写技术（将多种知识结合起来），也能丰富我们的思考方式。事实上，我认为与编译器相关的技术，例如解析器、自动机、中间表示等都是很实在的技术，日常学习或重温都很有好处。同样，作为程序员，也要时时温习数据结构与算法。

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条
 
* Tweet [https://twitter.com/nixzhu/status/866508356856328192](https://twitter.com/nixzhu/status/866508356856328192) 或
* 微博 [http://weibo.com/2076580237/F4gOY1vgz](http://weibo.com/2076580237/F4gOY1vgz)

以分享此文或参与讨论！
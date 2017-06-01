# 解析器组合子

在实现小巧的JSON解析器的过程中，体会函数式编程的含义。

作者：[@nixzhu](https://twitter.com/nixzhu)

---

[Coolie](https://github.com/nixzhu/Coolie)是我自学编译原理（也只学了皮毛）后的一个练习作（它的实现思路在[制作一个苦力](https://github.com/nixzhu/dev-blog/blob/master/2016-06-29-coolie.md)里有所介绍）。

在写完Coolie之后，我又发现了“解析器组合子”这种东西，据说是专门用来写解析器的。它看起来很酷，但可惜那时接触到的资料还不多，没有深入学习，更不敢用这种技术重新写Coolie的解析部分。

后来又断断续续地看了一些资料，也尝试着去了解一些相关如Monad的知识。到现在我依然觉得Monad太高深了（可能是没有对应的数学知识储备），不过我们并不需要完全理解它（本文亦不会再提及它）。

写解析器组合子的文章已经不少，为何我又来写一篇？因为，我希望读者看完本文后会有豁然开朗的感觉，至少我希望我能在半年前看到我自己写的这篇文章。

依然请先看看JSON的定义[http://www.json.org/json-zh.html](http://www.json.org/json-zh.html)，简单来说，一个JSON的Value可能是一个字符串，一个数字，一个对象，一个数组，布尔值以及null，而其中“对象”和“数组”又由Value递归定义，即，对象是关于Value的字典，数组是关于Value的数组。可用代码描述如下：

``` swift
enum Value {
    case null
    case bool(Bool)
    enum Number {
        case int(Int)
        case double(Double)
    }
    case number(Number)
    case string(String)
    indirect case object([String: Value])
    indirect case array([Value])
}
```

注意，如果可能，我希望这篇文章的读者能打开Xcode（版本8.3以上，支持Swift 3.1），新建一个Playground来实验本文列出的代码。如果有iPad，也可以使用Apple新近推出的[Playgrounds](http://www.apple.com/cn/swift/playgrounds/)应用。

接下来定义解析器。所谓解析器就是一个函数，它根据需要读取输入流（通常是一个字符串）开头的信息，如果满足它的需要，它就吃掉这些信息，并返回“结果”和剩下的输入流。这个“结果”可能有多种不同的类型，因此我们使用泛型，如下：

``` swift
typealias Stream = String.CharacterView
typealias Parser<A> = (Stream) -> (A, Stream)?
```

这里先定义一个`Stream`有性能方面的考虑，但不算重要。而结果是可选值表示解析有可能失败。

现在来实现一个解析字母`a`的解析器，它的类型一定是`Parser<Character>`：

``` swift
let a: Parser<Character>
```

我们已知Parser是一个函数，因此`a`的“值”也就是一个函数，因此我们用函数的闭包写法：

``` swift
let a: Parser<Character> = { stream in
    // TODO
}
```

函数体该怎么写呢？这个解析器要解析字母`a`，也就是说，`stream`必须以字母`a`开头，不然解析就会失败。这些思考已经足够写出：

``` swift
let a: Parser<Character> = { stream in
    guard let firstCharacter = stream.first, firstCharacter == "a" else { return nil }
    return (firstCharacter, stream.dropFirst())
}
```

我们利用guard来确保`stream`以字母`a`开头，不然返回`nil`表示解析失败。之后当然表示解析成功，因此直接返回符合Parser定义的tuple。

虽然这个解析器只能解析字母`a`，但我们也应该测试一下：

``` swift
a("abc".characters)
```

希望你已经在Playground的右边栏看到输出了（类似`(.0 "a", {{…}, _coreOffset 1})`），不过这个输出不太美观，因为tuple里包含的是Stream而不是String，而且输入参数也不能直接用String。那就新增一个函数：

``` swift
func test<A>(_ parser: Parser<A>, _ input: String) -> (A, String)? {
    guard let (result, remainder) = parser(input.characters) else { return nil }
    return (result, String(remainder))
}
```

这个函数也很简单，它将String转换为Stream再传给parser，最后把输出包装一下。然后执行：

``` swift
test(a, "abc")
```

右边栏看到的输出将类似`(.0 "a", .1 "bc")`，此tuple的第0个元素是字符`a`，第1个元素表示余下的输入部分，即字符串`bc`。

我们已经有了一个可以解析字母`a`的解析器，但它的用处不大。如果我们可以写一个函数，它能根据输入的字符帮我们生成一个解析器，那就很有用了，我们可以直接写出其签名：

``` swift
func character(_ character: Character) -> Parser<Character> {
    // TODO
}
```

它要返回的是一个Parser，也就是一个函数，因此很容易写出：

``` swift
func character(_ character: Character) -> Parser<Character> {
    let parser: Parser<Character> = { stream in
        // TODO
    }
    return parser
}
```

有了之前写`a`的经验，这里的TODO也很好填充：

``` swift
func character(_ character: Character) -> Parser<Character> {
    let parser: Parser<Character> = { stream in
        guard let firstCharacter = stream.first, firstCharacter == character else { return nil }
        return (firstCharacter, stream.dropFirst())
    }
    return parser
}
```

然后，我们可以很容易地写出`b`，并测试它：

``` swift
let b = character("b")
test(b, "bcd")
```

我们的能力又大了一些，但很可惜解析单个字母的用处仍然有限。比如对于单词`null`，我们目前就无能为力。那就来写一个函数，它接受一个单词，并返回一个Parser<String>：

``` swift
func word(_ string: String) -> Parser<String> {
    let parsers = string.characters.map({ character($0) })
    let parser: Parser<String> = { stream in
        var characters: [Character] = []
        var remainder = stream
        for parser in parsers {
            guard let (character, newRemainder) = parser(remainder) else { return nil }
            characters.append(character)
            remainder = newRemainder
        }
        return (String(characters), remainder)
    }
    return parser
}
```

这一回，我们的步子迈得有些大，但还不至于扯着蛋。仔细来看，我们先利用之前定义的`character`得到一个`parsers`数组，然后准备一个`parser`以便返回；这个`parser`自然接受一个Stream作为参数；因为我们要依次解析每一个字母，所以先准备一个`characters`数组做容器；然后就是依次调用各个字母parser，中间有任何一次失败就算失败，最后成功就将字符数组转换为String，和剩下的remainder一起返回。

有了这个新的工具函数，我们可以定义`null`并测试：

``` swift
let null = word("null")
test(null, "null!")
```

输出的tuple里第0个元素是字符串，我们希望将其改成Value的null case，该怎么办呢？我们可以将结果tuple送入一个函数，它将第0个元素做变换再输出。不过我们也可以写一个更通用的函数，思路与前面一样，它接受一个Parser和一个转换函数，并生成一个新的Parser，如下：

``` swift
func map<A, B>(_ parser: Parser<A>, _ transform: (A) -> B) -> Parser<B> {
    // TODO
}
```

根据函数的签名，我们需要返回一个新的Parser，transform只能执行在结果上，因此肯定要调用参数parser，于是：

``` swift
func map<A, B>(_ parser: @escaping Parser<A>, _ transform: @escaping (A) -> B) -> Parser<B> {
    let newParser: Parser<B> = { stream in
        guard let (result, remainder) = parser(stream) else { return nil }
        return (transform(result), remainder)
    }
    return newParser
}
```

注意又加了`@escaping`标记，因为`parser`也是函数，它和`transform`都会逃逸。

由此，我们可以重新定义`null`并测试（记得注释掉之前的null）:

``` swift
let null = map(word("null"), { _ in Value.null })
test(null, "null!")
```

或者用更好看的尾随闭包：

``` swift
let null = map(word("null")) { _ in Value.null }
test(null, "null!")
```

感觉很棒，而且我们还可以定义`true`和`false`：

``` swift
let `true` = map(word("true")) { _ in true }
let `false` = map(word("false")) { _ in false }
test(`true`, "true?")
test(`false`, "false?")
```

可惜Value里只有bool case，也就是说，我们需要将`true`或`false`转换为`bool`。很明显map可以做这件事，但我们缺少一个“或”。它当然也是一个函数，接受两个Parser，生成一个新的Parser：

``` swift
func or<A>(_ leftParser: @escaping Parser<A>, _ rightParser: @escaping Parser<A>) -> Parser<A> {
    let parser: Parser<A> = { stream in
        return leftParser(stream) ?? rightParser(stream)
    }
    return parser
}
```

我们在要返回的parser里，先尝试用leftParser解析stream，不行就换rightParser来解析stream，如果也不行，会返回nil

然后就可以定义`bool`并测试：

``` swift
let bool = map(or(`true`, `false`)) { bool in Value.bool(bool) }
test(bool, "true?")
test(bool, "false?")
```

我们已有了`null`和`bool`，接下来该`number`了。

我们知道一个数是由一个个数字组成的，因此我们先准备一些数字的Parser：

``` swift
let digitCharacters = "0123456789.-".characters.map { $0 }
let digitParsers = digitCharacters.map { character($0) }
```

稍微注意这里考虑了小数点和负数。

那么，一个`digit`Parser就是尝试从digitParsers中选一个来匹配输入的stream的第一个字符，所以我们需要一个新函数。它接受一个Parser数组，并生成一个新的Parser。它要做的，就是让这个Parser数组里的每一个都去尝试解析stream，若有一个成功就算成功，若都没有成功，就算失败：

``` swift
func one<A>(of parsers: [Parser<A>]) -> Parser<A> {
    let parser: Parser<A> = { stream in
        for parser in parsers {
            if let x = parser(stream) {
                return x
            }
        }
        return nil
    }
    return parser
}
```

由此：

``` swift
let digit = one(of: digitParsers)
test(digit, "123")
```

不过就算如此，我们也只能解析单个数字，而真实情况里，一个数通常都有多个数字（大于等于1个）。因此，又来，我们需要一个新的函数，它能将一个Parser变成一个能连续解析多次的Parser：

``` swift
func many<A>(_ parser: @escaping Parser<A>) -> Parser<[A]> {
    let parser: Parser<[A]> = { stream in
        var result = [A]()
        var remainder = stream
        while let (element, newRemainder) = parser(remainder) {
            result.append(element)
            remainder = newRemainder
        }
        return (result, remainder)
    }
    return parser
}
```

`many`很类似之前的`word`，我们从观察其签名开始，就大概知道该怎么实现它，希望你已经熟悉这种模式。它用一个while循环来不断尝试传入的parser，直到失败，然后返回结果数组。

我们也很容易知道，`many`是不会失败的，它至少能得到一个空数组。可我们知道，一个数不能没有任何数组，因此，我们需要一个有限制的many1：

``` swift
func many1<A>(_ parser: @escaping Parser<A>) -> Parser<[A]> {
    let parser: Parser<[A]> = { stream in
        guard let (element, remainder1) = parser(stream) else { return nil }
        if let (array, remainder2) = many(parser)(remainder1) {
            return ([element] + array, remainder2)
        } else {
            return ([element], remainder1)
        }
    }
    return parser
}
```

`many1`与`many`的不同之处在于，它会先解析一次，如果成功，再利用`many`得到新的解析器来解析剩下的部分，这时，就算剩下的部分不能解析出什么，它也至少能返回只有一个元素的数组。当然，`many1`是可能失败的。

有了上面的准备，`number`呼之欲出：

``` swift
let number: Parser<Value> = map(many1(digit)) {
    let numberString = String($0)
    if let int = Int(numberString) {
        return Value.number(.int(int))
    } else {
        let double = Double(numberString)!
        return Value.number(.double(double))
    }
}
```

再测试一下：

``` swift
test(number, "-123.34")
    .flatMap({ print($0) }) // 可能需要打印才能看到结果
```

为了写出`number`，我们耗费了不少脑细胞，不如再来回顾一下：

1. `digit`：从digitParsers中遍历选择某一个能够解析成功的parser去解析；
2. `many1`：将一个Parser至少重复一次，去不断地解析输入，直到失败为止；
3. `number`: 利用解析出的字符数组生成字符串，再先尝试构造为Int，不成功就一定是Double。

那么接下来就该`string`了。

JSON里的字符串，不论作为Key还是Value，都是由双引号包裹的，形如`"key"`、`"value"`。这给了我们灵感，我们需要解析一个引号，再解析一个符合条件的字符串，最后再解析一个引号。虽然要连续解析三个部分，但我们关心的其实是中间的部分。因此，我们要写一个函数。它接受3个Parser，返回一个和中间的Parser一样类型的Parser：

``` swift
func between<A, B, C>(_ a: @escaping Parser<A>, _ b: @escaping Parser<B>, _ c: @escaping Parser<C>) -> Parser<B> {
    let parser: Parser<B> = { stream in
        guard let (_, remainder1) = a(stream) else { return nil }
        guard let (result2, remainder2) = b(remainder1) else { return nil }
        guard let (_, remainder3) = c(remainder2) else { return nil }
        return (result2, remainder3)
    }
    return parser
}
```

很明显，a、b、c三者都需要消耗输入，但我们只在乎第二个结果。由此：

``` swift
let quotedString: Parser<String> = {
    let lowercaseParsers = "abcdefghijklmnopqrstuvwxyz".characters.map({ character($0) })
    let uppercaseParsers = "ABCDEFGHIJKLMNOPQRSTUVWXYZ".characters.map({ character($0) })
    let otherParsers = " \t_-".characters.map({ character($0) }) // TODO: more
    let letter = one(of: lowercaseParsers + uppercaseParsers + otherParsers)
    let _string = map(many1(letter)) { String($0) }
    let quote = character("\"")
    return between(quote, _string, quote)
}()
```

和定义`digit`类似，我们定义了`letter`，进而定义了`_string`，乃至利用`between`得到`quotedString`。

那么，`string`就有了，并测试：

``` swift
let string = map(quotedString) { Value.string($0) }
test(string, "\"name\"")
```

不知不觉，我们就要来到关键的地方，再观察一下Value：

``` swift
enum Value {
    case null
    case bool(Bool)
    enum Number {
        case int(Int)
        case double(Double)
    }
    case number(Number)
    case string(String)
    indirect case object([String: Value])
    indirect case array([Value])
}
```

我们已经能解析`null`、`bool`、`number`以及`string`，不过剩下的`object`和`array`不太一样，它们会递归使用Value，而我们的最终目的也是写一个`value`解析器。

也就是说，在我们实现`value`前，我们不能实现`object`和`array`，但`value`又包含`object`和`array`（需要从包括它们的case中选择）。

这算不算两难的境地呢？好在我们可以利用一个技巧，Swift的闭包特性，让`value`捕获一个变量，而这个变量将在之后被赋值：

``` swift
var _value: Parser<Value>?
let value: Parser<Value> = { stream in
    if let parser = _value {
        return parser(stream)
    }
    return nil
}
```

如上所示， 解析器`value`会利用`_value`的实现，而我们可以推迟实现`_value`。注意，其实`value`变成了一个“闭包”，因为它捕获了一个外部变量。

这样的话，我们就能定义`object`了：

``` swift
func and<A, B>(_ left: @escaping Parser<A>, _ right: @escaping Parser<B>) -> Parser<(A, B)> {
    return { stream in
        guard let (result1, remainder1) = left(stream) else { return nil }
        guard let (result2, remainder2) = right(remainder1) else { return nil }
        return ((result1, result2), remainder2)
    }
}

func eatRight<A, B>(_ left: @escaping Parser<A>, _ right: @escaping Parser<B>) -> Parser<A> {
    return { stream in
        guard let (result1, remainder1) = left(stream) else { return nil }
        guard let (_, remainder2) = right(remainder1) else { return nil }
        return (result1, remainder2)
    }
}

func list<A, B>(_ parser: @escaping Parser<A>, _ separator: @escaping Parser<B>) -> Parser<[A]> {
    return { stream in
        let separatorThenParser = and(separator, parser)
        let parser = and(parser, many(separatorThenParser))
        guard let (result, remainder) = parser(stream) else { return nil }
        let finalResult = [result.0] + result.1.map({ $0.1 })
        return (finalResult, remainder)
    }
}

let object: Parser<Value> = {
    let beginObject = character("{")
    let endObject = character("}")
    let colon = character(":")
    let comma = character(",")
    let keyValue = and(eatRight(quotedString, colon), value)
    let keyValues = list(keyValue, comma)
    return map(between(beginObject, keyValues, endObject)) {
        var dictionary: [String: Value] = [:]
        for (key, value) in $0 {
            dictionary[key] = value
        }
        return Value.object(dictionary)
    }
}()
```

请一定不要被这看似较大的一步吓倒，因为并没有发生多么复杂的事情。根据JSON的定义，我们知道object是一个字典，也就是由大括号包围的由逗号分隔的键值对。

为了实现键值对，我们实现了`and`和`eatRight`。其中`and`拼接两个Parser为一个，类似之前的`or`，而`eatRight`类似于`and`，只不过它丢弃了右边的结果。`list`要稍稍复杂一些，但也很好理解，它先利用`and`得到一个separatorThenParser，再利用`and`和`many`得到所需的parser，接着解析，再把结果整理出来。

这样，`object`的实现也就没有什么奇怪的地方了。

不过，若我们测试一下：

``` swift
test(object, "{\"name\":\"NIX\",\"age\":18}")
```

会发现并不会成功，原因也很明显，`_value`还没有具体的实现，不过我们还要先实现`array`：

``` swift
let array: Parser<Value> = {
    let beginArray = character("[")
    let endArray = character("]")
    let comma = character(",")
    let values = list(value, comma)
    return map(between(beginArray, values, endArray)) { Value.array($0) }
}()
```

得益于已有的`list`，依据JSON的定义，array不过是中括号包围的由逗号分隔的一些value而已。最后，我们再补上`_value`的实现：

``` swift
_value = one(of: [null, bool, number, string, array, object])
```

这时，再测试一下：

``` swift
test(object, "{\"name\":\"NIX\",\"age\":18}")
    .flatMap({ print($0) })
```

甚至更复杂的：

``` swift
let jsonString = "{\"name\":\"NIX\",\"age\":18,\"detail\":{\"skills\":[\"Swift on iOS\",\"C on Linux\"],\"projects\":[{\"name\":\"coolie\",\"intro\":\"Generate models from a JSON file\"},{\"name\":\"parser\",\"intro\":null}]}}"
test(value, jsonString)
    .flatMap({ print($0) })

let jsonString2 = "[{\"name\":\"coolie\",\"intro\":\"Generate models from a JSON file\"},{\"name\":\"parser\",\"intro\":null}]"
test(value, jsonString2)
    .flatMap({ print($0) })
```

输出大概类似：

``` swift
(__lldb_expr_71.Value.object(["name": __lldb_expr_71.Value.string("NIX"), "age": __lldb_expr_71.Value.number(__lldb_expr_71.Value.Number.int(18)), "detail": __lldb_expr_71.Value.object(["projects": __lldb_expr_71.Value.array([__lldb_expr_71.Value.object(["name": __lldb_expr_71.Value.string("coolie"), "intro": __lldb_expr_71.Value.string("Generate models from a JSON file")]), __lldb_expr_71.Value.object(["name": __lldb_expr_71.Value.string("parser"), "intro": __lldb_expr_71.Value.null])]), "skills": __lldb_expr_71.Value.array([__lldb_expr_71.Value.string("Swift on iOS"), __lldb_expr_71.Value.string("C on Linux")])])]), "")

(__lldb_expr_71.Value.array([__lldb_expr_71.Value.object(["name": __lldb_expr_71.Value.string("coolie"), "intro": __lldb_expr_71.Value.string("Generate models from a JSON file")]), __lldb_expr_71.Value.object(["name": __lldb_expr_71.Value.string("parser"), "intro": __lldb_expr_71.Value.null])]), "")

```

作为读者，如果你坚持到了这一步，那么我要恭喜你！如果中间有不明白的步骤，请多思考函数的类型（参数的类型，返回值的类型），结合JSON的定义。

最终的`value`解析器仍然有待改进，比如，现在还不能处理JSON里的空格，字符串与数字的定义有待完善等（我相信有心的读者可以自己修改实现），或者提供更好的错误提示，等等。

本文的Playground也放在了[GitHub](https://github.com/nixzhu/algorithm-playgrounds/blob/master/json-parser.playground/Contents.swift)，但我希望读者已经跟随本文写出了它，仅供参考。

补记：趁着周末的时间，利用解析器组合子，写了一个[Baby](https://github.com/nixzhu/Baby)。功能类似Coolie，从JSON文件生成Swift模型。目前还没有Coolie强，但它会变得更强。

## 小结

得益于Swift 3.1的改进，上面所有的解析器组合子都是函数。我们思考的过程也是先想要做某件事，假设有某个函数，再考虑这个函数的类型（参数和返回值），最后才考虑具体的实现。也就是说，这是一种自顶向下的思维方式。反过来，我们实现的复杂的解析器都是由简单的解析器通过一些函数组合而成的。

而且要注意，我故意没有使用任何自定义运算符，虽然它们可能会让代码看起来更酷，但会增加理解负担。

最后，我们似乎体会到一种Lisp的编码风格。😊　

---

参考资料：

1. [Functional Swift](https://www.objc.io/books/functional-swift/)
2. [The "Understanding Parser Combinators" series](http://fsharpforfunandprofit.com/series/understanding-parser-combinators.html)

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

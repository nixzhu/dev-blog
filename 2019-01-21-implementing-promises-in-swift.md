# 使用 Swift 实现 Promise

本文翻译自：[https://felginep.github.io/2019-01-06/implementing-promises-in-swift](https://felginep.github.io/2019-01-06/implementing-promises-in-swift)

原作者：[Pierre Felgines ](https://felginep.github.io/about/)

译者：[@nixzhu](https://twitter.com/nixzhu)

---

我最近在找如何使用 Swift 实现 Promise 的资料，因为没找到好的文章，所以我想自己写一篇。通过本文，我们将实现自己的 Promise 类型，以便明了其背后的逻辑。

要注意这个实现完全不适合生产环境。例如，我们的 Promise 没有提供任何错误机制，也没有覆盖线程相关的场景。我会在文章的后面提供一些有用的资源以及完整实现的链接，以飨愿深入挖掘的读者。

注：为了让本教程更有趣一点，我选择使用 TDD 来进行介绍。我们会先写测试，然后确保它们一个个通过。

## 第一个测试

先写第一个测试：

``` swift
test(named: "0. Executor function is called immediately") { assert, done in
    var string: String = ""
    _ = Promise { string = "foo" }
    assert(string == "foo")
    done()
}
```

通过此测试，我们想实现：传递一个函数给 Promise 的初始化函数，并立即调用此函数。

注：我们没有使用任何测试框架，仅仅使用一个自定义的`test`方法，它在 Playground 中模拟断言（[gist](https://gist.github.com/felginep/039ca3b21e4f0cabb1c06126d9164680)）。

当我们运行 Playground，编译器会报错：

```
error: Promise.playground:41:9: error: use of unresolved identifier 'Promise'
    _ = Promise { string = "foo" }
        ^~~~~~~
```

合理，我们需要定义 Promise 类。

``` swift
class Promise {

}
```

再运行，错误变为：

```
error: Promise.playground:44:17: error: argument passed to call that takes no arguments
    _ = Promise { string = "foo" }
                ^~~~~~~~~~~~~~~~~~
```

我们必须定义一个初始化函数，它接受一个闭包作为参数。而且这个闭包应该被立即调用。

``` swift
class Promise {

    init(executor: () -> Void) {
        executor()
    }
}
```

由此，我们通过第一个测试。目前我们还没有写出什么值得夸耀的东西，但耐心一点，我们的实现将在下一节继续增长。

```
• Test 0. Executor function is called immediately passed 
```

我们先将此测试注释掉，因为将来的 Promise 实现会变得有些不同。

## 最低限度

第二个测试如下：

``` swift
test(named: "1.1 Resolution handler is called when promise is resolved sync") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        resolve(string)
    }
    promise.then { (value: String) in
        assert(string == value)
        done()
    }
}
```

这个测试挺简单，但我们添加了一些新内容到 Promise 类。我们创建的这个 promise 有一个 resolution handler（即闭包的`resolve`参数），之后立即调用它（传递一个 value）。然后，我们使用 promise 的`then`方法来访问 value 并用断言确保其值。

在开始实现之前，我们需要引入另外一个不太一样的测试。

``` swift
test(named: "1.2 Resolution handler is called when promise is resolved async") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        after(0.1) {
            resolve(string)
        }
    }
    promise.then { (value: String) in
        assert(string == value)
        done()
    }
}
```

不同于测试 1.1，这里的`resove`方法被延迟调用。这意味着，在`then`里，value 不会立马可用（因为 0.1 秒的延迟，调用`then`时，`resolve`还未被调用）。

我们开始理解这里的“问题”。我们必须处理异步。

我们的 promise 是一个状态机。当它被创建时，promise 处于`pending`状态。一旦`resolve`方法被调用（与一个 value），我们的 promise 将转到`resolved`状态，并存储这个 value。

`then`方法可在任意时刻被调用，而不管 promise 的内部状态（即不管 promise 是否已有一个 value）。当这个 promise 处于`pending`状态时，我们调用`then`，value 将不可用，因此，我们需要存储此回调。之后一旦 promise 变成`resolved`，我们就能使用 resolved value 来触发同样的回调。

现在我们对要实现的东西有了更好的理解，那就先以修复编译器的报错开始。

```
error: Promise.playground:54:19: error: cannot specialize non-generic type 'Promise'
    let promise = Promise<String> { resolve in
                  ^      ~~~~~~~~
```

我们必须给`Promise`类型添加泛型。诚然，一个 promise 是这样的东西：它关联着一个预定义的类型，并能在被解决时，将一个此类型的 value 保留住。

``` swift
class Promise<Value> {

    init(executor: () -> Void) {
        executor()
    }
}
```

现在错误为：

```
error: Promise.playground:54:37: error: contextual closure type '() -> Void' expects 0 arguments, but 1 was used in closure body
    let promise = Promise<String> { resolve in
                                    ^
```

我们必须提供一个`resolve`函数传递给初始化函数（即 executor）。

``` swift
class Promise<Value> {

    init(executor: (_ resolve: (Value) -> Void) -> Void) {
        executor()
    }
}
```

注意这个 resolve 参数是一个函数，它消耗一个 value：`(Value) -> Void`。一旦 value 被确定，这个函数将被外部世界调用。

编译器依然不开心，因为我们需要提供一个`resolve`函数给`executor`。让我们创建一个`private`的吧。

``` swift
class Promise<Value> {

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    private func resolve(_ value: Value) -> Void {
        // To implement
        // This will be called by the outside world when a value is determined
    }
}
```

我们将在稍后实现`resolve`，当所有错误都被解决时。

下一个错误很简单，方法`then`还未定义。

```
error: Promise.playground:61:5: error: value of type 'Promise<String>' has no member 'then'
    promise.then { (value: String) in
    ^~~~~~~ ~~~~
```

让我们修复之。

``` swift
class Promise<Value> {

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    func then(onResolved: @escaping (Value) -> Void) {
        // To implement
    }

    private func resolve(_ value: Value) -> Void {
        // To implement
    }
}
```

现在编译器开心了，让我们回到开始的地方。

我们之前说过一个`Promise`就是一个状态机，它有一个`pending`状态和一个`resolved`状态。我们可以使用 enum 来定义它们。

``` swift
enum State<T> {
    case pending
    case resolved(T)
}
```

Swift 的美妙让我们可以直接存储 promise 的 value 在 enum 中。

现在我们需要在`Promise`的实现中定义一个状态，其默认值为`.pending`。我们还需要一个私有函数，它能在当前还处于`.pending`状态时更新状态。

``` swift
class Promise<Value> {

    enum State<T> {
        case pending
        case resolved(T)
    }

    private var state: State<Value> = .pending

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    func then(onResolved: @escaping (Value) -> Void) {
        // To implement
    }

    private func resolve(_ value: Value) -> Void {
        // To implement
    }

    private func updateState(to newState: State<Value>) {
        guard case .pending = state else { return }
        state = newState
    }
}
```

注意`updateState(to:)`函数先检查了当前处于`.pending`状态。如果 promise 已经处于`.resolved`状态，那它就不能再变成其他状态了。

现在是时候在必要时更新 promise 的状态，即，当`resolve`函数被外部世界传递 value 调用时。

``` swift
private func resolve(_ value: Value) -> Void {
    updateState(to: .resolved(value))
}
```

快好了，只缺少`then`方法还未实现。我们说过必须存储回调，并在 promise 被解决时调用回调。这就来实现之。

``` swift
class Promise<Value> {

    enum State<T> {
        case pending
        case resolved(T)
    }

    private var state: State<Value> = .pending
    // we store the callback as an instance variable
    private var callback: ((Value) -> Void)?

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    func then(onResolved: @escaping (Value) -> Void) {
        // store the callback in all cases
        callback = onResolved
        // and trigger it if needed
        triggerCallbackIfResolved()
    }

    private func resolve(_ value: Value) -> Void {
        updateState(to: .resolved(value))
    }

    private func updateState(to newState: State<Value>) {
        guard case .pending = state else { return }
        state = newState
        triggerCallbackIfResolved()
    }

    private func triggerCallbackIfResolved() {
        // the callback can be triggered only if we have a value,
        // meaning the promise is resolved
        guard case let .resolved(value) = state else { return }
        callback?(value)
        callback = nil
    }
}
```

我们定义了一个实例变量`callback`，以在 promise 处于`.pending`状态时保留回调。同时我们创建一个方法`triggerCallbackIfResolved`，它先检查状态是否为`.resolved`，然后传递拆包的 value 给回调。这个方法在两个地方被调用。一个是`then`方法中，如果 promise 已经在调用`then`时被解决。另一个在`updateState`方法中，因为那是 promise 更新其内部状态从`.pending`到`.resolved`的地方。

有了这些修改，我们的测试就成功通过了。

```
• Test 1.1 Resolution handler is called when promise is resolved sync passed (1 assertions)
• Test 1.2 Resolution handler is called when promise is resolved async passed (1 assertions)
```

我们走对了路，但我们还需要做出一点改变，以得到一个真正的`Promise`实现。先来看看测试。

``` swift
test(named: "2.1 Promise supports many resolution handlers sync") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        resolve(string)
    }
    promise.then { value in
        assert(string == value)
    }
    promise.then { value in
        assert(string == value)
        done()
    }
}
```

``` swift
test(named: "2.2 Promise supports many resolution handlers async") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        after(0.1) {
            resolve(string)
        }
    }
    promise.then { value in
        assert(string == value)
    }
    promise.then { value in
        assert(string == value)
        done()
    }
}
```

这回我们对每个 promise 都调用了两次`then`。

先看看测试输出。

```
• Test 2.1 Promise supports many resolution handlers sync passed (2 assertions)
• Test 2.2 Promise supports many resolution handlers async passed (1 assertions)
```

虽然测试通过了，但你可能也注意问题。测试 2.2 只有一个断言，但应该是两个。

如果我们思考一下，这其实符合逻辑。诚然，在异步的测试 2.2 中，当第一个`then`被调用时，promise 还处于`.pending`状态。如我们之前所见，我们存储了第一次`then`的回调。但当我们第二次调用`then`时，promise 还是没有被解决，依然处于`.pending`状态，于是，我们将回调擦除换成了新的。只有第二个回调会在将来被执行，第一个被忘记了。这使得测试虽然通过，但只有一个断言而不是两个。

解决办法也很简单，就是存储一个回调的数组，并在promise被解决时触发它们。

让我们更新一下。

``` swift
class Promise<Value> {

    enum State<T> {
        case pending
        case resolved(T)
    }

    private var state: State<Value> = .pending
    // We now store an array instead of a single function
    private var callbacks: [(Value) -> Void] = []

    init(executor: (_ resolve: @escaping (Value) -> Void) -> Void) {
        executor(resolve)
    }

    func then(onResolved: @escaping (Value) -> Void) {
        callbacks.append(onResolved)
        triggerCallbacksIfResolved()
    }

    private func resolve(_ value: Value) -> Void {
        updateState(to: .resolved(value))
    }

    private func updateState(to newState: State<Value>) {
        guard case .pending = state else { return }
        state = newState
        triggerCallbacksIfResolved()
    }

    private func triggerCallbacksIfResolved() {
        guard case let .resolved(value) = state else { return }
        // We trigger all the callbacks
        callbacks.forEach { callback in callback(value) }
        callbacks.removeAll()
    }
}
```

测试通过，而且都有两个断言。

```
• Test 2.1 Promise supports many resolution handlers sync passed (2 assertions)
• Test 2.2 Promise supports many resolution handlers async passed (2 assertions)
```

恭喜！我们已经创建了自己的`Promise`类。你已经可以使用它来抽象异步逻辑，但它还有限制。

注：如果从全局来看，我们知道`then`可以被重命名为`observe`。它的目的是消费 promise 被解决后的 value，但它不返回什么。这意味着我们暂时没法串联多个 promise。

## 串联多个 Promise

如果我们不能串联多个 promise，那我们的`Promise`实现就不算完整。

先来看看测试，它将帮助我们实现这个特性。

``` swift
test(named: "3. Resolution handlers can be chained") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        after(0.1) {
            resolve(string)
        }
    }
    promise
        .then { value in
            return Promise<String> { resolve in
                after(0.1) {
                    resolve(value + value)
                }
            }
        }
        .then { value in // the "observe" previously defined
            assert(string + string == value)
            done()
        }
}
```

如测试所见，第一个`then`创建了一个有新 value 的新`Promise`并返回了它。第二个`then`（我们前一节定义的，被称为`observe`）被串联在后面，它访问新的 value（其将是`"foofoo"`）。

我们很快在终端里看到错误。

```
error: Promise.playground:143:10: error: value of tuple type '()' has no member 'then'
        .then { value in
         ^
```

我们必须创建一个`then`的重载，它接受一个能返回 promise 的函数。为了能够串联调用`then`，这个方法必须也返回一个promise。这个`then`的原型如下。

``` swift
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    // to implement
}
```

注：细心的读者可能已经发现，我们在给`Promise`实现`flatMap`。就如给`Optional`和`Array`定义`flatMap`一样，我们也可以给`Promise`定义它。

困难来了。让我们一步步看看这个“flatMap”的`then`要怎么实现。

- 我们需要返回一个`Promise<NewValue>`
- 谁给我们这样一个 promise？ `onResolved` 方法
- 但`onResolved` 需要一个类型为`Value`的 value 为参数。我们该怎样得到这个 value？  我们可以使用之前定义的`then`(或者说 “`observe`”) 来在其可用时访问它

如果写成代码，大概如下：

``` swift
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    then { value in // the "observe" one
        let promise = onResolved(value) // `promise` is a Promise<NewValue>
        // problem: how do we return `promise` to the outside ??
    }
    return // ??!!
}
```

就快好了。但我们还有个小问题需要修复：这个`promise`变量被传递给`then`的闭包所限制。我们不能将其作为函数的返回值。

我们要使用的技巧是创建一个包装`Promise<NewValue>`，它将执行我们目前所写的代码，然后在`promise`变量解决时被同时解决。换句话说，当`onResolved`方法提供的 promise 被解决并从外部得到一个值，那包装的 promise 也就被解决并得到同样的值。

可能文字有些抽象，但如果我们写成代码，将看得更清楚：

``` swift
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    // We have to return a promise, so let's return a new one
    return Promise<NewValue> { resolve in
        // this is called immediately as seen in test 0.
        then { value in // the "observe" one
            let promise = onResolved(value) // `promise` is a Promise<NewValue>
            // `promise` has the same type of the Promise wrapper
            // we can make the wrapper resolves when the `promise` resolves
            // and gets a value
            promise.then { value in resolve(value) }
        }
    }
}
```

如果我们整理一下代码，我们将有这样两个方法：

``` swift
// observe
func then(onResolved: @escaping (Value) -> Void) {
    callbacks.append(onResolved)
    triggerCallbacksIfResolved()
}

// flatMap
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    return Promise<NewValue> { resolve in
        then { value in
            onResolved(value).then(onResolved: resolve)
        }
    }
}
```

最后，测试通过。

```
• Test 3. Resolution handlers can be chained passed (1 assertions)
```

## 串联多个 value

如果你能给某个类型实现`flatMap`，那你就能利用`flatMap`为其实现`map`。对于我们的`Promise`来说，`map`该是什么样子？

我们将使用如下测试：

``` swift
test(named: "4. Chaining works with non promise return values") { assert, done in
    let string: String = "foo"
    let promise = Promise<String> { resolve in
        after(0.1) {
            resolve(string)
        }
    }
    promise
        .then { value -> String in
            return value + value
        }
        .then { value in // the "observe" then
            assert(string + string == value)
            done()
        }
}
```

注意第一个`then`没有再返回一个`Promise`，而是将其接收的值做了一个变换。这个新的`then`就对应于我们想添加的`map`。

编译器报错说我们必须实现此方法。

```
error: Promise.playground:174:26: error: declared closure result 'String' is incompatible with contextual type 'Void'
        .then { value -> String in
                         ^~~~~~
                         Void
```

这个方法很接近`flatMap`，唯一的不同是其参数`onResolved`函数返回一个`NewValue`而不是`Promise<NewValue>`。

``` swift
// map
func then<NewValue>(onResolved: @escaping (Value) -> NewValue) -> Promise<NewValue> {
    // to implement
}
```

之前我们说可以利用`flatMap`实现`map`。在我们的情况里，我们看到我们需要返回一个`Promise<NewValue>`。如果我们使用这个“flatMap”的`then`，并创建一个promise，再以映射后的 value 来直接解决，我们就搞定了。让我来证明之。

``` swift
// map
func then<NewValue>(onResolved: @escaping (Value) -> NewValue) -> Promise<NewValue> {
    return then { value in // the "flatMap" defined before
        // must return a Promise<NewValue> here
        // this promise directly resolves with the mapped value
        return Promise<NewValue> { resolve in
            let newValue = onResolved(value)
            resolve(newValue)
        }
    }
}
```

再一次，测试通过。

```
• Test 4. Chaining works with non promise return values passed (1 assertions)
```

如果我们移除注释，再看看我们做出了什么。我们有三个`then`方法的实现，能被使用或串联。

``` swift
// observe
func then(onResolved: @escaping (Value) -> Void) {
    callbacks.append(onResolved)
    triggerCallbacksIfResolved()
}

// flatMap
func then<NewValue>(onResolved: @escaping (Value) -> Promise<NewValue>) -> Promise<NewValue> {
    return Promise<NewValue> { resolve in
        then { value in
            onResolved(value).then(onResolved: resolve)
        }
    }
}

// map
func then<NewValue>(onResolved: @escaping (Value) -> NewValue) -> Promise<NewValue> {
    return then { value in
        return Promise<NewValue> { resolve in
            resolve(onResolved(value))
        }
    }
}
```

## 使用示例

实现告一段落。我们的`Promise`类已足够完整来展示我们能够用它做什么。

假设我们的app有一些用户，结构如下：

``` swift
struct User {
    let id: Int
    let name: String
}
```

假设我们还有两个方法，一个获取用户id列表，另一个使用id获取某个用户。而且假设我们想显示第一个用户的名字。

通过我们的实现，我们可以这样做，使用之前定义的这三个`then`。

``` swift
func fetchIds() -> Promise<[Int]> {
    ...
}

func fetchUser(id: Int) -> Promise<User> {
    ...
}

fetchIds()
    .then { ids in // flatMap
        return fetchUser(id: ids[0])
    }
    .then { user in // map
        return user.name
    }
    .then { name in // observe
        print(name)
    }
```

代码变得十分易读、简洁，而且没有嵌套。

## 结论

本文结束，希望你喜欢它。

你可以在这个 [gist](https://gist.github.com/felginep/039ca3b21e4f0cabb1c06126d9164680) 找到完整代码。如果你想进一步理解，下面是一些我使用的资源。

- [Promises in Swift by Khanlou](http://khanlou.com/2016/08/promises-in-swift/)
- [JavaScript Promises … In Wicked Detail](https://www.mattgreer.org/articles/promises-in-wicked-detail/)
- [PromiseKit 6 Release Details](https://promisekit.org/news/2018/02/PromiseKit-6.0-Released/)
- [TDD Implementation of Promises in JavaScript](https://www.youtube.com/watch?v=C3kUMPtt4hY)

------

欢迎转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！
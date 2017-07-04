# App的环境

将散乱的配置包装起来

作者：[@nixzhu](https://twitter.com/nixzhu)

---

这里所说的“环境”并不是操作系统或硬件如内存之类，而是app的依赖基础。

比如，当前的语言、服务器域名、内部缓存的引用，或一些配置信息等。此外，如果你的app有用户系统，那还要包括当前登录的用户。

有了环境，就可以改变它。比如改变服务器域名，简单令其指向localhost就可以得到一个调试环境。从这个意义上讲，一切可配置或可替换的变量都是环境的一部分。

当我们有了这样的概念，我们就可以定义它。将我们所关心的环境做成一个封装，并定义为一个全局变量（这个变量的存放位置可能需要考虑一下）。

``` swift
struct Environment {
    enum Language {
        case en
        case zh
    }
    let language: Language 
    let serverDomain: String
    let user: User?
}

var environment = Environment(
    language: .zh,
    serverDomain: "example.com",
    user: nil
)
```

这只是一个简单的例子，你的app所关心的环境很可能不同，自行增减属性。

根据上面的代码，假如app打开一段时间后，用户进行登录，若成功就可以修改`environment`，给它一个新的有user的值。

如果你要写测试代码，则可以配置测试环境，如：

``` swift
let environment = Environment(
    language: .en,
    serverDomain: "localhost:8080",
    user: User(...)
)
```

甚至于你想给app增加一个测试模式（需要特殊条件触发），以便临时切换环境。那可以定义一个环境栈：

``` swift
struct AppEnvironment {
    private init() {
    }
    private static var stack: [Environment] = []
    static var current: Environment? {
        return stack.last
    }

    static func pushEnvironment(_ environment: Environment) {
        stack.append(environment)
    }

    @discardableResult
    static func popEnvironment() -> Environment? {
        let last = stack.popLast()
        return last
    }
}
```

这样，在进入测试模式时，就将一个测试环境压栈中。当然，对应业务代码里要改为使用`AppEnvironment.current`替换之前的全局变量。至于UI，你可以直接替换window或者present一个最底层的Container ViewController覆盖当前的UI。当你测试结束，弹出测试环境，UI也对应调整一下即可。

假如你的app支持多用户，用户会不停地的切换账号。而有了环境这种概念，是不是也觉得实现起来易如反掌？

---

本文属于阅读[kickstarter/ios-oss](https://github.com/kickstarter/ios-oss)的体会之一（我希望能写成一个系列）。

我之所以想起它，是因为最近我参与开发的一个app：[SpaceHub](https://duodian.com)，在处理某些推送时需要实现一种临时切换环境的功能，算得上有实际用上这种经验。这大概是阅读好代码的益处之一（尽管比较功利）。

---

更多文章请见：[dev-blog/README](https://github.com/nixzhu/dev-blog/blob/master/README.md)。

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)
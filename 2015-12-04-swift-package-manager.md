
# Swift 包管理器

译者：[@nixzhu](https://twitter.com/nixzhu)

=================================

# 开发包

引用：[Developing Packages](https://github.com/apple/swift-package-manager/blob/master/Documentation/DevelopingPackages.md)

简言之：一个包即一个有着语义版本 tag 的 git 仓库，其中包含 Swift 源代码，以及一个放在根目录的 `Package.swift` 清单文件。

## 转换库模块（Library Module）为外部包（External Package）

如果你在构建一个有着几个模块的 app，在某个时间点，你可能决定将这些模块放到一个外部包中。这样做可以让代码变成一个可靠的库让其他人使用。

有了包管理器，要做到这一点比较简单：

 1. 在 GitHub 上创建一个新的代码仓库
 2. 在终端里，进入模块所在目录
 3. `git init`
 4. `git remote add origin [github-URL]`
 5. `git tag 1.0.0`
 5. `git push origin master --tags`
 
现在删除子目录，并修改你的 `Package.swift`，让其 `package` 声明包含如下信息：

```swift
let package = Package(
    dependencies: [
        .Package(url: "…", versions: "1.0.0"),
    ]
)
```

现在输入命令 `swift build`

## 同时开发 app 和包

如果你在开发的 app 使用了某个包，而你也需要同时改进这个包，那你有如下几个选择：

 1. **编辑包管理器克隆的代码**
    
    克隆的代码位于 `./Packages` 中

 2. **修改你的 `Package.swift`，让其指向一个本地的包克隆**

    这可能很乏味，因为你每次做了改变后，都要强制做一个更新，当然包括更新版本 tag。

目前，这两个选择都不是很理想，因为新提交的代码可能很容易破坏同组人的使用，例如，如果你修改了 `Foo` 的代码并让你的 app 使用这些改动，却并没有提交这些改动到 `Foo`，那么你就可能将你的同事置于代码依赖的地狱（caused dependency hell）。

我们会努力改进工具以避免类似问题，但目前，希望你知悉这些问题。

=================================

# `Package.swift` — 清单文件

引用：[Package.swift — The Manifest File](https://github.com/apple/swift-package-manager/blob/master/Documentation/Package.swift.md)

指示如何构建一个包的清单文件，被称为 `Package.swift`。你可以定制此文件以声明构建 target 或依赖，引用或排除源文件，以及为模块或单个文件指定构建配置。

如下是一个 `Package.swift` 文件的例子：

```swift
import PackageDescription

let package = Package(
    name: "Hello",
    dependencies: [
        .Package(url: "ssh://git@example.com/Greeter.git", versions: "1.0.0"),
    ]
)
```

明显， `Package.swift` 文件是一个 Swift 文件，它用定义在 `PackageDescription` 模块的类型来声明一个包的配置。这个清单声明了一个对外部包 `Greeter` 的依赖。

果你的‘包’包含多个互相依赖的 target，那么你需要指明它们的相互依赖关系。如下例所示：

```swift
import PackageDescription

let package = Package(
    name: "Example",
    targets: [
        Target(
            name: "top",
            dependencies: [.Target(name: "bottom")]),
        Target(
            name: "bottom")
```

target 的名字就是你子目录的名字。

## 自定义构建

清单文件本质为 Swift 源码，可带来极强的自定义性，例如：

```swift
import PackageDescription

var package = Package()

#if os(Linux)
let target = Target(name: "LinuxSources/foo")
package.targets.append(target)
#endif
```

对于这样的特性，标准配置文件格式，如 JSON，将导致字典结构对每一个特性都增加很多复杂性。

## 依赖 Apple 模块（如 Foundation）

当前，还没有明确支持依赖于 Foundation、AppKit 等，尽管这些模块在合适的系统位置时应该正常工作。我们将为系统依赖添加明确的支持。注意，目前包管理器还没有支持 iOS、watchOS 或 tvOS 平台。



# Swift 包管理器

译者：[@nixzhu](https://twitter.com/nixzhu)

=================================
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

=================================
=================================

# 源码布局

引用：[Source Layouts](https://github.com/apple/swift-package-manager/blob/master/Documentation/SourceLayouts.md)

`swift build` 创建的模块取决于源文件在文件系统里的布局。

例如，如果你创建一个目录包含如下布局：

    example/
    example/Sources/bar.swift
    example/Sources/baz.swift

在 `example` 目录中运行 `swift build` 会产生单个库目标文件：`example/.build/debug/example.a`

要创建多个模块，就创建多个子目录：

    example/Sources/foo/foo.swift
    example/Sources/bar/bar.swift

此时运行 `swift build` 将产生两个库目标文件：

* `example/.build/debug/foo.a`
* `example/.build/debug/bar.a`

要生成一个可执行模块（而不是一个库模块），添加一个 `main.swift` 到那个模块的子目录即可：

    example/Sources/foo/main.swift
    example/Sources/bar/bar.swift

此时运行 `swift build` 将产生：

* `example/.build/debug/foo`
* `example/.build/debug/bar.a`

这里的 `foo` 就是一个可执行文件，而 `bar.a` 是一个静态库。

## 其它规则

* 命名为 `Tests` 的目录会被忽略
* 若目录的子目录命名为 `Sources`、`Source`、`srcs` 或 `src`，它们将成为模块
* 没有 `Sources` 目录是可接受的，在这种情况下，根目录就被当做单个模块（将你的源代码放在这里），或者根目录的子目录会被认为是模块。对于简单项目，这种布局比较方便。

=================================
=================================

# 系统模块

引用：[System Modules](https://github.com/apple/swift-package-manager/blob/master/Documentation/SystemModules.md)

你可以使用包管理器链接系统库。

要做到这一点，指定的包必须被发布，并包含一个模块地图（module map）。

Let’s use the example of [IJG’s JPEG library](http://www.ijg.org). This is the code we want to compile:

让我们以 [IJG’s JPEG 库](http://www.ijg.org) 为例。如下是我们要编译的代码：

```swift
import CJPEG

let jpegData = jpeg_common_struct()
print(jpegData)
```

将代码放入一个叫做 `example` 目录里：

    $ mkdir example
    $ cd example
    example$ touch main.swift Package.swift
    example$ open -t main.swift Package.swift

要 `import CJPEG`，包管理器要求这个 JPEG 库已被某个系统包管理器（如`apt`、`brew`、`yum`等）安装。

    /usr/lib/libjpeg.so      # .dylib on OS X
    /usr/include/jpeglib.h


为成为系统库而提供有模块地图的 Swift 包被处理的方式和常规 Swift 包不同。（Swift packages that provide module maps for system libraries are handled differently from regular Swift packages.）

创建一个叫做 `CJPEG` 的目录，与 `example` 目录在同一层级，然后创建一个叫做 `module.modulemap` 的文件：

    example$ cd ..
    $ mkdir CJPEG
    $ cd CJPEG
    CJPEG$ touch module.modulemap

编辑 `module.modulemap`，包含如下信息：

    module CJPEG [system] {
        header "/usr/include/jpeglib.h"
        link "jpeg"
        export *
    }


> 我们期望社区会遵循的惯例是在这样的模块名前冠以字母`C`，并驼峰命名模块，一如每个 Swift 模块命名的惯例。然后社区就可以自由命名另外的模块为 `JPEG`，它将包含更多原始 C API 的 更 Swifty 的函数包装。
> （The convention we hope the community will adopt is to prefix such modules with `C` and to camelcase the modules
> as per Swift module name conventions. Then the community is free to name another module simply `JPEG` which
> contains more “Swifty” function wrappers around the raw C interface.）

包就是 Git 仓库，以语义标签指明版本号，并在根目录包含一个 `Package.swift` 文件。因此，我们必须创建 `Package.swift` 并初始化一个至少有一个版本 tag 的 Git 仓库： 

    CJPEG$ touch Package.swift
    CJPEG$ git init
    CJPEG$ git add .
    CJPEG$ git ci -m "Initial Commit"
    CJPEG$ git tag 1.0.0

* * *

Now to consume JPEG we must declare our dependency in our example app’s `Package.swift`:

现在要使用 JPEG，我们必须在 example app 的 `Package.swift` 里声明我们的依赖：

```swift
import PackageDescription

let package = Package(
    dependencies: [
        .Package(url: "../CJPEG", majorVersion: 1)
    ]
)
```

> 这里我们使用了一个相对路径的 URL 以加快初始开发。如果（我们希望）你将你的模块地图包 push 到公共仓库里，你必须修改上面的 URL 引用，改为一个完整合格的 git URL。

现在，如果我们在 example app 目录里输入 `swift build`，我们将创建一个可执行文件：

    example$ swift build
    …
    example$ .build/debug/example
    jpeg_common_struct(err: 0x0000000000000000, mem: 0x0000000000000000, progress: 0x0000000000000000, client_data: 0x0000000000000000, is_decompressor: 0, global_state: 0)
    example$


## 有依赖的模块地图

让我们扩展 example 以包含 [JasPer](https://www.ece.uvic.ca/~frodo/jasper/)，它是一个 JPEG-2000 库，且依赖于 JPEG 库。

首先，创建一个叫做 `CJasPer` 的目录，与 `CJPEG` 和我们的 example 同级。

    CJPEG$ cd ..
    $ mkdir CJasPer
    $ cd CJasPer
    CJasPer$ touch module.modulemap Package.swift

JasPer 依赖 JPEG，因此任何使用 `CJasPer` 的包都必须知晓要 import `CJPEG`。我们通过在 CJasPer 的 `Package.swift` 里指定依赖来达成这一点。

```swift
import PackageDescription

let package = Package(
    dependencies: [
        .Package(url: "../CJPEG", majorVersion: 1)
    ])
```

CJasPer 的模块地图类似 CJPEG 的：

    module CJasPer [system] {
        header "/usr/local/include/jasper/jasper.h"
        link "jasper"
        export *
    }

**当心**，模块地图必须指明此系统包使用的所有头文件，***但是***，你一定不能指定你的头文件已经指定过的头文件（译者：应该是要防止重复引用头文件）。例如，使用 JasPer 的 example 会包含许多头文件，但所有被 `jasper.h` 包含的都要避免。如果你引用不对，那么你可能不时会遇到编译问题，很难调试。

一个包就是有着语义标签指明版本的 Git 仓库，并包含一个 `Package.swift` 文件，所以我们必须创建一个 Git 仓库：

    CJasPer$ git init
    CJasPer$ git add .
    CJasPer$ git ci -m "Initial Commit"
    CJasPer$ git tag 1.0.0

**注意！**包管理器克隆_标签_。如果你编辑了 `module.modulemap` 但没有 `git tag -f 1.0.0`，那么你将不能构建本地的修改。

* * *

回到我们的 example app 的 `Package.swift`，我们改变依赖到 `CJasPer`：

```swift
import PackageDescription

let package = Package(
    dependencies: [
        .Package(url: "../CJasPer", majorVersion: 1)
    ])
```

JasPer 依赖 CJPEG，所以我们不再需要在 example app 的 Package.swift 里指明我们依赖 CJPEG。

修改 example 的 `main.swift` 以测试 JasPer 支持：

```swift
import CJasPer

guard let version = String.fromCString(jas_getversion()) else {
    fatalError("Could not get JasPer version")
}

print("JasPer \(version)")
```

然后运行：

    example$ swift build
    …
    example$ .build/debug/example
    JasPer 1.900.1
    example$

> 注意我们没有命名模块为 `CLibjasper`。通常，避免 lib 前缀，除非包的作者总是这样使用。一个好的规则是检查头文件，在此我们可以看到头文件为 "jasper.h"，没有前缀。在非典型头文件（例如 `jpeglib.h`）的情况下，参考项目的主页，JPEG 库的作者们称其为“JPEG 库（The JPEG library）”而不是“jpeglib”或“jpeglib”。注意大小写；是 `CJPEG` 而不是 `CJpeg`，因为 JPEG 是一个缩写，通常全大写。是 `CJasPer` 而不是 `CJasper`，因为在项目自身的文档里就叫自己为“JasPer”。

---

请注意在 Ubuntu 15.10 上，上面步骤会失败：

    <module-includes>:1:10: note: in file included from <module-includes>:1:
    #include "/usr/include/jpeglib.h"
             ^
    /usr/include/jpeglib.h:792:3: error: unknown type name 'size_t'
      size_t free_in_buffer;        /* # of byte spaces remaining in buffer */
      ^

这是因为 `jpeglib.h` 不是一个正确绑定到 Ubuntu 的模块（而 Homebrew 的 jpeglib.h 是对的）。在 jpeglib.h 顶部添加 `#include <stdio.h>` 可修正此问题。

JPEG lib 自身需要打补丁，但由于这个情况比较普遍，我们打算添加一个解决方案到模块包里。

## 提供多个库的包

一些系统包提供多个库（例如 `.so` 和 `.dylib` 文件）。在这种情况下，你需要将所有的库信息添加到 Swift 模块地图包的 `.modulemap` 文件里：

    module CFoo [system] {
        header "/usr/local/include/foo/foo.h"
        link "foo"
        export *
    }
    
    module CFooBar [system] {
        header "/usr/include/foo/bar.h"
        link "foobar"
        export *
    }
    
    module CFooBaz [system] {
        header "/usr/include/foo/baz.h"
        link "foobaz"
        export *
    }

`foobar` 和 `foobaz` 链接到 `foo`；我们不需要模块地图里指定这个信息，因为头文件 `foo/bar.h` 和 `foo/baz.h` 都包含依赖的头文件，此外，当模块被引入 Swift，依赖模块不会被自动引入，将引起链接错误。如果链接错误在包的用户处发生，将导致你的包很难调试。（`foobar` and `foobaz` link to `foo`;
we don’t need to specify this information in the module-map because
the headers `foo/bar.h` and `foo/baz.h` both include `foo/foo.h`.
It is very important however that those headers do include their dependent headers,
otherwise when the modules are imported into Swift the dependent modules will not get
imported automatically and link errors will happen.
If these link errors occur to consumers of a package that consumes your
package the link errors can be especially difficult to debug.）

## 跨平台的模块地图

模块地图必须包含绝对路径，因此它们不是跨平台的。我们打算在包管理器里提供一个解决方案。

长期来看，我们希望系统库和系统包都提供模块地图，那是，包管理器的这个组建就会变得多余。

*Notably* the above steps will not work if you installed JPEG and JasPer with [Homebrew](http://brew.sh) since the files will
be installed to `/usr/local` for now adapt the paths, but as said, we plan to support basic relocations like these.

*值得注意的是*上述步骤在使用 [Homebrew](http://brew.sh) 安装 JPEG 和 JasPer 时将不会工作，因为目前文件会被安装到 `/usr/local`，但如提到的，我们打算支持基本重定位。

## 模块地图的版本

语义化模块地图的版本。语义版本的意思不太清晰，所以使用你最佳的判断。不要跟随模块所标示的系统库的版本，应该单独对待模块地图的版本。

遵循系统包的惯例；例如 python3 的 debian 包叫做 python3，as there is not a single package for python and python is designed to be installed side-by-side. 如果为 python3 做一个模块地图，你应该称其为 `CPython3`。

## 可选依赖的系统库

目前，你需要制作另外一个模块地图包来表示有着可选依赖的系统包。

例如，`libarchive` 可选依赖于 `xz`，这意味着它可以在 `xz` 支持下编译，但这不一定是必要的。要提供一个有着 xz 的 libarchive 包，你必须做一个 `CArchive+CXz` 包，其依赖于 `CXz`，同时提供 `CArchive`。


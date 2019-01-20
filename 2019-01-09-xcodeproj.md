# 使用 xcodeproj

作者：[@nixzhu](https://twitter.com/nixzhu)

---

本文介绍的是 xcodeproj 的 Swift 版本，其代码地址为 [https://github.com/tuist/xcodeproj](https://github.com/tuist/xcodeproj)，你可以通过 SwiftPM 引用它来制作自己的命令行工具。

xcodeproj 是一个能解析或编辑古老的 Xcode 配置文件的工具。关于这种配置文件的格式，你可[在此](http://www.monobjc.net/xcode-project-file-format.html)了解。大概来说，它是一种 plist 格式，但它并不是像`Info.plist`那种 XML 结构。因此，要解析或者修改它，需要有好用的工具来帮忙，而我们的工具就是 xcodeproj。

首先，通过

``` swift
// projectPath 是 *.xcodeproj 的 path（利用 xcodeproj 依赖的 PathKit 提供的 API 来获取）
let project = try XcodeProj(path: projectPath)
```

就可将配置解析出来。

如果我们想改变项目主 Target 的 bundle id，可利用

``` swift
let mainTarget = project.pbxproj.targets(named: "AppName").first!
```

拿到主 Target（注意替换 AppName），进而可通过

``` swift
let buildConfigurationList = mainTarget.buildConfigurationList!
```

来访问其配置。然后我们就可以修改其配置了

``` swift
let key = "PRODUCT_BUNDLE_IDENTIFIER"
for buildConfigurations in buildConfigurationList.buildConfigurations {
    if buildConfigurations.buildSettings[key] != nil {
        buildConfigurations.buildSettings[key] = value
    }
}
```

注意可能有多个配置，所以上面的 buildConfigurations 是一个数组。另外注意 buildSettings 是一个字典，这里为了不小心添加新的 key，先做了非空检查。

最后，我们需要把修改过的配置写到磁盘上，这样配置才会真正生效，因为目前修改的不过是内存中解析出的某个对象而已。

``` swift
try project.write(path: projectPath, override: true)
```

以上就是简单使用 xcodeproj 来修改 bundle id 的过程，可再适当封装一下。

如果你要修改其它配置，我推荐一个办法：先利用Xcode，手动修改项目里对应的值，然后利用`git diff`来对比，看看是什么 key 被改变了；之后利用上面的方法编写代码来修改，然后对比两次的`git diff`是否一致。

除了修改外，xcodeproj 也可以新增属性、配置，甚至于 Target。因为虽然它处理的格式很奇怪，但其本质上是一个树形结构，你可以想象为操作某个枝干或者叶子结点，增删自然不在话下。

最后的最后，查看 xcodeproj 的文档 [https://tuist.github.io/xcodeproj/](https://tuist.github.io/xcodeproj/)，对这棵树里有什么东西就更加清楚了。

具体使用时，多利用 Xcode 的代码提示功能，看看有什么属性可以用，某些结构里都有什么值，渐渐就熟悉了。

------

欢迎转载，但请一定注明出处：[https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog) ！
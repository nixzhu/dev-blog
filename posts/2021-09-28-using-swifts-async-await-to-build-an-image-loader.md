# 使用 Swift 的 async/await 构建一个图片加载器

本文翻译自：[https://www.donnywals.com/using-swifts-async-await-to-build-an-image-loader/](https://www.donnywals.com/using-swifts-async-await-to-build-an-image-loader/)

原作者：[Donny](https://www.donnywals.com)

译者：[@nixzhu](https://twitter.com/nixzhu)

---

Async/await 将是 iOS 15 及以上版本进行异步编程的事实方式。我已经写了[不少](https://www.donnywals.com/category/swift-concurrency/)关于 Swift 新的并发功能的文章，但依然还有很多东西可以写。在本文中，我将试图构建一个支持缓存的异步图片加载器。

iOS 15 上的 SwiftUI 已有一个可从网络加载图片的组件（AsyncImage），但它不支持缓存（除了 URLSession 本身提供的），而且它只接受 `URL` 而不接受 `URLRequest` 作为参数。它在多数场景都很好，但作为练习，我想探索一下自己实现这种组件需要做些什么。更具体地说，是我想体会一下使用 Swift 并发来实现这样的组件是什么感觉。

我们将从构建图片加载器对象本身开始。之后，我将展示如何构建一个简单的 SwiftUI 视图，以使用此图片加载器从网络（或从本地缓存，如果可能）加载图片。此加载器将接受  `URL`  或  `URLRequest`  作为参数，以实现最大的可配置性。

请注意，本文的重点不是向你展示一个完美的图片缓存方案。而是演示如何构建一个 `ImageLoader` 对象，该对象将检查本地是否有图片，并且只在请求的图片在本地没有的情况下使用网络。

设计图片加载器 API
------------------------------

我们的图片加载器的公开 API 很简单，只有两个方法：

1.  `public func fetch(_ url: URL) async throws -> UIImage`
2.  `public func fetch(_ urlRequest: URLRequest) async throws -> UIImage`

图片加载器会记录已发出的请求和已加载的图片。它将尽可能地重复使用图片或加载图片的任务。由此，我们想让图片加载器成为一个 `actor`。如果你对 actor 不熟悉，可以看看我之前发表的[这篇文章](https://www.donnywals.com/an-introduction-to-synchronizing-access-with-swifts-actors/)，来温习一下 Swift 并发里的 actor。

虽然公开 API 相对简单，但跟踪正在进行的请求和在可能的情况下从磁盘加载图片还是需要一点工作量。

定义 ImageLoader actor
------------------------------

我们将一步步实现一个功能齐全的加载器。让我们从定义 `ImageLoader` actor 的骨架开始。

```swift
actor ImageLoader {
    private var images: [URLRequest: LoaderStatus] = [:]

    public func fetch(_ url: URL) async throws -> UIImage {
        let request = URLRequest(url: url)
        return try await fetch(request)
    }

    public func fetch(_ urlRequest: URLRequest) async throws -> UIImage {
        // 通过 URLRequest 获取图片
    }

    private enum LoaderStatus {
        case inProgress(Task<UIImage, Error>)
        case fetched(UIImage)
    }
}
```

在上面的代码片断中，我实际上做了超出定义一个骨架的事情。例如，我定义了一个私有枚举 `LoaderStatus`。这个枚举将被用来跟踪我们从网络上加载的图片，以及哪些图片可以立即从内存中获得。我还实现了 `fetch(_:)` 方法，它需要一个 `URL` 作为参数。为了保持简单，它只是构建了一个没有额外配置的 `URLRequest`，并调用了 `fetch(_:)` 的重载，该重载需要一个 `URLRequest` 作为参数。

现在我们有了一个骨架，可以开始实现 `fetch(_:)` 方法。基本上，有三种不同的情况会被我们碰到。有趣的是，这三种情况与我在[之前与 Swift 并发相关的帖子](https://www.donnywals.com/building-a-token-refresh-flow-with-async-await-and-swift-concurrency/)中写的很相似，其中涉及刷新认证令牌。

这些情况可以大致定义如下：

1.  `fetch(_:)` 已经为这个 `URLRequest` 调用过了，所以将返回一个任务或加载的图片。
2.  我们可以从磁盘加载图片，并将其存储在内存中。
3.  我们需要从网络上加载图片，并将其存储在内存和磁盘上。

我将向你一步步展示 `fetch(_:)` 的实现。注意，在我们完成实现之前，代码不会通过编译。

首先，我们要检查 `images` 字典，看看我们是否可以重用现有的任务或直接从字典中拿到图片。

```swift
public func fetch(_ urlRequest: URLRequest) async throws -> UIImage {
    if let status = images[urlRequest] {
        switch status {
        case .fetched(let image):
            return image
        case .inProgress(let task):
            return try await task.value
        }
    }

    // 还需要更多实现才能通过编译
}
```

上面的代码看起来稀松平常。我们可以像平常一样简单地检查这个字典。由于  `ImageLoader`  是一个 actor，它会确保我们以线程安全的方式访问这个字典（如果你还不熟悉 actor 的话，别忘了参考我写的[关于 actor 的文章](https://www.donnywals.com/an-introduction-to-synchronizing-access-with-swifts-actors/)）。

如果我们找到一张图片，我们就把它返回。如果我们遇到了一个正在进行的任务，我们就等待该任务的值，以获得所要求的图片，而不创建一个新的（重复的）任务。

下一步是检查图片是否存在于磁盘上，以避免不必要的网络请求。

```swift
public func fetch(_ urlRequest: URLRequest) async throws -> UIImage {
    // ... 前面代码片段的代码

    if let image = try self.imageFromFileSystem(for: urlRequest) {
        images[urlRequest] = .fetched(image)
        return image
    }

    // 还需要更多实现才能通过编译
}
```

这段代码调用了一个名为 `imageFromFileSystem` 的私有方法，你很快会看到它的实现。首先，我想先简单介绍一下这段代码的作用。它试图从文件系统中获取请求的图片，这是同步进行的。当找到一张图片时，我们将其存储在 `images` 数组中，这样下次调用 `fetch(_:)` 时就可以从内存而不是文件系统中获得图片。

而且，这一切都以线程安全的方式完成，因为我们的 `ImageLoader` 是一个 actor。

正如承诺的那样，下面是 `imageFromFileSystem` 的样子。相当简明。

```swift
private func imageFromFileSystem(for urlRequest: URLRequest) throws -> UIImage? {
    guard let url = fileName(for: urlRequest) else {
        assertionFailure("无法为 \(urlRequest) 生成本地路径")
        return nil
    }

    let data = try Data(contentsOf: url)
    return UIImage(data: data)
}

private func fileName(for urlRequest: URLRequest) -> URL? {
    guard let fileName = urlRequest.url?.absoluteString.addingPercentEncoding(withAllowedCharacters: .urlPathAllowed),
          let applicationSupport = FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask).first 
    else {
        return nil
    }

    return applicationSupport.appendingPathComponent(fileName)
}
```

第三种也是最后一种我们可能遇到的情况是需要从网络获取图片。让我们具体看看。

```swift
public func fetch(_ urlRequest: URLRequest) async throws -> UIImage {
    // ... 前面代码片段的代码

    let task: Task<UIImage, Error> = Task {
        let (imageData, _) = try await URLSession.shared.data(for: urlRequest)
        let image = UIImage(data: imageData)!
        try self.persistImage(image, for: urlRequest)
        return image
    }

    images[urlRequest] = .inProgress(task)

    let image = try await task.value

    images[urlRequest] = .fetched(image)

    return image
}

private func persistImage(_ image: UIImage, for urlRequest: URLRequest) throws {
    guard let url = fileName(for: urlRequest),
          let data = image.jpegData(compressionQuality: 0.8) 
    else {
        assertionFailure("无法为 \(urlRequest) 生成保存路径")
        return
    }

    try data.write(to: url)
}
```

最后增加到 `fetch(_:)` 的代码创建了一个新的 `Task` 实例来从网络上获取图片数据。当数据被成功获取，并被转换为 `UIImage` 实例。然后使用我在这个片段中包含的 `persistImage(:for:)` 方法将这个图片持久化到磁盘。

创建任务后，我更新了 `images` 字典，使其包含新创建的任务。这将允许 `fetch(_:)` 的其他调用者重新使用这个任务。接下来，我等待任务的值，并更新 `images` 字典，使其包含获取的图片。最后，我返回图片。

你可能想知道为什么我需要在等待之前将正在进行的任务添加到 `images` 字典中。

原因是当 `fetch(_:)`被暂停以等待网络任务的值时，`fetch(_:)`的其他调用者将获得运行时间。这意味着当我们在等待任务值时，其他人可能会调用 `fetch(_:)` 方法并读取 `images` 字典。如果当时正在进行的任务没有被添加到字典中，我们将启动第二次获取。通过首先更新 `images` 字典，我们确保后续的调用者会重复使用正在进行的任务。

在这一点上，我们已经完成了一个完整的图片加载器。很不错，对吧？我总是很高兴地看到，actor 将需要仔细的同步来正确处理并发访问的流程简化。

下面是 `fetch(_:)` 方法最终的样子：

```swift
public func fetch(_ urlRequest: URLRequest) async throws -> UIImage {
    if let status = images[urlRequest] {
        switch status {
        case .fetched(let image):
            return image
        case .inProgress(let task):
            return try await task.value
        }
    }

    if let image = try self.imageFromFileSystem(for: urlRequest) {
        images[urlRequest] = .fetched(image)
        return image
    }

    let task: Task<UIImage, Error> = Task {
        let (imageData, _) = try await URLSession.shared.data(for: urlRequest)
        let image = UIImage(data: imageData)!
        try self.persistImage(image, for: urlRequest)
        return image
    }

    images[urlRequest] = .inProgress(task)

    let image = try await task.value

    images[urlRequest] = .fetched(image)

    return image
}
```

接下来，我们在某个 SwiftUI 视图中使用它来创建我们自己版本的 `AsyncImage`。

构建一个自定义的异步图片视图
--------------------------------------------

我们将在本节中创建自定义 SwiftUI 视图，主要是作为一个概念验证。我已经在一些场景中测试了它，但还不够彻底，不能有把握地说这是一个比内置的 `AsyncImage` 更好的异步图片视图。然而，我很确定这是一个在很多情况下都能正常工作的实现。

为了给我们的自定义图片视图提供一个 `ImageLoader` 的实例，我将使用 SwiftUI 的环境。要做到这一点，我们需要向`环境值`对象添加一个自定义值。

```swift
struct ImageLoaderKey: EnvironmentKey {
    static let defaultValue = ImageLoader()
}

extension EnvironmentValues {
    var imageLoader: ImageLoader {
        get { self[ImageLoaderKey.self] }
        set { self[ImageLoaderKey.self] = newValue }
    }
}
```

这段代码在 SwiftUI 环境中添加了一个 `ImageLoader` 的实例，使我们能够从我们的自定义视图中轻松访问它。

我们的 SwiftUI 视图将用 `URL` 或 `URLRequest` 进行初始化。为了保持简单，我们将始终在内部使用 `URLRequest`。

下面是此 SwiftUI 视图的实现：

```swift
struct RemoteImage: View {
    private let source: URLRequest
    @State private var image: UIImage?

    @Environment(\.imageLoader) private var imageLoader

    init(source: URL) {
        self.init(source: URLRequest(url: source))
    }

    init(source: URLRequest) {
        self.source = source
    }

    var body: some View {
        Group {
            if let image = image {
                Image(uiImage: image)
            } else {
                Rectangle()
                    .background(Color.red)
            }
        }
        .task {
            await loadImage(at: source)
        }
    }

    func loadImage(at source: URLRequest) async {
        do {
            image = try await imageLoader.fetch(source)
        } catch {
            print(error)
        }
    }
}
```

当我们实例化视图时，我们为它提供一个 `URL` 或 `URLRequest`。当视图第一次被渲染时，`image` 将是 `nil`，所以我们将只是渲染一个占位的矩形。我没有给它任何尺寸，这将由 `RemoteImage` 的用户来决定。

SwiftUI 视图有一个 `task` 修改器。这个修改器允许我们在首次创建视图时运行异步工作。在这种情况下，我们将使用一个任务来询问图片加载器的图片。当图片被加载时，我们更新 `@State var image`，这将触发视图的重绘。

这个 SwiftUI 视图是非常简单的，它没有处理像动画或稍后更新图片这样的事情。一些好的补充可以是增加使用 Placeholder，或者使 `source` 属性非私有化，并使用 `onChange` 修改器来启动一个新的任务，使用 `Task` 初始化器加载一个新的图片。

我将把这些功能留给你来实现。这个简单视图的重点只是向你展示这个自定义图片加载器如何在 SwiftUI 上下文中使用，而不是向你展示如何建立一个梦幻般的全功能的 SwiftUI 图片视图替换。

总结
----------

在这篇文章中，我们涵盖了很多内容。我是说，很多。你看到了如何建立一个 `ImageLoader`，通过让它成为一个actor 来优雅地处理并发调用。你看到了我们如何使用一个字典来跟踪正在获取的和已经获取的图片。我还向你展示了一个非常简单的文件系统缓存的实现。这使得我们可以在内存中缓存图片，并在需要时从文件系统中加载。最后，你看到了我们如何实现逻辑，在需要时从网络上加载我们的图片。

你了解到，当一个定义在 actor 上的异步函数被暂停时，actor 的状态可以被其他人读写。这意味着我们需要在等待任务结果之前将我们的图片加载任务分配给我们的字典，这样后续的调用者就会重复使用我们正在进行的任务。

之后，我向你展示了如何将我们构建的自定义图片加载器注入 SwiftUI 的环境中，以及如何使用它来构建一个非常简单的自定义异步图片视图。

总而言之，你在这篇文章中已经学到了很多东西。而最棒的是，在我看来，虽然底层逻辑和思维过程相当复杂，但Swift 并发允许我们以合理和可读的方式来表达这种逻辑，这真的很赞。

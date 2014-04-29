# 为一个 iOS 应用编写一个简单的 Node.js/MongoDB Web 服务

本文翻译自 [http://www.raywenderlich.com/61078/write-simple-node-jsmongodb-web-service-ios-app](http://www.raywenderlich.com/61078/write-simple-node-jsmongodb-web-service-ios-app)

原作者：[Michael Katz](http://www.raywenderlich.com/u/mkatz)

译者：[@nixzhu](https://twitter.com/nixzhu)

==========================================

在当今这个协作和社交应用的世界里，其关键是要有一个能简单构建和易于部署的后台。许多组织机构都依赖于一个应用栈（Application Stack），其使用下面三项技术：

- [Node.js](http://nodejs.org/)
- [Express](http://expressjs.com/)
- [MongoDB](https://www.mongodb.org/)

这个栈对于移动应用来说相当流行，因为原生数据格式是JSON，它容易被应用解析，例如通过使用 Cocoa 的 `NSJSONSerialization` 类或其它类似的解析器。

在本教程中，你将学会如何搭建了一个 Node.js 环境，驱动 Express；在此平台之上，你将构建一个通过 REST API 来提供一个 MongoDB 数据库的服务器，就像这样：

![The backend database rendered in a HTML table](http://cdn4.raywenderlich.com/wp-content/uploads/2014/02/tmt_db-480x270.png)  
在一个 HTML 表格中呈现的后端数据库

本教程的第二部分重点放在 iOS 应用端。你将构建一个很酷的叫做“有趣的地方”的应用，标记有趣的位置，让其它用户能够找出他们附近有趣的地方。下面稍微窥探一下你将构建的应用：

![The TourMyTown main view](http://cdn3.raywenderlich.com/wp-content/uploads/2014/02/tmt_basic-213x320.png)  
TourMyTown 的主视图

本教程假设你已经了解了 JavaScript 和 Web 开发的基础，但对 Node.js、Express 以及 MongoDB 都不熟悉。

## 一个 Node+Mongo 案例

大多数 Objective-C 开发者都不太熟悉 JavaScript ，但它对于 Web 开发者来说是极其常见的语言。因为这个原因，Node 作为一个 Web 框架收获了大量人气，但还有更多原因使其成为后端服务的绝好选择：

- 内建的服务器功能
- 通过它的包管理器做到良好的项目管理
- 一个快速的 JavaScript 引擎，也就是 V8
- 异步事件驱动编程模型

一个异步的关于事件和回调的编程模型非常适合服务器，它要等待许多事情，例如到来的请求以及通过其它服务（例如 MongoDB）的内部进程通信。

MongoDB 是一个低开销的数据库，其所有实体都是自由形式 BSON —— “二进制 JSON” —— 文档。这能让你同异构数据打交道，而且处理各种各样的数据格式也变得很容易。因为 BSON 与 JSON 兼容，构建一个 REST API 就很简单——服务器代码能够传递请求到数据驱动器而不需要很多的中间处理。

Node 和 MongoDB 在本质上都具有可扩展性，能够轻松地在跨越分布式模型中的多个机器，实现同步；这个组合对于不具有均匀分布负载的应用来说是一个理想选择。

## 入门

本教程假设你使用 OS X Mountain Lion 或 Mavericks ，Xcode 及其 command line tools 都已经安装好了。

第一步是安装 [Homebrew](http://brew.sh/) 。就像 CocoaPods 为 Cocoa 管理各种包 和 Gem 为 Ruby 管理各种包一样，Homebrew 管理 OS X 上的 Unix 工具。它构建在 Ruby 和 Git 之上，而且它具有高度的灵活性和可定制性。

如果你已经安装了 Homebrew ，那就可以跳过下面的步骤。不然，打开终端执行下列命令来安装 Homebrew ：

```Shell
ruby -e "$(curl -fsSL https://raw.github.com/Homebrew/homebrew/go/install)"
```

>注意：`cURL` 是使用 URL 请求来发送和接收文件与数据的称手工具。此处你使用它加载 Homebrew 安装脚本——在本教程后面，你还会使用它与 Node 服务器交互。

一旦安装好 Homebrew ，就在终端输入下面的命令：

```Shell
brew update
```

这只是更新 Homebrew ，让你拥有最新的软件包列表。

现在，通过 Homebrew 安装 MongoDB ，使用下面的命令：

```Shell
brew install mongodb
```

记下 MongoDB 被安装的位置，它就在输出的“Summary”中。稍后你将用它加载 MongoDB 服务。

从 [http://nodejs.org/download/](http://nodejs.org/download/) 下载并运行 Node.js 安装器。

一旦安装完成，你就马上测试 Node.js 是否安装成功。

在终端里输入：

```Shell
node
```

这能让你进入 Node.js 的交互式运行环境，在此你可以执行 JavaScript 表达式。

在提示符后输入下面的表达式：

```Shell
console.log("Hello World");
```

你将得到如下输出：

```Text
Hello World
undefined
```

`console.log` 在 Node.js 中相当于 `NSLog` 。当然，`console` 的输出流比 `NSLog` 的要复杂得多：它有 `console.info`、`console.assert`、`console.error` 以及你期望的从更先进的记录器例如 `CocoaLumberjack` 而来的其它流。

写在输出里的 “undefined” 值是 `console.log` 的返回值，而 `console.log` 没有返回值。 因为 Node.js 总是显示出所有表达式的输出，无论其返回值是否有定义。

>注意：如果你以前使用过 JavaScript ，你需要知道 Node.js 环境和浏览器环境之间有些许不同。`全局`对象被叫做 `global` 而不是 `window` 。在 Node.js 交互提示符后键入 `global` 并按下回车就会显示 global 命名空间里所有的方法和对象；当然，直接使用 [Node.js 文档](http://nodejs.org/api/) 来做参考更容易些。 :]  
`global` 对象有所有预定义的常数、函数以及数据类型，都可用于所有运行在 Node.js 环境里的程序。任何用户创造的变量同样也都添加到全局上下文对象。基本上 `global` 的输出将列出所有内存中可以访问的事物。

## 运行一个 Node.js 脚本

Node.js 的交互式环境对于玩耍和调试 JavaScript 表达式是很棒的，但通常你都会使用脚本文件来做实际的事情。就像 iOS 应用包含有 `Main.m` 作为其入口点，Node.js 的默认入口点就是 `index.js` 。然而，不同于 Objective-C ，这里没有 main 函数；相反， `index.js` 将从头到尾的执行。

按下 Control+C 两次以退出 Node.js Shell。执行下面的命令，新建一个目录以保存你的脚本：

```Shell
mkdir ~/Documents/NodeTutorial
```

然后执行下面的命令进入新建的目录并使用你默认的文本编辑器新建一个脚本文件：

```Shell
cd ~/Documents/NodeTutorial/; edit index.js
```

在 `index.js` 中添加如下代码：

```JavaScript
console.log("Hello World.");
```

保存你的工作，回到终端执行下面的命令看看你的脚本如何运行：

```Shell
node index.js
```

再一次，我们看到了熟悉的 “Hello World” 输出。你也可以执行 `node .` 来运行你的脚本，`.` 就会默认查找 `index.js` 。

![node_run](http://cdn3.raywenderlich.com/wp-content/uploads/2014/02/node_run-480x109.png)

固然，一个 “Hello World” 脚本成不了一个服务器，但这是测试你的安装是否成功的快速方式。下一节将向你介绍 Node.js 包的世界，这会成为你那闪亮的新 Web 服务器的基础！

## Node 包

Node.js 应用程序都被分成不同的包，这就是 Node.js 世界的“框架”。 Node.js 自带有几个基础且强大的包，但还有超过 50000 个由活跃的开发社区提供的公开包——如果你不能找到你需要的包，你自己也可以比较容易地创造。

>注意：查看 [https://npmjs.org/](https://npmjs.org/) 可得到所有可用包的列表

用下列代码替换 `index.js` 的内容：

```JavaScript
//1
var http = require('http');
 
//2 
http.createServer(function (req, res) {
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end('<html><body><h1>Hello World</h1></body></html>');
}).listen(3000);
 
console.log('Server running on port 3000.');
```

依次按照编号好的注释看看：

1. `require` 引入（import）模块（module）到当前文件。本次你引入了 HTTP 库。
2. 你创建一个 Web 服务，它对简单的 HTTP 请求的回应是发送一个 200 应答，并将页面内容放在应答里。

Node.js 作为一个运行时环境的最大的优势之一就是他的 `事件驱动模型（event-driven model）` 。它围绕着异步调用的回调函数的概念来设计。在上面的例子里，你正监听 3000 端口等着传入的 HTTP 请求。当你收到一个请求，你的脚本调用 `function (req, res) {…}` 并返回一个应答给调用者。

保存你的文件，回到终端并执行如下命令：

```Shell
node index.js
```

你将在控制台看到如下输出：

![node_server](http://cdn4.raywenderlich.com/wp-content/uploads/2014/02/node_server-480x88.png)

打开你最喜欢的浏览器导航至 [http://localhost:3000](http://localhost:3000) ；好好瞧着， Node.js 正在提供给你的是一个 “Hello World” 页面。

![web_helloworld](http://cdn1.raywenderlich.com/wp-content/uploads/2014/02/web_helloworld-480x230.png)

你的脚本还在哪里，耐心地等待从 3000 端口传入的 HTTP 请求。要干掉（kill）你的 Node 实例，只需在终端按下 `Ctrl+C`。

>注意：Node 包通常由顶层函数或引入的对象写就。通过使用 require ，这个函数在之后就被分配给一个顶层变量。这样有助于以一个健全的方式管理范围（scope）以及暴露（expose）模块的 API 。稍后在本教程中你会看到如何创建一个自定义模块，你将为 MongoDB 添加一个驱动器。

## NPM —— 使用外部 Node 模块

前一小节覆盖了 Node.js 内建的模块，那第三方的模块该怎么处理呢？例如你之后需要的 Express 模块，它为你的服务器平台提供路由中间件。

外部模块同样可以使用 require 函数引入到文件里，但你需要分开下载它们然后才能用于你的 Node 实例。

Node.js 使用　`npm` —— Node 包模块——来下载、安装以及管理包依赖。如果你熟悉 CocoaPods 或者 Ruby gems ，那么你对 `npm` 也会觉得熟悉。你的 Node.js 应用程序使用 `package.json` ，它专门定义配置和 `npm` 依赖。

## 使用 Express

Express 是一个流行的 Node.js 模块，提供路由中间件。为什么你会需要这个独立的包呢？考虑下面的情形。

如果你只使用 `http` 模块自身，你不得不分开解析每个请求的位置以找出提供什么内容给请求者——如此这般，事情很快就会变得难以处理。

然而，用 Express 你就能容易地为每个请求定义路由和回调。 Express 同样让为基于 HTTP 动词（例如 POST, PUT, GET, DELETE, HEAD, 等）以提供不同的回调变得很容易。

### HTTP 动词的简要介绍

一个 HTTP 请求包含一个方式——或者动词——的值。默认值是 `GET` ，它是为了获取数据，例如浏览器中 Web 页面。 `POST` 意味着上传数据，例如提交 Web 表单。对于 Web API 来说， `POST` 通常用于添加数据，但它同样可用于远程处理调用类型端点（but it can also be used for remote procedure call-type endpoints.）。

`PUT` 与 `POST` 的不同在于它通常用于替换已有数据。在实践中， `POST` 和 `PUT` 通常以同样的方式使用：在请求 Body 里提供实体以放进后端的数据存储里。 `DELETE`  用于从你的后端数据存储里移除条目。

`POST`、`GET`、`PUT` 以及 `DELETE` 就是 HTTP 实现的 CRUD 模型 —— Create、Read、Update 以及 Delete。

还有其它一些少有人知的 HTTP 动词。 `HEAD` 表现得像一个 `GET` 但只返回应答头而没有 Body 。这有助于最小化数据传输，如果应答头中的信息足够确定是否有可用的新数据。其它动词如 `TRACE` 和 `CONNECT` 用于网络路由。

## 添加一个包到 Node 实例

在终端里执行下列命令：

```Shell
edit package.json
```

这会创建一个新的 `package.json` ，它将包含你的包配置和依赖。

添加如下代码到 `package.json` 中：

```JSON
{
  "name": "mongo-server",
  "version": "0.0.1",
  "private": true,
  "dependencies": {
    "express": "3.3.4"
  }
}
```

这个文件定义了一些元数据，例如项目的名字和版本，一些脚本，以及对于你的目的来说最重要的包依赖。下面说明每行的意思：

- `name` 是项目的名字。
- `version` 是项目目前的版本。
- `private` 防止项目被意外地公开，如果你设置其为 true 。
- `dependencies` 是一个包含你应用使用的模块的列表。

依赖以 键/值 形式接受模块名和版本。你的依赖列表包含有 3.3.4 版本的 Express； 如果你想指明 Node.js 去使用最新版本的包，你可以使用通配符"*"。

保存文件，在终端里执行下列命令：

```Shell
npm install
```

你会看到如下输出：

![npm_install](http://cdn2.raywenderlich.com/wp-content/uploads/2014/02/npm_install-480x284.png)

`install` 下载并安装 `package.json` 指定的依赖——以及你的依赖本身的依赖！:] ——存进一个叫做 `node_modules` 的目录，并让你的应用程序使用它们。

一旦 `npm` 完成，你就可以在你的应用程序中使用 Express 了。

在 `index.js` 中找到下列行：

```JavaScript
var http = require('http');
```

并添加 Express 的 `require` 调用，如下所示：

```JavaScript
var http = require('http'),
    express = require('express');
```

这就引入了 Express 包，并将其存在变量 `express` 中。

添加如下代码到 `index.js`，就在刚在添加的区域的下面：

```JavaScript
var app = express();
app.set('port', process.env.PORT || 3000);
```

这就创建了一个 Express 应用并设置其默认端口为 3000 。你可以通过创建一个环境变量 `PORT` 来覆盖此默认值。这种类型的自定义在开发工具中非常方便，特别是如果你有多个应用程序监听这好几个端口。

添加如下代码到 `index.js` ，就在刚刚添加的区域的下面：

```JavaScript
app.get('/', function (req, res) {
  res.send('<html><body><h1>Hello World</h1></body></html>');
});
```

这就创建了一个`路由处理器（route handler）`，它是给定 URL 的请求处理者链的花哨名字。Express 匹配请求中指定的路径并执行适当的回调。

你上面的回调告诉 Express 去匹配根路径 "/" 并返回一个给定的 HTML 。 `send` 为你格式化各种响应头——例如 `content-type` 和 `status code` —— 如此你就能专注于编写伟大代码了。

最后，替换 `http.createServer(...)` 为下面的实现： 

```JavaScript
http.createServer(app).listen(app.get('port'), function(){
  console.log('Express server listening on port ' + app.get('port'));
});
```

这比之前的稍微紧凑些。 `app` 分开实现 `function(req,res)` 回调，而不是在  `createServer` 这里内联地包含它们。你同样添加了一个完成处理器回调，一旦端口准备好接收请求它就会被调用。现在你的应用在打印 “listening” 消息到控制台之前就等着端口准备好。

为了审查，`index.js` 整个看起来如下所示：

```JavaScript
var http = require('http'),
    express = require('express');
 
var app = express();
app.set('port', process.env.PORT || 3000); 
 
app.get('/', function (req, res) {
  res.send('<html><body><h1>Hello World</h1></body></html>');
});
 
http.createServer(app).listen(app.get('port'), function(){
  console.log('Express server listening on port ' + app.get('port'));
});
```

保存你的文件，并在终端执行下列命令：

```Shell
node index.js
```

回到浏览器，重新载入 [http://localhost:3000](http://localhost:3000) 去看看你的 Hello World 页面是否依然加载。

你的页面看起来与之前没有区别，但有不止一种方法可以查看引擎盖下发生了什么事。

创建终端的另一个实例，并执行如下命令：

```Shell
curl -v http://localhost:3000
```

你会看到如下输出：

![curl_v](http://cdn2.raywenderlich.com/wp-content/uploads/2014/02/curl_v-437x320.png)

`curl` 吐出你的 HTTP 请求的头和内容，给你显示服务传来的东西的原始细节。注意 `X-Powered-By : Express` 头；Express 会自动添加这个元数据到应答里。

## 使用 Express 提供内容

用 Express 提供静态文件非常容易。

添加如下语句到 `index.js` 顶部的 `require` 区域：

```JavaScript
path = require('path');
```

再添加下面一行到 `app.set` 语句之后：

```JavaScript
app.use(express.static(path.join(__dirname, 'public')));
```

这就告诉 Express 去使用 `express.static` 中间件 ，它为到来的请求提供静态文件作为应答。

`path.join(__dirname, 'public') ` 映射本地子目录 `public` 到基路由 `/` ； 它使用 Node.js `path` 模块创建一个平台无关的子目录字符串。

`index.js` 现在看起来如下：

```JavaScript
//1
var http = require('http'),
    express = require('express'),
    path = require('path');
 
//2 
var app = express();
app.set('port', process.env.PORT || 3000); 
app.use(express.static(path.join(__dirname, 'public')));
 
app.get('/', function (req, res) {
  res.send('<html><body><h1>Hello World</h1></body></html>');
});
 
http.createServer(app).listen(app.get('port'), function(){
  console.log('Express server listening on port ' + app.get('port'));
});
```

使用了静态处理器，任何在 `/public` 中的东西都可以由其名字访问到。

为了证明这一点，按下 Control+C 干掉你的 Node 实例，然后执行下面的命令：

```Shell
mkdir public; edit public/hello.html
```

添加如下代码到 `hello.html` ：

```HTML
<html></body>Hello World</body></html>
```

这就创建了一个新的 `public` 目录并创建了一个基础的静态 HTML 文件。

再次用命令 `node index.js` 重启你的 Node 实例。 浏览器打开 [http://localhost:3000/hello.html](http://localhost:3000/hello.html) 你就会看到这个新创建的页面，如下所示：

![web_hello](http://cdn5.raywenderlich.com/wp-content/uploads/2014/02/web_hello-480x230.png)

## 高级路由

静态页面是不错，但 Express 的真正威力是动态路由。 Express 在路由字符串上使用一个正则表达是匹配器，允许你为路由定义参数。

举个例子，路由字符串可以包含下列元素：

- 静态元素—— `/files` 只会匹配 `http://localhost:3000/pages` （译者注：似乎有点问题，应该只会匹配 `http://localhost:3000/files`）
- 以“:”开头的参数—— `/files/:filename` 匹配 `/files/foo` 和 `/files/bar`，但不能匹配 `/files`
- 以“?”结尾的可选参数——`/files/:filename?` 匹配 `/files/foo` 也能匹配 `/files`
- 正则表达式—— `/^\/people\/(\w+)/` 匹配 `/people/jill` 和 `/people/john`

要试试看，就添加下列路由到 `index.js` 中 `app.get` 语句后：

```JavaScript
app.get('/:a?/:b?/:c?', function (req,res) {
	res.send(req.params.a + ' ' + req.params.b + ' ' + req.params.c);
});
```

这就创建了一个新的路由，它接收三个路径层级并在应答 Body 中显示这些路径组件。任何由`:`开始的东西都映射到所提供名字的请求参数上。

重启你的 Node 实例，再将浏览器转到 [http://localhost:3000/hi/every/body](http://localhost:3000/hi/every/body) 。你将看到如下页面：

![web_hieverybody](http://cdn1.raywenderlich.com/wp-content/uploads/2014/02/web_hieverybody-480x230.png)

“hi” 是 `req.params.a` 的值，“every” 是`req.params.b` 的值，最后 “body” 分配给 `req.params.c` 。

路由匹配对于构建 REST API 来说很有用，你可以用动态路径指定后端数据存储的特定元素。

除了 `app.get` ， Express 还支持 `app.post`、`app.put`、`app.delete` 等等。

## 错误处理与模版化 Web 视图

服务器错误可用一到两种方式处理。你可以传递一个异常给调用栈——这样做可能干掉应用——或者你可以捕捉错误并返回一个合适的错误码。

[HTTP 1.1](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) 协议在 4xx 和 5xx 范围内定义了好几个错误码。 400 段的错误用于用户错误，例如请求一个不存在的条目：一个熟悉的错误码是 404 Not Found 错误。500 段的错误表示服务器错误，例如超时或者编程错误（比如 null 解除引用（null dereference））。

你将添加一个捕捉所有（catch-all）的路由并在请求内容不能被找到时返回一个 404 页面。因为路由处理器按照它们设置 `app.use` 或 app._verb_ 的顺序添加，一个捕捉所有（catch-all）的路由可以添加在路由链的最后。

添加如下代码到 `index.js` ，就在最后的 `app.get` 与调用 `http.createServer` 之间:

```JavaScript
app.use(function (req,res) { //1
    res.render('404', {url:req.url}); //2
});
```

这些代码会导致 404 页面的加载，如果此处没有前一个调用使用 `res.send()` .

这里有一些值得记录的点：

- `app.use(callback)` 匹配_所有_请求。当它被放在所有 `app.use` 和 app._verb_ 的列表的最后，callback 就会成为捕捉所有（catch-all）。
- `res.render(view, params)`调用使用`模版引擎( templating engine)`渲染的输出填充响应 Body 。 一个模版引擎使用磁盘上一个叫做“View”的模版文件并用一组键值参数替换其中的变量以生成一个新的文档。

等等——一个“模版引擎”？这货搞什么飞机？

Express 能使用好几种模版引擎来提供视图。要让这个例子工作起来，你将添加一个流行的 [Jade](http://jade-lang.com/) 模版引擎到你的应用程序中。

Jade 是一种简单的语言，它避开括号并使用空白符来代替，以确定 HTML 标签的顺序和内容。它同样可以使用变量、条件判断、迭代以及分支以便动态地创建 HTML 文档。

更新 `package.json` 中的依赖为：

```JSON
{
  "dependencies": {
    "express": "3.3.4",
    "jade": "1.1.5"
}
```

回到终端，干掉你的 Node 实例，并执行如下命令：

```Shell
npm update
```

这将下载并安装 `jade` 包。

添加如下代码到 `index.js` ，就在第一个 `app.set` 之后：

```JavaScript
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'jade');
```

上面的第一行指定了视图模版的位置，第二行就设置 jade 作为视图渲染引擎。

在终端执行如下命令：

```Shell
mkdir views; edit views/404.jade
```

添加下列代码到 `404.jade` 中：

```Jade
doctype html
body
    h1= 'Could not load page: ' + url
```

Jade 模版中的前两行创建了一个有 `body` 元素的新 HTML 文档。第三行通过缩进创在 `body` 内建了一个  `h1` 元素。间距在 Jade 中很重要！ :]

`h1` 元素的文本由  “Could not load page” 和作为 `index.js` 中 `res.render()` 的一部分传递的 `url` 参数串联而成。

作为一个快速检查，你的 `index.js` 现在看起来如下：

```JavaScript
var http = require('http'),
    express = require('express'),
    path = require('path');
 
var app = express();
app.set('port', process.env.PORT || 3000); 
app.set('views', path.join(__dirname, 'views')); //A
app.set('view engine', 'jade'); //B
 
app.use(express.static(path.join(__dirname, 'public')));
 
app.get('/', function (req, res) {
  res.send('<html><body><h1>Hello World</h1></body></html>');
});
 
app.use(function (req,res) {
    res.render('404', {url:req.url});
});
 
http.createServer(app).listen(app.get('port'), function(){
  console.log('Express server listening on port ' + app.get('port'));
});
```

重启你的 Node 实例，使用浏览器加载 URL [http://localhost:3000/show/a/404/page](http://localhost:3000/show/a/404/page) 。你将看到如下页面：

![web_404](http://cdn1.raywenderlich.com/wp-content/uploads/2014/02/web_404-480x230.png)

现在你的 `index.js` 中有足够的启动代码去接收传入的请求并提供一些基本的响应。而缺失的部分就是数据库持久化，它能将这些东西变成一个有用的Web应用程序，能够被一个移动应用所利用。

## 介绍 MongoDB

MongoDB 是一个存储 JSON 对象的数据库。不像 SQL 数据库，类似 Mongo 的 `NoSQL 数据库` 不支持实体关系。进一步说明，没有预定义的模式，所以同一集合里的实体不需要有同样的字段或符合预定义的模式。

MongoDB 同样提供了强大的的查询语言 `map-reduce` 以及对定位数据的支持。MongoDB 因其扩展、复制和碎片（scale, replicate and shard）能力而广受欢迎。扩展和高可用特性不在本教程覆盖范围。

MongoDB 最大的缺点是缺少关系支持，而且在内存映射实际的数据库文件时可能会占用过多内存。这些问题可以通过仔细构造的数据得到缓解；这将在本教程的第二部分进行说明。

因为 MongoDB 文档 和 JSON 的亲密关系，MongoDB 对于 Web 和移动应用都是很棒的选择。 MongoDB 不存储原始 JSON；而是叫做 BSON（即 Binary JSON） 格式的文档，这对于数据存储和查询来说更有效率。BSON 同时还支持比 JSON 更多的数据类型，例如日期和C类型（C-type）。

## 添加 MongoDB 到你的项目中

MongoDB 是一个原生应用程序，通过`驱动器（drivers）`访问。有好多种驱动器可用于几乎任何环境，当然包括 Node.js 。MongoDB 驱动器连接 MongoDB 服务器并发出命令去更新或读取数据。

这就意味着你需要运行一个 MongoDB 实例以在一个打开的端口上监听。幸运的是，这就是你的下一个步骤！:]

新开一个终端窗口并执行如下命令：

```Shell
cd /usr/local/opt/mongodb/; mongod
```

译者注：这里可能会发生错误 `ERROR: dbpath (/data/db) does not exist.`，试试先创建一个自定义路径，再用 `mongod --dbpath '~/somepath'` 来启动服务器。

这就能启动一个 MongoDB 守护服务器。

现在 MongoDB 已经启动，运行在默认端口 27017 上。

虽然 MongoDB 驱动器提供了数据库连接，但它依然需要被连接到服务器以便转换传入的 HTTP 请求为适当的数据库命令。

## 创建一个 MongoDB 集合驱动器（Collection Driver）

还记得你之前实现的 `/:a/:b/:c` 路由吗？如果你可以使用这个模式去查找数据库实体如何？

既然 MongoDB 文档被组织为集合，那么路由就可以很简单如： `/:collection/:entity` ，这能让你以超级 fashion 的 RESTful 的方式使用一个简单的地址系统去访问对象。

干掉你的 Node 实例并在终端执行下列命令：

```Shell
edit collectionDriver.js
```

添加如下代码到 `collectionDriver.js` ：

```JavaScript
var ObjectID = require('mongodb').ObjectID;
```

这一行引入了各个需要的包；在本例中，是来自 MongoDB 包的 `ObjectID` 。

>注意：如果你比较熟悉传统数据库，你可能明白术语“主键”。MongoDB有类似的概念：默认来说，新实体都会被分配一个唯一的 `_id` 字段，其类型为  `ObjectID` ，这是 MongoDB 用来优化查找和插入的。因为 ObjectID 是一个 BSON 类型而不是 JSON 类型，你必须转换任何传入的字符串为 ObjectID ，如果它们用于和一个　`_id` 字段进行比较。

添加如下代码到 `collectionDriver.js` 刚才那行后面：

```JavaScript
CollectionDriver = function(db) {
  this.db = db;
};
```

这个函数定义了 `CollectionDriver` 构造器方法；它存储一个 MongoDB 客户端实例以便之后使用。在 JavaScript 中， `this` 是当前上下文的引用，就像 Objective-C 中的 `self` 。

继续添加如下代码当刚刚添加的代码块下面：

```JavaScript
CollectionDriver.prototype.getCollection = function(collectionName, callback) {
  this.db.collection(collectionName, function(error, the_collection) {
    if( error ) callback(error);
    else callback(null, the_collection);
  });
};
```

这一段定义了一个帮助方法 `getCollection` 以便通过名字去获取一个 Mongo 集合。你通过添加函数到 `prototype` 定义了类方法。

`db.collection(name,callback)` 获取集合对象并返回集合——或一个错误——给回调。

继续添加如下代码到刚才添加的代码块下面：

```JavaScript
CollectionDriver.prototype.findAll = function(collectionName, callback) {
    this.getCollection(collectionName, function(error, the_collection) { //A
      if( error ) callback(error);
      else {
        the_collection.find().toArray(function(error, results) { //B
          if( error ) callback(error);
          else callback(null, results);
        });
      }
    });
};
```

A 行的 `CollectionDriver.prototype.findAll` 获取集合，如果没有如不能访问 MongoDB 服务器这样的错误，它就调用 B 行的 `find()` 。这将返回所有找到的对象。

`find()` 返回一个`数据游标（data cursor）`，它可用于遍历匹配对象。`find()` 同样能接受一个选择器对象来过滤结果。 `toArray()` 组织所有的结果为一个数组并将其传递给回调。最后回调返回给调用者一个找到的对象的数组或者一个错误。

继续添加如下代码到刚才添加的代码块下面：

```JavaScript
CollectionDriver.prototype.get = function(collectionName, id, callback) { //A
    this.getCollection(collectionName, function(error, the_collection) {
        if (error) callback(error);
        else {
            var checkForHexRegExp = new RegExp("^[0-9a-fA-F]{24}$"); //B
            if (!checkForHexRegExp.test(id)) callback({error: "invalid id"});
            else the_collection.findOne({'_id':ObjectID(id)}, function(error,doc) { //C
                if (error) callback(error);
                else callback(null, doc);
            });
        }
    });
};
```

在 A 行， `CollectionDriver.prototype.get` 使用 `_id` 从一个集合中获取单个条目。类似于 `prototype.findAll` 方法，这个调用首先获取一个集合对象然后在返回的对象上执行一个 `findOne` 。因为这匹配 `_id` 字段，本例中的一个 `find()` 或 `findOne()` 将会使用正确的[数据类型](http://docs.mongodb.org/manual/reference/bson-types/)来匹配它。

MongoDB 存储 `_id` 字段为 BSON 类型 `ObjectID` 。在上面的 C 行，`ObjectID()` 接受一个字符串并将其转换为一个 BSON ObjectID 去匹配集合。然而，`ObjectID()` 很小气，需要适当的十六进制字符串否则它会返回一个错误：因此，B 行会先用正则作检查。

这不能保证有一个与 `_id` 匹配的对象，但它保证 `ObjectID` 能够传递字符串。选择器 `{'_id':ObjectID(id)}` 使用提供的 `id` 匹配 `_id` 字段。

>注意：从一个不存在的集合或实体中读取不是一个错误—— MongoDB 驱动器只会返回一个空的容器。

继续添加如下代码到刚才添加的代码块下面：

```JavaScript
exports.CollectionDriver = CollectionDriver;
```

这一行定义或暴露实体用于其他应用程序，它们以一个需求模块列在 `collectionDriver.js` 中。

保存你的修改——你完成了这个文件！现在你需要一个方式去调用这个文件。

## 使用你的集合驱动器

要调用你的 `collectionDriver` ，首先添加下面一行到 `package.json` 中的 `dependencies` 内：

```JSON
    "mongodb":"1.3.23"
```

在终端执行下列命令：

```Shell
npm update
```

这将下载并安装 MongoDB 包。

在终端执行下列命令：

```Shell
edit views/data.jade
```

现在添加下列代码到 `data.jade` 中，注意缩进层级：

```Jade
body
    h1= collection
    #objects
        table(border=1)
          if objects.length > 0
              - each val, key in objects[0]
                  th= key 
          - each obj in objects
            tr.obj
              - each val, key in obj
                td.key= val
```

这个模版渲染一个集合到一个 HTML 表格中，使其对人类可读。

添加下列代码到 `index.js` ，就在 `path = require('path')` 那行下面：

```JavaScript
MongoClient = require('mongodb').MongoClient,
Server = require('mongodb').Server,
CollectionDriver = require('./collectionDriver').CollectionDriver;
```

这里你包含了来自 MongoDB 模块的 `MongoClient` 和  `Server` 对象以及你新创建的 `CollectionDriver` 。

添加下列代码到 `index.js` ，就在最后一行 `app.set` 的下面：

```JavaScript
var mongoHost = 'localHost'; //A
var mongoPort = 27017; 
var collectionDriver;
 
var mongoClient = new MongoClient(new Server(mongoHost, mongoPort)); //B
mongoClient.open(function(err, mongoClient) { //C
  if (!mongoClient) {
      console.error("Error! Exiting... Must start MongoDB first");
      process.exit(1); //D
  }
  var db = mongoClient.db("MyDatabase");  //E
  collectionDriver = new CollectionDriver(db); //F
});
```

上面 A 行假设 MongoDB 实例是本地运行在端口 27017 。如果你已经在其他地方运行过 MongoDB 服务器，那你就需要修改这些值，但在本教程中就留下它们吧。

B 行创建了一个新的 MongoClient 并调用 C 行的 `open` 试图建立一个连接。如果你的连接尝试失败了，那很可能是你还没有启动你的 MongoDB 服务器。在无连接的情况下，应用将在 D 行退出。 

如果客户端连接成功，它就在 E 行打开 `MyDatabase` 数据库。一个 MongoDB 实例可以包含多个数据库，每一个都有唯一的命名空间和唯一的数据。最后，你在 F 行创建 `CollectionDriver` 对象并传递一个处理器给 MongoDB 客户端。

用下列语句替换 `index.js` 中的头两行 `app.get` 调用：

```JavaScript
app.get('/:collection', function(req, res) { //A
   var params = req.params; //B
   collectionDriver.findAll(req.params.collection, function(error, objs) { //C
    	  if (error) { res.send(400, error); } //D
	      else { 
	          if (req.accepts('html')) { //E
    	          res.render('data',{objects: objs, collection: req.params.collection}); //F
              } else {
	          res.set('Content-Type','application/json'); //G
                  res.send(200, objs); //H
              }
         }
   	});
});
 
app.get('/:collection/:entity', function(req, res) { //I
   var params = req.params;
   var entity = params.entity;
   var collection = params.collection;
   if (entity) {
       collectionDriver.get(collection, entity, function(error, objs) { //J
          if (error) { res.send(400, error); }
          else { res.send(200, objs); } //K
       });
   } else {
      res.send(400, {error: 'bad url', url: req.url});
   }
});
```

这就创建了两个新路由 `/:collection` 和 `/:collection/:entity` 。它们分别调用 `collectionDriver.findAll` 和 `collectionDriver.get` 方法并返回 JSON 对象、HTML 文档或一个错误。

当你在 Express 中 定义 `/collection` ，它将明确匹配 “collection” 。然而，如果你定义如 A 行的路由 `/:collection` 那么它将匹配任何存储在 B 行的 `req.params` 集合中的第一层路径。在本例中，你使用 C 行的 `CollectionDriver` 的 `findAll` 定义的端点去匹配任何到 MongoDB 的 URL 。

如果查询成功，那么代码会在 E 行的头中检查，是否请求会接受一个 HTML 结果。如果是，那 F 行就从 `data.jade` 模版存储渲染过的 HTML 到`应答`中。这将简单地呈现集合内容到一个 HTML 表格中。

默认情况下，Web 浏览器会在它们的请求中指定它们接受 HTML 。当其他类型的客户端请求这个端点，例如 iOS 应用使用 `NSURLSession` ，这个方法就会在 G 行返回一个机器可读的 JSON 文档。 与 H 行， `res.send()` 会返回由集合驱动器生成的 JSON 文档和一个成功码。

这个例子中，对于两层 URL 指定的位置， I 行将其作为集合名和实体 `_id` 对待。之后你在 J 行使用 `collectionDriver` 的 `get()` 请求特定的实体。如果那个实体被找到，你就在 K 行将其作为 JSON 文档返回。

保存你的工作，重启你的 Node 实例， 检查你的 mongod 守护进程是否依然在运行，然后将浏览器指向 [http://localhost:3000/items](http://localhost:3000/items) ；你将看到如下页面：

![web_emptyitems](http://cdn2.raywenderlich.com/wp-content/uploads/2014/02/web_emptyitems-480x247.png)

怎么什么都没有？发生了什么事？

哦，等等——那是因为你还没有添加任何数据呢。是时候了！

## 与数据同行

从一个空空如也的数据库里读取对象一点儿也不有趣。要测试功能，就要有一个添加实体到数据库的途径。

在 `CollectionDriver.js` 中添加下列新的原型方法，就在 `exports.CollectionDriver` 行之前：

```JavaScript
//save new object
CollectionDriver.prototype.save = function(collectionName, obj, callback) {
    this.getCollection(collectionName, function(error, the_collection) { //A
      if( error ) callback(error)
      else {
        obj.created_at = new Date(); //B
        the_collection.insert(obj, function() { //C
          callback(null, obj);
        });
      }
    });
};
```

就像 `findAll` 和 `get` ，A 行的 `save` 首先检索集合对象。之后回调取得提供的实体并再添加一个字段记录创建的日期（如 B 行所示）。最后，你在 C 行插入修改后的对象到集合里。 `insert` 同时会自动添加一个  `_id` 。

添加下列代码到 `index.js` ，就在刚才添加的 `get` 方法之后： 

```JavaScript
app.post('/:collection', function(req, res) { //A
    var object = req.body;
    var collection = req.params.collection;
    collectionDriver.save(collection, object, function(err,docs) {
          if (err) { res.send(400, err); } 
          else { res.send(201, docs); } //B
     });
});
```

这就在 A 行为 `POST` 动词创建了一个新的路由，它通过调用你刚刚添加到你的驱动器里的 `save()` 将 Body 当作一个对象插入到指定的集合里。当资源被创建后，B 行就返回 HTTP 201 成功码。

只有最后一块了。添加下列代码到 `index.js` ，就在 `app.set` 行后面，但在 `app.use` 或 `app.get` 行之前： 

```JavaScript
app.use(express.bodyParser());
```

这会告诉 Express 去解析传入的 Body 数据；如果它是 JSON，那么用它创建一个 JSON 对象。通过将这个调用提前，Body 解析将在其他路由处理器之前调用。这样 `req.body` 就能直接作为 JavaScript 对象传递给驱动器。

再次重启你的 Node 实例，在终端里执行下列命令，插入一个测试对象到你的数据库：

```Shell
curl -H "Content-Type: application/json" -X POST -d '{"title":"Hello World"}' http://localhost:3000/items
```

你会在控制台看到记录的返回信息，如下所示：

![term_create](http://cdn5.raywenderlich.com/wp-content/uploads/2014/02/term_create-437x320.png)

现在转到你的浏览器，并重新加载 [http://localhost:3000/items](http://localhost:3000/items) ；你就会在表格中看到你插入的项目。

![web_createitem](http://cdn4.raywenderlich.com/wp-content/uploads/2014/02/web_createitem-480x247.png)

## 更新与删除数据

你已经实现了 CRUD 中的 Create 和 Read 操作——还剩下 Update 和 Delete 。这些都比较简单，遵循与其他两个一样的模式。

添加下列代码到 `CollectionDriver.js`，就在 `exports.CollectionDriver` 行之前：

```JavaScript
//update a specific object
CollectionDriver.prototype.update = function(collectionName, obj, entityId, callback) {
    this.getCollection(collectionName, function(error, the_collection) {
        if (error) callback(error);
        else {
            obj._id = ObjectID(entityId); //A convert to a real obj id
            obj.updated_at = new Date(); //B
            the_collection.save(obj, function(error,doc) { //C
                if (error) callback(error);
                else callback(null, obj);
            });
        }
    });
};
```

`update()` 函数接受一个对象，并在 C 行使用 `collectionDriver` 的 `save()` 方法在集合中更新它。这假设 Body 的 `_id` 与 A 行指定的路由一样。B 行添加一个 `updated_at` 字段作为对象更新时间。添加一个修改时间戳是一个好主意，有助于理解数据在你的应用程序的生命周期里是如何改变的。

注意这个更新用新对象操作取代了之前的对象——这里并没有属性级别的更新支持。

添加下列代码到 `CollectionDriver.js`，就在 `exports.CollectionDriver` 行之前：

```JavaScript
//delete a specific object
CollectionDriver.prototype.delete = function(collectionName, entityId, callback) {
    this.getCollection(collectionName, function(error, the_collection) { //A
        if (error) callback(error);
        else {
            the_collection.remove({'_id':ObjectID(entityId)}, function(error,doc) { //B
                if (error) callback(error);
                else callback(null, doc);
            });
        }
    });
};
```

`delete()` 与其他 CRUD 一样的操作。 在 A 行，它获取集合对象，然后在 B 行用提供的 `id` 调用 `remove()` 。

现在你需要两个新的路由来处理这些操作。幸运的是，`PUT` 和 `DELETE` 动词已经存在，所以你可以用与 `GET` 一样的语义创建处理器。

添加如下代码到 `index.js` ，就在 `app.post()` 调用之后：

```JavaScript
app.put('/:collection/:entity', function(req, res) { //A
    var params = req.params;
    var entity = params.entity;
    var collection = params.collection;
    if (entity) {
       collectionDriver.update(collection, req.body, entity, function(error, objs) { //B
          if (error) { res.send(400, error); }
          else { res.send(200, objs); } //C
       });
   } else {
       var error = { "message" : "Cannot PUT a whole collection" };
       res.send(400, error);
   }
});
```

这个 `put` 回调遵循同单实体 `get` 一样的模式：你在集合上匹配名字和 `_id` ，如 A 行所示。和 `post` 路由一样， 在 B 行 `put` 传递来自 Body 的 JSON 对象到 `collectionDriver` 里新写的 `update()` 方法中。

更新的对象将在应答中返回（C 行），所以客户端可以解析到任何服务器更新的字段，例如 `updated_at` 。

添加如下代码到 `index.js` ，就在刚添加的  `put` 方法之后：

```JavaScript
app.delete('/:collection/:entity', function(req, res) { //A
    var params = req.params;
    var entity = params.entity;
    var collection = params.collection;
    if (entity) {
       collectionDriver.delete(collection, entity, function(error, objs) { //B
          if (error) { res.send(400, error); }
          else { res.send(200, objs); } //C 200 b/c includes the original doc
       });
   } else {
       var error = { "message" : "Cannot DELETE a whole collection" };
       res.send(400, error);
   }
});
```

`delete` 端点非常类似于 `put`，如 A 行所示，除了 `delete` 不需要一个 Body。在 B 行，你传递参数给 `collectionDriver` 里的 `delete()` 方法，如果删除操作成功，那么你就在 C 行返回一个原始对象和一个 200 应答码。

如果上述操作中发生任何错误，你就返回一个适当的错误码。

保存你的工作，并重启你的 Node 实例。

在终端执行下列命令，替换 `{_id}`  为上一个 `POST` 调用的返回值：

```Shell
curl -H "Content-Type: application/json" -X PUT -d '{"title":"Good Golly Miss Molly"}' http://localhost:3000/items/{_id}
```

你会在终端看到如下应答：

![term_update](http://cdn2.raywenderlich.com/wp-content/uploads/2014/02/term_update-437x320.png)

转到浏览器，重新载入 [http://localhost:3000/items](http://localhost:3000/items) ；你会在表格中看到你修改的条目：

![web_updated](http://cdn1.raywenderlich.com/wp-content/uploads/2014/02/web_updated-480x247.png)

在终端里执行下列命令以删除你的记录：

```Shell
curl -H "Content-Type: application/json" -X DELETE  http://localhost:3000/items/{_id}
```

你会看到 curl 收到的响应：

![term_delete](http://cdn5.raywenderlich.com/wp-content/uploads/2014/02/term_delete-437x320.png)

重新载入 [http://localhost:3000/items](http://localhost:3000/items) ，我能确定，你的实体不见了。

![web_delete](http://cdn2.raywenderlich.com/wp-content/uploads/2014/02/web_delete-480x247.png)

就这样，你使用 Node.js、Express 以及 MongoDB 完成了你的整个 CRUD 模型！

## 下一步怎么走？

这里是完成的[示例项目](http://www.raywenderlich.com/61078/write-simple-node-jsmongodb-web-service-ios-app/mongodb_sample_project-3)，它包含有上面教程里所有的代码。

你的服务器现在准备好应对客户端的连接并开始传输数据。在本教程的下一部分里，你将构建一个 iOS 应用来连接你的新服务器，并利用一些 MongoDB 和 Express 的炫酷特性。

关于 MongoDB 的更多信息，看看 [官方的 MongoDB 文档](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-os-x/) 。

如果你有任何问题或评论，可自由地加入下方的讨论！ 

===============

译者注：欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条微博  [http://weibo.com/2076580237/B0JsD8YKe](http://weibo.com/2076580237/B0JsD8YKe) 以分享给更多人！

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳译者的幸苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

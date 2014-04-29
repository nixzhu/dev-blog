# 编写一个使用 Node.js/MongoDB Web 服务的 iOS 应用
本文翻译自 [http://www.raywenderlich.com/61264/write-ios-app-uses-node-jsmongodb-web-service](http://www.raywenderlich.com/61264/write-ios-app-uses-node-jsmongodb-web-service)

原作者：[Michael Katz](http://www.raywenderlich.com/u/mkatz)

译者：[@nixzhu](https://twitter.com/nixzhu)

==========================================

欢迎回到本教程系列的第二部分，创建一个以 Node.js 和 MongoDB 为后端的 iOS 应用。

在本系列的[第一部分][2]中，你创建了一个简单的 Node.js 服务器，它通过 REST API 暴露一个 MongoDB 数据库。

在本系列的第二部分（也是最后一部分），你将创建一个有趣的 iPhone 应用，它能让用户标记他们附近有趣的地点，这样就能让更多其他用户发现它们。

作为这个过程的一部分，你将使用一个我们提供的启动项目，然后在它之上添加几个功能：一个使用 `NSURLSession` 的网络层，对地理查询的支持，以及将图像存储到后端的能力。

## 开始

首先 [下载启动项目][3] 并将其解压到你系统中合适的位置。

这个 zip 文件包含两个文件夹：

  * _server_ 包含来自[前一个教程][2]的 Javascript 服务器代码。
  * _TourMyTown_ 包含 Xcode 启动项目，已预设好 UI，但还没有网络相关的代码。

打开 _TourMyTown\TourMyTown.xcodeproj_ 然后编译运行。你会看到如下界面：

译者注：因为要使用用户的地理位置，最好在真机上运行，或者记得设置模拟器的位置。

![starter project][4]

目前为止没有太多事情发生，但在本教程完成时，你会看到的应用大概如下所示：

![TourMyTown screenshot][5]  
TourMyTown 截图

用户添加新的地点标记（Location Markers）到应用里，地点标记包括描述、分类以及图片。按下 _Add_ 按钮将会在地图的中央放置一个标记，用户还可以拖动此标记到所需的地点。另外，按住屏幕并稍微保持一会儿将会在所选择的地方放置一个标记。

视图 delegate 使用 Core Location 的地理编码功能去查找地点的地址和名字，如果它存在的话。在 Annotation View 上点击 _Info_ 按钮将显示细节编辑屏幕。

![Edit point of interest data][6]  
编辑地点数据

这个应用会将所有数据保存到后端，这样它在将来就可以重复使用这些数据了。

![The map with an annotation.][7]  
有一个 Annotation 的地图

要将应用变成我们所希望的状态，还有不少事情要做，所以让我们开始编码吧！

## 设置你的 Node.js 实例

如果你没有完成本系列教程的第一部分，或者你不想使用你自己跟着教程走时所编写的项目，那你可以使用包含在 _server_ 目录的文件作为起始点。

下列说明会带领你设置 Node.js 实例；如果你已经有了来自本教程第一部分的可工作实例，那么可以跳过这些说明，直接进入下一节。

打开终端并导航至 MongoDB 安装目录——例如 _/usr/local/opt/mongodb/_ ，在你的系统上可能稍有差异。

执行下列命令以启动 MongoDB 守护进程：

```Shell
mongod
```

译者注：如我在第一部分的翻译中所提及的，可能并不需要进入 MongoDB 的安装目录，而且直接运行 mongod 可能会出错， `ERROR: dbpath (/data/db) does not exist.`，试试先创建一个自定义路径，再用 `mongod --dbpath '~/somepath'` 来启动服务器。

现在导航至你解压出的 _server_ 目录，执行下列命令：

```Shell
npm install
```

它会读取 _package.json_ 文件并安装服务器的相关依赖。

最后，用下面的命令启动你的 Node.js 服务器：

```Shell
node .
```

>_Note:_ 启动项目已配置为连接到 `localhost` ，端口 3000 。这对于在模拟器中本地运行应用来说很好，但如果你想在物理设备上部署这个应用，如果你的 Mac 和 iOS 设备处于同一个网络，你需要将 `localhost` 改为 `.local` 。而如果它们没有处于同一个网络，你需要将其设置为你的机器的 IP 地址。你会在 _Locations.m_ 的顶部附近找到这些值。

## 应用的数据模型 

项目中的 _Location_ 类表示单个有趣的地点及其关联数据，它会做下列事情：

  * 保持地点的数据，包括它的坐标、描述和类别。
  * 知道如何将对象序列化为 JSON 兼容的 `NSDictionary` ，并能反序列化。
  * 实现 `MKAnnotation` 协议，因此它能作为一个大头针被放置在某个 `MKMapView` 实例上。 
  * 有零个或多个定义在  _Categories.m_ 里的类别。

_Locations_ 类代表应用中 Location 对象的集合，以及从服务器加载这些对象的机制，这个类负责：

  * 提供一个叫做 `filteredObjects` 的可过滤的地点列表作为应用的数据模型。
  * 通过 `import`、`persist` 和 `query` 与服务器通信以加载或保存地点条目。

_Categories_ 包含一个类别列表， Location 就可以属于这些类别，然后可以通过类别来过滤地点列表。_Categories_ 还会做如下这些事情：

  * 存有 `allCategories` ，它提供类别的主列表。你还可以添加额外的类别到这个数组里。
  * 提供一个活动地点集合中所有类别的列表。
  * 通过类别过滤地点。

## 从服务器加载地点

用下列代码替换 _Locations.m_ 中 `import` 的实现：

```Objective-C
- (void)import
{
    NSURL* url = [NSURL URLWithString:[kBaseURL stringByAppendingPathComponent:kLocations]]; //1
 
    NSMutableURLRequest* request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET"; //2
    [request addValue:@"application/json" forHTTPHeaderField:@"Accept"]; //3
 
    NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration]; //4
    NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
 
    NSURLSessionDataTask* dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) { //5
        if (error == nil) {
            NSArray* responseArray = [NSJSONSerialization JSONObjectWithData:data options:0 error:NULL]; //6
            [self parseAndAddLocations:responseArray toArray:self.objects]; //7
        }
    }];
 
    [dataTask resume]; //8
}
```

此处 `import` 做了：

  1. 最重要的信息是 URL 和请求头。 URL 是由 base URL 与 “locations” 简单串联而来。
  2. 因为是从服务器读取数据，所以使用 `GET` 。GET 是默认的方法，是不需要特别指明的，但这里指明这有助于完整性和清晰性。
  3. 服务器代码使用 `Accept` 头的内容作为提示以确定哪一种响应会被发出。通过在请求中指定会接受 JSON 作为响应，那么返回的数据就是 JSON 而不是通常的 HTML 。
  4. 在此你使用默认配置创建了一个 _NSURLSession_ 实例。
  5. 一个 _data task_ 是 `NSURLSession` 的基本任务，用于从web服务里传输数据。还有其他特定用于上传和下载的任务，用于长期运行的传输和后台操作。一个数据任务在后台线程上异步运行，所以你要使用一个回调 Block 来响应操作的完成或失败。
  6. 完成处理器（Completion Handler）先检查是否有任何错误；如果没有错误，它就使用 `NSJSONSerialization` 类方法来反序列化数据。
  7. 假设返回值是一个地点数组，`parseAndAddLocations:` 解析对象并通知视图控制器已有数据更新。
  8. 很奇怪，数据任务使用 `resume` 消息来开始。当你创建一个 NSURLSessionTask 实例后，它就处于  “暂停（paused）” 状态，所以简单的调用 `resume` 使其开始。

还是在同一个文件里，用下列代码替换 `parseAndAddLocations:` 的实现：

```Objective-C
- (void)parseAndAddLocations:(NSArray*)locations toArray:(NSMutableArray*)destinationArray //1
{
    for (NSDictionary* item in locations) { 
        Location* location = [[Location alloc] initWithDictionary:item]; //2
        [destinationArray addObject:location];        
    }
 
    if (self.delegate) {
        [self.delegate modelUpdated]; //3
    }
}
```

按顺序看看每个注释：

  1. 遍历 JSON 字典组成的列表并为每个条目创建一个新的 Location 对象。
  2. 使用自定义的初始化方法将反序列化的 JSON 字典变为一个 _Location_ 实例。
  3. 模型通知 UI 有新的对象。

合在一起，这两个方法让你的应用在启动时从服务器加载数据。 `import` 依赖  `NSURLSession` 去处理繁重的网络操作。对于 `NSURLSession` 的内部运作情况，请看看本站的 [NSURLSession][16] 

注意到 `Location` 类已经有了如下的初始化方法，它从字典中获取各种值然后设置到相应对象合适的属性上。

```Objective-C
- (instancetype) initWithDictionary:(NSDictionary*)dictionary
{
    self = [super init];
    if (self) {
        self.name = dictionary[@"name"];
        self.location = dictionary[@"location"];
        self.placeName = dictionary[@"placename"];
        self.imageId = dictionary[@"imageId"];
        self.details = dictionary[@"details"];
        _categories = [NSMutableArray arrayWithArray:dictionary[@"categories"]];
    }
    return self;
}
```

## 保存地点到服务器

很明显，从一个空空如也的数据库里加载地点信息并不有趣。你接下来的任务是实现将地点信息保存到数据库的功能。

用下列代码替换 _Locations.m_ 中的 `persist:` 实现：

```Objective-C
- (void) persist:(Location*)location
{
    if (!location || location.name == nil || location.name.length == 0) {
        return; //input safety check
    }
 
 
    NSString* locations = [kBaseURL stringByAppendingPathComponent:kLocations];
 
    BOOL isExistingLocation = location._id != nil;
    NSURL* url = isExistingLocation ? [NSURL URLWithString:[locations stringByAppendingPathComponent:location._id]] :
    [NSURL URLWithString:locations]; //1
 
    NSMutableURLRequest* request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = isExistingLocation ? @"PUT" : @"POST"; //2
 
    NSData* data = [NSJSONSerialization dataWithJSONObject:[location toDictionary] options:0 error:NULL]; //3
    request.HTTPBody = data;
 
    [request addValue:@"application/json" forHTTPHeaderField:@"Content-Type"]; //4
 
    NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
 
    NSURLSessionDataTask* dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) { //5
        if (!error) {
            NSArray* responseArray = @[[NSJSONSerialization JSONObjectWithData:data options:0 error:NULL]];
            [self parseAndAddLocations:responseArray toArray:self.objects];
        }
    }];
    [dataTask resume];
}
```

`persist:` 类似 `import` ，同样使用一个 `NSURLSession` 请求后端的 `locations`。然而，它们还有以下不同之处：

  1. 有两个保存对象的端点：如果是添加一个新地点，使用 `/locations`，如果是更新一个已存在的地点，则使用 `/locations/_id` ，`id` 就表明了已存在的地点。
  2. 这个请求使用 `PUT` 用于已存在对象，或使用  `POST` 用于新对象。服务器代码会据此调用合适的处理器而不是使用默认的 `GET` 处理器。
  3. 因为你是在更新一个实体，你在请求中提供了一个 `HTTPBody` ，它是由 `NSJSONSerialization` 类创建的 `NSData` 实例。
  4. 你提供一个 `Content-Type` 而不是 `Accept` 头。这会告知服务器上的 `bodyParser` 如何处理 body 里的数据。
  5. 完成处理器再一次接受从服务器返回的修改后的实体，解析它并将其放入本地的 Location 对象集合里。

注意到，如同 `initWithDictionary:` ，_Location.m_ 已经有了一个帮助模块可以处理 Location 对象到 JSON 兼容的字典的转换，如下所示：

```Objective-C
#define safeSet(d,k,v) if (v) d[k] = v;
- (NSDictionary*) toDictionary
{
    NSMutableDictionary* jsonable = [NSMutableDictionary dictionary];
    safeSet(jsonable, @"name", self.name);
    safeSet(jsonable, @"placename", self.placeName);
    safeSet(jsonable, @"location", self.location);
    safeSet(jsonable, @"details", self.details);
    safeSet(jsonable, @"imageId", self.imageId);
    safeSet(jsonable, @"categories", self.categories);
    return jsonable;
}
```


`toDictionary` 包含一个神奇的宏：`safeSet()` 。它在将某个值放入 NSDictionary 前会先检查那值是否不是 `nil` ；这件避免了抛出  `NSInvalidArgumentException` 异常。你需要这个检查是因为你的应用不能强制你的对象填充哪些属性。

你可能会问：“为何不使用 `NSCoder` ？” 因为 `NSCoding` 协议用 `NSKeyedArchiver` 可以做到与 `toDictionary` 和 `initWithDictionary` 一样的事情；即，提供一个键值对象转换。

然而，`NSKeyedArchiver` 是为了和 `plists` 一起工作而设计的，这是一个不同的格式，有着略微不同的数据类型。上面所用的方式比利用 `NSCoding` 机制稍微简单。
 

## 保存图像到服务器


启动项目已经有了给地点添加图片的机制；这是一种在应用内浏览数据的好方式。图片在地图 Annotation 上和细节屏幕上都以缩略图形式显示。Location 对象已经有一个 `imageId` 属性，它用于保存服务器上某个文件的链接。

添加图像需要做两件事：客户端调用保存和加载图像，服务器端需要存储图像。

回到终端，确保你在 _server_ 目录，执行下列明明以创建一个新文件放置你的文件处理器代码：

```Shell
edit fileDriver.js
```

添加下列代码到 _fileDriver.js_ ：

```JavaScript
var ObjectID = require('mongodb').ObjectID, 
    fs = require('fs'); //1
 
FileDriver = function(db) { //2
  this.db = db;
};
```

它用下列步骤设置你的 _FileDriver_ 模块：

  1. 这个模块使用文件系统模块  _fs_ 去读写硬盘。 
  2. 这个构造器接受一个到 MongoDB 数据库驱动器的引用，以便在后续方法中使用。

添加下列代码到 _fileDriver.js_ ，就在上面添加的代码之后：

```JavaScript
FileDriver.prototype.getCollection = function(callback) {
  this.db.collection('files', function(error, file_collection) { //1
    if( error ) callback(error);
    else callback(null, file_collection);
  });
};
```

`getCollection()` 查找 `files` 集合；除了文件本身的内容，每个文件还有一个条目在 `files` 集合里，它保存文件的元数据，包括它在硬盘上存放的位置。

添加如下代码到刚才你添加的代码块后面：


```JavaScript
//find a specific file
FileDriver.prototype.get = function(id, callback) {
    this.getCollection(function(error, file_collection) { //1
        if (error) callback(error);
        else {
            var checkForHexRegExp = new RegExp("^[0-9a-fA-F]{24}$"); //2
            if (!checkForHexRegExp.test(id)) callback({error: "invalid id"});
            else file_collection.findOne({'_id':ObjectID(id)}, function(error,doc) { //3
                if (error) callback(error);
                else callback(null, doc);
            });
        }
    });
};
```

下面说明以上代码的功能：

  1. `get` 从数据库中获取 files 集合。
  2. 由于输入到这个函数的字符串表示对象的 `_id`，你必须将其转换为一个 BSON ObjectID 对象。 
  3. `findOne()` 找到那个匹配的实体（如果它存在的话）。

添加如下代码到刚才你添加的代码块后面：


```JavaScript
FileDriver.prototype.handleGet = function(req, res) { //1
    var fileId = req.params.id;
    if (fileId) {
        this.get(fileId, function(error, thisFile) { //2
            if (error) { res.send(400, error); }
            else {
                    if (thisFile) {
                         var filename = fileId + thisFile.ext; //3
                         var filePath = './uploads/'+ filename; //4
    	                 res.sendfile(filePath); //5
    	            } else res.send(404, 'file not found');
            }
        });        
    } else {
	    res.send(404, 'file not found');
    }
};
```

`handleGet` 是一个 Express 路由器要使用的请求处理器。它通过从 _index.js_ 中抽象出文件处理，简化了服务器代码。它执行下列操作：

  1. 通过提供的 id 从数据库中获取文件实体。
  2. 添加存储在数据库条目里的扩展名到 id 上，以创建文件名
  3. 将文件存储在本地的 `uploads` 目录里。
  4. 在应答对象上调用 `sendfile()` ；这个方法知道如何传输文件和设置合适的响应头。

再一次，添加如下代码到刚才你添加的代码块后面：


```JavaScript
//save new file
FileDriver.prototype.save = function(obj, callback) { //1
    this.getCollection(function(error, the_collection) {
      if( error ) callback(error);
      else {
        obj.created_at = new Date();
        the_collection.insert(obj, function() {
          callback(null, obj);
        });
      }
    });
};
```

上面的 `save()` 和 _collectionDriver_ 里的那个一样；它插入一个新的对象到 files 集合里。

添加如下代码到刚才你添加的代码块后面：


```JavaScript
FileDriver.prototype.getNewFileId = function(newobj, callback) { //2
	this.save(newobj, function(err,obj) {
		if (err) { callback(err); } 
		else { callback(null,obj._id); } //3
	});
};
```

  1. `getNewFileId()` 是 `save` 的包装器，以便创建一个新的文件实体并单独返回 `id` 。
  2. 只返回新创建对象的 `id` 。

添加如下代码到刚才你添加的代码块后面：

```JavaScript
FileDriver.prototype.handleUploadRequest = function(req, res) { //1
    var ctype = req.get("content-type"); //2
    var ext = ctype.substr(ctype.indexOf('/')+1); //3
    if (ext) {ext = '.' + ext; } else {ext = '';}
    this.getNewFileId({'content-type':ctype, 'ext':ext}, function(err,id) { //4
        if (err) { res.send(400, err); } 
        else { 	         
             var filename = id + ext; //5
             filePath = __dirname + '/uploads/' + filename; //6
 
	     var writable = fs.createWriteStream(filePath); //7
	     req.pipe(writable); //8
             req.on('end', function (){ //9
               res.send(201,{'_id':id});
             });               
             writable.on('error', function(err) { //10
                res.send(500,err);
             });
        }
    });
};
 
exports.FileDriver = FileDriver;
```

这个方法做了不少事情，花点时间根据注释看一看：

  1. `handleUploadRequest` 创建一个新的对象到 files 集合里，它使用 `Content-Type` 来决定文件的扩展名并返回新对象的 `_id`。
  2. 查找由应用设置的 `Content-Type` 的值
  3. 尝试通过 Content Type 得到文件扩展名。例如， `image/png` 对应着 `png` 这个扩展名。
  4. 将 `Content-Type` 和 `extension` 存储到文件集合实体里。
  5. 通过将合适的扩展名连接到新的 `id` 后面来创建一个文件名。
  6. 文件的特定路径是服务器的根目录，在  _uploads_ 子目录之下，`__dirname` 是 Node.js 执行脚本的目录。
  7. `fs` 包含 `writeStream` 它——如你所猜测的——是一个输出流。
  8. 请求对象同样有一个 `readStream` ，所以你可以用 `pipe()` 函数将其导入到某个 write stream 里。这些 [Stream][19] 对象是 Node.js 时间驱动范式的绝好例子。
  9. `on()` 用一个回调关联 stream 事件。在这个例子里，`readStream` 的 `end` 事件将在 pipe 操作完成时发生，返回给 Express 的应答是一个 201 状态码和一个新的文件 `_id`。
  10. 如果 write stream 抛出一个 `error` 事件，那么将有一个 error 被写入文件。服务器应答返回一个 500 内部服务器错误与一个适当的文件系统错误。

由于上面的代码需要有一个 _uploads_ 子目录，那就在终端里执行下列命令：

```Shell
mkdir uploads
```

添加如下代码到 _index.js_ 顶部的  `require` 代码块之后：

```JavaScript
FileDriver = require('./fileDriver').FileDriver;
```

接下来，再添加下面的代码到 _index.js_ ，就在 `var mongoPort = 27017;` 这一行下面：

```JavaScript
var fileDriver;
```

添加下面的代码到 _index.js_ ，就在 `var db = mongoClient.db("MyDatabase");` 这一行下面：

在 monglClient 设置回调创建一个 FileDriver 实例，就在 CollectionDriver 创建后：


```JavaScript
fileDriver = new FileDriver(db);
```

这会创建一个新的 `FileDriver` 实例。

添加如下代码到  _index.js_ ，就在通用的 `/:collection` 路由之前：

```JavaScript
app.use(express.static(path.join(__dirname, 'public')));
app.get('/', function (req, res) {
  res.send('<html><body><h1>Hello World</h1></body></html>');
});
 
app.post('/files', function(req,res) {fileDriver.handleUploadRequest(req,res);});
app.get('/files/:id', function(req, res) {fileDriver.handleGet(req,res);});
```

将这些代码放在通用的 `/:collection` 路由之前意味着将 _files_ 不同于通用的 _files_ 集合来对待。

保存你的工作，若有必要，通过 Ctrl + C 干掉你的 Node 实例再使用如下命令重启它：

```Shell
node index.js
```

现在你的服务器可以处理文件了，这也意味着你需要修改应用，让它 Post 图像到服务器。

## 在应用中保存图像

Location 类有两个属性：`image` 和 `imageId`。 `imageId` 是一个后端属性，它将 `locations` 集合的某个实体链接一个 `files` 集合的实体。如果这是关系数据库，你可以使用一个外键来表示这个链接。`image` 存储着实际的 `UIImage` 对象。

保存和加载文件要求一个额外的请求，每个对象都要再传输文件数据。操作的顺序很重要，以确保文件 id 与合适的对象关联。当你保存一个文件，你必须先发送文件以接收到关联的 id 再将其链接到地点数据上。

添加如下代码到 _Locations.m_ 的底部：


```Objective-C
- (void) saveNewLocationImageFirst:(Location*)location
{
    NSURL* url = [NSURL URLWithString:[kBaseURL stringByAppendingPathComponent:kFiles]]; //1
    NSMutableURLRequest* request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST"; //2
    [request addValue:@"image/png" forHTTPHeaderField:@"Content-Type"]; //3
 
    NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
 
    NSData* bytes = UIImagePNGRepresentation(location.image); //4
    NSURLSessionUploadTask* task = [session uploadTaskWithRequest:request fromData:bytes completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) { //5
        if (error == nil && [(NSHTTPURLResponse*)response statusCode] < 300) {
            NSDictionary* responseDict = [NSJSONSerialization JSONObjectWithData:data options:0 error:NULL];
            location.imageId = responseDict[@"_id"]; //6
            [self persist:location]; //7
        }
    }];
    [task resume];
}
```

这是个相当忙碌的模块，但只要你一句一句地看，它还是比较简单的：

  1. URL 表示 _files_ 端点。
  2. 使用 `POST` 触发 `fileDriver` 的 `handleUploadRequest` 以保存文件。
  3. 设置 Content Type 以确保文件被正确保存到服务器上。`Content-Type` 头对于服务器决定文件扩展名来说很重要。
  4. 将一个 `UIImage` 实例变为 PNG 文件数据。
  5. `NSURLSessionUploadTask` 让你发送 `NSData` 到服务器。例如，上传任务自动根据数据长度设置 `Content-Length` 头。上传任务会报告进度并能在后台运行，但对于这两个特性，这里都没有使用。
  6. 响应里包含有新的文件数据实体，所以你保存 `_id` 到地点对象中以便日后使用。
  7. 一旦图像被保存而且 `_id` 被记录，那么地点实体就可以保存到服务器上了。

添加如下代码到  _Location.m_ 中的 `persist:` 里，就在 `if (!location || location.name == nil || location.name.length == 0)` 语句块的后面：

```Objective-C
- (void) persist:(Location*)location
 
    //if there is an image, save it first
    if (location.image != nil && location.imageId == nil) { //1
        [self saveNewLocationImageFirst:location]; //2
        return;
    }
```

这会检查新图像的存在性，并第一次保存图像。看看注释的语句说了什么：

  1. 如果有图像但没有图像 id，那么图像没有被保存过。
  2. 调用新方法保存图像，然后退出。

一旦保存完成，`persist:` 会被再次调用，但在那时 `imageId` 不会是 nil ，and the code will proceed into the existing procedure for saving the Location entity.


接下来，用下列代码替换 _Location.m_ 中的 `loadImage:` ：

```Objective-C
- (void)loadImage:(Location*)location
{
    NSURL* url = [NSURL URLWithString:[[kBaseURL stringByAppendingPathComponent:kFiles] stringByAppendingPathComponent:location.imageId]]; //1
 
    NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
 
    NSURLSessionDownloadTask* task = [session downloadTaskWithURL:url completionHandler:^(NSURL *fileLocation, NSURLResponse *response, NSError *error) { //2
        if (!error) {
            NSData* imageData = [NSData dataWithContentsOfURL:fileLocation]; //3
            UIImage* image = [UIImage imageWithData:imageData];
            if (!image) {
                NSLog(@"unable to build image");
            }
            location.image = image;
            if (self.delegate) {
                [self.delegate modelUpdated];
            }
        }
    }];
 
    [task resume]; //4
}
```

下面解释上面的代码做了什么：

  1. 就像加载一个特定的地点一样，图像的 id 连接到代表 `files` 端点的路径之后。
  2. 下载任务是第三种  `NSURLSession` ；它下载一个文件到临时存储位置并返回一个到那个位置的 URL，而不是原始的  `NSData` 对象，因为原始对象可能相当大。
  3. 临时存储位置只保证在完成 Block 执行期间可用，所以你必须将其装载进入内存，或者将其移动到其他地方。
  4. 如同所有的 `NSURLSession` 任务，你使用 `resume` 让其开始。

接下来，用下列代码替换当前的 `parseAndAddLocations:toArray:` ：


```Objective-C
- (void)parseAndAddLocations:(NSArray*)locations toArray:(NSMutableArray*)destinationArray
{
    for (NSDictionary* item in locations) {
        Location* location = [[Location alloc] initWithDictionary:item];
        [destinationArray addObject:location];
 
        if (location.imageId) { //1
            [self loadImage:location]; 
        }
    }
 
    if (self.delegate) {
        [self.delegate modelUpdated];
    }
}
```

更新后的 `parseAndAddLocations:toArray:` 的会检查 imageId；如果找到了，它就调用 `loadImage:` 。

## 文件处理的快速小结

总的来说：文件传输在 iOS 应用工作概念上和常规的数据传输的方式一样。大的区别是你使用的 `NSURLSessionUploadTask` 和 `NSURLSessionDownloadTask` 在对象和语义上与 `NSURLSessionDataTask` 略有不同。

在服务器端，文件处理是另外一头不同的野兽。它要求一个特别的处理器对象，这个对象与文件系统而不是 Mongo 数据库通信，但依然需要存储一些元数据到数据库中以便更容易地检索。

特殊路由被设置为映射传入的 HTTP 动词和端点到文件驱动器。你也 _能够_ 用通用数据端点来做到这一点，但当要确定在何处持久化数据时，代码会变得相当复杂。

## 测试

编译并运行你的应用，然后点击右上角的按钮添加一个新的地点。

作为创建新地点的一部分，要添加一个图像。记住你可以在模拟器的 Safari 中通过长按图片添加多个图像到模拟器中的相机胶卷了，便于测试。

一旦你保存了新的地点，就重启应用——然后观察，应用会顺利地重新载入你的数据，截图如下：

![Adding an image to a Location in Tour My Town.][21]  
添加一个图像到地点中

![Location annotation with an image.][22]  
一个有图像的地点 Annotation

## 查询地点

你这超级流行的 Tour My Town 应用会在释出后以令人难以置信的速度收集到大量数据。为了避免下载所有数据到应用里造成漫长的等待，你可以通过基于位置的过滤来限制数据量。这样你就只需要检索出那些只会出现在屏幕上的数据。

MongoDB 有一个强大的特性，即能找到匹配给定条件（criteria）的实体。这些[条件][23]可以是基本比较、类型检查、表达式求值（包括正则表达式和任意JavaScript），以及地理空间查询（Geospatial Querying）。

MongoDB 的地理空间查询对于基于映射的应用来说实乃天作之合。你可以使用地图视图的范围来获得那些只需要显示在屏幕上的数据组成的子集

下一个任务是修改  _collectionDriver.js_ 以通过 GET 请求提供过滤条件。

添加如下方法到 _collectionDriver.js_ 中最后一行 `exports` 之上：


```JavaScript
//Perform a collection query
CollectionDriver.prototype.query = function(collectionName, query, callback) { //1
    this.getCollection(collectionName, function(error, the_collection) { //2
      if( error ) callback(error)
      else {
        the_collection.find(query).toArray(function(error, results) { //3
          if( error ) callback(error)
          else callback(null, results)
        });
      }
    });
};
```

下面说明这个方法的功能：

  1. `query` 类似于已存在的 `findAll`，除了它有一个 `query` 参数指定过滤条件。
  2. 获取集合以访问对象，就像其他方法一样。
  3. `CollectionDriver` 的 `findAll` 方法使用没有参数的 `find()` ，但这里 `query` 对象以参数的形式传递进去。这会通过 MongoDB 的评估，因此只有匹配的文档才会在结果中返回。

>_Note:_ 这里直接传递 query 对象到 MongoDB里。在公开 API 的情况下，这可能会非常危险，因为 MongoDB 允许通过  `$where` 查询操作符执行任意 JavaScript 代码。这会带来一些风险，包括运行崩溃、无法预料的结果或安全问题；但在本教程的项目中只使用了有限的一组操作，只是一个微不足道的问题。

回到 _index.js_ 用下列代码替换 `app.get('/:collection'...` 代码块：


```JavaScript
app.get('/:collection', function(req, res, next) {  
   var params = req.params;
   var query = req.query.query; //1
   if (query) {
        query = JSON.parse(query); //2
        collectionDriver.query(req.params.collection, query, returnCollectionResults(req,res)); //3
   } else {
        collectionDriver.findAll(req.params.collection, returnCollectionResults(req,res)); //4
   }
});
 
function returnCollectionResults(req, res) {
    return function(error, objs) { //5
        if (error) { res.send(400, error); }
	        else { 
                    if (req.accepts('html')) { //6
                        res.render('data',{objects: objs, collection: req.params.collection});
                    } else {
                        res.set('Content-Type','application/json');
                        res.send(200, objs);
                }
        }
    };
};
```

  1. HTTP 查询可以添加到 URL 后面，形式为 `http://domain/endpoint?key1=value1&amp;key2=value2...` 。用 `req.query` 可以得到传入的 URL 中的整个“查询”部分。
  2. 这个查询值在 MongoDB 条件对象里应该以一个字符串呈现。`JSON.parse()` 将 JSON字符串变为一个 JavaScript 对象，然后直接传入 MongoDB。
  3. 如果一个查询被提供给端点，就调用  `collectionDriver.query()`；`returnCollectionResults` 是一个通用的帮助函数，它负责格式化请求的输出。
  4. 如果没有查询被提供，那么 `collectionDriver.findAll` 返回集合所有的条目。
  5. 由于 `returnCollectionResults()` 在它被调用时就执行，这个函数为集合驱动器返回一个回调函数。
  6. 如果请求指定 HTML 作为响应答，那么将数据渲染为 HTML 表格；否则在 body 中返回一个 JSON 文件。

保存你的工作，干掉 Node.js 实例并重启它：

```Shell
node index.js
```

现在服务器已设置好查询，你该在应用里添加地理查询功能了。

用下列代码替换 _Locations.m_  中 `queryRegion` 的实现：


```Objective-C
- (void) queryRegion:(MKCoordinateRegion)region
{
    //note assumes the NE hemisphere. This logic should really check first.
    //also note that searches across hemisphere lines are not interpreted properly by Mongo
    CLLocationDegrees x0 = region.center.longitude - region.span.longitudeDelta; //1
    CLLocationDegrees x1 = region.center.longitude + region.span.longitudeDelta;
    CLLocationDegrees y0 = region.center.latitude - region.span.latitudeDelta;
    CLLocationDegrees y1 = region.center.latitude + region.span.latitudeDelta;
 
    NSString* boxQuery = [NSString stringWithFormat:@"{\"$geoWithin\":{\"$box\":[[%f,%f],[%f,%f]]}}",x0,y0,x1,y1]; //2
    NSString* locationInBox = [NSString stringWithFormat:@"{\"location\":%@}", boxQuery]; //3
    NSString* escBox = (NSString *)CFBridgingRelease(CFURLCreateStringByAddingPercentEscapes(NULL,
                                                                                  (CFStringRef) locationInBox,
                                                                                  NULL,
                                                                                  (CFStringRef) @"!*();':@&=+$,/?%#[]{}",
                                                                                  kCFStringEncodingUTF8)); //4
    NSString* query = [NSString stringWithFormat:@"?query=%@", escBox]; //5
    [self runQuery:query]; //7
}
```

这些代码相当简单； `queryRegion:` 将一个从 MKMapView 生成的 Map Kit 区域变为 bounded-box 查询，它是这样做的：

  1. 这四行用 bounding box 的对角计算地图坐标。
  2. 这里使用 MongoDB 的特定查询语言为查询定义一个 JSON 结构。一个有 `$geoWithin` 键值的查询指定一个搜索条件，结构里的一切都通过所提供的值来定义。`$box` 指定通过提供的坐标定义的矩形，并被作为一个有着两个对角的“经度纬度对”的数组。
  3. `boxQuery` 定义了条件值；你同时还需要提供搜索键值字段与 `boxQuery` 一道作为一个 JSON 对象传到 MongoDB。
  4. 然后你 Escape 整个查询对，它会作为 URL 的一部分被 Post；你需要确保内部的引号、括号、逗号以及其他非字母数字符号不会被解释为 HTTP 查询参数的一部分。`CFURLCreateStringByAddingPercentEscapes` 是一个 CoreFoundation 方法，用于创建URL 编码的字符串
  5. 字符串构建的最后部分是设置整个 Escaped MongoDB 查询作为 URL 中的查询值。
  6. 然后使用你的新查询从服务器请求匹配的值。

>_Note:_ 在 MongoDB 里，坐标对被指定为 [longitude（经度）, latitude（纬度）] ，它刚好和常见的 lat/long 对相反（例如 Google Maps API 里使用的）。

用下列代码替换  _Locations.m_ 中 `runQuery:` 的实现：

```Objective-C
- (void) runQuery:(NSString *)queryString
{
    NSString* urlStr = [[kBaseURL stringByAppendingPathComponent:kLocations] stringByAppendingString:queryString]; //1
    NSURL* url = [NSURL URLWithString:urlStr];
 
    NSMutableURLRequest* request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"GET";
    [request addValue:@"application/json" forHTTPHeaderField:@"Accept"];
 
    NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
 
    NSURLSessionDataTask* dataTask = [session dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error == nil) {
            [self.objects removeAllObjects]; //2
            NSArray* responseArray = [NSJSONSerialization JSONObjectWithData:data options:0 error:NULL];
            NSLog(@"received %d items", responseArray.count);
            [self parseAndAddLocations:responseArray toArray:self.objects];
        }
    }];    
     [dataTask resume];
}
```

`runQuery:` 非常类似于 `import` ，但有两个重要区别：

  1. 你要添加 `queryRegion:` 生成的查询字符串到 locations 端点 URL 的后面。
  2. 你还要丢弃前一组地点并用从服务器返回的过滤集合替换。

编译并运行；创建一写新的有趣的地点，分散在地图上。放大一点点，然后滑动并缩放地图并观察 NSLog 显示不断改变的地图范围内/外的条目数量，如下所示：
 
![Debugger output while panning and zooming map.][24]  
滑动和缩放地图时的调试输出


## 使用查询按类别筛选


最后一点是添加 _categories_ 到你的 Locations ，这样用户能用来过滤。这个过滤能够重用服务器在前一节使用 MongoDB 的数组条件运算完成的工作。

用下列代码替换 _Categories.m_ 中 `query` 的实现：


```Objective-C
+ (NSString*) query
{
    NSArray* a = [self filteredCategories:YES]; //1
    NSString* query = @"";
    if (a.count > 0) {
 
        query = [NSString stringWithFormat:@"{\"categories\":{\"$in\":[%@]}}", [a componentsJoinedByString:@","]]; //2
        query = (NSString *)CFBridgingRelease(CFURLCreateStringByAddingPercentEscapes(NULL,
                                                                                           (CFStringRef) query,
                                                                                           NULL,
                                                                                           (CFStringRef) @"!*();':@&=+$,/?%#[]{}",
                                                                                           kCFStringEncodingUTF8));
 
        query = [@"?query=" stringByAppendingString:query];
    }
    return query;
}
```

这里创建的查询字符串类似前面地理查询里使用的那个，但有如下不同：

  1. 这是一个所选类别的列表。
  2. `$in` 操作符接收一个 MongoDB 文件，如果指定的 `categories` 属性有一个值能匹配相应数组中的任意某些条目。

编译并运行；添加一些地点，并给它们分配一个或多个类别。点击文件夹图标，选择一个类别开始过滤。地图会重新加载那些只符合选定类别的地点Annotations，如下所示：

![A map with many locations][25]  
有多个地点的地图

![Select just the "Park" category][26]  
只选择 Park 类别

![Map after filtering][27]  
过滤后的地图

## 下一步怎么走？

你可以在此[下载完成的示例项目][28]。

在本教程中，你覆盖了 MongoDB 存储的基本内容——但还有其它非常多的功能没有被覆盖到。

MongoDB 对于从数据库中筛选数据提供了众多选项；也有很多服务器端的特性可用于管理缩放和安全。而且，你的 Node.js 应用一定可以通过添加用户验证和更多围绕数据的隐私来得到改善。

至于你的 iOS 应用，你还可以添加一些有趣的功能，包括：

  * 将用户导向感兴趣的地点
  * 给地点添加更多媒体信息
  * 改进文本编辑体验

另外，每个像样的网络应用都应该在本地缓存数据，这样当网络连接不稳定时，它仍然具有功能性。

希望你在这个尝试 Node.js 、 Express 和 MongoDB 的过程中收获了乐趣——如果你有任何问题或评论，欢迎加入下面的讨论！

===============

译者注：欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳译者的幸苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)



   [1]: http://cdn2.raywenderlich.com/wp-content/uploads/2014/03/MongoDB.png
   [2]: https://github.com/nixzhu/dev-blog/blob/master/2014-04-21-write-a-simple-nodejs-mongodb-web-service-for-an-ios-app.md
   [3]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/03/node_tutorial_starter_project.zip
   [4]: http://cdn1.raywenderlich.com/wp-content/uploads/2014/03/iOS-Simulator-Screen-shot-Apr-8-2014-12.17.09-PM-281x500.png
   [5]: http://cdn4.raywenderlich.com/wp-content/uploads/2014/01/Tour-My-Town-Screenshot-1-180x320.png
   [6]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/01/TourMyTown-Screenshot-2-180x320.png
   [7]: http://cdn4.raywenderlich.com/wp-content/uploads/2014/01/TourMyTown-Screenshot-3-180x320.png
   [8]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSURL_Class/
   [9]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSMutableURLRequest_Class/
   [10]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSData_Class/
   [11]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSURLResponse_Class/
   [12]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSError_Class/
   [13]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSArray_Class/
   [14]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSMutableArray_Class/
   [15]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSDictionary_Class/
   [16]: http://www.raywenderlich.com/51127/nsurlsession-tutorial (NSURLSession Tutorial)
   [17]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSString_Class/
   [18]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSMutableDictionary_Class/
   [19]: http://nodejs.org/api/stream.htm
   [20]: http://developer.apple.com/documentation/Cocoa/Reference/Foundation/Classes/NSHTTPURLResponse_Class/
   [21]: http://cdn2.raywenderlich.com/wp-content/uploads/2014/02/TMT_add_image-213x320.png
   [22]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/02/TMT_marker_with_image-213x320.png
   [23]: http://docs.mongodb.org/manual/reference/operator/query/
   [24]: http://cdn3.raywenderlich.com/wp-content/uploads/2014/02/debug_output_filter.png
   [25]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/02/iOS-Simulator-Screen-shot-Feb-8-2014-8.54.05-PM-213x320.png
   [26]: http://cdn5.raywenderlich.com/wp-content/uploads/2014/02/iOS-Simulator-Screen-shot-Feb-8-2014-8.53.59-PM-213x320.png
   [27]: http://cdn3.raywenderlich.com/wp-content/uploads/2014/02/iOS-Simulator-Screen-shot-Feb-8-2014-8.54.02-PM-213x320.png
   [28]: http://www.raywenderlich.com/61264/write-ios-app-uses-node-jsmongodb-web-service/node_tutorial_sample_app_complete
  
# 编写一个使用 Node.js/MongoDB Web 服务的 iOS 应用
本文翻译自 [http://www.raywenderlich.com/61264/write-ios-app-uses-node-jsmongodb-web-service](http://www.raywenderlich.com/61264/write-ios-app-uses-node-jsmongodb-web-service)

原作者：[Michael Katz](http://www.raywenderlich.com/u/mkatz)

译者：[@nixzhu](https://twitter.com/nixzhu)

==========================================

Welcome back to the second part of this two-part tutorial series on creating an iOS app with a Node.js and MongoDB back-end.

欢迎回到本教程系列的第二部分，创建一个以 Node.js 和 MongoDB 为后端的 iOS 应用。

In the [first part][2] of this series, you created a simple Node.js server to expose MongoDB through a REST API.

在本系列的第一部分中，你创建了一个简单的 Node.js 服务器，它通过 REST API 暴露 MongoDB 数据库。

In this second and final part of the series, you’ll create a fun iPhone application that lets users tag interesting places near them so other users can discover them.

在本系列的第二部分（也是最后一部分），你将创建一个有趣的 iPhone 应用，它能让用户标记他们附近的有趣的位置，这样就能让其他用户发现它们。

As part of this process you’ll take the starter app provided and add several things: a networking layer using `NSURLSession`, support for geo queries and the ability to store images on the backend.

作为这个过程的一部分，你将使用一个我们提供的启动项目，然后在它之上添加几个功能：一个使用 `NSURLSession` 的网络层，支持地理查询，以及将图像存储到后端的能力。

## 开始

First things first: [download the starter project][3] and extract it to a convenient location on your system.

首先 [下载启动项目][3] 并将其解压到你系统中合适的位置。

The zip file contains two folders:

这个 zip 文件包含两个文件夹：

  * _server_ contains the javascript server code from the [previous tutorial][2]. _server_ 包含来自[前一个教程][2]的 Javascript 服务器代码。
  * _TourMyTown_ contains the starter Xcode project with the UI pre-built, but no networking code added yet. _TourMyTown_ 包含Xcode启动项目，已预设好 UI，但还没有网络相关的代码。

Open _TourMyTown\TourMyTown.xcodeproj_ and build and run. You should see something like this:

打开 _TourMyTown\TourMyTown.xcodeproj_ 然后编译运行。你会看到如下界面：

译者注：因为要使用用户的地理位置，最好在真机上运行。

![starter project][4]

Right now not much happens, but here’s a sneak peek of what the app will look like when you finish this tutorial:

目前为止还没有什么事情会发生，但在本教程完成时，你会看到的应用大概如下所示：

![TourMyTown screenshot][5]  
TourMyTown 截图

Users add new location markers to the app along with descriptions, categories and pictures. The _Add_ button places a marker at the center of the map, and the user can drag the marker to the desired location. Alternatively, a tap and hold places a marker at the selected location.

用户添加新的位置标记（location markers）到应用里，位置标记包括描述、分类以及图片。按下 _Add_ 按钮将会在地图的中央放置一个标记，用户还可以拖动此标记到所需的位置。另外，按住屏幕并稍微保持一会儿将会在所选择的地方放置一个标记。

The view delegate uses Core Location’s geo coder functionality to look up the address and place name of the location, if it’s available. Tapping the _Info_ button on the annotation view presents the detail editing screen.

视图 delegate 使用 Core Location 的地理编码功能去查找位置的地址和名字，如果它存在的话。在 annotation view 上点击 _Info_ 按钮将显示细节编辑屏幕。

![Edit point of interest data][6]  
Edit point of interest data

The app saves all data to the backend so that it can be recalled in future sessions.

这个应用将所有数据保存到后端，这样它在将来就可以重复使用它们了。

![The map with an annotation.][7]  
The map with an annotation.

You’ve got a lot of work to do to transition the app to this state, so let’s get coding!

要将应用变成我们所希望的状态，还有不少事情要做，所以让我们开始编码吧！

## 设置你的 Node.js 实例

If you didn’t complete the first part of this tutorial or don’t want to use your existing project, you can use the files contained in the _server_ directory as a starting point.

如果你没有完成本系列教程的第一部分，或这不想使用你自己根据教程走时编写的项目，那你可以使用包含在 _server_ 目录的文件作为起始点。

The following instructions take you through setting up your Node.js instance; if you already have your working instance from Part 1 of this tutorial then feel free to skip straight to the next section.

下列说明会带领你设置 Node.js 实例；如果你已经有了来自本教程第一部分的可工作实例，那么可以跳过这些说明，直接进入下一节。

Open Terminal and navigate to the MongoDB install directory — likely _/usr/local/opt/mongodb/_ but this may be slightly different on your system.

打开终端并导航至 MongoDB 安装目录——例如 _/usr/local/opt/mongodb/_ ，在你的系统上可能稍有差异。

Execute the following command in Terminal to start the MongoDB daemon:

执行下列命令以启动 MongoDB 守护进程：

```Shell
mongod
```

译者注：如我在第一部分的翻译中所提及的，可能并不需要进入 MongoDB 的安装目录，而且直接运行 mongod 可能会出错， `ERROR: dbpath (/data/db) does not exist.`，试试先创建一个自定义路径，再用 `mongd --dbpath '~/somepath'` 来启动服务器。

Now navigate to the _server_ directory you extracted above. Execute the following command:

现在导航至你解压出的 _server_ 目录，执行下列命令：

```Shell
npm install
```

This reads the _package.json_ file and installs the dependencies for your new server code.

它会读取 _package.json_ 文件并安装服务器的相关依赖。

Finally, launch your Node.js server with the following command:

最好，用下面的命令启动你的 Node.js 服务器：

```Shell
node .
```

>_Note:_ The starter project is configured to connect to `localhost`, port 3000. This is fine when you’re running the app locally on your simulator, but if you want to deploy the app to a physical device you’ll have to change `localhost` to `.local` if your Mac and iOS device are on the same network. If they’re not on the same network, then you’ll need to set it to the IP address of your machine. You’ll find these values near the top of _Locations.m_.

启动项目已配置为连接到 `localhost` ，端口 3000 。这对于在模拟器中本地运用应用来说很好，但如果你想在物理设备上部署这个应用，你需要将 `localhost` 改为 `.local` 如果你的 Mac 和 iOS 设备处于同一个网络。而如果它们没有处于同一个网络，你需要将其设置为你的机器的 IP 地址。你会在 _Locations.m_ 的顶部附近找到这些值。

## 应用的数据模型 

The _Location_ class of your project represents a single point of interest and its associated data. It does the following things:

项目中的 _Location_ 类表示单个有趣的位置及其关联数据，它会做下列事情：

  * Holds the location’s data, including its coordinates, description, and categories. 保持未知的数据，包括它的坐标、描述和类别。
  * Knows how to serialize and deserialize the object to a JSON-compatible `NSDictionary`. 知道如何 将对象序列化为
  * Conforms to the `MKAnnotation` protocol so it can be placed on an instance of `MKMapView` as a pin.
  * Has zero or more categories as defined in _Categories.m_.

The _Locations_ class represents your application’s collection of Location objects and the mechanisms that load the objects from the server. This class is responsible for:

  * Serving as the app’s data model by providing a filterable list of locations via `filteredObjects`.
  * Communicating with the server by loading and saving items via `import`, `persist` and `query`.

The _Categories_ class contains the list of categories that a Location can belong to and provides the ability to filter the list of locations by category. _Categories_ also does the following:

  * Houses `allCategories` which provides the master list of categories. You can also add additional categories to its array.
  * Provides a list of all categories in the active set of locations.
  * Filters the locations by categories.

## Loading Locations from the Server

Replace the stubbed-out implementation of `import` in _Locations.m_ with the following code:


    - (void)import
    {
        [NSURL][8]* url = [[NSURL][8] URLWithString:[kBaseURL stringByAppendingPathComponent:kLocations]]; //1
    &nbsp;
        [NSMutableURLRequest][9]* request = [[NSMutableURLRequest][9] requestWithURL:url];
        request.HTTPMethod = @"GET"; //2
        [request addValue:@"application/json" forHTTPHeaderField:@"Accept"]; //3
    &nbsp;
        NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration]; //4
        NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
    &nbsp;
        NSURLSessionDataTask* dataTask = [session dataTaskWithRequest:request completionHandler:^([NSData][10] *data, [NSURLResponse][11] *response, [NSError][12] *error) { //5
            if (error == nil) {
                [NSArray][13]* responseArray = [NSJSONSerialization JSONObjectWithData:data options:0 error:NULL]; //6
                [self parseAndAddLocations:responseArray toArray:self.objects]; //7
            }
        }];
    &nbsp;
        [dataTask resume]; //8
    }

Here’s what `import` does:

  1. The most important bits of information are the URL and request headers. The URL is simply the result of concatenating the base URL with the “locations” collections.
  2. You’re using `GET` since you’re reading data from the server. GET is the default method so it’s not necessary to specify it here, but it’s nice to include it for completeness and clarity.
  3. The server code uses the contents of the `Accept` header as a hint to which type of response to send. By specifying that your request will accept JSON as a response, the returned bytes will be JSON instead of the default format of HTML.
  4. Here you create an instance of _NSURLSession_ with a default configuration.
  5. A _data task_ is your basic `NSURLSession` task for transferring data from a web service. There are also specialized upload and download tasks that have specialized behavior for long-running transfers and background operation. A data task runs asynchronously on a background thread, so you use a callback block to be notified when the operation completes or fails.
  6. The completion handler checks for any errors; if it finds none it tries to deserialize the data using a `NSJSONSerialization` class method.
  7. Assuming the return value is an array of locations, `parseAndAddLocations:` parses the objects and notifies the view controller with the updated data.
  8. Oddly enough, data tasks are started with the `resume` message. When you create an instance of NSURLSessionTask it starts in the “paused” state, so to start it you simply call `resume`.

Still working in the same file, replace the stubbed-out implementation of `parseAndAddLocations:` with the following code:


    - (void)parseAndAddLocations:([NSArray][13]*)locations toArray:([NSMutableArray][14]*)destinationArray //1
    {
        for ([NSDictionary][15]* item in locations) {
            Location* location = [[Location alloc] initWithDictionary:item]; //2
            [destinationArray addObject:location];
        }
    &nbsp;
        if (self.delegate) {
            [self.delegate modelUpdated]; //3
        }
    }

Taking each numbered comment in turn:

  1. You iterate through the array of JSON dictionaries and create a new Location object for each item.
  2. Here you use a custom initializer to turn the deserialized JSON dictionary into an instance of _Location_.
  3. The model signals the UI that there are new objects available.

Working together, these two methods let your app load the data from the server on startup. `import` relies on `NSURLSession` to handle the heavy lifting of networking. For more information on the inner workings of `NSURLSession`, check out the [NSURLSession][16] on this site.

Notice the `Location` class already has the following initializer which simply takes the various values in the dictionary and sets the corresponding object properties appropriately:


    - (instancetype) initWithDictionary:([NSDictionary][15]*)dictionary
    {
        self = [super init];
        if (self) {
            self.name = dictionary[@"name"];
            self.location = dictionary[@"location"];
            self.placeName = dictionary[@"placename"];
            self.imageId = dictionary[@"imageId"];
            self.details = dictionary[@"details"];
            _categories = [[NSMutableArray][14] arrayWithArray:dictionary[@"categories"]];
        }
        return self;
    }

## Saving Locations to the Server

Unfortunately, loading locations from an empty database isn’t super interesting. Your next task is to implement the ability to save Locations to the database.

Replace the stubbed-out implementation of `persist:` in _Locations.m_ with the following code:


    - (void) persist:(Location*)location
    {
        if (!location || location.name == nil || location.name.length == 0) {
            return; //input safety check
        }
    &nbsp;
    &nbsp;
        [NSString][17]* locations = [kBaseURL stringByAppendingPathComponent:kLocations];
    &nbsp;
        BOOL isExistingLocation = location._id != nil;
        [NSURL][8]* url = isExistingLocation ? [[NSURL][8] URLWithString:[locations stringByAppendingPathComponent:location._id]] :
        [[NSURL][8] URLWithString:locations]; //1
    &nbsp;
        [NSMutableURLRequest][9]* request = [[NSMutableURLRequest][9] requestWithURL:url];
        request.HTTPMethod = isExistingLocation ? @"PUT" : @"POST"; //2
    &nbsp;
        [NSData][10]* data = [NSJSONSerialization dataWithJSONObject:[location toDictionary] options:0 error:NULL]; //3
        request.HTTPBody = data;
    &nbsp;
        [request addValue:@"application/json" forHTTPHeaderField:@"Content-Type"]; //4
    &nbsp;
        NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration];
        NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
    &nbsp;
        NSURLSessionDataTask* dataTask = [session dataTaskWithRequest:request completionHandler:^([NSData][10] *data, [NSURLResponse][11] *response, [NSError][12] *error) { //5
            if (!error) {
                [NSArray][13]* responseArray = @[[NSJSONSerialization JSONObjectWithData:data options:0 error:NULL]];
                [self parseAndAddLocations:responseArray toArray:self.objects];
            }
        }];
        [dataTask resume];
    }

`persist:` parallels `import` and also uses a `NSURLSession` request to the `locations` endpoint. However, there are just a few differences:

  1. There are two endpoints for saving an object: `/locations` when you’re adding a new location, and `/locations/_id` when updating an existing location that already has an `id`.
  2. The request uses either `PUT` for existing objects or `POST` for new objects. The server code calls the appropriate handler for the route rather than using the default `GET` handler.
  3. Because you’re updating an entity, you provide an `HTTPBody` in your request which is an instance of `NSData` object created by the `NSJSONSerialization` class.
  4. Instead of an `Accept` header, you’re providing a `Content-Type`. This tells the `bodyParser` on the server how to handle the bytes in the body.
  5. The completion handler once again takes the modified entity returned from the server, parses it and adds it to the local collection of Location objects.

Notice just like `initWithDictionary:`, _Location.m_ already has a helper module to handle the conversion of Location object into a JSON-compatible dictionary as shown below:


    #define safeSet(d,k,v) if (v) d[k] = v;
    - ([NSDictionary][15]*) toDictionary
    {
        [NSMutableDictionary][18]* jsonable = [[NSMutableDictionary][18] dictionary];
        safeSet(jsonable, @"name", self.name);
        safeSet(jsonable, @"placename", self.placeName);
        safeSet(jsonable, @"location", self.location);
        safeSet(jsonable, @"details", self.details);
        safeSet(jsonable, @"imageId", self.imageId);
        safeSet(jsonable, @"categories", self.categories);
        return jsonable;
    }

`toDictionary` contains a magical macro: `safeSet()`. Here you check that a value isn’t `nil` before you assign it to a NSDictionary; this avoids raising an `NSInvalidArgumentException`. You need this check as your app doesn’t force your object’s properties to be populated.

“Why not use an `NSCoder`?” you might ask. The `NSCoding` protocol with `NSKeyedArchiver` does many of the same things as `toDictionary` and `initWithDictionary`; namely, provide a key-value conversion for an object.

However, `NSKeyedArchiver` is set up to work with `plists` which is a different format with slightly different data types. The way you’re doing it above is a little simpler than repurposing the `NSCoding` mechanism.

## Saving Images to the Server

The starter project already has a mechanism to add photos to a location; this is a nice visual way to explore the data in the app. The pictures are displayed as thumbnails on the map annotation and in the details screen. The Location object already has a stub `imageId` which provides a link to to a stored file on the server.

Adding an image requires two things: the client-side call to save and load images and the server-side code to store the images.

Return to Terminal, ensure you’re in the _server_ directory, and execute the following command to create a new file to house your file handler code:

Add the following code to _fileDriver.js_:


    var ObjectID = require('mongodb').ObjectID,
        fs = require('fs'); //1
    &nbsp;
    FileDriver = function(db) { //2
      this.db = db;
    };

This sets up your _FileDriver_ module as follows:

  1. This module uses the filesystem module _fs_ to read and write to disk.
  2. The constructor accepts a reference to the MongoDB database driver to use in the methods that follows.

Add the following code to _fileDriver.js_, just below the code you added above:


    FileDriver.prototype.getCollection = function(callback) {
      this.db.collection('files', function(error, file_collection) { //1
        if( error ) callback(error);
        else callback(null, file_collection);
      });
    };

`getCollection()` looks through the `files` collection; in addition to the content of the file itself, each file has an entry in the `files` collection which stores the file’s metadata including its location on disk.

Add the following code below the block you just added above:


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

Here’s what’s going on in the code above:

  1. `get` fetches the files collection from the database.
  2. Since the input to this function is a string representing the object’s `_id`, you must convert it to a BSON ObjectID object.
  3. `findOne()` finds a matching entity if one exists.

Add the following code directly after the code you added above:


    FileDriver.prototype.handleGet = function(req, res) { //1
        var fileId = req.params.id;
        if (fileId) {
            this.get(fileId, function(error, thisFile) { //2
                if (error) { res.send(400, error); }
                else {
                        if (thisFile) {
                             var filename = fileId %2B thisFile.ext; //3
                             var filePath = './uploads/'%2B filename; //4
        	                 res.sendfile(filePath); //5
        	            } else res.send(404, 'file not found');
                }
            });
        } else {
    	    res.send(404, 'file not found');
        }
    };

`handleGet` is a request handler used by the Express router. It simplifies the server code by abstracting the file handling away from _index.js_. It performs the following actions:

  1. Fetches the file entity from the database via the supplied id.
  2. Adds the extension stored in the database entry to the id to create the filename.
  3. Stores the file in the local `uploads` directory.
  4. Calls `sendfile()` on the response object; this method knows how to transfer the file and set the appropriate response headers.

Once again, add the following code directly underneath what you just added above:


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

`save()` above is the same as the one in _collectionDriver_; it inserts a new object into the files collection.

Add the following code, again below what you just added:


    FileDriver.prototype.getNewFileId = function(newobj, callback) { //2
    	this.save(newobj, function(err,obj) {
    		if (err) { callback(err); }
    		else { callback(null,obj._id); } //3
    	});
    };

  1. `getNewFileId()` is a wrapper for `save` for the purpose of creating a new file entity and returning `id` alone.
  2. This returns only `_id` from the newly created object.

Add the following code after what you just added above:


    FileDriver.prototype.handleUploadRequest = function(req, res) { //1
        var ctype = req.get("content-type"); //2
        var ext = ctype.substr(ctype.indexOf('/')%2B1); //3
        if (ext) {ext = '.' %2B ext; } else {ext = '';}
        this.getNewFileId({'content-type':ctype, 'ext':ext}, function(err,id) { //4
            if (err) { res.send(400, err); }
            else {
                 var filename = id %2B ext; //5
                 filePath = __dirname %2B '/uploads/' %2B filename; //6
    &nbsp;
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
    &nbsp;
    exports.FileDriver = FileDriver;

There’s a lot going on in this method, so take a moment and review the above comments one by one:

  1. `handleUploadRequest` creates a new object in the file collection using the `Content-Type` to determine the file extension and returns the new object’s `_id`.
  2. This looks up the value of the `Content-Type` header which is set by the mobile app.
  3. This tries to guess the file extension based upon the content type. For instance, an `image/png` should have a `png` extension.
  4. This saves `Content-Type` and `extension` to the file collection entity.
  5. Create a filename by appending the appropriate extension to the new `id`.
  6. The designated path to the file is in the server’s root directory, under the _uploads_ sub-folder. `__dirname` is the Node.js value of the executing script’s directory.
  7. `fs` includes `writeStream` which — as you can probably guess — is an output stream.
  8. The request object is also a `readStream` so you can dump it into a write stream using the `pipe()` function. These [stream][19] objects are good examples of the Node.js event-driven paradigm.
  9. `on()` associates stream events with a callback. In this case, the `readStream’s` `end` event occurs when the pipe operation is complete, and here the response is returned to the Express code with a 201 status and the new file `_id`.
  10. If the write stream raises an `error` event then there is an error writing the file. The server response returns a 500 Internal Server Error response along with the appropriate filesystem error.

Since the above code expects there to be an _uploads_ subfolder, execute the command below in Terminal to create it:

Add the following code to the end of the `require` block at the top of _index.js_:


        FileDriver = require('./fileDriver').FileDriver;

Next, add the following code to _index.js_ just below the line `var mongoPort = 27017;`:

Add the following line to _index.js_ just after the line ` var db = mongoClient.db("MyDatabase");`:

In the mongoClient setup callback create an instance of FileDriver after the CollectionDriver creation:


    fileDriver = new FileDriver(db);

This creates an instance of your new `FileDriver`.

Add the following code just before the generic `/:collection` routing in _index.js_:


    app.use(express.static(path.join(__dirname, 'public')));
    app.get('/', function (req, res) {
      res.send('Hello World');
    });
    &nbsp;
    app.post('/files', function(req,res) {fileDriver.handleUploadRequest(req,res);});
    app.get('/files/:id', function(req, res) {fileDriver.handleGet(req,res);});

Putting this before the generic `/:collection` routing means that _files_ are treated differently than a generic _files_ collection.

Save your work, kill your running Node instance with _Control%2BC_ if necessary and restart it with the following command:

Your server is now set up to handle files, so that means you need to modify your app to post images to the server.

## Saving Images in your App

The Location class has two properties: `image` and `imageId`. `imageId` is the backend property that links the entity in the `locations` collection to the entity in the `files` collection. If this were a relational database, you’d use a foreign key to represent this link. `image` stores the actual `UIImage` object.

Saving and loading files requires an extra request for each object to transfer the file data. The order of operations is important to make sure the file id is property associated with the object. When you save a file, you must send the file first in order to receive the associated id to link it with the location’s data.

Add the following code to the bottom of _Locations.m_:


    - (void) saveNewLocationImageFirst:(Location*)location
    {
        [NSURL][8]* url = [[NSURL][8] URLWithString:[kBaseURL stringByAppendingPathComponent:kFiles]]; //1
        [NSMutableURLRequest][9]* request = [[NSMutableURLRequest][9] requestWithURL:url];
        request.HTTPMethod = @"POST"; //2
        [request addValue:@"image/png" forHTTPHeaderField:@"Content-Type"]; //3
    &nbsp;
        NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration];
        NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
    &nbsp;
        [NSData][10]* bytes = UIImagePNGRepresentation(location.image); //4
        NSURLSessionUploadTask* task = [session uploadTaskWithRequest:request fromData:bytes completionHandler:^([NSData][10] *data, [NSURLResponse][11] *response, [NSError][12] *error) { //5
            if (error == nil &amp;&amp; [([NSHTTPURLResponse][20]*)response statusCode] &lt; 300) {
                [NSDictionary][15]* responseDict = [NSJSONSerialization JSONObjectWithData:data options:0 error:NULL];
                location.imageId = responseDict[@"_id"]; //6
                [self persist:location]; //7
            }
        }];
        [task resume];
    }

This is a fairly busy module, but it’s fairly straightforward when you break it into small chunks:

  1. The URL is the _files_ endpoint.
  2. Using `POST` triggers `handleUploadRequest` of `fileDriver` to save the file.
  3. Setting the content type ensures the file will be saved appropriately on the server. The `Content-Type` header is important for determining the file extension on the server.
  4. `UIImagePNGRepresentation` turns an instance of `UIImage` into PNG file data.
  5. `NSURLSessionUploadTask` lets you send `NSData` to the server in the request itself. For example, upload tasks automatically set the `Content-Length` header based on the data length. Upload tasks also report progress and can run in the background, but neither of those features is used here.
  6. The response contains the new file data entity, so you save `_id` along with the location object for later retrieval.
  7. Once the image is saved and `_id` recorded, then the main Location entity can be saved to the server.

Add the following code to `persist:` in _Location.m_ just after the `if (!location || location.name == nil || location.name.length == 0)` block’s closing brace:


    - (void) persist:(Location*)location
    &nbsp;
        //if there is an image, save it first
        if (location.image != nil &amp;&amp; location.imageId == nil) { //1
            [self saveNewLocationImageFirst:location]; //2
            return;
        }

This checks for the presence of a new image, and saves the image first. Taking each numbered comment in turn, you’ll find the following:

  1. If there is an image but no image id, then the image hasn’t been saved yet.
  2. Call the new method to save the image, and exits.

Once the save is complete, `persist:`will be called again, but at that point `imageId` will be non-nil, and the code will proceed into the existing procedure for saving the Location entity.

Next replace the stub method `loadImage:` in _Location.m_ with the following code:


    - (void)loadImage:(Location*)location
    {
        [NSURL][8]* url = [[NSURL][8] URLWithString:[[kBaseURL stringByAppendingPathComponent:kFiles] stringByAppendingPathComponent:location.imageId]]; //1
    &nbsp;
        NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration];
        NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
    &nbsp;
        NSURLSessionDownloadTask* task = [session downloadTaskWithURL:url completionHandler:^([NSURL][8] *fileLocation, [NSURLResponse][11] *response, [NSError][12] *error) { //2
            if (!error) {
                [NSData][10]* imageData = [[NSData][10] dataWithContentsOfURL:fileLocation]; //3
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
    &nbsp;
        [task resume]; //4
    }

Here’s what’s going on in the code above:

  1. Just like when loading a specific location, the image’s id is appended to the path along with the name of the endpoint: `files`.
  2. The download task is the third kind of `NSURLSession`; it downloads a file to a temporary location and returns a URL to that location, rather than the raw `NSData` object, as the raw object can be rather large.
  3. The temporary location is only guaranteed to be available during the completion block’s execution, so you must either load the file into memory, or move it somewhere else.
  4. Like all `NSURLSession` tasks, you start the task with `resume`.

Next, replace the current `parseAndAddLocations:toArray:` with the following code:


    - (void)parseAndAddLocations:([NSArray][13]*)locations toArray:([NSMutableArray][14]*)destinationArray
    {
        for ([NSDictionary][15]* item in locations) {
            Location* location = [[Location alloc] initWithDictionary:item];
            [destinationArray addObject:location];
    &nbsp;
            if (location.imageId) { //1
                [self loadImage:location];
            }
        }
    &nbsp;
        if (self.delegate) {
            [self.delegate modelUpdated];
        }
    }

This updated version of `parseAndAddlocations` checks for an imageId; if it finds one, it calls `loadImage:`.

## A Quick Recap of File Handling

To summarize: file transfers in an iOS app work conceptually the same way as regular data transfers. The big difference is that you’re using `NSURLSessionUploadTask` and `NSURLSessionDownloadTask` objects and semantics that are slightly different from `NSURLSessionDataTask`.

On the server side, file wrangling is a fairly different beast. It requires a special handler object that communicates with the filesystem instead of a Mongo database, but still needs to store some metadata in the database to make retrieval easier.

Special routes are then set up to map the incoming HTTP verb and endpoint to the file driver. You _could_ accomplish this with generic data endpoints, but the code would get quite complicated when determining where to persist the data.

## Testing it Out

Build and run your app and add a new location by tapping the button in the upper right.

As part of creating your new location, add an image. Note that you can add images to the simulator by long-pressing on pictures in Safari.

Once you’ve saved your new location, restart the app — and lo and behold, the app reloads your data without a hitch, as shown in the screenshot below:

![Adding an image to a Location in Tour My Town.][21]

Adding an image to a Location in Tour My Town.

![Location annotation with an image.][22]

Location annotation with an image.

## Querying for Locations

Your ultra-popular Tour My Town app will collect a ton of data incredibly quickly after it’s released. To prevent long wait times while downloading all of the data for the app, you can limit the amount of data retrieved by using location-based filtering. This way you only retrieve the data that’s going to be shown on the screen.

MongoDB has a powerful feature for finding entities that match a given criteria. These [criteria][23] can be basic comparisons, type checking, expression evaluation (including regular expression and arbitrary javascript), and geospatial querying.

The geospatial querying of MongoDBis a natural fit with a map-based application. You can use the extents of the map view to obtain only the subset of data that will be shown on the screen.

Your next task is to modify _collectionDriver.js_ to supply filter criteria with a GET request.

Add the following method above the final `exports` line in _collectionDriver.js_:


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

Here’s how the above code functions:

  1. `query` is similar to the existing `findAll`, except that it has a `query` parameter for specifying the filter criteria.
  2. You fetch the collection access object just like all the other methods.
  3. `CollectionDriver`‘s `findAll` method used `find()` with no arguments, but here the `query` object is passed in as an argument. This will be passed along to MongoDB for evaluation so that only the matching documents will be returned in the result.

_Note:_ This passes in the query object directly to MongoDB. In an open API case, this can be dangerous since MongoDB permits arbitrary JavaScript using the `$where` query operator. This runs the risk of crashes, unexpected results, or security concerns; but in this tutorial project which uses a limited set of operations, it is a minor concern.

Go back to _index.js_ and replace the current `app.get('/:collection'...` block with the following:


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
    &nbsp;
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

  1. HTTP queries can be added to the end of a URL in the form `http://domain/endpoint?key1=value1&amp;key2=value2...`. `req.query` gets the whole “query” part of the incoming URL. For this application the key is “query” (hence `req.query.query`)
  2. The query value should be a string representing a MongoDB condition object. `JSON.parse()` turns the JSON-string into a javascript object that can be passed directly to MongoDB.
  3. If a query was supplied to the endpoint, call `collectionDriver.query()`; `returnCollectionResults` is a common helper function that formats the output of the request.
  4. If no query was specified, then `collectionDriver.findAll` returns all the items in the collection.
  5. Since `returnCollectionResults()` is evaluated at the time it is called, this function returns a callback function for the collection driver.
  6. If the request specified HTML for the response, then render the data table in HTML; otherwise return it as a JSON document in the body.

Save your work, kill your Node.js instance and restart it with the following command:

Now that the server is set up for queries, you can add the geo-querying functions to the app.

Replace the stubbed-out implementation of `queryRegion` of _Locations.m_ with the following code:


    - (void) queryRegion:(MKCoordinateRegion)region
    {
        //note assumes the NE hemisphere. This logic should really check first.
        //also note that searches across hemisphere lines are not interpreted properly by Mongo
        CLLocationDegrees x0 = region.center.longitude - region.span.longitudeDelta; //1
        CLLocationDegrees x1 = region.center.longitude %2B region.span.longitudeDelta;
        CLLocationDegrees y0 = region.center.latitude - region.span.latitudeDelta;
        CLLocationDegrees y1 = region.center.latitude %2B region.span.latitudeDelta;
    &nbsp;
        [NSString][17]* boxQuery = [[NSString][17] stringWithFormat:@"{\"$geoWithin\":{\"$box\":[[%f,%f],[%f,%f]]}}",x0,y0,x1,y1]; //2
        [NSString][17]* locationInBox = [[NSString][17] stringWithFormat:@"{\"location\":%@}", boxQuery]; //3
        [NSString][17]* escBox = ([NSString][17] *)CFBridgingRelease(CFURLCreateStringByAddingPercentEscapes(NULL,
                                                                                      (CFStringRef) locationInBox,
                                                                                      NULL,
                                                                                      (CFStringRef) @"!*();':@&amp;=%2B$,/?%#[]{}",
                                                                                      kCFStringEncodingUTF8)); //4
        [NSString][17]* query = [[NSString][17] stringWithFormat:@"?query=%@", escBox]; //5
        [self runQuery:query]; //7
    }

This is a fairly straightforward block of code; `queryRegion:` turns a Map Kit region generated from a MKMapView into a bounded-box query. Here’s how it does it:

  1. These four lines calculate the map-coordinates of the two diagonal corners of the bounding box.
  2. This defines a JSON structure for the query using MongoDB’s specific query language.
A query with a `$geoWithin` key specifies the search criteria as everything located within the structure defined by the provided value. `$box` specifies the rectangle defined by the provided coordinates and supplied as an array of two longitude-latitude pairs at opposite corners.
  3. `boxQuery` just defines the criteria _value_; you also have to provide the search key field along `boxQuery` as a JSON object to MongoDB.
  4. You then escape the entire query object as it will be posted as part of a URL; you need to ensure that that internal quotes, brackets, commas, and other non-alphanumeric bits won’t be interpreted as part of the HTTP query parameter. `CFURLCreateStringByAddingPercentEscapes` is a CoreFoundation method for creating URL-encoded strings.
  5. The final piece of the string building sets the entire escaped MongoDB query as the query value in the URL.
  6. You then request matching values from the server with your new query.

_Note:_ In MongoDB coordinate pairs are specified as [longitude, latitude], which is the opposite of the usual lat/long pairing you’d see in things like the Google Maps API.

Replace the stubbed-out implementation of `runQuery:` in _Locations.m_ with the following code:


    - (void) runQuery:([NSString][17] *)queryString
    {
        [NSString][17]* urlStr = [[kBaseURL stringByAppendingPathComponent:kLocations] stringByAppendingString:queryString]; //1
        [NSURL][8]* url = [[NSURL][8] URLWithString:urlStr];
    &nbsp;
        [NSMutableURLRequest][9]* request = [[NSMutableURLRequest][9] requestWithURL:url];
        request.HTTPMethod = @"GET";
        [request addValue:@"application/json" forHTTPHeaderField:@"Accept"];
    &nbsp;
        NSURLSessionConfiguration* config = [NSURLSessionConfiguration defaultSessionConfiguration];
        NSURLSession* session = [NSURLSession sessionWithConfiguration:config];
    &nbsp;
        NSURLSessionDataTask* dataTask = [session dataTaskWithRequest:request completionHandler:^([NSData][10] *data, [NSURLResponse][11] *response, [NSError][12] *error) {
            if (error == nil) {
                [self.objects removeAllObjects]; //2
                [NSArray][13]* responseArray = [NSJSONSerialization JSONObjectWithData:data options:0 error:NULL];
                NSLog(@"received %d items", responseArray.count);
                [self parseAndAddLocations:responseArray toArray:self.objects];
            }
        }];
         [dataTask resume];
    }

`runQuery:` is very similar to `import` but has two important differences:

  1. You add the query string generated in `queryRegion:` to the end of the locations endpoint URL.
  2. You also discard the previous set of locations and replace them with the filtered set returned from the server. This keeps the active results at a manageable level.

Build and run your app; create a few new locations of interest that are spread out on the map. Zoom in a little, then pan and zoom the map and watch NSLog display the changing count of the items both inside and outside the map range, as shown below:

![Debugger output while panning and zooming map.][24]

Debugger output while panning and zooming map.

## Using Queries to Filter by Category

The last bit to add _categories_ to your Locations that users can filter on. This filtering can re-use the server work done in the previous section through the use of MongoDB’s array conditional operators.

Replace the stubbed-out implementation of `query` in _Categories.m_ with the following code:


    %2B ([NSString][17]*) query
    {
        [NSArray][13]* a = [self filteredCategories:YES]; //1
        [NSString][17]* query = @"";
        if (a.count &gt; 0) {
    &nbsp;
            query = [[NSString][17] stringWithFormat:@"{\"categories\":{\"$in\":[%@]}}", [a componentsJoinedByString:@","]]; //2
            query = ([NSString][17] *)CFBridgingRelease(CFURLCreateStringByAddingPercentEscapes(NULL,
                                                                                               (CFStringRef) query,
                                                                                               NULL,
                                                                                               (CFStringRef) @"!*();':@&amp;=%2B$,/?%#[]{}",
                                                                                               kCFStringEncodingUTF8));
    &nbsp;
            query = [@"?query=" stringByAppendingString:query];
        }
        return query;
    }

This creates a query string similar to the one used by the geolocation query that has the following differences:

  1. This is the list of selected categories.
  2. The `$in` operator accepts a MongoDB document if the specified property `categories` has a value matching any of the items in the corresponding array.

Build and run your app; add a few Locations and assign them one or more categories. Tap the folder icon and select a category to filter on. The map will reload just the Location annotations matching the selected categories as shown below:

![A map with many locations][25]

A map with many locations

![Select just the "Park" category][26]

Select just the “Park” category

![Map after filtering][27]

Map after filtering

## Where to Go From Here?

You can [download the completed sample project][28] here.

In this tutorial you covered the basics of MongoDB storage — but there’s a ton of functionality beyond what you covered here.

MongoDB offers a multitude of options for selecting data out of the database; as well there are a host of server-side features to manage scaling and security. As well, your Node.js installation could definitely be improved by adding user authentication and more privacy around the data.

As for your iOS app, you could add a pile of interesting features, including the following:

  * Routing users to points of interest
  * Adding additional media to locations
  * Improved text editing

Additionally, every decent networked app should cache data locally so it remains functional when data connections are spotty.

Hopefully you’ve enjoyed this small taste of Node.js, Express and MongoDB — if you have any questions or comments please come join the discussion below!

===============

译者注：欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳译者的幸苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

   [1]: http://cdn2.raywenderlich.com/wp-content/uploads/2014/03/MongoDB.png
   [2]: http://www.raywenderlich.com/61078/write-simple-node-jsmongodb-web-service-ios-app
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
  
# 入门 GraphQL & Apollo

本文翻译自 [https://www.raywenderlich.com/158433/getting-started-graphql-apollo-ios](https://www.raywenderlich.com/158433/getting-started-graphql-apollo-ios)

原作者：[Nikolas Burk](https://www.raywenderlich.com/u/nikolasburk)

译者：[@nixzhu](https://twitter.com/nixzhu)

---

当你配合一个REST API工作时，因为某个端点没有给你所需要的数据用于app的某些视图，你是否有感到过沮丧呢？例如需要多个请求才能从服务器获取到正确的信息，或者你需要报bug通知后端开发者才能调整API？现在，你无需再担忧，_GraphQL_和_Apollo_将拯救你。

[GraphQL][2]是一种新的API设计范式，由Facebook在2015年开源，不过早在2012，他们就开始用于给自家的app提供支持了。它消除了当今[REST][3] API的许多低效之处。对比REST，GraphQL只需要暴露一个端点，API的消费者就能精确地指定他们所需要的数据。

在本教程中，你将构建一个iPhone app来帮助用户计划参与哪些iOS会议。你将设置自己的GraphQL服务器并在app中用[Apollo iOS Client][4]与之交互。Apollo是一个网络库，它能让我们与GraphQL API工作起来如沐春风。

此app将包含如下特性：

* 显示iOS会议列表
* 标记自己想参加或不想参加某个会议
* 查看参加某个会议的同伴

为了本教程的缘故，你要先通过[Node Package Manager][5]安装一些工具，请确保你已经安装有npm，且版本在4.5.0（或以上）。

## 开始

下载并打开用于本教程的[starter project][6]。它已包含有必要的UI组件，因此你可以专注于同API打交道，把数据正确地带到app中。

Storyboard看起来如下：

![Application Main Storyboard][7]

此app使用[CocoaPods][8]，所以在下载好第三方包后，你要打开_ConferencePlanner.xcworkspace_。`Apollo`的pod已经被包含在项目中，不过你应该确保安装的是最新的版本。

打开终端，导航到项目所在目录，并执行`pod install`来更新：

![Pod Install Output][9]

## 为何使用GraphQL?

REST API暴露多个端点，每个端点都返回特定的信息。例如，你可能有如下端点：

* `/conferences`：返回一个所有会议的列表，每个会议都有`id`、`name`、`city`和`year`字段
* `/conferences/__id__/attendees`：返回某个会议所有的参加者（包括`id`和`name`）和会议`id`

想象一下，你所写的app要显示所有会议的列表，加上每个会议最近注册的三个参加者。你有何选择？

![iOS Screen Application Running][10]

_选项1：修改API_

告诉后端开发者修改API，这样每次调用`/conferences`时都返回最近注册的三个参加者：

![Conferences REST API Endpoint Data][11]

_选项2：发送n+1次请求_

发送`n+1`次请求（此处`n`是会议的数量）来获取所需的信息，并且接受你可能会耗尽用户的网络流量的后果，因为虽然每个会议你只需要最近的三个与会者却也必须下载全部。

![Conference Attendee REST Endpoint Data][12]

这两个选项都不是很有吸引力，在更大的开发项目里也不具有很好的伸缩性！

使用GraphQL，你就可以简单地指定你在单个请求里所需要的数据，使用GraphQL语法以申明式的方式描述即可：

``` graphql
    {
      allConferences {
        name
        city
        year
        attendees(last: 3) {
          name
        }
      }
    }
```

这个查询的应答将包含一个会议的列表，其中每个会议都有`name`、`city`和`year`，而且带有三个最新的与会者。

### 在iOS中通过Apollo使用GraphQL

GraphQL在移动开发者社区中还不是很流行，不过很可能会随着更多相关工具的改进而改变。这种势头表现在[Apollo iOS客户端][4]上，它实现了许多方便特性，让你与API合作时更轻松。

目前，它的主要特性如下：

1. 基于数据要求的静态类型生成（译者注：即自动生成模型代码）
2. 缓存与监听查询

通过本教程，你对这两个特性都会有所体会。

### 与GraphQL交互

与API交互的主要目标通常是：

* 获取数据
* 创建、更新和删除数据

在GraphQL中，_获取_数据通过_query_完成，而写入数据库通过_mutation_来达成。

一个mutation看起来很像一个query，同样允许你申明需要服务器返回的信息，这就确保了在一个单程往返中你就可以获取更新后的信息。

考虑如下两个简单例子：

``` graphql
    query AllConferences {
      allConferences {
        id
        name
      }
    }
```

此query获取所有的会议并返回一个JSON数组，其中每个对象包带有会议的`id`和`name`。

``` graphql
    mutation CreateConference {
      createConference(name: "WWDC", city: "San Jose", year: "2017") {
        id 
      }
    }
```

此mutation创建了一个新会议并类似地返回其`id`。

如果你对这种语法还没什么感觉，别担心，稍后会探讨更多细节。

## 准备GraphQL服务器

为了本教程之目的，你将基于一个数据模型使用一个叫做[Graphcool][13]的服务来生成一个GraphQL服务器。

说到数据模型，下面是其样子，它的语法叫做[GraphQL Interface Definition Language][14] (IDL)：
    
``` graphql
    type Conference {
      id: String!
      name: String!
      city: String!
      year: String!
      attendees: [Attendee] @relation(name: "Attendees")
    }
    
    type Attendee {
      id: String!
      name: String!
      conferences: [Conference] @relation(name: "Attendees")
    }
```

GraphQL有它自己的类型系统，你可在其上构建自己的类型。如本例中的`Conference`和`Attendee`类型。每个类型都有一些属性，GraphQL术语叫做_字段（field）_。注意跟在每个类型后的`!`，它表示此字段是必须的。

话不多说，开始创建你的GraphQL服务器吧！
    
通过npm安装[Graphcool CLI][15]。打开终端，键入：
    
``` bash
    npm install -g graphcool
```

使用`graphcool`创建服务器：
    
``` bash
    graphcool init --schema http://graphqlbin.com/conferences.graphql --name ConferencePlanner
```

这个命令会创建一个叫做`ConferencePlanner`的Graphcool项目。在此之前，它还会打开一个浏览器窗口，你需要创建一个Graphcool账户（译者注：推荐使用GitHub登录）。一旦建立好，你就能访问GraphQL的全部功能了。

![GraphCool command output showing Simple API and Relay API][16]

拷贝`Simple API`端点留作之后使用。

搞定！你现在能访问完整的GraphQL API并通过[Graphcool console][17]管理。

### 输入初始会议数据

在继续之前，先准备一些初始数据到数据库中。

将前一步中`Simple API`返回的端点拷贝到浏览器地址栏中。这将打开一个[GraphQL Playground][18]，你可在其中用交互式的方式探索API。

添加如下GraphQL代码到Playground左边的输入框，以创建一些初始数据：
    
``` graphql
    mutation createUIKonfMutation {
      createConference(name: "UIKonf", city: "Berlin", year: "2017") {
        id
      }
    }
    
    mutation createWWDCMutation {
      createConference(name: "WWDC", city: "San Jose", year: "2017") {
        id
      }
    }
```

这个代码片段包含两个GraphQL mutation。点击Play按钮并通过下拉菜单将每个mutaiton都选择一次（译者注：每次执行一个mutation）：

![GraphQL playground][19]

这就创建了两个新会议。为了验证已经创建成功，你可以通过[Graphcool console][17]中的[data browser][20]查看当前数据库的状态，或者发送之前见过的`allConferences` query。

![GraphQL Playground AllConferences query result][21]

## 配置Xcode并设置Apollo iOS客户端

之前提到，Apollo iOS客户端能做_静态类型生成_。也就是说，你不再需要写模型来表示来自你应用域名的信息了。作为替代，Apollo iOS客户端使用GraphQL query中的信息来生成对应的Swift类型。

_注意_：这种方式消除了Swift中JSON解析的不便。由于JSON没有类型，唯一真正安全的解析方式是在Swift类型上使用可选值，因为你不能100%确定JSON数据中包含某个特定类型的属性。

要在Xcode中实现静态类型生成，你要先配置几步：

_1. 安装apollo-codegen_

[`apollo-codegen`][22]会搜索Xcode项目中的GraphQL代码，并生成Swift类型。

打开终端，输入：
    
``` bash
    npm install -g apollo-codegen
```

![NPM results from apollo-codegen installation][23]

_2. 添加build phase_

在Xcode中，选择_Project Navigator_中的_ConferencePlanner_。选择名为_ConferencePlanner_的target。选择_Build Phases_栏，再点击左上角的_+_按钮。

从菜单中选择_New Run Script Phase_：

![Xcode altering build phases adding new run script][24]

重命名新建的build phase为_Generate Apollo GraphQL API_。并拖动它到_Compile Sources_上方。

拷贝如下代码片段到输入框（目前写着_Type a script or drag a script file from your workspace to insert its path_）中：
    
``` bash
    APOLLO_FRAMEWORK_PATH="$(eval find $FRAMEWORK_SEARCH_PATHS -name "Apollo.framework" -maxdepth 1)"
    
    if [ -z "$APOLLO_FRAMEWORK_PATH" ]; then
    echo "error: Couldn't find Apollo.framework in FRAMEWORK_SEARCH_PATHS; make sure to add the framework to your project."
    exit 1
    fi
    
    cd "${SRCROOT}/${TARGET_NAME}"
    $APOLLO_FRAMEWORK_PATH/check-and-run-apollo-codegen.sh generate $(find . -name '*.graphql') --schema schema.json --output API.swift
```

检查一下，你的_Build Phases_应该看起来如下：

![Build Script Run Phase Code to Run][25]

_3. 添加schema文件_

在此你又会用到`Simple API`的端点。打开终端，输入如下命令（替换其中的`__SIMPLE_API_ENDPOINT__`为你之前生成的自定义GraphQL端点）：
    
``` bash
    apollo-codegen download-schema __SIMPLE_API_ENDPOINT__ --output schema.json
```

_注意_：如果你丢失了GraphQL端点，你可以在[Graphcool console][17]中点击左下角的_ENDPOINTS_按钮找回。
![GraphCool Endpoint Console][26]

接下来，将此文件放到Xcode工程的根目录。和_AppDelegate.swift_在同一个目录，即_ConferencePlanner-starter/ConferencePlanner_。

![Finder File Listing showing Schema.json][27]

快速小结一下刚刚做的事：

* 首先安装`apollo-codegen`，它用于生成Swift类型
* 接着，在Xcode中添加一个build phase，它会在编译之前让`apollo-codegen`介入
* 之后到你实际的GraphQL guery（稍后添加），`apollo-codegen`要求项目根目录里有一个上一步已下载的schema文件

## 初始化ApolloClient

终于要写点儿实际的代码了！

打开_AppDelegate.swift_，添加如下代码，注意替换其中的`__SIMPLE_API_ENDPOINT__`为你自己的端点。
    
``` swift
    import Apollo
    
    let graphQLEndpoint = "__SIMPLE_API_ENDPOINT__"
    let apollo = ApolloClient(url: URL(string: graphQLEndpoint)!)
```

传递了`Simple API`端点后，`ApolloClient`就知道该和哪个GraphQL服务器通信了。生成的`apollo`对象将是你与API打交道的主要界面。

## 创建与会者并查询会议列表

所有与GraphQL API交互的准备工作都已搞定！首先，确保使用此app的用户能通过取个名字来进行注册。

### 编写第一个Mutation

在Xcode中的_GraphQL_ group中新建一个文件，记得用_Other_下的_Empty_模版，命名为_RegisterViewController.graphql_：

![Xcode New File Picker][28]

接下来，添加如下mutation到此文件中：
    
``` graphql
    # 1
    mutation CreateAttendee($name: String!) {
      # 2
      createAttendee(name: $name) {
        # 3
        id
        name
      }
    }
```

稍微解释一下：

1. 这里用了mutation签名（类似Swift的函数）。这个mutation叫做`CreateAttendee`，接受一个名为`name`且类型为`String`的参数。感叹号表示这个参数必须传递。
2. `createAttendee`引用了GraphQL API所暴露的一个mutaion。Graphcool Simple API给每个类型都默认提供一个`create`-mutation。
3. 此mutation的_payload_，即你希望服务器执行此mutation后返回的数据。

在下一次构建项目时，`apollo-codegen`就会找到这些代码并为这个mutation生成Swift表示。用快捷键_Cmd+B_构建吧。

_注意_：如果你想给GraphQL代码增加语法高亮，请参考[这里][29]的指导。

当`apollo-codegen`初次运行，它会在项目根目录创建一个叫做_API.swift_的新文件。所有后续的生成只会更新此文件。

生成的_API.swift_文件位于项目根目录，但你还需要将其添加到Xcode中，拖动它到_GraphQL_ group里。注意不要选中_Copy items if needed_！

![Xcode Project Navigator showing API.swift in the GraphQL group][30]

观察_API.swift_文件的内容，你会看到一个叫做`CreateAttendeeMutation`的类。它的初始化方法接受一个`name`变量作为参数。它还有一个嵌套的结构体，叫做`Data`，它又嵌套另一个结构体，叫做`CreateAttendee`。这会作为mutation的返回数据，包含attendee的`id`和`name`。

接下来，你要使用此mutation。打开_RegisterViewController.swift_并实现`createAttendee`方法如下：

``` swift
    func createAttendee(name: String) {
      activityIndicator.startAnimating()
      
      // 1  
      let createAttendeeMutation = CreateAttendeeMutation(name: name)
      
      // 2
      apollo.perform(mutation: createAttendeeMutation) { [weak self] result, error in
         self?.activityIndicator.stopAnimating()
          
        if let error = error {
          print(error.localizedDescription)
          return
        }
          
        // 3
        currentUserID = result?.data?.createAttendee?.id
        currentUserName = result?.data?.createAttendee?.name
          
        self?.performSegue(withIdentifier: "ShowConferencesAnimated", sender: nil)
      }
    }
```

上面的代码：

1. 使用用户提供的名字初始化mutation
2. 使用`apollo`实例发送此mutation到API
3. 接受从服务器返回的数据，并存在全局变量上，代表当前的用户

_注意_：本教程所有的API调用都遵循如下模式：首先生成一个query或mutation的实例，然后将其传递给`ApolloClient`，最后在回调中使用结果。

因为允许用户改名，你可添加第二个mutation。打开_RegisterViewController.graphql_并添加如下代码：
    
``` graphql
    mutation UpdateAttendeeName($id: ID!, $newName: String!) {
      updateAttendee(id: $id, name: $newName) {
        id
        name
      }
    }
```

按下_Cmd+B_让`apollo-codegen`为此mutation生成Swift代码。之后，打开_RegisterViewController.swift_并替换`updateAttendee`如下：
    
``` swift
    func updateAttendee(id: String, newName: String) {
      activityIndicator.startAnimating()
      
      let updateAttendeeNameMutation = UpdateAttendeeNameMutation(id: id, newName: newName)
      apollo.perform(mutation: updateAttendeeNameMutation) { [weak self] result, error in
        self?.activityIndicator.stopAnimating()
        
        if let error = error {
          print(error.localizedDescription)
          return
        }
        
        currentUserID = result?.data?.updateAttendee?.id
        currentUserName = result?.data?.updateAttendee?.name
        
        self?.performSegue(withIdentifier: "ShowConferencesAnimated", sender: nil)
      }
    }
```

这些代码和`createAttendee`几乎一样，除了要传递一个`id`来指代你要更新的是哪个用户。

编译并运行，输入一个名字，然后点击_Save_按钮。一个新的attentee就在GraphQL后端被创建好了。

![User Settings page for the application][31]

你可在Playground中验证，发送`allAttendees` query即可：

![GraphCool Playground showing AllAttendees query][32]

### 查询所有会议

下一步要在`ConferencesTableViewController`中显示所有的会议。

在GraphQL group中新建一个文件，命名为_ConferenceTableViewController.graphql_并添加如下代码：
    
``` graphql
    fragment ConferenceDetails on Conference {
      id
      name
      city
      year
      attendees {
        id
      }
    }
    
    query AllConferences {
      allConferences {
        ...ConferenceDetails
      }
    }
```

这里的_fragment_是什么东西？

简单来说，[Fragments][33]是可重用的零件，它能包装GraphQL类型的一些字段。它们与静态类型一起使用非常方便，因为它们增强了GraphQL服务器返回的信息的可重用性，并且每个片段将由其自己的结构体表示。

Fragment能通过`...`加上片段名集成到任何query或mutation中。当`AllConferences`query被发出时，`...ConferenceDetails`就被`ConferenceDetails`片段中包含的所有字段替换。（译者注：就像宏替换一样）

接下来该使用query填充table view了。

按下_Cmd+B_以确保生成新的query和fragment的Swift代码，然后打开_ConferencesTableViewController.swift_并添加如下属性到其顶部：
    
``` swift
    var conferences: [ConferenceDetails] = [] {
      didSet {
        tableView.reloadData()
      }
    }
```

在`viewDidLoad`最后，添加如下代码以发送query并显示结果：
    
``` swift
    let allConferencesQuery = AllConferencesQuery()
    apollo.fetch(query: allConferencesQuery) { [weak self] result, error in
      guard let conferences = result?.data?.allConferences else { return }  
      self?.conferences = conferences.map { $0.fragments.conferenceDetails }
    }
```

和你在第一个mutation所看到的一样，你使用的是同样的模式，不过这次你发送的是一个query。在初始化此query之后，你将其传递给`apollo`实例并在回调中获取会议列表。这个列表的类型是[`AllConferencesQuery.Data.AllConference]`，所以为了使用这个信息，你要访问每个元素的`fragments`并映射出`ConferenceDetails`。

剩下的就是告知`UITableView`如何显示会议数据了。

打开_ConferenceCell.swift_并添加如下属性：

``` swift
    var conference: ConferenceDetails! {
      didSet {
        nameLabel.text = "(conference.name) (conference.year)"
        let attendeeCount = conference.numberOfAttendees
        infoLabel.text = 
          "\(conference.city) (\(attendeeCount) \(attendeeCount == 1 ? "attendee" : "attendees"))"
      }
    } 
```

注意代码还不能通过编译，因为`numberOfAttendees`还不可用。你稍后会修复它。

接下来，打开_ConferencesTableViewController.swift_并替换`UITableViewDataSource`实现如下：

``` swift
    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
      return conferences.count
    }
    
    override func tableView(_ tableView: UITableView,
                            cellForRowAt indexPath: IndexPath) -> UITableViewCell {
      let cell = tableView.dequeueReusableCell(withIdentifier: "ConferenceCell") as! ConferenceCell
      let conference = conferences[indexPath.row]
      cell.conference = conference
      cell.isCurrentUserAttending = conference.isAttendedBy(currentUserID!)
      return cell
    }
```

这是标准的`UITableViewDataSource`实现。然而，编译器会抱怨不能在`ConferenceDetails`中找到`isAttendedBy`。

`numberOfAttendees`和`isAttendedBy`都表示了有用的信息，我们希望它们作为“模型”的工具函数。然而，记得吗，`ConferenceDetails`是自动生成在_API.swift里的。你不应该手动修改这个文件，因为你的改动会在Xcode下次构建时被覆盖！

逃离此困境的方法之一是在另外一个文件中创建一个_extension_来实现你想要的功能。打开_Utils.swift_并添加如下extension：
    
``` swift
    extension ConferenceDetails {
      
      var numberOfAttendees: Int {
        return attendees?.count ?? 0
      }
      
      func isAttendedBy(_ attendeeID: String) -> Bool {
        return attendees?.contains(where: { $0.id == attendeeID }) ?? false
      }
    }
```

运行app，你会看到之前添加的会议都出现在列表中。

![Listing of the conferences][34]

## 显示会议的详细信息

`ConferenceDetailViewController`会显示被选中的会议的详细信息，包括与会者的列表。

你将通过编写GraphQL query并生成Swift类型来准备。

创建一个新文件_ConferenceDetailViewController.graphql_并添加如下代码：
    
``` graphql
    query ConferenceDetails($id: ID!) {
      conference: Conference(id: $id) {
        ...ConferenceDetails
      }
    }
    
    query AttendeesForConference($conferenceId: ID!) {
      conference: Conference(id: $conferenceId) {
        id
        attendees {
          ...AttendeeDetails
        }
      }
    }
    
    fragment AttendeeDetails on Attendee {
      id
      name
      _conferencesMeta {
        count
      }
    }
```

在第一个query中，你提供一个`id`来请求特定的会议。第二个query返回此会议所有的与会者，对于每一个与会者，所有信息都由`AttendeeDetails`指定，包括`id`和`name`，以及与会者参加的所有会议的数量。

The `_conferencesMeta` field in `AttendeeDetails` fragment allows you to retrieve additional information about the relation. Here you're asking for the number of attendees using `count`.

`AttendeeDetails`fragment中的`_conferencesMeta`字段让你可以获取额外的关系信息。你通过使用`count`得到与会者所有会议的数量。（译者注：这里的原文是"Here you're asking for the number of attendees using `count`"，疑似有误）

构建应用生成Swift类型。

接下来，打开_ConferenceDetailViewController.swift_并添加如下属性到`IBOutlet`声明的下方：

``` swift
    var conference: ConferenceDetails! {
      didSet {
        if isViewLoaded {
          updateUI()
        }
      }
    }
      
    var attendees: [AttendeeDetails]? {
      didSet {
        attendeesTableView.reloadData()
      }
    }
    
    var isCurrentUserAttending: Bool {
      return conference?.isAttendedBy(currentUserID!) ?? false
    }
```

头两个属性实现的`didSet`属性监听器确保UI的即时更新。最后一个计算当前用户参与的会议的意图是否被显示。

`updateUI`方法将用选择的会议信息来配置UI元素。实现如下：
    
``` swift
    func updateUI() {
      nameLabel.text = conference.name
      infoLabel.text = "(conference.city), (conference.year)"
      attendingLabel.text = isCurrentUserAttending ? attendingText : notAttendingText
      toggleAttendingButton.setTitle(isCurrentUserAttending ? attendingButtonText : notAttendingButtonText, for: .normal)
    }
```

最后，在_ConferenceDetailViewcontroller.swift_中替换`tableView(_:numberOfRowsInSection:)`和`tableView(_:cellForRowAt:)`的实现如下：

``` swift
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return attendees?.count ?? 0
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
      guard let attendees = self.attendees else { return UITableViewCell() }
      
      let cell = tableView.dequeueReusableCell(withIdentifier: "AttendeeCell")!
      let attendeeDetails = attendees[indexPath.row]
      cell.textLabel?.text = attendeeDetails.name
      let otherConferencesCount = attendeeDetails.numberOfConferencesAttending - 1
      cell.detailTextLabel?.text = "attends (otherConferencesCount) other conferences"
      return cell
    }
```   

如之前所见，编译器会抱怨`AttendeeDetails`中没有`numberOfConferencesAttending`。你同样通过将其实现在extension中来修复。
 
打开_Utils.swift_添加如下extension：

``` swift
    extension AttendeeDetails {
      
      var numberOfConferencesAttending: Int {
        return conferencesMeta.count
      }

    }
```

最后通过在`viewDidLoad`中加载数据来完成`ConferenceDetailViewController`：

``` swift
    let conferenceDetailsQuery = ConferenceDetailsQuery(id: conference.id) 
    apollo.fetch(query: conferenceDetailsQuery) { result, error in
      guard let conference = result?.data?.conference else { return }
      self.conference = conference.fragments.conferenceDetails
    }
    
    let attendeesForConferenceQuery = AttendeesForConferenceQuery(conferenceId: conference.id)
    apollo.fetch(query: attendeesForConferenceQuery) { result, error in
      guard let conference = result?.data?.conference else { return }
      self.attendees = conference.attendees?.map { $0.fragments.attendeeDetails }
    }
```

最后，你需要传递被选中会议的信息给`ConferenceDetailViewController`，这刚好可在segue被执行时搞定。

打开_ConferencesTableViewController.swift_并实现`prepare(for:sender:)`如下：

``` swift
    override func prepare(for segue: UIStoryboardSegue, sender: Any?) {
      let conferenceDetailViewController = segue.destination as! ConferenceDetailViewController
      conferenceDetailViewController.conference = conferences[tableView.indexPathForSelectedRow!.row]
    }
```

好了！运行app并从列表中选择一个会议。在详细界面，你会看到被选中会议的细节信息。

![][35]

## 改变参加意愿后自动更新UI

使用Apollo iOS客户端的一大优势是它对之前查询的[规范化和缓存][36]。当发送一个mutation时，它知道数据中的哪些比特改变了，然后特定地更新缓存，而不需要重新发送原始的query。一个很棒的副作用是它允许“自动UI更新”，接下来体会。

在`ConferenceDetailViewController`中，有个按钮允许用户修改他们参加会议的意愿。为了改变后端的状态，你需要先在_ConferenceDetailViewController.graphql_中创建两个mutation：

``` graphql
    mutation AttendConference($conferenceId: ID!, $attendeeId: ID!) {
      addToAttendees(conferencesConferenceId: $conferenceId, attendeesAttendeeId: $attendeeId) {
        conferencesConference {
          id
          attendees {
            ...AttendeeDetails
          }
        }
      }
    }
    
    mutation NotAttendConference($conferenceId: ID!, $attendeeId: ID!) {
      removeFromAttendees(conferencesConferenceId: $conferenceId, attendeesAttendeeId: $attendeeId) {
        conferencesConference {
          id
          attendees {
            ...AttendeeDetails
          }
        }
      }
    }
```

第一个mutation用于添加一个与会者到会议；第二个移除一个与会者。

构建应用确保所有的mutation的Swift类型都被创建。

打开_ConferenceDetailViewController.swift_并替换`attendingButtonPressed`方法如下：

``` swift
    @IBAction func attendingButtonPressed() {
      if isCurrentUserAttending {
        let notAttendingConferenceMutation = 
          NotAttendConferenceMutation(conferenceId: conference.id,
                                      attendeeId: currentUserID!)
        apollo.perform(mutation: notAttendingConferenceMutation, resultHandler: nil)
      } else {
        let attendingConferenceMutation = 
          AttendConferenceMutation(conferenceId: conference.id,
                                   attendeeId: currentUserID!)
        apollo.perform(mutation: attendingConferenceMutation, resultHandler: nil)
      }
    }
```

如果你现在运行app，你将能改变参加的某个会议的状态（你可通过在[Graphcool console][17]的数据浏览器中验证）。但目前这个改变还不会反映到UI上。

别担心，Apollo iOS客户端会罩着你！通过使用[`GraphQLQueryWatcher`][37]，你能监听mutation带来的改变。要使用`GraphQLQueryWatcher`，只需做如下少量修改。
    
首先，打开_ConferenceDetailViewController.swift_并添加两个属性在顶部：

``` swift
    var conferenceWatcher: GraphQLQueryWatcher<ConferenceDetailsQuery>?
    var attendeesWatcher: GraphQLQueryWatcher<ConferenceDetailsQuery>?
```

接下来，你要修改`viewDidLoad`中发送query的方式，使用方法`watch`替换`fetch`，并将返回值赋给上面创建的属性：

``` swift
    ...
    let conferenceDetailsQuery = ConferenceDetailsQuery(id: conference.id)
    conferenceWatcher = apollo.watch(query: conferenceDetailsQuery) { [weak self] result, error in      guard let conference = result?.data?.conference else { return }
      self?.conference = conference.fragments.conferenceDetails
    }
    ...
```

以及

``` swift    
    ...
    let attendeesForConferenceQuery = AttendeesForConferenceQuery(conferenceId: conference.id)
    attendeesWatcher = apollo.watch(query: attendeesForConferenceQuery) { [weak self] result, error in
      guard let conference = result?.data?.conference else { return }
      self?.attendees = conference.attendees?.map { $0.fragments.attendeeDetails }
    }
    ...
```    

与`ConferenceDetailsQuery`相关的数据或与`AttendeesForConferenceQuery`在缓存中的修改相关的数据，都会导致传递给`watch`的尾随闭包的的执行，从而更新UI。

最后一件为了让watchers正常工作的事是实现`ApolloClient`实例的`cacheKeyForObject`方法。这个方法告诉Apollo你想如何唯一识别它放入缓存的对象。在此例中，检查属性`id`就很好。
    
实现`cacheKeyForObject`的绝佳之处是在app初次启动时。打开_AppDelegate.swift_并添加如下代码到`application(_:didFinishLaunchingWithOptions:)`的`return`语句前：

``` swift
    apollo.cacheKeyForObject = { $0["id"] }
```

_注意_：如果你想知道更多关于为何这是必要的以及Apollo缓存的工作方式，你可阅读 [Apollo blog][38]。

再次运行app，并修改你参加会议的意图，现在UI会立即更新。然而，当你回到`ConferencesTableViewController`，你还不会看到会议cell中状态的更新。

![][39]

为了修复此问题，同样是用一个`GraphQLQueryWatcher`。打开_ConferencesTableViewController.swift_并增加如下属性到类的顶部：

``` swift
    var allConferencesWatcher: GraphQLQueryWatcher?
```

接着，更新`viewDidLoad`中的query：
    
``` swift
    ...
    let allConferencesQuery = AllConferencesQuery()
    allConferencesWatcher = apollo.watch(query: allConferencesQuery) { result, error in
      guard let conferences = result?.data?.allConferences else { return }
      self.conferences = conferences.map { $0.fragments.conferenceDetails }
    }
    ...
```

这就确保了传递给`watch`的尾随闭包在每次缓存中与`AllConferencesQuery`相关的改变发生时自动执行。

## 接下来如何?

看看此教程的[final project][40]，也许你需要对比一下。

如果你想学到更多关于GraphQL的知识，你可以开始阅读它良好的[文档][2]，或者订阅[GraphQL weekly][41]。

更多关于GraphQL社区的精彩内容可在[Apollo][42]和[Graphcool][43]的博客中找到。

作为一个挑战，你可实现添加新会议的功能。这个特性同样已包含在样本代码中。

我们希望你学习GraphQL时感到有趣！请让我们知道你对这种新的API范式的想法，加入下面的论坛讨论。

---

欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

[1]: https://koenig-media.raywenderlich.com/uploads/2017/05/apollo-feature.png
[2]: http://www.graphql.org
[3]: https://en.wikipedia.org/wiki/Representational_state_transfer
[4]: https://github.com/apollographql/apollo-ios
[5]: https://www.npmjs.com/
[6]: https://koenig-media.raywenderlich.com/uploads/2017/05/ConferencePlanner-starter.zip
[7]: https://koenig-media.raywenderlich.com/uploads/2017/04/storyboard.png
[8]: https://cocoapods.org/
[9]: https://koenig-media.raywenderlich.com/uploads/2017/04/pod-install.png
[10]: https://koenig-media.raywenderlich.com/uploads/2017/04/last3.png
[11]: https://koenig-media.raywenderlich.com/uploads/2017/04/Screen-Shot-2017-04-08-at-20.07.15-650x481.png
[12]: https://koenig-media.raywenderlich.com/uploads/2017/04/Screen-Shot-2017-04-08-at-20.07.27-650x494.png
[13]: http://www.graph.cool
[14]: https://www.graph.cool/docs/faq/graphql-schema-definition-idl-kr84dktnp0/
[15]: https://www.npmjs.com/package/graphcool
[16]: https://koenig-media.raywenderlich.com/uploads/2017/04/endpoints.png
[17]: https://console.graph.cool
[18]: https://www.graph.cool/docs/faq/tips-and-tricks-graphql-playground-ook6luephu/
[19]: https://koenig-media.raywenderlich.com/uploads/2017/04/demo-mutations-1-650x354.png
[20]: https://www.youtube.com/watch?v=XeLKw2BSdI4&t=16s
[21]: https://koenig-media.raywenderlich.com/uploads/2017/04/allConferences-650x334.png
[22]: https://github.com/apollographql/apollo-codegen
[23]: https://koenig-media.raywenderlich.com/uploads/2017/04/install-apollo-codegen.png
[24]: https://koenig-media.raywenderlich.com/uploads/2017/04/run-script-phase-2-650x402.png
[25]: https://koenig-media.raywenderlich.com/uploads/2017/04/verify-build-phases-650x484.png
[26]: http://imgur.com/jeBgjkH.png
[27]: http://imgur.com/GCNQRiP.png
[28]: https://koenig-media.raywenderlich.com/uploads/2017/04/other-650x449.png
[29]: http://dev.apollodata.com/ios/installation.html#installing-xcode-add-ons
[30]: https://koenig-media.raywenderlich.com/uploads/2017/04/file-tree-380x500.png
[31]: https://koenig-media.raywenderlich.com/uploads/2017/04/phone1-240x500.png
[32]: https://koenig-media.raywenderlich.com/uploads/2017/04/allAttendees-650x274.png
[33]: http://graphql.org/learn/queries/#fragments
[34]: https://koenig-media.raywenderlich.com/uploads/2017/04/phone2-238x500.png
[35]: https://koenig-media.raywenderlich.com/uploads/2017/04/phone3-240x500.png
[36]: http://dev.apollodata.com/core/how-it-works.html#normalize
[37]: http://dev.apollodata.com/ios/watching-queries.html#watching-queries
[38]: https://dev-blog.apollodata.com/the-concepts-of-graphql-bc68bd819be3
[39]: https://koenig-media.raywenderlich.com/uploads/2017/04/phone4-241x500.png
[40]: https://koenig-media.raywenderlich.com/uploads/2017/05/ConferencePlanner-final-1.zip
[41]: https://graphqlweekly.com/
[42]: https://dev-blog.apollodata.com/
[43]: https://www.graph.cool/blog/



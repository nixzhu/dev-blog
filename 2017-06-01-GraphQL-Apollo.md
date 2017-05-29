[Source](https://www.raywenderlich.com/158433/getting-started-graphql-apollo-ios "Permalink to Getting started with GraphQL & Apollo on iOS")

# Getting started with GraphQL & Apollo on iOS

![Getting started with GraphQL & Apollo on iOS][1]

Have you even felt frustrated when working with a REST API, because the endpoints didn't give you the data you needed for the views in your app? Getting the right information from the server either required multiple requests or you had to bug the backend developers to adjust the API? Worry no more — it's _GraphQL_ and _Apollo_ to the rescue!

当你配合一个REST API工作时，因为某个端点没有给你所需要的数据用于app的某些视图，你是否有感到沮丧呢？需要多个请求才能从服务器获取到正确的信息，或者你需要报bug通知后端开发者才能调整API？现在，你不需要再担忧，_Apollo_将拯救你。

[GraphQL][2] is a new API design paradigm open-sourced by Facebook in 2015, but has been powering their mobile apps since 2012. It eliminates many of the inefficiencies with today's [REST][3] API. In contrast to REST, GraphQL APIs only expose a single endpoint and the consumer of the API can specify precisely what data they need.

[GraphQL][2]是一种新的API设计范式，由Facebook在2015年开源，不过在2012，他们就开始用于给自家的app提供支持了。它消除了当今[REST][3] API的许多低效之处。对比REST，GraphQL只需要暴露一个端点，API的消费者就能精确地指定他们所需要的数据。

In this GraphQL & Apollo on iOS tutorial, you're going to build an iPhone app that helps users plan which iOS conferences they'd like to attend. You'll setup your own GraphQL server and interact with it from the app using the [Apollo iOS Client][4], a networking library that makes working with GraphQL APIs a breeze :]

在本教程中，你将构建一个iPhone app来帮助用户计划参与那些iOS会议。你将设置自己的GraphQL服务器并在app中用[Apollo iOS Client][4]与之交互。Apollo是一个网络库，它能让我们与GraphQL API工作起来如沐春风。

The app will have the following features:

* Display list of iOS conferences
* Mark yourself as attending / not attending
* View who else is going to attend a conference

此app将包含如下特性：

* 显示iOS会议列表
* 标记自己想参加或不想参加
* 查看其他参加某个会议的人

For this GraphQL & Apollo on iOS tutorial, you'll have to install some tooling using the [Node Package Manager][5], so make sure to have npm version 4.5.0 (or higher) installed before continuing!

为了本教程的缘故，你要先通过[Node Package Manager][5]安装一些工具，请确保你已经安装有npm，且版本在4.5.0（或以上）。

## Getting Started

Download and open the [starter project][6] for this GraphQL & Apollo on iOS tutorial. It already contains the required UI components, so you can focus on interacting with the API and bringing the right data into the app.

下载并打开用于本教程的[starter project][6]。它已包含有必要的UI组件，因此你可以专注于与API打交道，把数据正确地带到app中。

Here is what the _Storyboard_ looks like:

Storyboard看起来如下：

![Application Main Storyboard][7]

You're using [CocoaPods][8] for this app, so you'll have to open the _ConferencePlanner.xcworkspace_ after you've downloaded the package. The `Apollo` pod is already included in the project. However, you should ensure you have the most recent version installed.

此app使用[CocoaPods][8]，所以在下载好第三方包后，你要打开_ConferencePlanner.xcworkspace_。`Apollo`的pod已经被包含在项目中，不过你应该确保安装的是最新的版本。

Open a new Terminal window, navigate to the directory where you downloaded the starter project to, and execute `pod install` to update to the latest version:

打开终端，导航到项目所在目录，并执行`pod install`来更新：

![Pod Install Output][9]

## Why GraphQL?

REST APIs expose multiple endpoints where each endpoint returns specific information. For example, you might have the following endpoints:

REST API暴露多个端点，每个端点都返回特定的信息。例如，你可能有如下端点：

* `/conferences`: Returns a list of all the conferences where each conferences has `id`, `name`, `city` and `year`
* `/conferences/__id__/attendees`: Returns a list of all conference attendees (each having an `id` and `name`) and the conference `id`. 

* `/conferences`：返回一个所有会议的列表，每个会议都有`id`、`name`,、`city`和`year`字段
* `/conferences/__id__/attendees`：返回某个会议所有的参加者（包括`id`和`name`）和会议`id`

Imagine you're writing an app to display a list of all the conferences, plus the three latest registered attendees per conference. What are your options?

想象一下，你所写的app要显示所有会议的列表，加上每个会议最近注册的三个参加者。你有何选择？

![iOS Screen Application Running][10]

_Option 1: Adjust the API_

选项1：修改API

Tell your backend developers to change the API so each call to `/conferences` also returns the last three registrations:

告诉后端开发者修改API，这样每次调用`/conferences`是都返回最近注册的三个参加者：

![Conferences REST API Endpoint Data][11]

_Option 2: Make n+1 requests_

选项2：发送n+1次请求

Send `n+1` requests (where `n` is the number of conferences) to retrieve the required information, accepting you might exhaust the user's data plan because you're downloading all the conferences' attendees but actually only display the last three:

发送`n+1`次请求（此处`n`是会议的数量）来获取所需的信息，并且接受你可能会耗尽用户的网络流量，因为虽然每个会议你只需要最近的三个参与者确也必须下载全部。

![Conference Attendee REST Endpoint Data][12]

Both options are not terribly compelling, and won't scale well in larger development projects!

两个选项都不是很有吸引力，在更大的开发项目里也不具有很好的伸缩性！

Using GraphQL, you're able to simply specify your data requirements in a single request using GraphQL syntax and describe the data you need in a declarative fashion:

使用GraphQL，你就可以简单地指定你在单个请求里所需要的数据，只需要使用GraphQL语法以申明式的方式描述即可：
    
    
    
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
    

The response of this query will contain an array of conferences, each carrying a `name`, `city` and `year` as well as the last three `attendees`.

这个查询的应答将包含一个会议的列表，其中每个都有`name`、`city`和`year`，而且带有三个最新的参与者。

### Using GraphQL On iOS with Apollo

GraphQL isn't very popular in the mobile developer communities (yet!), but that might change with more tooling evolving around it. A first step in that direction is the [Apollo iOS client][4], which implements handy features you'll need when working with APIs.

GraphQL在移动开发者社区中还不是很流行，不过很可能会随着更多相关工具的改进而改变。这种势头表现在[Apollo iOS client][4]上，它实现了许多方便特性，让你与API合作时更轻松。

Currently, its major features are as follows:

1. Static type generation based on your data requirements
2. Caching and watching queries

目前，它的主要特性如下：

1. 基于数据要求的静态类型生成
2. 缓存与察看查询

You'll get experience with both of these extensively throughout this GraphQL & Apollo on iOS tutorial.

通过本教程，你对这两个特性都会有所体会。

### Interacting with GraphQL

When interacting with an API, the main goals generally are:

* Fetching data
* Creating, updating and deleting data

当与API交互时，主要目标通常是：

* 获取数据
* 创建、更新和删除数据

In GraphQL, _fetching_ data is done using _queries_, while _writing_ to the database can be achieved through _mutations_.

在GraphQL中，_获取_数据通过_queries_完成，而写入数据库通过_mutations_来达成。

A mutation, much like a query, also allows you to declare information to be returned by the server and thus enables you to retrieve the updated information in a single roundtrip!

一个mutation看起来很像一个query，一样允许你申明需要服务器返回的信息，这就确保了在一个单程往返中你就可以获取更新后的信息。

Consider the following two simple examples:
    
考虑如下两个简单例子：
    
    query AllConferences {
      allConferences {
        id
        name
      }
    }
    

This query retrieves all the conferences and returns a JSON array where each object carries the `id` and `name` of a conference.
    
次查询获取所有的会议并返回一个JSON数组，其中每个对象包含有会议的`id`和`name`。
    
    mutation CreateConference {
      createConference(name: "WWDC", city: "San Jose", year: "2017") {
        id 
      }
    }
    

This mutation creates a new conference and likewise returns its `id`.

这个mutation创建了一个新会议并类似地返回其`id`。

Don't worry if you don't quite grok the syntax yet — it'll be discussed in more detail later!

如果你对这种语法还没什么感觉，别担心，稍后会探讨更多细节。

## Preparing Your GraphQL Server

For the purpose of this GraphQL & Apollo on iOS tutorial, you're going to use a service called [Graphcool][13] to generate a full-blown GraphQL server based on a data model.

Speaking of the data model, here is what it looks like for the application, expressed in a syntax called [GraphQL Interface Definition Language][14] (IDL):
    
    
    
    type Conference {
      id: String!
      name: String!
      city: String!
      year: String!
      attendees: [Attendee] @relation(name: Attendees)
    }
    
    type Attendee {
      id: String!
      name: String!
      conferences: [Conference] @relation(name: Attendees)
    }
    

GraphQL has its own type system you can build upon. The types in this case are `Conference` and `Attendee`. Each type has a number of properties, called _fields_ in GraphQL terminology. Notice the `!` following the type of each field, which means this field is required.

Enough talking, go ahead and create your well-deserved GraphQL server!

Install the [Graphcool CLI][15] with npm. Open a Terminal window and type the following:
    
    
    
    npm install -g graphcool
    

Use `graphcool` to create your GraphQL server by typing the following into a Terminal window:
    
    
    
    graphcool init --schema http://graphqlbin.com/conferences.graphql --name ConferencePlanner
    

This command will create a Graphcool project named `ConferencePlanner`. Before the project is created, it'll also open up a browser window where you need to create a Graphcool account. Once created, you'll have access to the full power of GraphQL:

![GraphCool command output showing Simple API and Relay API][16]

Copy the endpoint for the `Simple API` and save it for later usage.

That's it! You now have access to a fully-fledged GraphQL API you can manage in the [Graphcool console][17].

### Entering Initial Conference Data

Before continuing, you'll add some initial data to the database.

Copy the endpoint from the `Simple API` you received in the previous step and paste it in the address bar of your browser. This will open a [GraphQL Playground][18] that lets you explore the API in an interactive manner.

To create some initial data, add the following GraphQL code into the left section of the Playground:
    
    
    
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
    

This snippet contains code for two GraphQL mutations. Click the _Play_ button and _select each of the mutations displayed in the dropdown only once_:

![GraphQL playground][19]

This will create two new conferences. To verify the conferences have been created, you can either view the current state of your database using the [data browser][20] in the [Graphcool console][17] or send the `allConferences` query you saw before in the Playground:

![GraphQL Playground AllConferences query result][21]

## Configuring Xcode and Setting Up the Apollo iOS Client

As mentioned before, the Apollo iOS client features _static type generation_. This means you effectively don't have to write the _model_ types which you'd use to represent the information from your application domain. Instead, the Apollo iOS client uses the information from your GraphQL queries to generate the Swift types you need!

_Note:_ This approach eliminates the inconvenience of parsing JSON in Swift. Since JSON is not typed, the only real safe approach to parse it is by having _optional_ properties on the Swift types, since you can never be 100% sure whether a particular property is actually included in the JSON data. 

To benefit from static type generation in Xcode, you'll have to go through some configuration steps:

_1\. Install apollo-codegen_

[`apollo-codegen`][22] will search for GraphQL code in the Xcode project and generate the Swift types.

Open a Terminal window and type the following command:
    
    
    
    npm install -g apollo-codegen
    

![NPM results from apollo-codegen installation][23]

_2\. Add a build phase_

In Xcode, select the _ConferencePlanner_ in the _Project Navigator_. Select the application target called _ConferencePlanner_. Select the _Build Phases_ tab on top, and click the _+_ button on the top left. 

Select _New Run Script Phase_ from the menu: 

![Xcode altering build phases adding new run script][24]

Rename the newly added build phase to _Generate Apollo GraphQL API_. Drag and drop the build phase to be above the _Compile Sources_.

Copy the following code snippet into the field that currently says: _Type a script or drag a script file from your workspace to insert its path_:
    
    
    
    APOLLO_FRAMEWORK_PATH="$(eval find $FRAMEWORK_SEARCH_PATHS -name "Apollo.framework" -maxdepth 1)"
    
    if [ -z "$APOLLO_FRAMEWORK_PATH" ]; then
    echo "error: Couldn't find Apollo.framework in FRAMEWORK_SEARCH_PATHS; make sure to add the framework to your project."
    exit 1
    fi
    
    cd "${SRCROOT}/${TARGET_NAME}"
    $APOLLO_FRAMEWORK_PATH/check-and-run-apollo-codegen.sh generate $(find . -name '*.graphql') --schema schema.json --output API.swift
    

Verify your _Build Phases_ look like this:

![Build Script Run Phase Code to Run][25]

_3\. Add the schema file_

This is where you need the endpoint for the `Simple API` again. Open a Terminal window and type the following command (replacing `__SIMPLE_API_ENDPOINT__` with the custom GraphQL endpoint you previously generated):
    
    
    
    apollo-codegen download-schema __SIMPLE_API_ENDPOINT__ --output schema.json
    

_Note:_ If you lose your GraphQL endpoint, you can always find it in the [Graphcool console][17] by clicking the _ENDPOINTS_ button in the bottom-left corner:  
![GraphCool Endpoint Console][26]

Next, move this file into the root directory of the Xcode project. This is the same directory where _AppDelegate.swift_ is located — _ConferencePlanner-starter/ConferencePlanner_:

![Finder File Listing showing Schema.json][27]

Here is a quick summary of what you just did:

* You first installed `apollo-codegen`, the command-line tool that generates the Swift types.
* Next, you added a build phase to the Xcode project where `apollo-codegen` will be invoked on every build just before compilation.
* Next to your actual GraphQL queries (which you're going to add in just a bit), `apollo-codegen` requires a schema file to be available in the root directory of your project which you downloaded in the last step.

## Instantiate the ApolloClient

You're finally going to write some actual code!

Open _AppDelegate.swift_, and add the following code replacing `__SIMPLE_API_ENDPOINT__` with your own endpoint for the `Simple API`:
    
    
    
    import Apollo
    
    let graphQLEndpoint = "__SIMPLE_API_ENDPOINT__"
    let apollo = ApolloClient(url: URL(string: graphQLEndpoint)!)
    

You need to pass the endpoint for the `Simple API` to the initializer so the `ApolloClient` knows which GraphQL server to talk to. The resulting `apollo` object will be your main interface to the API.

## Creating Your Attendee and Querying the Conference List

You're all set to start interacting with the GraphQL API! First, make sure users of the app can register themselves by picking a username.

### Writing Your First Mutation

Create new file in the _GraphQL_ Xcode group using the _Empty_ file template from the _Other_ section and name it _RegisterViewController.graphql_:

![Xcode New File Picker][28]

Next, add the following mutation into that file:
    
    
    
    # 1
    mutation CreateAttendee($name: String!) {
      # 2
      createAttendee(name: $name) {
        # 3
        id
        name
      }
    }
    

Here's what's going on in that mutation:

1. This part represents the _signature_ of the mutation (somewhat similar to a Swift function). The mutation is named `CreateAttendee` and takes an argument called `name` of type `String`. The exclamation mark means this argument is required.
2. `createAttendee` refers to a mutation exposed by the GraphQL API. Graphcool Simple API provides `create`-mutations for each type out of the box.
3. The _payload_ of the mutation, i.e. the data you'd like the server to return after the mutation was performed.

On next build of the project, `apollo-codegen` will find this code and generate a Swift representation for the mutation from it. Hit _Cmd+B_ to build the project.

_Note:_ If you'd like to have _syntax highlighting_ for your GraphQL code, you can follow the instructions [here][29] to set it up. 

The first time `apollo-codegen` runs, it creates a new file in the root directory of the project named _API.swift_. All subsequent invocations will just update the existing file.

The generated _API.swift_ file is located in the root directory of the project, but you still need to add it to Xcode. Drag and drop it into the _GraphQL_ group. Make sure to _uncheck_ the _Copy items if needed_ checkbox!

![Xcode Project Navigator showing API.swift in the GraphQL group][30]

When inspecting the contents of _API.swift_, you'll see a class named `CreateAttendeeMutation`. Its initializer takes the `name` variable as an argument. It also has a nested struct named `Data` which nests a struct called `CreateAttendee`. This will carry the `id` and the `name` of the attendee you specified as return data in the mutation.

Next, you'll incorporate the mutation. Open _RegisterViewController.swift_ and implement the `createAttendee` method like so:
    
    
    
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
    

In the code above, you:

1. Instantiate the mutation with the user provided string.
2. Use the `apollo` instance to send the mutation to the API.
3. Retrieve the data returned by the server and store it globally as information about the current user.

_Note:_ All the API calls you'll be doing in this GraphQL & Apollo on iOS tutorial will follow this pattern: First instantiate a query or mutation, then pass it to the `ApolloClient` and finally make use of the results in a callback. 

Since users are allowed to change their usernames, you can add the second mutation right away. Open _RegisterViewController.graphql_ and add the following code at the end:
    
    
    
    mutation UpdateAttendeeName($id: ID!, $newName: String!) {
      updateAttendee(id: $id, name: $newName) {
        id
        name
      }
    }
    

Press _Cmd+B_ to make `apollo-codegen` generate the Swift code for this mutation. Next, open _RegisterViewController.swift_ and replace `updateAttendee` with the following:
    
    
    
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
    

The code is almost identical to `createAttendee`, except this time you also pass the `id` of the user so the GraphQL server knows which user it should update.

Build and run the app, type a name into the text field, then click the _Save_ button. A new attendee will be created in the GraphQL backend.

![User Settings page for the application][31]

You can validate this by checking the data browser or sending the `allAttendees` query in a Playground:

![GraphCool Playground showing AllAttendees query][32]

### Querying All Conferences

The next goal is to display all the conferences in the `ConferencesTableViewController`.

Create a new file in the GraphQL group, name it _ConferenceTableViewController.graphql_ and add the following GraphQL code:
    
    
    
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
    

What's that _fragment_ thing there?

[Fragments][33] are simply _reusable sub-parts_ that bundle a number of fields of a GraphQL type. They come in very handy in combination with the static type generation since they enhance the reusability of the information returned by the GraphQL server, and each fragment will be represented by its own struct.

Fragments can be integrated in any query or mutation using `...` plus the fragment name. When the `AllConferences` query is sent, `...ConferenceDetails` is replaced with all the fields contained within the `ConferenceDetails` fragment.

Next it's time to use the query to populate the table view.

Press _Cmd+B_ to make sure the types for the new query and fragment are generated, then open _ConferencesTableViewController.swift_ and add a new property at the top:
    
    
    
    var conferences: [ConferenceDetails] = [] {
      didSet {
        tableView.reloadData()
      }
    }
    

At the end of `viewDidLoad`, add the following code to send the query and display the results:
    
    
    
    let allConferencesQuery = AllConferencesQuery()
    apollo.fetch(query: allConferencesQuery) { [weak self] result, error in
      guard let conferences = result?.data?.allConferences else { return }  
      self?.conferences = conferences.map { $0.fragments.conferenceDetails }
    }
    

You're using the same pattern you saw in the first mutations, except this time you're sending a query instead. After instantiating the query, you pass it to the `apollo` instance and retrieve the lists of conferences in the callback. This list is of type [`AllConferencesQuery.Data.AllConference]`, so in order to use its information you first must retrieve the values of type `ConferenceDetails` by mapping over it and accessing the `fragments`.

All that's left to do is tell the `UITableView` how to display the conference data.

Open _ConferenceCell.swift_, and add the following property:
    
    
    
    var conference: ConferenceDetails! {
      didSet {
        nameLabel.text = "(conference.name) (conference.year)"
        let attendeeCount = conference.numberOfAttendees
        infoLabel.text = 
          "(conference.city) ((attendeeCount) (attendeeCount == 1 ? "attendee" : "attendees"))"
      }
    } 
    

Notice the code doesn't compile since `numberOfAttendees` is not available. You'll fix that in a second

Next, open _ConferencesTableViewController.swift_, and replace the current implementation of `UITableViewDataSource` with the following:
    
    
    
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
    

This is a standard implementation of `UITableViewDataSource`. However, the compiler complains `isAttendedBy` can't be found on the `ConferenceDetails` type.

Both `numberOfAttendees` and `isAttendedBy` represent useful information that could be expected as utility functions on the "model" `ConferenceDetails`. However, remember `ConferenceDetails` is a generated type and lives in _API.swift_. You should never make manual changes in that file, since they'll be overridden the next time Xcode builds the project!

A way out of this dilemma is to create an _extension_ in a different file where you implement the desired functionality. Open _Utils.swift_ and add the following extension:
    
    
    
    extension ConferenceDetails {
      
      var numberOfAttendees: Int {
        return attendees?.count ?? 0
      }
      
      func isAttendedBy(_ attendeeID: String) -> Bool {
        return attendees?.contains(where: { $0.id == attendeeID }) ?? false
      }
    }
    

Run the app and you'll see the conferences you added in the beginning displayed in the table view:

![Listing of the conferences][34]

## Displaying Conference Details

The `ConferenceDetailViewController` will display information about the selected conference, including the list of attendees.

You'll prepare everything by writing the GraphQL queries and generating the required Swift types.

Create a new file named _ConferenceDetailViewController.graphql_ and add the following GraphQL code:
    
    
    
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
    

In the first query, you ask for a specific conference by providing an `id`. The second query returns all attendees for a specific conference where for each attendee, all the info is specified in `AttendeeDetails` will be returned by the server. That includes the attendee's `id`, `name` and the number of conferences they're attending.

The `_conferencesMeta` field in `AttendeeDetails` fragment allows you to retrieve additional information about the relation. Here you're asking for the number of attendees using `count`.

Build the application to generate the Swift types.

Next, open _ConferenceDetailViewController.swift_ and add the following properties below the  
`IBOutlet` declarations:
    
    
    
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
    

The first two properties implement the `didSet` property observer to make sure the UI gets updated after they're set. The last one computes if the current user attends the conference being displayed.

The `updateUI` method will configure the UI elements with the information about the selected conference. Implement it as follows:
    
    
    
    func updateUI() {
      nameLabel.text = conference.name
      infoLabel.text = "(conference.city), (conference.year)"
      attendingLabel.text = isCurrentUserAttending ? attendingText : notAttendingText
      toggleAttendingButton.setTitle(isCurrentUserAttending ? attendingButtonText : notAttendingButtonText, for: .normal)
    }
    

Finally, in _ConferenceDetailViewcontroller.swift_, replace the current implementation of `tableView(_:numberOfRowsInSection:)` and `tableView(_:cellForRowAt:)` with the following:
    
    
    
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
    

Similarly to what you saw before, the compiler complains about `numberOfConferencesAttending` not being available on `AttendeeDetails`. You'll fix that by implementing this in an extension of `AttendeeDetails`.

Open _Utils.swift_ and add the following extension:
    
    
    
    extension AttendeeDetails {
      
      var numberOfConferencesAttending: Int {
        return conferencesMeta.count
      }
      
    }
    

Finish up the implementation of `ConferenceDetailViewController` by loading the data about the conference in `viewDidLoad`:
    
    
    
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
    

Finally, you need to pass the information about which conference was selected to the `ConferenceDetailViewController`, this can be done right before the segue is performed.

Open _ConferencesTableViewController.swift_ and implement `prepare(for:sender:)` like so:
    
    
    
    override func prepare(for segue: UIStoryboardSegue, sender: Any?) {
      let conferenceDetailViewController = segue.destination as! ConferenceDetailViewController
      conferenceDetailViewController.conference = conferences[tableView.indexPathForSelectedRow!.row]
    }
    

That's it! Run the app and select one of the conferences in the table view. On the details screen, you'll now see the info about the selected conference being displayed:

![][35]

## Automatic UI Updates When Changing the Attending Status

A major advantage of working with the Apollo iOS client is it [normalizes and caches][36] the data from previous queries. When sending a mutation, it knows what bits of data changed and can update these specifically in the cache without having to resend the initial query. A nice side-effect is it allows for "automatic UI updates", which you'll explore next.

In `ConferenceDetailViewController`, there's a button to allow the user to change their attending status of the conference. To change that status in the backend, you first have to create two mutations in _ConferenceDetailViewController.graphql_:
    
    
    
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
    

The first mutation is used to add an attendee to a conference; the second, to remove an attendee.

Build the application to make sure the types for these mutations are created.

Open, _ConferenceDetailViewController.swift_ and replace the `attendingButtonPressed` method with the following:
    
    
    
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
    

If you run the app now, you'll be able to change your attending status on a conference (you can verify this by using the data browser in the [Graphcool console][17]). However, this change is not yet reflected in the UI.

No worries: The Apollo iOS client has you covered! With the [`GraphQLQueryWatcher`][37], you can observe changes occurring through mutations. To incorporate the `GraphQLQueryWatcher`, a few minor changes are required.

First, open _ConferenceDetailViewController.swift_ and add two more properties to the top:
    
    
    
    var conferenceWatcher: GraphQLQueryWatcher?
    var attendeesWatcher: GraphQLQueryWatcher?
    

Next, you have to change the way you send the queries in `viewDidLoad` by using the method `watch` instead of `fetch` and assigning the return value of the call to the properties you just created:
    
    
    
    ...
    let conferenceDetailsQuery = ConferenceDetailsQuery(id: conference.id)
    conferenceWatcher = apollo.watch(query: conferenceDetailsQuery) { [weak self] result, error in      guard let conference = result?.data?.conference else { return }
      self?.conference = conference.fragments.conferenceDetails
    }
    ...
    

and
    
    
    
    ...
    let attendeesForConferenceQuery = AttendeesForConferenceQuery(conferenceId: conference.id)
    attendeesWatcher = apollo.watch(query: attendeesForConferenceQuery) { [weak self] result, error in
      guard let conference = result?.data?.conference else { return }
      self?.attendees = conference.attendees?.map { $0.fragments.attendeeDetails }
    }
    ...
    

Every time data related to the `ConferenceDetailsQuery` or to the `AttendeesForConferenceQuery` changes in the cache, the trailing closure you're passing to the call to `watch` will be executed, thus taking care of updating the UI.

One last thing you've to do for the watchers to work correctly is implement the `cacheKeyForObject` method on the instance of the `ApolloClient`. This method tells Apollo how you'd like to uniquely identify the objects it's putting into the cache. In this case, that's simply by looking at the `id` property.

A good place to implement `cacheKeyForObject` is when the app launches for the first time. Open _AppDelegate.swift_ and add the following line in `application(_:didFinishLaunchingWithOptions:)` before the `return` statement:
    
    
    
    apollo.cacheKeyForObject = { $0["id"] }
    

_Note:_ If you want to know more about why that's required and generally how the caching in Apollo works, you can read about it on the [Apollo blog][38]. 

Running the app again and changing your attending status on a conference will now immediately update the UI. However, when navigating back to the `ConferencesTableViewController`, you'll notice the status is not updated in the conference cell:

![][39]

To fix that, you can use the same approach using a `GraphQLQueryWatcher` again. Open _ConferencesTableViewController.swift_ and add the following property to the top of the class:
    
    
    
    var allConferencesWatcher: GraphQLQueryWatcher?
    

Next, update the query in `viewDidLoad`:
    
    
    
    ...
    let allConferencesQuery = AllConferencesQuery()
    allConferencesWatcher = apollo.watch(query: allConferencesQuery) { result, error in
      guard let conferences = result?.data?.allConferences else { return }
      self.conferences = conferences.map { $0.fragments.conferenceDetails }
    }
    ...
    

This will make sure to execute the trailing closure passed to `watch` when the data in the cache relating to `AllConferencesQuery` changes.

## Where to Go From Here?

Take a look at the [final project][40] for this GraphQL & Apollo on iOS tutorial in case you want to compare it against your work.

If you want to learn more about GraphQL, you can start by reading the excellent [docs][2] or subscribe to [GraphQL weekly][41].

More great content around everything that's happening in the GraphQL community can be found on the [Apollo][42] and [Graphcool][43] blogs.

As a challenge, you can try to implement functionality for adding new conferences yourself! This feature is also included in the sample solution.

We hope you enjoyed learning about GraphQL! Let us know what you think about this new API paradigm by joining the discussion in the forum below.

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

  
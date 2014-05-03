# 创建自定义 AngularJS 指令，第一部分，基础知识

本文翻译自 [http://weblogs.asp.net/dwahlin/archive/2014/04/29/creating-custom-angularjs-directives-part-i-the-fundamentals.aspx](http://weblogs.asp.net/dwahlin/archive/2014/04/29/creating-custom-angularjs-directives-part-i-the-fundamentals.aspx)

原作者：[@DanWahlin](https://twitter.com/DanWahlin)

译者：[@nixzhu](https://twitter.com/nixzhu)

==========================================

[AngularJS][3] 提供了很多指令（Directive）用于操作 DOM、将事件路由到事件处理器、执行数据绑定、关联控制器/scope到View上，等等很多。例如 **ng-click**、 **ng-show/ng-hide**、 **ng-repeat** ，以及许多其它能在 AngularJS 核心代码里找到的指令能让我们非常轻松地开始使用这一框架。

尽管內建指令已经覆盖了许多不同的场景，但可能有好几次你发现自己还是需要自定义指令，却不清楚如何开始编写它们。在本系列教程中，我会一步一步地提供一个研究指令如何工作以及你如何开始使用它们的方法。总体目标是慢慢前进并提供简单、易消化的指令示例，且随着本系列的进展不断深入。

我假设你已经知道指令是什么并懂得如何使用它们。如果你完全不知道指令是什么，看看我制作的 [60 分钟 AngularJS 视频教程 (或者获取 eBook 版本)][5] 或我新写的 [AngularJS JumpStart][1] 课程。


## 编写自定义指令很容易吗？

在你第一次看到 AngularJS 指令的时候，它们可能会吓着你。它们具有不同的选项，还有一些神秘功能（而“神秘”是我的政治正确术语，表示“他们到底在想什么？”），所以初次看到确实是个挑战。然而，一旦你明白一些片段并知道它们如何结合在一起，你就会发现它们并没有那么糟糕。我将其和学习一种乐器做比较。你第一次弹钢琴或吉他时，你的表现不会很好，而且听起来完全不对劲。然而，投入必要的时间去练习后，最终你能相当不错地处理更复杂的音乐。对于指令来说也一样——它们需要一些练习，而一旦你掌握了它们的诀窍，你就能做出许多强大的东西。我期待着某一天 [Polymer][6] 项目和 [Web Components][7] spec 中的特性能取代我们今天编写的指令。如果你想一窥指令的未来发展方向，就看看这些链接。


## 开始自定义指令

你为什么想写一个自定义指令呢？比方说你一直负责将一个客户集合转换为某些类型的网格以显示在视图里。虽然你确实可以通过添加 DOM 的方式来处理这个问题，但这样做会很难测试，也打破了分离原则——你不应该在 AngularJS 里这样做。相反，这种类型的功能应该用一个自定义指令来实现。在其它情况下你可能有一些数据绑定用于好几个视图，而且你想重用数据绑定表达式。使用 ng-include 加载的一个子视图就可以被使用，指令同样能很好的工作在这样的情况下。还有其它许多方式可以使用自定义指令，这些例子还只是冰山一角而已。

让我们从一个非常基本的 AngularJS 指令的例子开始。假设我们在应用里定义了如下模块和控制器（我也会在将来的文章里提到这些模块和控制器）：


```JavaScript
var app = angular.module('directivesModule', []);

app.controller('CustomersController', ['$scope', function ($scope) {
    var counter = 0;
    $scope.customer = {
        name: 'David',
        street: '1234 Anywhere St.'
    };
    
    $scope.customers = [
        {
            name: 'David',
            street: '1234 Anywhere St.'
        },
        {
            name: 'Tina',
            street: '1800 Crest St.'
        },
        {
            name: 'Michelle',
            street: '890 Main St.'
        }
    ];

    $scope.addCustomer = function () {
        counter++;
        $scope.customers.push({
            name: 'New Customer' + counter,
            street: counter + ' Cedar Point St.'
        });
    };

    $scope.changeData = function () {
        counter++;
        $scope.customer = {
            name: 'James',
            street: counter + ' Cedar Point St.'
        };
    };
}]);
```

假设我们发现自己一遍又一遍的在应用的视图中写下类似下面的数据绑定表达式：

```HTML
Name: {{customer.name}} 
<br />
Street: {{customer.street}}
```

一个能促进重用的技术是将数据绑定模版放在一个子视图里（我称其为 **myChildView.html** ）并在父视图里使用 **ng-include** 指令来引用它。这将使得 **myChildView.html** 在整个应用里都可以在需要的地方被重用。

```HTML
<div ng-include="'myChildView.html'"></div>
```

虽然这能搞定事情，但还有另外一个可用的技术是将数据绑定表达式放入一个指令里。要创建一个指令，你首先定位到指令会被放入的目标模块，并调用它的 **directive()** 函数。这个 **directive()** 函数接受指令的名字和一个函数。下面是一个简单的嵌入数据绑定表达式的指令：


```JavaScript
angular.module('directivesModule').directive('mySharedScope', function () {
    return {
        template: 'Name: {{customer.name}}<br /> Street: {{customer.street}}'
    };
});
```

这给我们带来了比使用 **ng-include** 指令加载子视图更多的东西吗？目前还没有。然而，指令能够通过添加一点点代码来做到更多事情，而且它们能够让元素轻易地附上功能。一个将 **mySharedScope** 指令附加到一个 **<div>** 的例子如下所示：

```HTML
<div my-shared-scope></div>
```

当指令运行时，它会基于之前展示的控制器里的数据输出下面的内容：

```
Name: David  
Street: 1234 Anywhere St.
```

你会注意到的第一个情况是 **mySharedScope** 指令在视图里是用 **my-shared-scope** 来引用的。为什么是这个格式？答案是指令名以驼峰语法定义。例如，当你使用 **ngRepeat** 指令时你使用的是连字符版的 **ng-repeat** 。

另一个有趣的情况是它默认从视图继承了 scope 。如果之前展示的控制器（CustomersController）被绑定到视图，那么  **$scope**  的 **customer** 属性就被关联到指令了。这被称为 **shared scope** 并且非常适合你知道许多 parent scope 信息的情况。在你想重用指令的情况下，你不能依赖于知道该 scope 的属性，虽然可能会使用称为 **isolate scope** 的东西。我会下一篇教程里提供更多 isolate scope 的细节。


## 指令属性

在前一个 **mySharedScope** 指令中有一个单独的属性叫做 **template** ，它被定义在从函数返回的对象中。这个属性负责定义模版代码（目前是一个数据绑定表达式）用于生成 HTML 。还有什么其它可用的属性呢？

自定义指令通常会返回一个对象，它负责定义指令所需的属性，例如模版、控制器（如果使用了）、负责 DOM 操作的代码，还有更多。有不同的几种属性可以使用（你可以[在此找到它们的完整列表][8]）。下面看看一些你可能遇到的关键属性，以及一个使用它们的实例：

```JavaScript
angular.module('moduleName')
    .directive('myDirective', function () {
    return {
        restrict: 'EA', //E = element（元素）, A = attribute（属性）, C = class, M = comment         
        scope: {
            //@ reads the attribute value, = provides two-way binding, & works with functions
            //@ 读取属性值， = 提供双向绑定， & 以函数一起工作
            title: '@'         },
        template: '<div>{{ myVal }}</div>',
        templateUrl: 'mytemplate.html',
        controller: controllerFunction, //Embed a custom controller in the directive 在指令中嵌入一个自定义控制器
        link: function ($scope, element, attrs) { } //DOM manipulation DOM 操作
    }
});
```

每个属性的简短说明如下所示：

<table cellspacing="0" cellpadding="2" width="500" border="1"><tbody>
  <tr>
    <td valign="top" width="128"><strong>属性</strong></td>

    <td valign="top" width="372"><strong>描述</strong></td>
  </tr>

  <tr>
    <td valign="top" width="128">restrict</td>

    <td valign="top" width="372">决定一个指令可如何被使用（例如元素、属性、CSS class 或 注释）。</td>
  </tr>

  <tr>
    <td valign="top" width="128">scope</td>

    <td valign="top" width="372">用于创建一个子 scope  或孤立的 scope 。</td>
  </tr>

  <tr>
    <td valign="top" width="128">template</td>

    <td valign="top" width="372">定义指令的输出内容。可以包含 HTML 、数据绑定表达式，甚至是其它指令。</td>
  </tr>

  <tr>
    <td valign="top" width="128">templateUrl</td>

    <td valign="top" width="372">提供指令所用模版的路径。如果模版被定义在 &lt;script&gt; 内，那它可以包含一个 DOM 元素的 id 。</td>
  </tr>

  <tr>
    <td valign="top" width="128">controller</td>

    <td valign="top" width="372">用于定义和指令模版关联的控制器。 </td>
  </tr>

  <tr>
    <td valign="top" width="128">link</td>

    <td valign="top" width="372">用于 DOM 操作任务的函数</td>
  </tr>
</tbody></table>


## 操作 DOM

除了在模版上执行数据绑定操作（我会在将来的文章里讲到更多相关的情况）外，指令同样可用于操作 DOM 。即通过之前提到的 link 函数来做。

link 函数通常接受 3 个参数（虽然某些情况下还可以传递其它参数），包括 scope 、与指令关联的元素，以及目标元素的属性（attribute）。下面是指令如何处理某个元素上 click、mouseenter，以及 mouseleave 事件的例子：

```JavaScript
app.directive('myDomDirective', function () {
    return {
        link: function ($scope, element, attrs) {
            element.bind('click', function () {
                element.html('You clicked me!');
            });
            element.bind('mouseenter', function () {
                element.css('background-color', 'yellow');
            });
            element.bind('mouseleave', function () {
                element.css('background-color', 'white');
            });
        }
    };
});
```

要使用此指令，只需将如下代码添加到视图：

```HTML
<div my-dom-directive>Click Me!</div>
```

当鼠标进入或离开时，背景色会在黄色和白色之间切换（本例使用的是嵌入式样式，但 CSS 类形式的样式一样可以工作）。当目标元素被点击，内部的 HTML 就会变成 “You clicked me!” 。当然了，对于 DOM 的操作，还有非常非常多的事情可以做，这里只是简单地帮助你开始而已。


## 构造 AngularJS 指令代码

虽然 **mySharedScope** 和 **myDomDirective** 指令能很好的工作，但在定义指令和其它 AngularJS 组件时，我会遵循一个特定的模式。下面是一个例子：

```JavaScript
(function () {

    var directive = function () {
        return {

        };
    };

    angular.module('moduleName')
        .directive('directiveName', directive);

}());
```

这些代码是用一个会立即执行的函数包装了所有的东西，以将这些东西拉到全局 scope 之外。然后它在函数内定义了一个指令功能函数并将其分配给  **directive** 变量。最后，调用模块的 **directive()** 函数并传递  **directive** 变量。还有其它几种技术可用于构造代码（看看我关于本主题的[旧文章][9]），但这就是我通常会遵循的模式。

## 总结

在这关于 AngularJS 指令的第一篇文章中，你了解了一些基础知识，并学会了如何创建两个基本的指令。但我们还只是懂了一点儿皮毛而已。在下一篇文章中我会讨论孤立 scope 和一些不同的属性（它们被称为 **局部 scope 属性** ，可用于数据绑定等更多情况上）。

===============

译者注：欢迎转载，但请一定注明出处！ [https://github.com/nixzhu/dev-blog](https://github.com/nixzhu/dev-blog)

欢迎转发此条微博 [http://weibo.com/2076580237/B2Bs62jQG](http://weibo.com/2076580237/B2Bs62jQG) 以分享给更多人。

如果你认为这篇翻译不错，也有闲钱，那你可以用支付宝随便捐助一点，以慰劳译者的幸苦：

![nixzhu的支付宝二维码](https://github.com/nixzhu/dev-blog/raw/master/images/nixzhu_alipay.png)

   [1]: http://weblogs.asp.net/dwahlin/archive/2014/03/24/the-angularjs-jumpstart-video-training-course-has-been-released.aspx
   [2]: http://weblogs.asp.net/blogs/dwahlin/AngularJSCourseLogoYellow.png
   [3]: http://www.angularjs.org
   [4]: https://docs.angularjs.org/api/ng/directive
   [5]: http://weblogs.asp.net/dwahlin/archive/2013/08/15/angularjs-in-60-ish-minutes-the-ebook.aspx
   [6]: http://www.polymer-project.org/
   [7]: http://w3c.github.io/webcomponents/explainer/
   [8]: http://docs.angularjs.org/api/ng/service/$compile
   [9]: http://weblogs.asp.net/dwahlin/archive/2013/12/01/structuring-angularjs-code.aspx
  
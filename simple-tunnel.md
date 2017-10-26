
# SimpleTunnel阅读笔记

[SimpleTunnel](https://developer.apple.com/library/content/samplecode/SimpleTunnel/Introduction/Intro.html)是Apple官方出的关于NetworkExtension框架的demo。我想，阅读过后定有收获。

作者：[@nixzhu](https://twitter.com/nixzhu)

---

首先阅读README。

……

接下来观察Storyboard。

首先，VPN列表显示在`ConfigurationListController`，它通过`reloadManagers()`将所有的VPN配置加载为列表。通过segue，用户可以新增VPN配置、编辑已有VPN配置，最重要的是查看某个VPN的状态。

在`StatusViewController`中，被查看的VPN由`targetManager`表示（在VPN列表里准备segue时赋值）。其中，用户可以enable/disable VPN；在VPN enabled时，用户可以打开或关闭VPN。在`viewWillAppear(_:)`时，除了设置UI外，通过IPC给TunnelProvider（也就是PacketTunnelProvider扩展）发送了一个消息。消息内容并不重要，`NETunnelProviderSession.sendProviderMessage()`的文档说明，此消息意在唤醒PacketTunnelProvider扩展。两个UISwitch的target-action里处理了VPN的enable/disable，以及VPN的开关。其它还有监听VPN状态的通知等，主要是更新UI，不表。

通过搜索`IPC message`，我们可以到达`PacketTunnelProvider`的`handleAppMessage()`方法，它简单地应答了一个消息。值得注意的是，这个IPC通讯的过程用了`simpleTunnelLog()`（调用NSLog）来打印消息，应该有便于调试扩展的功效。

现在，我们来分析一下VPN建立的具体过程。

当用户按下开关，`startVPNTunnel()`被调用，它的文档里说：“此函数使用当前VPN configuration来开始一个VPN tunnel，VPN tunnel connection进程将马上开始，而此函数会立即返回。”

在`PacketTunnelProvider`中，我们可观察到`startTunnel()`方法，其文档说：“此方法被系统调用用于开始一个network tunnel。当Packet Tunnel Provider以nil参数执行completionHandler时，即通知系统它已经准备好处理网络数据。因此，Packet Tunnel Provider需要调用`setTunnelNetworkSettings(_:completionHandler:)`并等待其完成再执行completionHandler。”

具体到`startTunnel()`的代码，它新建了一个`ClientTunnel`，设置其`delegate`为`self`，并让其执行内部的`startTunnel()`，最后把`completionHandler`保存下来备用。

既然如此，我们把目光转向这个真正做事的`ClientTunnel`。

# ClientTunnel

`ClientTunnel`位于`SimpleTunnelServices` target中，其`startTunnel()`首先根据来自`PacketTunnelProvider`（它已被通过参数传递）的配置，创建好`endpoint`，它是一个`NWEndpoint`，包装了`hostname`和`port`。至于缺少信息而创建`NWBonjourServiceEndpoint`类型的`endpoint`就不理会了。

接下来就是最重要的一步，让`PacketTunnelProvider`通过`createTCPConnectionToEndpoint:enableTLS:TLSParameters:delegate:`发起到`endpoint`的TCP链接，并将链接记到`connection`上，且观察其`state`。

那么我们就转到KVO的处理上。

在`observeValue`方法中，当链接状态为`connected`，我们把`remoteAddress`的`hostname`记下来为`remoteHost`；然后用`readNextPacket()`读取下一个包，并让`delegate`（也就是`PacketTunnelProvider`）知道`tunnel`打开了。其它状的处理暂时不理。

让我们转回到`PacketTunnelProvider`实现的`tunnelDidOpen`delegate方法，它利用tunnel建立了一个`ClientTunnelConnection`记为`tunnelConnection`并`open`它。

那我们就再来看看`ClientTunnelConnection`是个什么东西。其`open`方法里，利用`clientTunnel`（也就是初始化时传递的`tunnel`)的`sendMessage`发送了一个字典，这个字典里有identifier、值为open的command以及为值ip的tunnel type。




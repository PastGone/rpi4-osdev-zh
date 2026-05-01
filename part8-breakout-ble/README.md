为 Raspberry Pi 4 编写「裸机」操作系统（第八部分）
===================================================================

[< 返回part7-bluetooth](../part7-bluetooth)

接收蓝牙数据
------------------------
我们已经掌握了广播，正在向世界广播数据。但这只是故事的一半！在这一部分，我们将探索如何从外部来源接收数据。这更加令人兴奋，因为我们可以开始使用其他设备作为遥控器。

事实上，我在这一部分的想法是通过蓝牙接收我MacBook Pro触控板的数据来控制我们在part6中编写的打砖块游戏！很棒，对吧？

构建蓝牙打砖块控制器
----------------------------------------
首先，让我们构建用于从笔记本电脑广播的控制器代码。我们本质上需要在代码中创建一个BLE外设。

既然我们不再是裸机了，我们不需要从头开始构建一切！我使用了[Bleno库](https://github.com/noble/bleno)，它在这类工作中非常流行。它需要一些[Node.js](https://nodejs.org/en/download/)知识，但我开始之前也没有，所以我相信你也会没事的。

安装Node.js后，使用`npm`安装Bleno：`npm install bleno`。因为我在Mac上运行最新版本的Mac OS X（> Catalina），我需要改用[这三个步骤](https://punchthrough.com/how-to-use-node-js-to-speed-up-ble-app-development/)：

* `npm install github:notjosh/bleno#inject-bindings`
* `npm install github:notjosh/bleno-mac`
* `npm install github:sandeepmistry/node-xpc-connection#pull/26/head`

我使用了Bleno仓库中的[echo示例](https://github.com/noble/bleno/tree/master/examples/echo)作为我的基础代码。这个示例实现了一个蓝牙外设，公开了一个服务，该服务：

* 允许连接的设备读取本地存储的字节值
* 允许连接的设备更新本地存储的字节值（它也可以在本地更改！）
* 允许连接的设备订阅以在本地存储的字节值更改时接收更新
* 允许连接的设备取消订阅接收更新

你不会惊讶地知道，我的设计是让我们的Raspberry Pi订阅接收运行在我笔记本电脑上的这个服务的更新。那个本地存储的字节值将在本地更新，以反映当前鼠标光标位置的变化。然后，每当我在MacBook Pro上移动鼠标时，我们的Raspberry Pi就会神奇地收到通知！

你可以在这个part8-breakout-ble的_controller_子目录中看到我对Bleno echo示例所做的修改，以实现这一点。它们归结为使用[iohook](https://github.com/wilix-team/iohook)，我使用`npm install iohook`安装了它。这是有趣的部分（其余只是管道）：

```c
var ioHook = require('iohook');

var buf = Buffer.allocUnsafe(1);
var obuf = Buffer.allocUnsafe(1);
const scrwidth = 1440;
const divisor = scrwidth / 100;

ioHook.on( 'mousemove', event => {
   buf.writeUInt8(Math.round(event.x / divisor), 0);

   if (Buffer.compare(buf, obuf)) {
      e._value = buf;
      if (e._updateValueCallback) e._updateValueCallback(e._value);
      buf.copy(obuf);
   }
});

ioHook.start();
```

在这里，我捕获鼠标光标的x坐标，并将其转换为0（屏幕最左侧）到100（屏幕最右侧）之间的数字。如果它与我们之前看到的值不同，我们更新回调值（我们的Raspberry Pi只需要知道位置何时改变）。随着回调值的更新，所有订阅的设备都将收到通知。

我们有了一个工作的蓝牙游戏控制器，尽管有点简陋！你可以用`node main.js`命令运行它，但如果没有连接的设备，它不会做太多事情。

从Raspberry Pi连接到我们的打砖块控制器
-----------------------------------------------------------
运行时的_main.js_正在公开广播打砖块控制器的可用性，使用与我们的Eddystone信标完全相同的技术。现在我们在Pi上的任务是：

* 开始监听（称为"扫描"）这些广告，以便我们知道echo服务在哪里
* 找到echo服务后连接到它
* 订阅接收其光标位置更新
* 监听这些更新并在收到时采取行动

还记得我提到我们会在part7-bluetooth中讨论`run_search()`函数吗？嗯，这正是它的作用。这次注释掉`run_eddystone()`函数，让`run_search()`运行，同时你的echo服务在笔记本电脑上运行。当你在触控板上移动手指时，你应该会在Pi上看到更新！

与广播不同，`run_search()`将蓝牙控制器置于扫描模式（请参阅part7-bluetooth中的链接了解如何完成此操作）。然后重复调用_kernel.c_中的`bt_search()`，直到它收到一些特定的广告数据——即echo服务的服务ID（十六进制数字0xEC00）的可用性通知，以及其广播名称'echo'。如果它在同一个广告报告中看到两者，它就会假设找到了它要找的东西。记录原始蓝牙设备地址。

我们向该地址发送`connect()`请求（TI文档中的LE Create Connection），现在重复调用`bt_conn()`，直到收到连接成功完成的通知。发生这种情况时，我们得到一个非零的`connection_handle`。从现在开始，我们将使用它来标识往返echo服务的通信。

接下来，我们在_bt.c_的`sendACLsubscribe()`中使用该句柄向服务发送订阅请求。我们告诉它我们有兴趣接收其存储值（或"特征"）的更新。实际上，我做了大量反向工程才得到这段代码。HCI上的ACL数据包没有广泛记录。阅读[这个论坛帖子](https://www.raspberrypi.org/forums/viewtopic.php?t=233140)，看看我为成功所做的事情。Raspbian上的`gatttool`和`hcitool`原来是我的好朋友！

最后，我们重复调用`acl_poll()`来查看是否有任何更新等待。数据以ACL数据包的形式到达，其中标识了连接句柄（值得与我们记录的句柄进行检查，以便我们知道它是给我们的）以及数据长度和操作码。

![ATT handle value notification opcode 1b](images/8-opcode-1b.png)

操作码0x1B代表"ATT handle value notification"（TI文档中的ATT_HandleValueNoti）。这些是我们正在寻找的更新。在part7中，我们只是将更新打印到调试来显示它已收到。

最后一步
-------------
有了这个，将part6和part7代码合并以形成一个通过蓝牙控制的工作打砖块实现是一个很好的练习！毕竟，这正是我最终得到part8代码的方式...如果你卡住了，我的仓库里都有。

祝你好运！下次，我们将看看如何在游戏中添加声音...

[前往part9-sound >](../part9-sound)
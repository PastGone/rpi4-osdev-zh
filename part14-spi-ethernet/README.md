为 Raspberry Pi 4 编写「裸机」操作系统（第十四部分）
====================================================================

[< 返回part13-interrupts](../part13-interrupts)

不到10英镑的裸机以太网
---------------------------------
构建自己的操作系统很令人兴奋，但除非你赋予它与外界通信的能力，否则你的可能性是有限的。事实上，我们简单的蓝牙通信让我们启动并运行了——但如果我们要做任何有意义的事情，我们需要适当的网络连接。

在本教程中，我们将使用RPi4的串行外设接口（SPI）连接到外部以太网控制器（如果你愿意，可以称之为网卡）。

你需要的东西：

* 一个[ENC28J60以太网模块](https://www.amazon.co.uk/dp/B00DB76ZSK)——我花了不到6英镑，每一分钱都值得（注意：代码仅在这个确切型号上测试过）
* 一些[母对母跳线电缆](https://www.amazon.co.uk/dp/B072LN3HLG)——花了我不到2.50英镑
* 一根以太网电缆连接到你的互联网路由器

连接ENC28J60以太网模块
------------------------------------------
我遵循了[这里](https://www.instructables.com/Super-Cheap-Ethernet-for-the-Raspberry-Pi/)非常有帮助的说明，将ENC28J60连接到RPi4的SPI0接口。

我们暂时不连接中断线，所以只需要连接六根跳线（我建议了颜色）：

| Pi引脚 | Pi GPIO     | 跳线颜色 | ENC28J60引脚 |
| ------ | ----------- | -------- | ------------ |
| Pin 17 | +3V3电源    | 红色     | VCC          |
| Pin 19 | GPIO10/MOSI | 绿色     | SI           |
| Pin 20 | GND         | 黑色     | GND          |
| Pin 21 | GPIO09/MISO | 黄色     | SO           |
| Pin 23 | GPIO11/SCLK | 蓝色     | SCK          |
| Pin 24 | GPIO08/CE0  | 绿色     | CS           |

![GPIO位置](../part3-helloworld/images/3-helloworld-pinloc.png)

这是我的RPi4正确连接的（不是很有用的）照片：

![ENC28J60连接](images/14-spi-ethernet-photo.jpg)

SPI库
---------------
让我们先看看如何实现SPI通信。

我不打算写一篇长篇大论来介绍SPI的工作原理以及为什么我们需要它，因为[其他地方有很好的文档](https://learn.sparkfun.com/tutorials/serial-peripheral-interface-spi/)。建议进行背景阅读，但如果你只想让某些东西工作，这不是必需的。

查看_lib/spi.c_。它使用了_lib/io.c_中的一些现有函数，你会从之前的教程中记住这些函数。事实上，我在_include/io.h_头文件中添加了两个函数，以便我们可以从SPI库中调用它们：

```c
void gpio_setPinOutputBool(unsigned int pin_number, unsigned int onOrOff);
void gpio_initOutputPinWithPullNone(unsigned int pin_number);
```

具体来说，`spi_init()`将GPIO 7、9、10和11设置为使用ALT0功能。参考[BCM2711 ARM外设文档](https://datasheets.raspberrypi.com/bcm2711/bcm2711-peripherals.pdf)第77页，你会看到这将SPI0映射到GPIO接头。GPIO 8被映射为输出引脚，因为我们将使用它向ENC28J60发出信号，表示我们想要通信。事实上，`spi_chip_select()`函数接受一个true/false（布尔）参数，用于设置或清除此引脚。

查看第136页的SPI0寄存器映射，我们在`REGS_SPI0`结构中看到了这一点。这使我们能够方便地访问SPI0外设的内存映射寄存器。

然后，我们的`spi_send_recv()`函数为某些通信做好准备：

* 将DLEN寄存器设置为要传输的字节数（我们传递给函数的长度）
* 清除RX和TX FIFO
* 设置传输活动（TA）标志

然后，当有数据要写或有数据要读时（并且我们还没有写/读超过我们要求的字节数），我们使用传入的缓冲区写入/读取FIFO。一旦我们认为完成了，我们等待直到SPI接口同意，即CS寄存器中的DONE标志被设置。如果有多余的字节要读取，我们就把它们扔掉（嗯，现在先把它们转储到屏幕上，因为这不应该发生）。

最后，为了绝对确定，我们清除TA标志。

然后我设置了两个方便的函数——`spi_send()`和`spi_recv()`——它们调用`spi_send_recv()`，主要是为了使未来的代码更易读。

ENC28J60驱动程序
--------------------
现在让我们查看_net/_子目录。

_enc28j60.c_和_enc28j60.h_共同构成了ENC28J60以太网模块的驱动代码。虽然我们可以花几个月的时间根据[模块的数据表](http://ww1.microchip.com/downloads/en/devicedoc/39662c.pdf)编写自己的驱动程序，但我选择利用别人的辛勤工作。能够毫不费力地将别人的好代码带入我自己的操作系统，感觉像是一种胜利！然而，我确保我理解代码在每个环节都在做什么（可选！）。

感谢[这个Github仓库](https://github.com/wolfgangr/enc28j60)为我节省了数月的工作。我对代码做了一些非常小的改动，但这里没有什么值得记录的。如果你想看看我需要做多少改动，克隆仓库并充分利用`diff`命令。

我需要做的是在驱动程序和RPi4硬件之间编写一些桥接代码。本质上，我正在谈论将我们的SPI库连接到驱动程序——这就是_encspi.c_的全部原因。

它定义了驱动程序需要的四个函数（在_enc28j60.h_文件中有很好的文档记录）：

```c
void ENC_SPI_Select(unsigned char truefalse) {
    spi_chip_select(!truefalse); // 如果为true，选择0（ENC），如果为false，选择1（即取消选择ENC）
}

void ENC_SPI_SendBuf(unsigned char *master2slave, unsigned char *slave2master, unsigned short bufferSize) {
    spi_chip_select(0);
    spi_send_recv(master2slave, slave2master, bufferSize);
    spi_chip_select(1); // 取消选择ENC
}

void ENC_SPI_Send(unsigned char command) {
    spi_chip_select(0);
    spi_send(&command, 1);
    spi_chip_select(1); // 取消选择ENC
}

void ENC_SPI_SendWithoutSelection(unsigned char command) {
    spi_send(&command, 1);
}
```

也许最令人困惑的方面是芯片选择。通过一些反复试验，我发现当GPIO08为低电平时，设备被选中，当它为高电平时，设备被取消选中。事实证明这是因为ENC28J60上的芯片选择引脚是低电平有效，并且板上没有反相器（可能是为了节省成本），所以GPIO08必须为低电平才能启用IC。

更多计时器功能
-------------------------
我们的ENC28J60驱动程序唯一需要的其他东西是访问几个定义良好的计时器函数：

* `HAL_GetTick()` - 返回自启动以来的当前计时器滴答数
* `HAL_Delay()` - 延迟指定的毫秒数

这些在_kernel/kernel.c_中快速实现，在_part13-interrupts_之后并不是太大的扩展：

```c
unsigned long HAL_GetTick(void) {
    unsigned int hi = REGS_TIMER->counter_hi;
    unsigned int lo = REGS_TIMER->counter_lo;

    // 再次检查hi值在设置后是否改变...
    if (hi != REGS_TIMER->counter_hi) {
        hi = REGS_TIMER->counter_hi;
        lo = REGS_TIMER->counter_lo;
    }

    return ((unsigned long)hi << 32) | lo;
}

void HAL_Delay(unsigned int ms) {
    unsigned long start = HAL_GetTick();

    while(HAL_GetTick() < start + (ms * 1000));
}
```

让我们连接！
--------------
所以我们有一个工作的驱动程序，它通过_net/encspi.c_和_kernel/kernel.c_中的一些计时器函数与我们的硬件接口。现在怎么办？

我们内核的网络演示的设计目标是：

1. 证明我们可以与硬件通信
2. 成功启动网络
3. 证明我们可以连接到网络上的其他设备并获得响应

我对如何实现这些目标的建议是：

1. 证明我们可以检测网络链路是否已在物理层面建立（CAT5电缆已插入并连接到工作的交换机）
2. 依靠ENC28J60驱动程序告诉我们启动成功
3. 手工制作并发送[ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol)请求，并等待来自我的互联网路由器的ARP响应（设备从一无所知开始在网络上"找到彼此"的传统方式）

查看_kernel/arp.c_。首先，我们创建一个句柄来引用我们的驱动程序实例`ENC_HandleTypeDef handle`。然后我们在`init_network()`中初始化这个结构：

```c
handle.Init.DuplexMode = ETH_MODE_HALFDUPLEX;
handle.Init.MACAddr = myMAC;
handle.Init.ChecksumMode = ETH_CHECKSUM_BY_HARDWARE;
handle.Init.InterruptEnableBits = EIE_LINKIE | EIE_PKTIE;
```

这将模块启动为半双工模式（不能同时发送和接收），设置其MAC地址（我的最爱：`C0:FF:EE:C0:FF:EE`），告诉硬件添加自己的数据包校验和（我们不想在软件中创建它们），并启用"链路断开/连接"和"收到数据包"的中断消息。

然后我们调用驱动程序例程`ENC_Start(&handle)`并检查它返回true（这满足设计要求2——驱动程序告诉我们启动正确）。我们继续使用`ENC_SetMacAddr(&handle)`设置MAC地址。

这行等待物理网络链路建立（满足设计要求1）：

```c
while (!(handle.LinkStatus & PHSTAT2_LSTAT)) ENC_IRQHandler(&handle);
```

驱动程序的`ENC_IRQHandler(&handle)`例程通常会在引发中断时被调用，以刷新驱动程序状态标志。因为我们没有连接中断线，为了保持简单，我们现在只是在软件中轮询。当我们看到`handle.LinkStatus`标志设置了`PHSTAT2_LSTAT`位时，我们知道链路已连接（在模块的数据表第24页有文档记录）。

在我们完成之前，我们必须重新启用以太网中断（`ENC_IRQHandler()`禁用它们，但不重新启用它们——这是我通过阅读代码发现的）。

发送/接收ARP
------------------------
要在以太网上传输，我们需要正确格式化数据包。ENC28J60处理物理层（包括我们要求的校验和），所以我们只需要关注数据链路层——由头部和有效载荷组成。

头部（我们的`EtherNetII`结构体）只是目标和源MAC地址，以及一个[16位数据包类型](https://en.wikipedia.org/wiki/EtherType)。例如，ARP数据包的类型为`0x0806`。你会注意到在我们的`#define ARPPACKET`中，我们交换了两个字节。这是因为大端序是网络协议中的主要顺序，而RPi4是小端架构（可能需要一些阅读！）。我们必须全面这样做。

有效载荷是在`ARP`结构体中定义的完整[ARP数据包](https://en.wikipedia.org/wiki/Address_Resolution_Protocol)。`SendArpPacket()`函数在结构体中设置我们需要的数据（在代码注释中有文档记录），然后使用驱动程序调用来传输数据包：

```c
// 发送数据包

if (ENC_RestoreTXBuffer(&handle, sizeof(ARP)) == 0) {
   debugstr("Sending ARP request.");
   debugcrlf();

   ENC_WriteBuffer((unsigned char *)&arpPacket, sizeof(ARP));
   handle.transmitLength = sizeof(ARP);

   ENC_Transmit(&handle);
}
```

`ENC_RestoreTXBuffer()`只是准备传输缓冲区，如果成功则返回0。`ENC_WriteBuffer()`通过SPI将数据包发送到ENC28J60。然后我们在驱动程序状态标志中设置传输缓冲区长度，并调用`ENC_Transmit()`告诉ENC将数据包发送到网络上。

你会看到`arp_test()`函数以这种方式发送我们的第一个ARP。我们告诉它路由器的IP（在我的情况下是`192.168.0.1`），但我们不知道它的MAC地址——这正是我们想要找出的。一旦发送了ARP，`arp_test()`然后等待收到的以太网数据包，检查它们是否是发给我们的，如果它们来自路由器的IP地址（因此很可能是对我们请求的ARP响应），我们打印出路由器的MAC地址。

这满足了设计要求3，因此我们完成了！我们需要做的就是确保_kernel/kernel.c_以正确的顺序调用我们的网络例程。我选择在核心3上执行此操作，并从_part13-interrupts_结束的地方做一些易于遵循的更改。基本上这些是我们需要的所有调用：

```c
spi_init();
init_network();
arp_test();
```

_想象一下，当我看到路由器的（正确的！）MAC地址出现在屏幕上时，我有多高兴——这是生命的迹象，证明我的操作系统实际上正在联网！_

![ARP响应](images/14-spi-ethernet-arp.jpg)

[前往part15-tcpip-webserver >](../part15-tcpip-webserver)
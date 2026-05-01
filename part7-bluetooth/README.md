为 Raspberry Pi 4 编写「裸机」操作系统（第七部分）
===================================================================

[< 返回part6-breakout](../part6-breakout)

启动蓝牙
--------------------

仅通过UART连接的笔记本电脑控制RPi4并不是很有趣。我们的打砖块游戏值得一个更好的控制器——理想情况下是无线控制器。

在这一部分中，我们设置第二个UART来与RPi4的板载蓝牙调制解调器通信。蓝牙并不简单，但它至少比USB简单，这就是我选择追求它的原因。

Broadcom固件
---------------------

蓝牙调制解调器是一个Broadcom芯片（BCM43455），在对我们有用之前，它需要加载专有软件。我从[Bluez仓库](https://github.com/RPi-Distro/bluez-firmware/tree/master/broadcom)获取了他们的_BCM4345C0.hcd_文件。

由于我们还没有任何文件系统，我们无法在运行时加载它，所以我们需要将它构建到我们的内核中。我们可以使用`objcopy`来构建一个我们可以链接的_.o_文件。我们在_Makefile_中添加这些行：

```c
BCM4345C0.o : BCM4345C0.hcd
	$(GCCPATH)/aarch64-none-elf-objcopy -I binary -O elf64-littleaarch64 -B aarch64 $< $@
```

我们还需要修改我们的`kernel8.img`依赖项以包含我们新的_.o_文件：

```c
kernel8.img: boot.o $(OFILES) BCM4345C0.o
	$(GCCPATH)/aarch64-none-elf-ld -nostdlib -nostartfiles boot.o $(OFILES) BCM4345C0.o -T link.ld -o kernel8.elf
	$(GCCPATH)/aarch64-none-elf-objcopy -O binary kernel8.elf kernel8.img
```

如果你现在构建内核，你会看到新的映像大得多——因为它包含固件字节。你可以运行`objdump -x kernel8.elf`，你还会在其中看到一些新符号：

* _binary_BCM4345C0_hcd_start
* _binary_BCM4345C0_hcd_size
* _binary_BCM4345C0_hcd_end

我们稍后将使用这些符号从我们的C代码中引用固件。

设置UART
-------------------

看我们新的_bt.c_。我们现在专注于`// UART0`部分。许多技术你应该很熟悉，因为这个硬件也使用MMIO技术，我们正在实现许多与我们使串行调试工作相同的功能。

为了在进行串行调试的同时使用蓝牙，我们需要重新映射一些GPIO引脚。引脚30、31、32和33都需要采用它们的_备用功能3_，以便我们访问CTS0、RTS0、TXD0和RXD0。你可以在[BCM2711 ARM外设文档](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2711/rpi_DATA_2711_1p0.pdf)的第5.3节中阅读所有相关内容。

我们已经在_io.c_中有一个函数来做这件事，所以我们将`gpio_useAsAlt3`的函数定义添加到_io.h_中，以便从这里访问它。在`bt_init()`中，我们继续重新映射GPIO引脚，然后刷新接收缓冲区以确保安全。接下来的MMIO写入设置为115200波特率和[8-N-1通信](https://en.wikipedia.org/wiki/8-N-1)——我们知道蓝牙调制解调器可以处理这个。也许最重要的一行是：

```c
mmio_write(ARM_UART0_CR, 0xB01);
```

它启用UART（位0开启），启用TX和RX（位8和9开启），并将RTS0驱动为低电平（位11开启）——这非常重要，因为蓝牙调制解调器在看到这一点之前会保持无响应状态。**我花了很多时间才弄明白这一点。**

我们现在应该能够通过我们新的UART与蓝牙调制解调器通信。

与蓝牙调制解调器通信
------------------------------

蓝牙规范非常庞大，实现完整的驱动程序需要很长时间。我现在满足于"生命证明"。不过我们还有一段路要走...

我们使用**HCI命令**与蓝牙调制解调器通信。我喜欢阅读[TI HCI文档](http://software-dl.ti.com/simplelink/esd/simplelink_cc13x2_sdk/1.60.00.29_new/exports/docs/ble5stack/vendor_specific_guide/BLE_Vendor_Specific_HCI_Guide/hci_interface.html)作为介绍。

`bt_reset()`只是调用`hci_Command`，后者又调用`hci_CommandBytes`，将字节写入UART，告诉蓝牙芯片重置并等待固件。这是一个供应商特定的调用，所以你不会在任何地方找到它的文档。我使用Raspberry Pi Linux发行版中的以下文件反向工程了这些调用：

* https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/bluetooth/btbcm.c
* https://github.com/raspberrypi/linux/blob/rpi-5.10.y/drivers/bluetooth/hci_bcm.c
* https://github.com/raspberrypi/linux/blob/rpi-5.10.y/include/net/bluetooth/hci.h

`hci_CommandBytes`然后在成功返回之前等待一个非常特定的响应——"命令完成"响应。

加载固件
--------------------

现在设备正在等待。我们需要向它发送我们包含在内核中的固件字节：

```c
void bt_loadfirmware()
{
    volatile unsigned char empty[] = {};
    if (hciCommand(OGF_VENDOR, COMMAND_LOAD_FIRMWARE, empty, 0)) uart_writeText("loadFirmware() failed\n");

    extern unsigned char _binary_BCM4345C0_hcd_start[];
    extern unsigned char _binary_BCM4345C0_hcd_size[];

    unsigned int c=0;
    unsigned int size = (long)&_binary_BCM4345C0_hcd_size;

    unsigned char opcodebytes[2];
    unsigned char length;
    unsigned char *data = &(_binary_BCM4345C0_hcd_start[0]);

    while (c < size) {
        opcodebytes[0] = *data;
        opcodebytes[1] = *(data+1);
        length =         *(data+2);
        data += 3;

        if (hciCommandBytes(opcodebytes, data, length)) {
           uart_writeText("Firmware data load failed\n");
           break;
        }

        data += length;
        c += 3 + length;
    }

    wait_msec(0x100000);
}
```

首先，我们发送一个命令告诉芯片我们即将发送固件。你会看到我们然后引用我们的新符号，它们指向我们的固件字节。我们现在知道固件的大小，所以我们遍历它。

固件只是一系列遵循以下格式的HCI命令：

* 2字节的操作码
* 1字节告诉我们后续数据的长度
* _length_字节的数据

我们在进行过程中检查每个HCI命令是否成功，等待一秒钟然后返回。如果它没有错误地运行，那么我们已经加载了固件，我们准备好开始一些蓝牙通信。

然后我选择实现`bd_setbaud()`和`bt_setbdaddr()`。这设置了蓝牙调制解调器将使用的速度（就像我们在UART示例中做的那样）以及它的唯一[蓝牙设备地址](https://macaddresschanger.com/what-is-bluetooth-address-BD_ADDR)。

构建Eddystone信标
----------------------------

也许最简单的蓝牙设备是"信标"。它只是公开广播少量数据，以便任何经过的接收器都可以查看数据。一个典型的用例是为基于位置的营销目的广播一个Web URL。

谷歌定义了[Eddystone格式](https://en.wikipedia.org/wiki/Eddystone_(Google))，它被广泛采用。我们将实现这个作为我们的例子。这是我们需要实现的：

* 设置LE事件掩码，确保蓝牙控制器被所有传入流量中断
* 设置广告参数
* 设置广告数据
* 启用广告

[markfirmware的代码](https://github.com/markfirmware/zig-bare-metal-raspberry-pi/blob/master/src/ble.zig)极大地促进了我的学习。我不是简单地盲目复制代码到我的_bt.c_中，而是将其与重量级的[蓝牙规范](https://www.bluetooth.com/specifications/specs/core-specification-5-2/)的相关部分一起阅读。如果你有兴趣真正理解这里发生的事情，我建议你也这样做。

在构建广告数据时，我还参考了[PiMyLifeUp关于Eddystone的文章](https://pimylifeup.com/raspberry-pi-eddystone-beacon/)。

要测试代码，确保在_kernel.c_中取消注释`run_eddystone()`而不是`run_search()`（我们将在part8-breakout-ble中更详细地讨论`run_search()`）。

构建并运行后，我使用[eBeacon iPhone应用](https://apps.apple.com/us/app/ebeacon-ble-scanner/id730279939)检查我的Eddystone信标是否正在广播。下面的截图显示我的URL如预期地自豪地被广告：

![RPi4上工作的Eddystone信标](images/7-eddystone-beacon.png)

[前往part8-breakout-ble >](../part8-breakout-ble)
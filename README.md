为 Raspberry Pi 4 编写一个「裸机」操作系统
==========================================================

简介
------------

作为一名科技公司CEO，我已经不再写代码了。最近我意识到我是多么怀念写代码的时光。

由于新冠疫情导致全国「封锁」（也省去了我平时的通勤时间），我发现自己每天有了更多的时间。我利用这段时间实现了一个儿时的梦想——编写一个能在商用硬件上运行的**裸机**操作系统。

什么是裸机？
--------------------------

当我们购买电脑、平板或智能手机时，它通常预装了一些基础软件。你可能很熟悉在首次开机（或**启动**）时看到微软Windows、Mac OS、iOS、Android甚至Linux的启动画面。这些都是操作系统——设计用来让电脑芯片开箱即用的软件。它们帮助我们通过绘制屏幕、处理键盘和鼠标等设备的输入、通过网络硬件连接互联网、播放声音等等来与机器交互。

全世界的软件开发人员在这些操作系统之上构建应用程序（apps）。这些应用通过操作系统（**OS**）与硬件通信，因此这些复杂的代码不必重复编写。因此，软件开发人员可以不必了解太多硬件知识！正是操作系统做了那些繁重的工作，让我们能够使用像Facebook、Instagram、WhatsApp、TikTok等应用。

可以说，_没有操作系统，计算机无法做任何有用的事情_。它们只是坐在那里等待被告诉该做什么。那么，为什么只有微软、苹果和谷歌这样的软件巨头才能在大多数电脑开机时告诉它们该做什么呢？为什么我们不能？实际上我们可以，这就是裸机编程。

硬件选择
------------------

如果你对指挥计算机做什么感到兴奋，那么你需要对硬件感兴趣。执行你指令的计算机芯片叫做**CPU**（中央处理器）——它是每台计算机设备的心脏。多年来许多公司都设计了这样的CPU，但Intel和Arm两家公司的产品被最广泛采用。它们各有优缺点。如果你拥有一部智能手机，它很可能运行在Arm设计的芯片上。如果你拥有一台运行微软Windows的笔记本电脑，它很可能运行在Intel芯片上。最终你需要了解这两种**架构**，但在这个项目中我选择了Arm设备。

新款的[Raspberry Pi 4 Model B](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/)是一款低成本电脑，运行在1.5 GHz的64位四核Arm Cortex-A72处理器上。它是一款全球数百万人使用的设备，因此为它编写裸机代码非常令人兴奋。想象一下有一天别人可能会使用你的操作系统！**RPi4**还有一些有用的附加硬件，这将帮助我们完成项目。

硬件前提条件
----------------------

开始编写操作系统需要一些硬件：

* 一台RPi4，配有专用电源和HDMI线
* 通过HDMI连接到RPi4的显示器/电视
* 一张用于启动RPi4的micro-SD卡
* 一台编写代码的计算机，例如Windows/Mac笔记本电脑（**开发机器**）

你需要确保可以使用开发机器向micro-SD卡写入数据。对我来说，这意味着购买一个SD卡适配器，因为micro-SD卡对于我笔记本电脑的卡槽来说太小了。你可能也需要同样的东西，如果你的笔记本电脑没有内置读卡器，甚至可能需要一个USB SD卡读卡器。

其他非常有用且必不可少的硬件：

* 一把眉毛镊子（我从我妻子那里借来的！）——用于将micro-SD卡插入/从RPi4上的小插槽中取出
* 一根[USB转串口TTL线](https://www.amazon.co.uk/gp/product/B01N4X3BJB/ref=ppx_yo_dt_b_asin_title_o00_s00?ie=UTF8&psc=1)——在能够向屏幕写入信息之前很久，就可以用它来查看操作系统正在做什么

软件前提条件
----------------------

如果你无法让别人的操作系统运行，你可能无法编写自己的操作系统。因此我首先用Raspbian（Raspberry Pi推荐的操作系统）刷写SD卡。我使用了他们网站上提供的非常简洁的[Imager工具](https://www.raspberrypi.org/downloads/)来完成这项工作。

连接你的RPi4并确保它能启动进入Raspbian。网上有很多资源可以帮助你完成/排除故障。启动Raspbian将测试你的硬件设置是否正常工作。注意：因为我将RPi4连接到我的（不太好的）电视上，我需要在SD卡上的_config.txt_文件中进行编辑（将`hdmi_safe`参数设置为1）以确保我能看到屏幕。否则，屏幕就是黑的。如果你仍然遇到问题，可以查看Raspberry Pi网站上的其他_config.txt_视频选项[这里](https://www.raspberrypi.com/documentation/computers/config_txt.html#video-options)。

在Raspbian运行之前不要继续！

---

RPi4运行在Arm Cortex-A72处理器上。你的开发机器可能运行在Intel处理器上。因此你需要一些软件来帮助你构建能在不同架构上运行的代码。这叫做**交叉编译器**。

使用Arm的Linux编译器
------------------------------

下载并解压[Arm的gcc编译器](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads)。由于一些我在这里不深入的原因，你需要使用"AArch64 ELF裸机目标"版本。由于我在Windows 10上使用WSL模拟Ubuntu，我下载了x86_64 Linux托管的交叉编译器。

我还建议安装GNU make——你很快就会需要它。因为我使用WSL，对我来说只需输入`sudo apt install make`并输入密码即可。

在Mac OS X上使用clang（Apple Silicon或Intel）
------------------------------------------------

从App Store下载并安装XCode。这将免费为你提供大量开发工具，包括`make`。

我建议使用[Homebrew](https://docs.brew.sh/Installation)来安装LLVM。对我来说，Homebrew已经安装好了，所以只需输入`brew install llvm`即可。

LLVM将为你提供在Mac上为Raspberry Pi裸机开发所需的一切。它甚至可以在我的Apple Silicon M1 MacBook Pro上运行，该电脑运行的是ARM处理器而不是Intel处理器。

直接在Raspberry Pi 4上构建
-------------------------------------

你可以在[这里](./RPI-BUILD.md)阅读更多关于如何在Pi上构建的信息。

_现在你准备好开始编写操作系统了！_

[前往part1-bootstrapping >](./part1-bootstrapping/)

致谢
----------------
这里的代码并非都是我原创的，而是从一些出色的贡献者那里收集和启发而来。

感谢：

* Zoltan Baldaszti的"裸机Raspberry Pi 3教程" [(github)](https://github.com/bztsrc/raspi3-tutorial/)

如果你也想在这里被致谢，请联系我！
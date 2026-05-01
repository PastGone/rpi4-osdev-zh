为 Raspberry Pi 4 编写「裸机」操作系统
==========================================================

在RPi4上构建
---------------------------

无需额外的构建设备，就可以在Raspberry Pi上遵循本教程（但不是超级简单）。

也许最简单的方法是首先重新镜像你的Pi以使用64位Raspberry Pi OS（Beta版），然后使用预构建的交叉编译器：

* 从[64位镜像列表](https://downloads.raspberrypi.org/raspios_arm64/images/)下载压缩的_.img_镜像文件，选择最新的更新
* 解压它并使用[Raspberry Pi Imager](https://www.raspberrypi.org/software/)将其写入你的SD卡，从选项中选择"使用自定义"并指向你下载的_.img_文件
* 启动Pi并按照设置向导操作，确保你有一个可用的互联网连接
* 为了好运，运行`sudo apt update`

然后你需要从Arm网站下载一个交叉编译器。

你正在寻找当前的[AArch64 ELF裸机目标（aarch64-none-elf）](https://developer.arm.com/-/media/Files/downloads/gnu-a/10.2-2020.11/binrel/gcc-arm-10.2-2020.11-aarch64-aarch64-none-elf.tar.xz)。如果这个链接以某种方式中断了，你可以使用谷歌搜索"Arm GNU-A linux hosted cross compilers"。

然后使用`tar -xf <filename>`解压归档文件。你最终会得到一个_gcc_目录（尽管名称稍长），它本身包含一个_bin_子目录，在那里你会找到_gcc_可执行文件（同样——名称较长！）。记住这个路径。

注意：你可以避免重新镜像Pi，而是[自己构建交叉编译器](https://wiki.osdev.org/GCC_Cross-Compiler)。

现在让我们构建一些东西：

* 使用`git`克隆此仓库：`git clone https://github.com/babbleberry/rpi4-osdev.git`
* 决定要构建哪个部分——我喜欢用_part5-framebuffer_测试（它是可视化的，所以你会知道它什么时候工作！）
* 将_Makefile.gcc_复制到_Makefile_
* 编辑_Makefile_并确保`GCCPATH`变量指向你的交叉编译器所在的_bin_子目录
* 在命令行输入`make`，它应该能无错误地构建

如果你想随后用它启动，你需要将_kernel8.img_文件复制到一个准备好的SD卡上，如教程所述。为了测试这个过程，我做了以下操作（注意：除非你备份旧文件以便以后可以移回，否则它会破坏你的操作系统安装）：

* `sudo cp kernel8.img /boot`
* 然后编辑_/boot/config.txt_使其只包含这些行（无论如何对于_part5-framebuffer_，否则请完整阅读教程以了解其他部分的任何必要配置更改...）：

```c
hdmi_group=1
hdmi_mode=16
core_freq_min=500
```

重新启动，你应该会看到_part5-framebuffer_演示启动！

[前往part1-bootstrapping >](./part1-bootstrapping/)
为 Raspberry Pi 4 编写「裸机」操作系统（第二部分）
===================================================================

[< 返回part1-bootstrapping](../part1-bootstrapping/)

制作Makefile
-----------------

我现在可以告诉你构建这个非常简单的内核所需的一个接一个的命令，但让我们尝试为未来做一点准备。我预计我们的内核会变得更复杂，需要构建多个C文件。因此制作一个**Makefile**是有意义的。Makefile是用（另一种）语言编写的，它为我们自动化构建过程。

如果你在Linux上使用Arm gcc，请将以下内容保存为_Makefile_（在仓库中为_Makefile.gcc_）：

```
CFILES = $(wildcard *.c)
OFILES = $(CFILES:.c=.o)
GCCFLAGS = -Wall -O2 -ffreestanding -nostdinc -nostdlib -nostartfiles
GCCPATH = ../../gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf/bin

all: clean kernel8.img

boot.o: boot.S
	$(GCCPATH)/aarch64-none-elf-gcc $(GCCFLAGS) -c boot.S -o boot.o

%.o: %.c
	$(GCCPATH)/aarch64-none-elf-gcc $(GCCFLAGS) -c $< -o $@

kernel8.img: boot.o $(OFILES)
	$(GCCPATH)/aarch64-none-elf-ld -nostdlib boot.o $(OFILES) -T link.ld -o kernel8.elf
	$(GCCPATH)/aarch64-none-elf-objcopy -O binary kernel8.elf kernel8.img

clean:
	/bin/rm kernel8.elf *.o *.img > /dev/null 2> /dev/null || true
```

* CFILES是当前目录中已存在的_.c_文件列表（我们的输入）
* OFILES是相同的列表，但将每个文件名中的_.c_替换为_.o_——这些将是我们的**目标文件**，包含二进制代码，它们将由编译器生成（我们的输出）
* GCCFLAGS是一组参数，告诉编译器我们正在为裸机构建，因此它不能依赖任何标准库（它通常用来实现简单函数）——在裸机上没有免费的东西！
* GCCPATH是我们编译器二进制文件的路径（你之前下载的Arm工具的解压位置）

然后是一个目标列表，依赖项列在冒号后面。每个目标下面的缩进命令将被执行以构建该目标。希望很容易看出，要构建_boot.o_，我们依赖于源代码文件_boot.S_的存在。然后我们使用正确的标志运行编译器，将_boot.S_作为输入并生成_boot.o_。

`%`是Makefile中的匹配通配符。因此，当我读取下一个目标时，我看到要构建任何其他以_.o_结尾的文件，我们需要其同名的_.c_文件。下面的命令列表在执行时，`$<`将被_.c_文件名替换，`$@`将被_.o_文件名替换。

继续，要构建_kernel8.img_，我们必须首先构建_boot.o_以及OFILES列表中的所有其他_.o_文件。如果已经构建，我们运行`ld`链接器，使用我们的链接脚本_link.ld_将_boot.o_与其他目标文件连接起来，以定义我们创建的_kernel8.elf_映像的布局。遗憾的是，ELF格式设计为由另一个操作系统运行，因此对于裸机目标，我们使用`objcopy`从ELF文件中提取正确的部分到_kernel8.img_中。这就是我们最终将用来启动RPi4的内核映像。

我希望"clean"和"all"目标现在已经不言自明了。

其他平台上的Makefile
----------------------------

如果你在Mac OS X上使用clang，仓库中已命名为_Makefile_的文件将是你需要的。当然，确保LLVMPATH设置正确。它看起来与Arm gcc的版本没有太大不同，所以上面的解释大部分适用。

同样，如果你在Windows上本地使用Arm gcc，part8-breakout-ble有一个_Makefile.gcc.windows_作为示例。

构建
--------

现在我们已经有了_Makefile_，我们只需输入`make`来构建我们的内核映像。由于"all"是我们Makefile中列出的第一个目标，`make`将构建它，除非你另行指定。构建"all"时，它将首先清理任何旧的构建，然后为我们构建一个新的_kernel8.img_。

将内核映像复制到SD卡
---------------------------------------

希望你已经有一张带有工作Raspbian映像的micro-SD卡。要启动我们的内核而不是Raspbian，我们需要用我们自己的内核映像替换任何现有的内核映像，同时注意保持其余目录结构完整。

在你的开发机器上，挂载SD卡并删除所有以_kernel_开头的文件。更谨慎的方法可能是简单地将这些文件从SD卡移动到本地硬盘上的备份文件夹中。如果需要，你可以轻松恢复这些文件。

现在我们将_kernel8.img_复制到SD卡上。这个名字很有意义，它向RPi4发出信号，表明我们希望它以64位模式启动。我们也可以通过在[config.txt](https://www.raspberrypi.org/documentation/configuration/config-txt/boot.md)中将`arm_64bit`设置为非零值来强制这一点。将我们的操作系统启动到64位模式意味着我们可以利用RPi4可用的更大内存容量。

启动
-------

从开发机器安全卸载SD卡，将其放回RPi4中并通电。

_你刚刚启动了自己的操作系统！_

尽管听起来很令人兴奋，但在RPi4自己的"彩虹启动画面"之后，你很可能只会看到一个空的黑屏。然而，我们不应该感到惊讶：我们还没有要求它做任何事情，除了在无限循环中旋转。

不过基础已经奠定，我们现在可以开始做一些令人兴奋的事情了。**恭喜你走到这一步！**

[前往part3-helloworld >](../part3-helloworld)
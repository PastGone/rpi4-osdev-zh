为 Raspberry Pi 4 编写「裸机」操作系统（第十三部分）
====================================================================

[< 返回part12-wgt](../part12-wgt)

什么是中断？
--------------------
如果你花时间查看这些教程中的蓝牙代码，你会注意到我们总是在"轮询"更新。事实上，在_part11-breakout-smp_中，我们占用了整个核心只是为了等待事情发生。这显然不是CPU时间的最佳利用。幸运的是，世界多年前就用_中断_为我们解决了这个问题。

理想情况下，我们希望告诉一个硬件去做某事，当工作完成时简单地通知我们，这样我们就可以在这段时间里继续我们的生活。这些通知被称为_中断_，因为它们中断正常的程序执行并强制CPU立即运行_中断处理程序_。

最简单的中断设备
-----------------------------------
一个有用的内置硬件是系统计时器，可以对其进行编程以定期中断，例如每秒一次。如果你想在单个核心上调度多个进程运行，例如使用时间切片原则，你将需要这个。

然而，目前我们只是要学习如何编程计时器并响应其中断。

代码库
------------
让我快速解释一下你在_part13-interrupts_代码中看到的内容：

* _boot/_ : 来自_part12-wgt_的相同_boot_代码目录
* _include/_ : 直接从_part11-multicore_复制的一些有用的头文件
* _lib/_ : 直接从_part11-multicore_复制的一些有用的库
* _kernel/_ : 本教程中我们需要关注的唯一新代码

请注意：我也做了一些工作来整理_Makefile_并尊重这个目录结构，但没什么值得大书特书的！

新代码
------------
你会从_part10-multicore_中认出很多_kernel.c_，除了现在我们只使用核心0和1，并且另外利用两个计时器中断来显示四个进度条。所以，`main()`例程启动核心1，设置计时器，然后最后启动核心0的工作负载。

使用这些调用设置计时器：

```c
irq_init_vectors();
enable_interrupt_controller();
irq_barrier();
irq_enable();
timer_init();
```

初始化异常向量表
---------------------------------------
实际上，中断是一种更具体的_异常_——当"触发"时，需要处理器立即关注的事情。异常可能发生的一个完美例子是当错误代码尝试做一些"不可能"的事情时，例如除以零。CPU需要知道如何响应何时/如果发生这种情况，即跳转到要运行的某些代码的地址来优雅地处理此异常，例如通过向屏幕打印错误。这些地址存储在_异常向量表_中。

_irqentry.S_设置一个名为`vectors`的列表，其中包含各个_向量条目_。这些向量条目只是跳转到处理程序代码的跳转指令。

在_kernel.c_中的`main()`调用`irq_init_vectors()`期间，CPU被告知此异常向量表存储在哪里。你可以在_utils.S_中找到此代码：

```c
irq_init_vectors:
    adr x0, vectors
    msr vbar_el1, x0
    ret
```

它只是将向量基地址寄存器设置为`vectors`列表的地址。

中断处理
------------------
就本教程而言，我们真正关心的唯一向量条目是`handle_el1_irq`。这是对在EL1（内核执行级别）传入的任何中断请求（IRQ）的通用处理程序。

如果你确实想要更深入的理解，我强烈推荐阅读s-matyukevich的工作[这里](https://github.com/s-matyukevich/raspberry-pi-os/blob/master/docs/lesson03/rpi-os.md)。

```c
handle_el1_irq:
	kernel_entry
	bl      handle_irq
	kernel_exit
```

简单来说，`kernel_entry`在中断处理程序运行之前保存寄存器状态，`kernel_exit`在我们返回之前恢复此寄存器状态。当我们_中断_正常程序执行时，我们希望确保将事情恢复到原来的状态，以便当我们的内核代码恢复时不会发生不可预测的事情。

在中间，我们只是调用一个名为`handle_irq()`的函数，该函数在_irq.c_中用C语言编写。它的目的是更仔细地查看中断请求，找出是哪个设备负责生成中断，并运行正确的子处理程序：

```c
void handle_irq() {
    unsigned int irq = REGS_IRQ->irq0_pending_0;

    while(irq & (SYS_TIMER_IRQ_1 | SYS_TIMER_IRQ_3)) {
        if (irq & SYS_TIMER_IRQ_1) {
            irq &= ~SYS_TIMER_IRQ_1;

            handle_timer_1();
        }

        if (irq & SYS_TIMER_IRQ_3) {
            irq &= ~SYS_TIMER_IRQ_3;

            handle_timer_3();
        }
    }
}
```

如你所见，我们在此代码中处理两个不同的计时器中断。实际上，`handle_timer_1()`和`handle_timer_3()`在_kernel.c_中实现，通过增加进度计数器并更新其值的图形表示来演示计时器已触发。计时器3配置为以计时器1的4倍速度运行。

中断控制器
------------------------
中断控制器是负责在中断发生时通知CPU的硬件。我们可以使用中断控制器作为看门狗，允许/阻止（或启用/禁用）中断。我们也可以使用它来找出哪个设备生成了中断，就像我们在`handle_irq()`中所做的那样。

在`enable_interrupt_controller()`中（从_kernel.c_中的`main()`调用），我们允许计时器1和计时器3中断通过，在`disable_interrupt_controller()`中我们阻止所有中断：

```c
void enable_interrupt_controller() {
    REGS_IRQ->irq0_enable_0 = SYS_TIMER_IRQ_1 | SYS_TIMER_IRQ_3;
}

void disable_interrupt_controller() {
    REGS_IRQ->irq0_enable_0 = 0;
}
```

屏蔽/取消屏蔽中断
----------------------------
要开始接收中断，我们需要再走一步：取消屏蔽所有类型的中断。

屏蔽是CPU用来防止特定代码被中断停止的技术。它用于保护*必须*完成的重要代码。想象一下，如果我们的`kernel_entry`代码（保存寄存器状态）在中途被中断会发生什么！在这种情况下，寄存器状态将被覆盖并丢失。这就是为什么CPU在执行异常处理程序时自动屏蔽所有中断。

_utils.S_中的`irq_enable`和`irq_disable`函数负责屏蔽和取消屏蔽中断。

`irq_barrier`函数帮助它们确保`enable_interrupt_controller()`调用在`irq_enable()`调用之前正确完成：

```c
.globl irq_enable
irq_enable:
    msr daifclr, #2
    ret

.globl irq_disable
irq_disable:
    msr daifset, #2
    ret

.globl irq_barrier
irq_barrier:
    dsb sy
    ret
```

一旦从_kernel.c_中的`main()`调用`irq_enable()`，当计时器中断触发时，计时器处理程序就会运行。嗯，差不多！

初始化系统计时器
-----------------------------
我们仍然需要初始化计时器。

RPi4的系统计时器再简单不过了。它有一个计数器，每个时钟滴答增加1。然后它有4个中断线（0和2保留给GPU，本教程中我们使用1和3！），带有4个相应的比较寄存器。当计数器的值变得等于其中一个比较寄存器中的值时，相应的中断就会触发。

因此，在我们接收任何计时器中断之前，我们还必须将正确的比较寄存器设置为非零值。`timer_init()`函数（从_kernel.c_中的`main()`调用）获取当前计时器值，添加计时器间隔，并将比较寄存器设置为该总和，因此当正确数量的时钟滴答过去时，中断就会触发。它对计时器1和计时器3都这样做，将计时器3设置为以4倍速度运行。

处理计时器中断
-----------------------------
这是最简单的部分。

我们更新比较寄存器，以便下一次中断将在相同的间隔后再次生成。重要的是，我们然后通过设置控制状态寄存器的正确位来确认中断。

然后我们更新屏幕以显示我们的进度！

_然后...嘿 presto！你就像专业人士一样处理两个系统计时器中断！_

![计时器在Raspberry Pi 4上全力运行](images/13-interrupts-running.jpg)

[前往part14-spi-ethernet >](../part14-spi-ethernet)
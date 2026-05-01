为 Raspberry Pi 4 编写「裸机」操作系统（第十部分）
====================================================================

[< 返回part9-sound](../part9-sound)

使用多个CPU核心
------------------------
我建议我们可以使用第二个CPU核心来播放音频，同时我们的主核心继续运行，而不是进行后台DMA传输。我也说过在Raspberry Pi 4上这很难...确实如此。

我参考了[Sergey Matyukevich的工作](https://github.com/s-matyukevich/raspberry-pi-os/tree/master/src/lesson02)编写了这段代码，对此我非常感激。它确实需要一些修改，以确保辅助核心在适当的时候被唤醒。这段代码还不是特别"安全"，但它足以在原则上证明这个概念。

你需要修改SD卡上的_config.txt_文件，包含以下行：

```c
kernel_old=1
disable_commandline_tags=1
arm_64bit=1
```

也许这里最重要的是`kernel_old=1`指令。这告诉bootloader在偏移量`0x00000`而不是`0x80000`处期望内核。因此，我们需要从_link.ld_中删除这一行：

```c
. = 0x80000;     /* Kernel load address for AArch64 */
```

它也不会在启动时为我们锁定辅助核心，所以我们仍然能够访问它们（稍后会详细介绍）。

设置主计时器
-------------------------
现在我们需要自己处理另一个重要的设置——建立主计时器。我们在_boot.S_的顶部添加以下`#define`块：

```c
#define LOCAL_CONTROL   0xff800000
#define LOCAL_PRESCALER 0xff800008
#define OSC_FREQ        54000000
#define MAIN_STACK      0x400000
```

`LOCAL_CONTROL`是ARM_CONTROL寄存器的地址。在我们的`_start:`部分的顶部，我们将其设置为零，实际上告诉ARM主计时器使用晶体时钟作为源，并将增量值设置为1：

```c
ldr     x0, =LOCAL_CONTROL   // 处理计时器
str     wzr, [x0]
```

我们继续设置预分频器——把它想象成另一个等效的时钟除数。这样设置它实际上会使这个除数为1（即它不会有任何效果）：

```c
mov     w1, 0x80000000
str     w1, [x0, #(LOCAL_PRESCALER - LOCAL_CONTROL)]
```

你应该从part9记住预期的振荡器频率54 MHz。我们用以下行设置它：

```c
ldr     x0, =OSC_FREQ
msr     cntfrq_el0, x0
msr     cntvoff_el2, xzr
```

我们的计时器现在已经按我们需要的方式设置好了。

启动主核心
---------------------
我们继续像往常一样检查处理器ID。如果它是零，那么我们在主核心上，并跳转到标签`2:`。这次，我们必须稍微不同地设置我们的堆栈指针。我们不能在代码下方设置它，因为它现在在0x00000！相反，我们使用我们之前定义为顶部的`MAIN_STACK`地址：

```c
// 将堆栈设置在安全的地方开始
mov     sp, #MAIN_STACK
```

然后我们继续像往常一样清除BSS，并跳转到我们C代码中的`main()`函数。如果它确实返回，我们分支回`1:`来停止核心。

设置辅助核心
------------------------------
以前，我们通过在标签`1:`处的无限循环中旋转其他核心来明确地停止它们。相反，现在每个核心将在其自己指定的内存地址处监视一个值，该值在_boot.S_的底部初始化为零，并命名为`spin_cpu0-3`。如果这个值变为非零，那么这是一个唤醒并跳转到该内存位置的信号，执行那里的任何代码。一旦该代码返回，我们再次开始循环和监视。

```c
    adr     x5, spin_cpu0        // 基础监视地址
1:  wfe
    ldr     x4, [x5, x1, lsl #3] // 将(8 * core_number)添加到基础地址，并将那里的内容加载到x4
    cbz     x4, 1b               // 如果为零则循环，否则继续

    ldr     x2, =__stack_start   // 获取一个新的堆栈 - 位置取决于请求的CPU核心
    lsl     x1, x1, #9           // 将core_number乘以512
    add     x3, x2, x1           // 添加到地址
    mov     sp, x3

    mov     x0, #0               // 清零寄存器x0-x3，以防万一
    mov     x1, #0
    mov     x2, #0
    mov     x3, #0
    br      x4                   // 运行x4中地址处的代码
    b       1b
```

你会注意到我们在其他地方设置了堆栈指针，每个核心都有自己指定的堆栈地址。这是为了避免与其他核心上的活动冲突。我们通过在_link.ld_中添加以下内容来建立指向安全内存区域的必要指针：

```c
.cpu1Stack :
{
    . = ALIGN(16);               // 16位对齐
    __stack_start  = .;          // 指向开始的指针
    . = . + 512;                 // 512字节长
    __cpu1_stack  = .;           // 指向末尾的指针（堆栈向下增长）
}
.cpu2Stack :
{
    . = . + 512;
    __cpu2_stack  = .;
}
.cpu3Stack :
{
    . = . + 512;
    __cpu3_stack  = .;
}
```

呼！bootloader代码就这样了。如果你使用这个新的bootloader和你现有的代码，RPi4应该像以前一样启动和运行。我们现在需要继续实现所需的信号传递，以在这些现在可供我们使用的辅助核心上执行代码。

从C唤醒辅助核心
---------------------------------
查看_multicore.c_。

在这里，我们本质上为每个核心复制两个函数：

```c
void start_core1(void (*func)(void))
{
    store32((unsigned long)&spin_cpu1, (unsigned long)func);
    asm volatile ("sev");
}

void clear_core1(void) 
{
    store32((unsigned long)&spin_cpu1, 0);
}
```

第一个`start_core1()`使用`store32()`函数（也在_multicore.c_中）将一个地址写入我们预定义的`spin_cpu1`内存位置。这使它变为非零，告诉核心1醒来时跳到哪里。既然我们用`wfe`（等待事件）指令让它睡眠，我们使用`sev`（设置事件）指令再次唤醒它。

第二个`clear_core1()`可以被执行函数用来将`spin_cpu1`重置为零，这样当执行代码返回时，核心不会再次跳转。

更多的main()！
---------------------
最后，我们看一下_kernel.c_，我们现在有一个单一的`main()`，但还有：

* `core0_main()` - 每1秒（大约）增加一个进度条
* `core1_main()` - 有一个两步进度条，使用CPU在50%时播放音频样本，完成时直接跳到100%
* `core2_main()` - 设置DMA音频传输，然后每半秒（大约）增加一个进度条，在播放完成时跳到100%
* ...以及`core3_main()` - 每四分之一秒（大约）增加一个进度条

`main()`是核心0的入口点，最终会执行到`core0_main()`，但在此之前它通过将`core3_main()`和`core1_main()`传递给它们各自的启动函数来启动它们。当`core1_main()`完成时，它启动`core2_main()`。

_当你运行这个时，你会看到这些函数在各自的核心上并行运行。欢迎来到对称多处理！_

**如果启动时你只看到彩虹屏幕，请尝试首先使用Raspbian中的`rpi-update`更新你的固件。**

![代码现在在Raspberry Pi 4的所有四个核心上运行](images/10-multicore-running.jpg)

在第11部分中，我们将把所有这些工作整合在一起，创建我们打砖块游戏的多核版本。

[前往part11-breakout-smp >](../part11-breakout-smp)
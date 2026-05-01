为 Raspberry Pi 4 编写「裸机」操作系统（第三部分）
===================================================================

[< 返回part2-building](../part2-building)

让事情发生
-----------------------

到目前为止，我们的操作系统只产生一个黑屏。我们怎么能确定我们的代码实际上在运行呢？让我们做一些更有趣的事情来真正证明我们控制了硬件。

通常，软件开发人员学到的第一件事是向屏幕打印"Hello world!"。然而，在裸机开发中，向屏幕打印可能是一个相当大的挑战，所以我们首先做一些更简单的事情。

介绍UART
--------------------

也许我们可以从操作系统"发送消息"的最简单方法是通过**UART**或串行通信电路。UART代表通用异步收发器，它是一种非常古老且相当简单的接口，只使用两根线在两个设备之间通信。在USB出现之前，鼠标、打印机和调制解调器等设备都是以这种方式连接的。

我们将把你的开发机器直接连接到RPi4，并让RPi4向你的开发机器发送"Hello world!"消息！你的开发机器会将其打印到屏幕上。

你需要：

* 一根[USB转串口TTL线](https://www.amazon.co.uk/gp/product/B01N4X3BJB/ref=ppx_yo_dt_b_asin_title_o00_s00?ie=UTF8&psc=1)
* [下载并安装电缆驱动](https://www.silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers)
* 在你的开发机器上[下载并安装PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
* 如果你使用Mac，我建议[安装Serial Tools](https://apps.apple.com/gb/app/serialtools/id611021963?mt=12)作为PuTTY的替代

如果你想在开始前阅读有关串行通信的内容，我建议查看[SparkFun网站](https://learn.sparkfun.com/tutorials/serial-communication/all)。

连接电缆
--------------------

如果你已安装驱动程序，请继续将电缆连接到开发机器上的一个空闲USB端口。几乎不会发生什么事情，但如果你现在打开控制面板，点击设备管理器并打开端口部分，你应该会看到一个"Prolific"条目。这告诉我们你的电缆工作正常。

这是我的机器的样子：

![Windows控制面板安装了电缆](images/3-helloworld-ctlpanel.png)

记下Prolific条目后括号中的COMx编号——在我的例子中，是**COM5**。

同样的电缆在Mac上无需安装任何驱动程序即可工作。

现在我们需要查看RPi4，确定如何连接电缆的另一端。你会找到**GPIO引脚**，共40个，就在Raspberry Pi版权声明的上方。

下图显示了你需要连接的位置。我推荐的电缆有颜色编码的分线：

* BLACK = 接地
* RED = +5v电源
* GREEN = TX（从USB端口传输到RPi4）
* WHITE = RX（从RPi4接收至USB端口）

接地线（在我的例子中是黑色）连接到RPi4的接地引脚（引脚6），RX线（白色）连接到TXD（GPIO 14/引脚8），TX线（绿色）连接到RXD（GPIO 15/引脚10）。请注意，RX和TX需要交叉连接，即将RX连接到TX，反之亦然。由于我们使用专用电源为RPi4供电，请确保你**不要连接红色连接器**。

![GPIO位置](images/3-helloworld-pinloc.png)

这是我的RPi4正确连接电缆的样子：

![连接电缆的GPIO照片](images/3-helloworld-cable.jpg)

设置PuTTY
----------------

* 在你的开发机器上运行PuTTY
* 点击左侧窗格中的"Session"类别
* 将"Connection type"设置为Serial
* 点击左侧窗格中"Connection"类别下的"Serial"
* 将"Serial line to connect to"设置为我们上面找到的COMx编号，我的是COM5
* 将"Speed (baud)"设置为115200
* 确保"Data bits"为8，"Stop bits"为1，"Parity"为None，"Flow control"为None
* 点击左侧窗格中的"Session"类别，你应该会看到更改的设置
* 通过在"Saved Sessions"下的文本框中输入名称（例如"Raspberry Pi 4"）并点击Save来保存这些设置
* 你现在可以通过双击"Raspberry Pi 4"来启动连接——如果你这样做，目前你只会看到一个空的黑色窗口

如果你使用不同的终端模拟器，你需要按照应用程序供应商的软件使用说明使用上述相同设置。例如，Mac上的Serial Tools在[这里](https://www.w7ay.net/site/Applications/Serial%20Tools/)有说明。

快速修改config.txt
-------------------------

你还记得在第一个教程中，我必须编辑SD卡上的_config.txt_文件才能让Raspbian在我的电视屏幕上运行吗？现在我们需要添加一行来确保我们的UART连接可靠。

UART通信在很大程度上与计时有关，两端就发送/接收数据的确切速度达成一致非常重要。当我们设置PuTTY时，我们告诉它以115200波特率通信，我们需要RPi4以相同的速率通信。事实上，我们不能确定它会这样做——它可能会根据CPU的繁忙程度而通信得更快或更慢。

添加这行到你的_config.txt_来解决这个问题：

```c
core_freq_min=500
```

在代码中启用UART
------------------------------

首先，让我们更新_kernel.c_来进行一些新的调用：

```c
#include "io.h"

void main()
{
    uart_init();
    uart_writeText("Hello world!\n");
    while (1);
}
```

我们首先包含一个新的**头文件**_io.h_。这允许我们在_kernel.c_文件之外编写一些新代码，并在需要时调用它。

你会注意到我们的`main()`例程也有一些新行。首先我们调用一个函数来初始化UART，然后我们调用另一个函数来向它写入"Hello world!"。字符串末尾的奇怪字符——`\n`——是我们如何在文本末尾添加换行符，就像在文字处理器中按Enter键一样！

现在让我们创建包含以下内容的_io.h_：

```c
void uart_init();
void uart_writeText(char *buffer);
```

这是一个非常短的文件，包含两个**函数定义**。`uart_init()`是一个**无返回值函数**，没有**参数**，就像`main()`一样。这意味着它不需要调用者提供任何数据来完成其工作，并且完成后不会向调用者发送任何数据。你会注意到`uart_writeText`也是一个无返回值函数，但它确实需要一个参数，因为我们需要告诉它要写什么文本！

我们将这些函数的实际代码放在另一个新文件_io.c_中：

```c
// GPIO

enum {
    PERIPHERAL_BASE = 0xFE000000,
    GPFSEL0         = PERIPHERAL_BASE + 0x200000,
    GPSET0          = PERIPHERAL_BASE + 0x20001C,
    GPCLR0          = PERIPHERAL_BASE + 0x200028,
    GPPUPPDN0       = PERIPHERAL_BASE + 0x2000E4
};

enum {
    GPIO_MAX_PIN       = 53,
    GPIO_FUNCTION_ALT5 = 2,
};

enum {
    Pull_None = 0,
};

void mmio_write(long reg, unsigned int val) { *(volatile unsigned int *)reg = val; }
unsigned int mmio_read(long reg) { return *(volatile unsigned int *)reg; }

unsigned int gpio_call(unsigned int pin_number, unsigned int value, unsigned int base, unsigned int field_size, unsigned int field_max) {
    unsigned int field_mask = (1 << field_size) - 1;
  
    if (pin_number > field_max) return 0;
    if (value > field_mask) return 0; 

    unsigned int num_fields = 32 / field_size;
    unsigned int reg = base + ((pin_number / num_fields) * 4);
    unsigned int shift = (pin_number % num_fields) * field_size;

    unsigned int curval = mmio_read(reg);
    curval &= ~(field_mask << shift);
    curval |= value << shift;
    mmio_write(reg, curval);

    return 1;
}

unsigned int gpio_set     (unsigned int pin_number, unsigned int value) { return gpio_call(pin_number, value, GPSET0, 1, GPIO_MAX_PIN); }
unsigned int gpio_clear   (unsigned int pin_number, unsigned int value) { return gpio_call(pin_number, value, GPCLR0, 1, GPIO_MAX_PIN); }
unsigned int gpio_pull    (unsigned int pin_number, unsigned int value) { return gpio_call(pin_number, value, GPPUPPDN0, 2, GPIO_MAX_PIN); }
unsigned int gpio_function(unsigned int pin_number, unsigned int value) { return gpio_call(pin_number, value, GPFSEL0, 3, GPIO_MAX_PIN); }

void gpio_useAsAlt5(unsigned int pin_number) {
    gpio_pull(pin_number, Pull_None);
    gpio_function(pin_number, GPIO_FUNCTION_ALT5);
}

// UART

enum {
    AUX_BASE        = PERIPHERAL_BASE + 0x215000,
    AUX_ENABLES     = AUX_BASE + 4,
    AUX_MU_IO_REG   = AUX_BASE + 64,
    AUX_MU_IER_REG  = AUX_BASE + 68,
    AUX_MU_IIR_REG  = AUX_BASE + 72,
    AUX_MU_LCR_REG  = AUX_BASE + 76,
    AUX_MU_MCR_REG  = AUX_BASE + 80,
    AUX_MU_LSR_REG  = AUX_BASE + 84,
    AUX_MU_CNTL_REG = AUX_BASE + 96,
    AUX_MU_BAUD_REG = AUX_BASE + 104,
    AUX_UART_CLOCK  = 500000000,
    UART_MAX_QUEUE  = 16 * 1024
};

#define AUX_MU_BAUD(baud) ((AUX_UART_CLOCK/(baud*8))-1)

void uart_init() {
    mmio_write(AUX_ENABLES, 1); //enable UART1
    mmio_write(AUX_MU_IER_REG, 0);
    mmio_write(AUX_MU_CNTL_REG, 0);
    mmio_write(AUX_MU_LCR_REG, 3); //8 bits
    mmio_write(AUX_MU_MCR_REG, 0);
    mmio_write(AUX_MU_IER_REG, 0);
    mmio_write(AUX_MU_IIR_REG, 0xC6); //disable interrupts
    mmio_write(AUX_MU_BAUD_REG, AUX_MU_BAUD(115200));
    gpio_useAsAlt5(14);
    gpio_useAsAlt5(15);
    mmio_write(AUX_MU_CNTL_REG, 3); //enable RX/TX
}

unsigned int uart_isWriteByteReady() { return mmio_read(AUX_MU_LSR_REG) & 0x20; }

void uart_writeByteBlockingActual(unsigned char ch) {
    while (!uart_isWriteByteReady()); 
    mmio_write(AUX_MU_IO_REG, (unsigned int)ch);
}

void uart_writeText(char *buffer) {
    while (*buffer) {
       if (*buffer == '\n') uart_writeByteBlockingActual('\r');
       uart_writeByteBlockingActual(*buffer++);
    }
}
```

你会看到我们在_io.h_头文件中定义的两个函数现在有了一些实际代码，以及一些其他支持函数。我将在下一个教程中解释这段代码中发生了什么，但现在让我们直接进入操作！

有了新的_io.c_和_io.h_文件，以及对_kernel.c_所做的更改，运行`make`来构建你的新操作系统。

然后：

* 将新构建的_kernel8.img_复制到SD卡，然后将SD卡放入你的RPi4
* 确保你的USB转串口TTL线连接正确
* 运行你的终端模拟器（例如PuTTY）并连接到你之前设置的"Raspberry Pi 4"会话——你应该看到一个空的黑屏且没有错误
* 打开RPi4的电源

如果你遵循了所有这些说明，几秒钟后你会在开发机器上的终端模拟器窗口中看到"Hello world!"出现。

_这是来自你的RPi4的一条消息，告诉你你的操作系统正在工作。终于有证据了！_

[前往part4-miniuart >](../part4-miniuart)
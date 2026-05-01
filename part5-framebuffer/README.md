为 Raspberry Pi 4 编写「裸机」操作系统（第五部分）
===================================================================

[< 返回part4-miniuart](../part4-miniuart)

与屏幕交互
-----------------------

尽管UART很令人兴奋，但在本教程中，我们终于要把"Hello world!"显示在屏幕上了！现在我们已经是MMIO专家了，我们准备好处理**邮箱**（mailboxes）了。这是我们与VideoCore多媒体处理器通信的方式。我们可以向它发送消息，它也可以回复。把它想象成电子邮件就好了。

让我们创建_mb.c_：

```c
#include "io.h"

// 缓冲区必须对齐16字节，因为地址的高28位才能通过邮箱传递
volatile unsigned int __attribute__((aligned(16))) mbox[36];

enum {
    VIDEOCORE_MBOX = (PERIPHERAL_BASE + 0x0000B880),
    MBOX_READ      = (VIDEOCORE_MBOX + 0x0),
    MBOX_POLL      = (VIDEOCORE_MBOX + 0x10),
    MBOX_SENDER    = (VIDEOCORE_MBOX + 0x14),
    MBOX_STATUS    = (VIDEOCORE_MBOX + 0x18),
    MBOX_CONFIG    = (VIDEOCORE_MBOX + 0x1C),
    MBOX_WRITE     = (VIDEOCORE_MBOX + 0x20),
    MBOX_RESPONSE  = 0x80000000,
    MBOX_FULL      = 0x80000000,
    MBOX_EMPTY     = 0x40000000
};

unsigned int mbox_call(unsigned char ch)
{
    // 28位地址（高位）和4位值（低位）
    unsigned int r = ((unsigned int)((long) &mbox) &~ 0xF) | (ch & 0xF);

    // 等待直到可以写入
    while (mmio_read(MBOX_STATUS) & MBOX_FULL);
    
    // 将缓冲区地址写入邮箱，并附加通道号
    mmio_write(MBOX_WRITE, r);

    while (1) {
        // 有回复吗？
        while (mmio_read(MBOX_STATUS) & MBOX_EMPTY);

        // 是对我们消息的回复吗？
        if (r == mmio_read(MBOX_READ)) return mbox[1]==MBOX_RESPONSE; // 成功了吗？
           
    }
    return 0;
}
```

首先，我们包含_io.h_，因为我们需要访问`PERIPHERAL_BASE`定义，并且也要使用_io.c_提供的`mmio_read`和`mmio_write`函数。我们之前的MMIO经验在这里很有用，因为发送/接收邮箱请求/响应是使用相同的技术实现的。正如你在代码中看到的，我们将只是从`PERIPHERAL_BASE`访问不同的偏移量。

重要的是，我们的邮箱缓冲区（消息将存储在这里）需要在内存中正确对齐。这是我们需要严格要求编译器而不是让它自行处理的一个例子！通过确保缓冲区是"16字节对齐"的，我们知道它的内存地址是16的倍数，即4个最低有效位未设置。这很好，因为只有28个最高有效位可以用作地址，留下4个最低有效位来指定**通道**。

我建议阅读[邮箱属性接口](https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface)。你会看到通道8保留用于ARM发送给VideoCore响应的消息，所以我们将使用这个通道。

`mbox_call`实现了我们发送消息（假设它已经在缓冲区中设置好了）并等待回复所需的所有MMIO操作。VideoCore会直接将回复写入我们原来的缓冲区。

帧缓冲区
---------------

现在看一下_fb.c_。`fb_init()`例程使用_mb.h_中的一些定义进行我们的第一次邮箱调用。还记得电子邮件的类比吗？嗯，既然可以通过电子邮件要求一个人做不止一件事，我们也可以同时向VideoCore要求几件事。这条消息要求两件事：

* 帧缓冲区起始指针（`MBOX_TAG_GETFB`）
* 间距（`MBOX_TAG_GETPITCH`）

你可以在我之前分享的[邮箱属性接口](https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface)页面上阅读更多关于消息结构的信息。

**帧缓冲区**只是一块内存区域，包含一个驱动视频显示的位图。换句话说，我们可以通过写入特定的内存地址直接操纵屏幕上的像素。不过，我们首先需要了解这块内存是如何组织的。

在这个例子中，我们向VideoCore请求：

* 一个简单的1920x1080（1080p）帧缓冲区
* 每像素32位的深度，RGB像素顺序

所以每个像素由8位红色值、8位绿色值、8位蓝色值和8位Alpha通道（表示透明度/不透明度）组成。我们要求像素在内存中按红色字节在前，然后是绿色，然后是蓝色的顺序排列——RGB。实际上，Alpha字节总是在所有这些之前，所以实际上是ARGB。

然后我们使用通道8（`MBOX_CH_PROP`）发送消息，并检查VideoCore返回的内容是否与我们请求的一致。它还应该告诉我们帧缓冲区组织难题中缺失的部分——每行的字节数或**间距**（pitch）。

如果一切按预期返回，那么我们就准备好写入屏幕了！

绘制像素
---------------

为了节省我们记住RGB颜色组合的时间，让我们设置一个简单的16色**调色板**。有人记得旧的[EGA/VGA调色板](https://en.wikipedia.org/wiki/Enhanced_Graphics_Adapter)吗？如果你看一下_terminal.h_，你会看到`vgapal`数组设置了相同的调色板，黑色是项目0，白色是项目15，中间有许多色调！

我们的`drawPixel`例程可以接受一个(x, y)坐标和一个颜色。我们使用一个`unsigned char`（8位）来同时表示两个调色板索引，高4位表示背景颜色，低4位表示前景颜色。你可能会明白为什么现在只有16色调色板是有帮助的！

```c
void drawPixel(int x, int y, unsigned char attr)
{
    int offs = (y * pitch) + (x * 4);
    *((unsigned int*)(fb + offs)) = vgapal[attr & 0x0f];
}
```

我们首先计算帧缓冲区中的字节偏移量。(y * pitch)让我们到达坐标(0, y)——pitch是每行的字节数。然后我们加上(x * 4)到达(x, y)——每个像素有4个字节（或32位！）（ARGB）。然后我们可以将帧缓冲区中的该字节设置为我们的前景色（这里我们不需要背景色）。

绘制线条、矩形和圆形
-------------------------------------

现在检查并理解我们的`drawRect`、`drawLine`和`drawCircle`例程。填充多边形时，我们使用背景色进行填充，使用前景色进行轮廓绘制。

我建议阅读[Bresenham](https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm)关于绘制图形原语的内容。我的线条和圆形绘制算法来自他，它们设计为只使用简单的数学运算。他的著作非常有趣，他的算法在今天仍然非常重要。

向屏幕写入字符
--------------------------------

我告诉过你在裸机编程中没有什么是免费的，对吧？那么，如果我们想在屏幕上写消息，我们需要一种字体。所以，就像我们构建调色板一样，我们现在需要为我们的代码构建一种字体。幸运的是，字体只是简单**位图**的数组——用于描述图片的1和0。我们将定义一个类似于MS-DOS使用的8x8字体。

想象一个8x8的"A"：

```c
0 0 0 0 1 1 0 0 = 0x0C
0 0 0 1 1 1 1 0 = 0x1E
0 0 1 1 0 0 1 1 = 0x33
0 0 1 1 0 0 1 1 = 0x33
0 0 1 1 1 1 1 1 = 0x3F
0 0 1 1 0 0 1 1 = 0x33
0 0 1 1 0 0 1 1 = 0x33
0 0 0 0 0 0 0 0 = 0x00
```

这个位图可以只用8个字节（=号后面的十六进制数字）来表示。当你查看_terminal.h_时，你会看到我们已经为[代码页437](https://en.wikipedia.org/wiki/Code_page_437)中的许多有用字符做了这个。

`drawChar`现在应该相当不言自明了。

* 我们设置一个指针`glyph`指向我们要绘制的字符的位图
* 我们遍历位图数组，从第一行开始，然后是第二行，然后是第三行等等
* 对于行中的每个像素，我们确定它应该设置为背景色（相应的字形位为0）还是前景色（位为1）
* 我们在正确的坐标处绘制相应的像素

`drawString`毫不奇怪地使用`drawChar`来打印整个字符串。

更新我们的内核使其更具艺术性
---------------------------------------

最后，我们可以在屏幕上创作一件艺术品了！我们更新后的_kernel.c_练习所有这些图形例程来绘制下面的图片。

构建内核，将其复制到你的SD卡。你可能需要再次更新你的_config.txt_。如果你之前设置了`hdmi_safe`参数来启动Raspbian，你现在可能不需要它了。然而，你可能需要专门设置`hdmi_mode`和`hdmi_group`以确保我们进入1080p模式。

现在是了解RPi4的[屏幕分辨率设置](https://pimylifeup.com/raspberry-pi-screen-resolution/)的好时机。因为我使用的是普通电视，我的_config.txt_文件现在包含三行（包括我们已经为UART添加的那一行）：

```c
core_freq_min=500
hdmi_group=1
hdmi_mode=16
```

现在启动RPi4！

_我们已经在屏幕上做了比基本的"Hello world!"更多的事情了！_ 坐下来，放松，欣赏你的艺术品。在下一个教程中，我们将把图形与来自UART的键盘输入结合起来，创建我们的第一个游戏。

![我们的第一个屏幕艺术品](images/5-framebuffer-screen.jpg)

[前往part6-breakout >](../part6-breakout)
为 Raspberry Pi 4 编写「裸机」操作系统（第十五部分）
====================================================================

[< 返回part14-spi-ethernet](../part14-spi-ethernet)

添加TCP/IP协议栈
---------------------
在_part14-spi-ethernet_中从我们的以太网模块实现"生命证明"后，你无疑想知道如何从那里发展到提供网页、在Twitter上发布推文，或者甚至只是简单地响应ping！

这就是你需要一个成熟的TCP/IP协议栈的地方，它远远超出手工制作的ARP，实现更多协议以实现高效的双向通信。

在这一部分，我们利用了[tuxgraphics.org](http://tuxgraphics.org/)的Guido Socher的一些代码，旨在为嵌入式设备提供轻量级的TCP/IP协议栈。我选择它是因为它非常容易工作（或"移植"），但如果你需要更高级的东西，你可能想看看[LwIP](https://en.wikipedia.org/wiki/LwIP)。

代码
--------
大多数新代码在_tcpip/_子目录中。我实际上是在[这个tarball](http://tuxgraphics.org/common/src2/article09051/eth_tcp_client_server-dhcp-5.10.tar.gz)中找到它的，同样，只做了一些非常小的表面改动（`diff`是你的朋友！）。

它确实需要我公开我们在_lib/fb.c_中实现的`strlen()`函数，所以这被添加到_include/fb.h_中。同样，我们公开我们在_kernel/kernel.c_中实现的`memcpy()`函数，所以这被添加到_kernel/kernel.h_中。

我还需要一个告诉ENC发送数据包的函数。这里没有新东西，只是不同的封装：

```c
void enc28j60PacketSend(unsigned short buflen, void *buffer) {
   if (ENC_RestoreTXBuffer(&handle, buflen) == 0) {
      ENC_WriteBuffer((unsigned char *) buffer, buflen);
      handle.transmitLength = buflen;
      ENC_Transmit(&handle);
   }
}
```

这也被添加到_kernel/kernel.h_中。

_arp.c_怎么了？
-------------------------
你会注意到我已经合并了_arp.c_和_kernel.c_。我也不再使用多核或IRQ计时器，以使这个内核保持简单。我们仍然以完全相同的方式初始化网卡，但是当我们完成后，我们调用Guido代码中的这个函数：

```c
init_udp_or_www_server(myMAC, deviceIP);
```

这告诉TCP/IP库我们是谁，所以我们都在同一页面上！

最后，除了一些清理工作（例如将HAL/系统计时器函数移动到_lib/io.c_，并对_include/io.h_进行相应的更改），主要更改是新的`serve()`函数：

```c
void serve(void)
{
   while (1) {
      while (!ENC_GetReceivedFrame(&handle));

      uint8_t *buf = (uint8_t *)handle.RxFrameInfos.buffer;
      uint16_t len = handle.RxFrameInfos.length;
      uint16_t dat_p = packetloop_arp_icmp_tcp(buf, len);

      if (dat_p != 0) {
         debugstr("Incoming web request... ");

         if (strncmp("GET ", (char *)&(buf[dat_p]), 4) != 0) {
            debugstr("not GET");
            dat_p = fill_tcp_data(buf, 0, "HTTP/1.0 401 Unauthorized\r\nContent-Type: text/html\r\n\r\n<h1>ERROR</h1>");
         } else {
            if (strncmp("/ ", (char *)&(buf[dat_p+4]), 2) == 0) {
               // 只是网页服务器"根目录"中的一个网页
               debugstr("GET root");
               dat_p = fill_tcp_data(buf, 0, "HTTP/1.0 200 OK\r\nContent-Type: text/html\r\n\r\n<h1>Hello world!</h1>");
            } else {
               // 不是网页服务器"根目录"中的一个网页
               debugstr("GET not root");
               dat_p = fill_tcp_data(buf, 0, "HTTP/1.0 200 OK\r\nContent-Type: text/html\r\n\r\n<h1>Goodbye cruel world.</h1>");
            }
         }

         www_server_reply(buf, dat_p); // 发送网页数据
         debugcrlf();
      }
   }
}
```

这是一个无限循环，等待传入的数据包，然后首先将其传递给Guido的`packetloop_arp_icmp_tcp()`函数。这个函数实现了一些有用的东西，比如响应ping。我修改了例程，在发送"pong"时向屏幕打印一条消息（查看_tcpip/ip_arp_udp_tcp.c_的第1371行），这样我们就可以看到它何时在运行！

检查`packetloop_arp_icmp_tcp()`的返回值然后允许我们检查是否有传入的Web请求，因为我们已经在_tcpip/ip_config.h_中使用`#define WWW_server`将TCP/IP库配置为Web服务器。

然后我们根据三种可能的情况提供响应：

* 传入请求不是GET请求（例如，可能是HEAD请求）——你可以使用`curl`工具模拟：`curl -I 192.168.0.66`
* 传入请求是对根网页`/`的GET请求——`curl 192.168.0.66/`
* 传入请求是对任何非根网页的GET请求——例如`curl 192.168.0.66/babbleberry/monkey`

我建议阅读[这篇文章](http://tuxgraphics.org/electronics/200905/embedded-tcp-ip-stack.shtml)以获得完整的解释。我移植的代码与你在那里看到的非常相似。

_想象一下，当我构建、运行并能够在192.168.0.66处ping我的RPi4，并在我的笔记本电脑和iPhone上的浏览器中获得Web响应时，我有多兴奋！_

![从我的iPhone ping](images/15-tcpip-webserver-pinging.jpg)
![从我的笔记本电脑浏览](images/15-tcpip-webserver-browser.png)
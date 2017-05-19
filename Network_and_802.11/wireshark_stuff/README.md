# [Wireshark指南(by bt3)](http://bt3gl.github.io/wiresharking-for-fun-or-profit.html)


[Wireshark](https://www.wireshark.org/)是一个开源的**网络封包分析**软件，允许实时流量分析并支持多种协议。

Wireshark也允许**网络取证**，对CTF比赛非常有用，比如说(看我的writeup，[D-CTF Quals 2014](http://bt3gl.github.io/exploring-d-ctf-quals-2014s-exploits.html)，CSAW Quals 2014 在 [Networking](http://bt3gl.github.io/csaw-ctf-2014-networking-100-big-data.html) 和 [Forensics](http://bt3gl.github.io/csaw-ctf-2014-forensics-200-why-not-sftp.html)).



------------------------------------------------------
# 网络架构

在我们理解并分析网络流量包之前，我们必须理解网络栈是如何工作的。


## OSI模型

[开放系统互联](http://en.wikipedia.org/wiki/OSI_model) (OSI)模型于1983年发布，是一种概念模型，通过将通信系统内部功能划分为抽象层来表示和标准化内部功能层。

![](http://i.imgur.com/dZyiOTX.png)

协议根据自身功能分离，层次结构使网络通信更加易于理解：



### 第1层: 物理层

表示传输网络数据的物理和电路媒介。

它包括了所有的硬件，集线器，网络适配器，电缆。

### 第2层: 数据链路层

提供了通过物理网络*传输数据*的方法。网桥和交换机是这一层的物理设备。

它负责提供可用于识别物理设备的寻址方案：[MAC地址](http://en.wikipedia.org/wiki/MAC_address)。

举例来说，这一层的协议有：[Ethernet](http://en.wikipedia.org/wiki/Ethernet)，[Token Ring](http://en.wikipedia.org/wiki/Token_ring)，[AppleTalk](http://en.wikipedia.org/wiki/AppleTalk)，和[Fiber Distributed Data Interface](http://en.wikipedia.org/wiki/Fiber_Distributed_Data_Interface) (FDDI)。

### 第3层: 网络层

它负责在物理网络之间路由数据，分配网络主机的*逻辑寻址*。它也负责处理*数据包碎片*和*错误检测*。

路由器和*路由表*属于这一层。这一层的协议有，比如：[Internet Protocol](http://en.wikipedia.org/wiki/Internet_Protocol) (IP)，[Internetwork Packet Exchange](http://en.wikipedia.org/wiki/Internetwork_Packet_Exchange)，和[Internet Control Message Protocol](http://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) (ICMP)。


### 第4层: 传输层

提供两个主机之间的数据流量控制。许多防火墙和配置协议都在这一层工作。

这一层的协议有，比如：[UDP](http://en.wikipedia.org/wiki/User_Datagram_Protocol)和[TCP](http://en.wikipedia.org/wiki/Transmission_Control_Protocol)。

### 第5层: 会话层
负责两台计算机之间的*会话*，管理诸如正常终止连接的操作。它还可以确定连接是[双工或半双工](http://en.wikipedia.org/wiki/Duplex_%28telecommunications%29)。

这一层的协议有，比如：[NetBIOS](http://en.wikipedia.org/wiki/NetBIOS)和[NWLink](http://en.wikipedia.org/wiki/NWLink)。

### 第6层: 表示层

将接受的数据转换为应用层可以读取的格式，例如编码/解码和几种用于保护数据的加密/解密形式。

这一层的协议有，比如：[ASCII](http://en.wikipedia.org/wiki/ASCII)，[MPEG](http://en.wikipedia.org/wiki/Moving_Picture_Experts_Group)，[JPEG](http://en.wikipedia.org/wiki/JPEG)，和[MIDI](http://en.wikipedia.org/wiki/MIDI).

### 第7层: 应用层

为最终用户提供访问网络资源的详细信息。

这一层的协议有，比如：[HTTP](http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)，[SMTP](http://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol)，[FTP](http://en.wikipedia.org/wiki/File_Transfer_Protocol)，和[Telnet](http://en.wikipedia.org/wiki/Telnet).

---

## 数据封装

OSI模型不同层上的协议通过*数据封装*的方式，堆栈中的每个层都向报文添加一个头或者尾。

封装协议创建了一个[协议数据单元](http://en.wikipedia.org/wiki/Protocol_data_unit) (PDU)，包括添加了头和尾信息的数据。我们所说的*包*是完整的协议数据单元。

例如，在Wireshark中，我们可以跟踪更高层的PDU启动和停止的序列号。这允许我们测量传输PDU所需的时间(*显示过滤器*是**tcp.pdu.time**)。


---


## 交换机和路由器
捕获来自在**交换**网络上的目标设备的流量主要有四种方法：使用**集线器**，使用**tap**,通过端口镜像，或者通过ARP欺骗/缓存投毒。前两个方法显然需要一个集线器或者一个tap。端口镜像需要交换机的转发功能。参考资料[1]借鉴了一种决定使用哪种方法的比较好的途径：

![](http://i.imgur.com/aRUfmsp.png)


所有用于交换网络的技术也可以在**路由**网络中使用。但是，对于路由器，嗅探器的放置位置显得更加重要，因为设备的广播域仅在到达路由器之前扩展。

---
## 数据包的类型

在网络中有三种类型的数据包：

* **广播包**: 发送到网段上的所有端口。广播MAC地址是*ff:ff:ff:ff:ff:ff*（第二层）或者可能性最大的IP地址（第三层）。

* **组播包**: 从单源发送到多个目的端，以简化并尽可能少地使用带宽。

* **单播包**: 从一个端系统到另一个。


---
## 各层通用协议

### 地址解析协议（ARP） (第2层)

**逻辑地址**和**物理地址**都用于网络通信。逻辑地址用于多个网络（间接连接的设备）之间的通信。物理地址用于单个网络（如使用交换机彼此连接的设备）之间的通信。


[ARP](http://en.wikipedia.org/wiki/Address_Resolution_Protocol)是用于确定[MAC地址](http://en.wikipedia.org/wiki/MAC_address) (如物理地址为00:09:5B:01:02:03，属于第2层)对应一个特定的IP地址（如逻辑地址为10.100.20.1，属于第3层）的协议。

ARP解析过程使用两个包（*ARP请求*和*ARP响应*）来查找匹配的MAC地址，向域中的每个设备发送一个**广播包**，等待正确的客户端的响应。这样做是基于交换机使用一个MAC表来确定向哪个端口发送流量。

在Wireshark中，用这样的语句可以很容易获得ARP：**"Who has 192.168.11/ Tell 192.168.1.1"**。另外，你可以使用这种方法查询你的设备的ARP表：

```
$ arp -a
```

### Internet协议(第3层)

互联网上的每个接口必须拥有唯一的网络地址。Internet协议基于分组报头中的IP地址，在主机之间传递分组。

[IPv4](http://en.wikipedia.org/wiki/IPv4)地址是用于唯一标识网络中连接的设备的32位地址。它们由4组8个bit的，使用点分十进制表示法表示0~255之间十进制数。

此外，IP地址由两部分组成：**网络地址**和**主机地址**。网络地址表示*局域网*（LAN），主机地址标识该网络中的设备。

这两个部分的确定由另一组寻址信息给出，即**网络掩码**（网络掩码或子网掩码），也是32位长。子网掩码中，标识为1的位属于IP地址中的网络号，剩余位标识了主机号：

![](http://i.imgur.com/a7Evq9z.png)

此外，IP包头包含以下信息：

* **版本号**: 使用的IP版本

* **头长度**: IP头部长度

* **服务类型**: 路由器使用的优先处理流量的标识

* **总长度**: 包含IP头部的数据报的总长度

* **标识位**: 标识包或者分段分组序列

* **分段偏移**: 识别分组是否是段

* **生存时间**: 定义数据包的生命周期，计算通过路由器的速度（跳数/秒）。当一个数据包被创建时，会定义一个TTL，每当数据包被路由器转发是，TTL会减一。

* **协议**: 识别序列中下一个数据包的类型

* **头部校验和**: 错误侦测机制

* **源IP地址**.

* **目的IP地址**.

* **选项**: 路由和时间戳

* **数据**.



### 互联网控制信息协议（ICMP） (第3层)

ICMP是TCP/IP的一个效用协议，负责提供有关设备，服务，或网络上的路由的可用性信息。

使用ICMP的服务有，如：**ping**:

```
$ ping www.google.com
PING www.google.com (74.125.228.210) 56(84) bytes of data.
64 bytes from iad23s23-in-f18.1e100.net (74.125.228.210): icmp_seq=1 ttl=53 time=21.5 ms
64 bytes from iad23s23-in-f18.1e100.net (74.125.228.210): icmp_seq=2 ttl=53 time=22.5 ms
64 bytes from iad23s23-in-f18.1e100.net (74.125.228.210): icmp_seq=3 ttl=53 time=21.4 ms
```


还有**traceroute** (Windows发送ICMP数据包，Linux发送UDP):

```
$ traceroute www.google.com
traceroute to www.google.com (173.194.46.84), 30 hops max, 60 byte packets
 1  * * *
 2  67.59.254.85 (67.59.254.85)  30.078 ms  30.452 ms  30.766 ms
 3  67.59.255.137 (67.59.255.137)  33.889 ms 67.59.255.129 (67.59.255.129)  33.426 ms 67.59.255.137 (67.59.255.137)  34.007 ms
 4  rtr101.wan.hcvlny.cv.net (65.19.107.109)  34.004 ms 451be075.cst.lightpath.net (65.19.107.117)  32.743 ms rtr102.wan.hcvlny.cv.net (65.19.107.125)  33.951 ms
 5  64.15.3.222 (64.15.3.222)  34.972 ms 64.15.0.218 (64.15.0.218)  35.187 ms  35.120 ms
 6  * 72.14.215.203 (72.14.215.203)  29.225 ms  29.646 ms
 7  209.85.248.242 (209.85.248.242)  29.361 ms 209.85.245.116 (209.85.245.116)  39.780 ms  42.108 ms
 8  209.85.249.212 (209.85.249.212)  33.220 ms 209.85.252.242 (209.85.252.242)  33.500 ms  33.786 ms
 9  216.239.50.248 (216.239.50.248)  53.231 ms  57.314 ms 216.239.46.215 (216.239.46.215)  52.140 ms
10  216.239.50.237 (216.239.50.237)  52.022 ms 209.85.254.241 (209.85.254.241)  48.517 ms  48.075 ms
11  209.85.243.55 (209.85.243.55)  56.220 ms  45.359 ms  44.934 ms
12  ord08s11-in-f20.1e100.net (173.194.46.84)  43.184 ms  39.770 ms  45.095 ms
```

traceroute的工作方式是通过发送IP头中有特定功能的回显请求：**the TTL is 1**。这意味着数据包在第一跳后被丢弃。第二个数据包经过第一跳，在第二跳被丢弃(TTL is 2)，依此类推。

为了进行这个工作，路由器用*double-headed packet*来回复响应，包含IP头的副本和在原始回显请求中发送的数据。

PS: 看这篇Julia Evans的文章，学习如何创建一个简单的[*在15行代码内使用python的scapy实现的Traceroute*](http://jvns.ca/blog/2013/10/31/day-20-scapy-and-traceroute/).


### 传输控制协议(第4层)

通过**三次握手**在两个主机之间提供可靠的数据流。目的是使发送主机能够确保目的主机已经启动，并且让发送主机能够检查端口的可用性。

握手是这样进行的：

1. 主机A发送初始的无数据包，发送SYN标识和初始同步序列编号以及双方交互的[最大段长度](http://en.wikipedia.org/wiki/Maximum_segment_size) (MSS)。
2. 主机B返回一个SYN请求和ACK确认标识，及它的初始序列编号。
3. 主机A发送(ACK)确认包。

当通信完成之后，**TCP teardown**进程将开始运行，正常地结束两个设备间的连接。这个过程包含4个包：
1. 主机A发送有FIN+ACK标识的包。
2. 主机B发送一个ACK确认包，再发送一个FIN/ACK包。
4. 主机A发送ACK确认包。


但有时，连接可能会突然终端（比如由于潜在的攻击者发出端口扫描或者由于一个错误配置的主机）。在这些情况下，TCP使用RST标识重置数据包。这表明一个连接突然被关闭了或者连接请求被拒绝了。

还有，在使用TCP协议通信的时候，有65,535个端口可供使用，我们经常把他们分成两组：

* **标准端口**: 1到1023，提供给特定服务使用

* **临时端口**: 1024到65535，由服务随机选择。

最后，TCP头部包含以下信息：

* **源端口**.
* **目的端口**.
* **序列号**: 标识TCP报文段
* **确认号**: 另一个设备返回的包的序列号
* **Flags**: URG, ACK, PSH, RST, SYN, FIN标识，确认正在传输的TCP包的类型
* **窗口大小**: TCP收到的数据包的大小（字节为单位）
* **校验**: 确保TCP头部信息有效性
* **紧急数据点**: CPU应当读取的包中的附加指示信息
* **选项**: 可选字段


### 用户数据报协议(第4层)

TCP被设计用于可靠的数据传输，UDP专注于速度。UDP从一个主机向另一个主机发送**数据报**的数据包，但不保证它们到达另一端。

与TCP不同，UDP不正式建立和终止主机之间的连接。因此，它通常依赖于内置的可靠性服务（如DNS和DHCP等协议）。

UDP报头的字段比TCP少：

* **源端口**.
* **目的端口**.
* **包长度**.
* **确认号**.


###  动态主机配置协议 (第7层)

在互联网发展初期，但设备需要通过网络通信时，它将被手动分配一个地址。

随着互联网的发展，**引导程序协议**（BOOTP）出现，为设备自动分配地址。后来，BOOOTP被DHCP代替。


### 超文本传输协议 (第7层)

HTTP是允许浏览器连接到Web服务器以查看网页的机制。HTTP数据包建立在TCP的顶部，使用八种不同的请求方法中的一个来识别。


------------------------------------------------------


#  在Wireshark中分析数据包

在Wireshark中，整个的网络嗅探过程可以被分为三步：

1.  **收集**:  将选中的网络接口转换为混杂模式，这样可以捕捉到源二进制数据。

2.  **转化**:  收集起来的二进制块被转化成可读的形式。

3.  **分析**: 处理协议类型，通信通道，端口号，协议头。

## 收集数据包
只有当**网络接口**（NIC）被转换为**混杂模式**时，网络流量嗅探是可行的。这允许将所有接收到的流量传输到CPU（而不是处理打算接收的帧）。如果NIC未设置为混杂模式，那么不发送到控制器的数据包将会被丢弃。

##  Wireshark主要的GUI
Wireshark主要的GUI包含四个部分：

* **捕捉中的选项**.
* **数据包列表**: 列出抓取的文件中的所有包。可以编辑以显示包的标号，时间，源，目的，协议，等等。
* **数据包细节**: 层次化的显示单个数据包的信息。
* **Packet Bytes**: 数据包未经处理的形式

为了开始抓包，你所需要做的就是选择网络接口。你也可以在在收集包之前设置*捕捉过滤器*。


## 配色方案

数据包列表面板通过不同的颜色（可设置）显示不同类型的流量。比如：

* 绿色是TCP (于是是HTTP),
* 深蓝是DNS,
* 浅蓝是UDP,
* 浅黄是ARP,
* 黑色标识有问题的TCP数据包。

##  数据包的可视化和统计

Wireshark有几个可以用来学习包和网络的工具：

* **Statistics -> IO Graphs**: 允许图形化的数据吞吐量。例如，你可以使用图形来查找数据中的峰值，发现各个协议中的性能瓶颈，并比较数据流。此接口提供过滤功能（例如，显示ARP和DHCP流量）。

* **Statistics -> TCP -> Stream Graph -> Round Trip Time Graph**: 允许制作给定捕获的文件的 **来回通讯延迟** (RTT)。这是从数据包发送到接收确认信息所需的时间。

* **Statistics -> Flow Graph**: 基于时间轴的通信统计表示（基于时间间隔）。它允许可视化连接和数据随时间的变化。流程图包含主机之间基于列的连接视图，并组织流量。这个分析可以展示慢点（slow point）和瓶颈，并决定是否有延迟。

* **Statistics -> Summary**: 返回一个对于整个进程的各项性能，如接口，抓包延迟和序号，以及数据包的大小。

* **Statistics -> Protocol Hierarchy**: 以 *nodal形式* 展示不同协议的统计数据。它根据所处的层安排协议，将它们以百分比形式展示。比如，如果你值得你的网络通常是15%的流量，如果你看见如50%的值，你就知道有错误产生了。

* **Statistics -> Conversations**: 展示对话中的端点地址。

* **Statistics -> Endpoints**: 和对话相同，显示了从一个IP发出和到达的流量统计。比如，对于TCP，它将是 **SYN, SYN/ACK, SYN**。

* **Edit-> Finding Packet or CTRL-F**: 找到与某个标准相匹配的数据包。这里有三个选项：
    * *Display filter*: 基于表达式的过滤器(例如 **not ip**, **ip addr==192.168.0.10**，或者 **arp**).
    * *Hex value*: 十六进制数据包（例如 00:ff, ff:ff).
    * *String*:  具有文本字符串的数据包(例如 admin 或者 workstation).

* **右击 -> Follow TCP Stream**: 将TCP数据流变成可读的形式（而不是以块存储的数据）。 *红色* 文本表示从源到目的的流量， *蓝色* 表示相反方向的流量。如果你知道流序号（可以用来得到多种数据包的值），你也可以使用下面的过滤器来达到同样的结果：

```
tcp.stream eq <number>
```

* **右击 -> Mark Packet or CTRL+M**: 有助于组织相关数据包。




---
##  过滤器

### Berkeley数据包过滤器语法

Wireshark的过滤是一个非常强大的特性。它使用[Berkeley数据包过滤器](http://en.wikipedia.org/wiki/Berkeley_Packet_Filter) (BFP)语法。语法对应一个有多个 **原语** 组成的 **表达式** 。这些原语可以有一个或多个 **限定符** ，定义如下：

* **Type**:  ID名或者序号(例如： **host**, **net**, **port**).
* **Dir**: 改变发到这个ID名或序号/从此ID名或序号发出的方向 (例如: **src** and **dst**).
* **Proto**: 限制与一个特定的协议匹配(例如: **ether**, **ip**, **tcp**, **udp**, or **http**)

一个原语的示例：
```
dst host 192.168.0.10
```
这里面 **dst host** 是限定符，IP地址是ID。

### 过滤器类型
数据包可以以两种方式过滤：

* **抓包过滤器**: 在捕获数据包的时候指定。这种方法对于大流量的捕获的性能是很好的。
* **显示过滤器**: 应用于现有收集的数据包。这种方法提供了多样功能，因为你可以使用整个数据。

在下面的部分我会展示几个capture和display过滤器的例子。

### 针对主机地址和主机名抓包过滤

* 与主机的IPV4地址相关联的流量（也适用于IPV6网络）。
```
host 172.16.16.150
```

* 发送到或者来自一个网段内的IP地址的流量：

```
net 192.168.0.0/24
```

* 设备的主机名和主机限定符：

```
host testserver
```

* 如果你担心主机的IP地址会发生更改，你可以基于MAC地址过滤：

```
ether host ff-ff-ff-ff-ff-aa
```

* 只有来自特定主机的流量（主机是一个可选择的限定符）：

```
src host 172.16.16.150
```

* 从某个主机发出的所有流量：

```
dst host 172.16.16.150
```

* 只从IP地址为173.15.2.1的主机发送或来自此主机的流量：

```
host 173.15.2.1
```

* 来自一个网段内的IP地址：

```
src net 192.168.0.0/24
```


### 针对端口抓包过滤

* 仅在8000端口上的流量:

```
port 8000
```

* 除了端口443上的所有流量:

```
!port 443
```

* 监听80端口的主机的流量:

```
dst port 80
```

* 一系列端口上的流量:

```
tcp portrange 1501-1549
```

* 在端口80和21进和出的流量:

```
port 80 || port == 21
```

* 只有非http和非SMTP的流量（等同）：

```
host www.example.com and not (port 80 or port 25)
```

### 针对协议抓包过滤

* 捕捉单播流量(有助于回避网络噪声):

```
not broadcast and not multicast
```

*  仅ICMP流量：

```
icmp
```


* 丢弃ARP数据包:

```
!arp
```

* 丢弃IPv6流量:

```
!ipv6
```

* DNS流量:

```
dns
```

* 清空文本邮件流量:

```
smtp || pop || imap
```

### 针对数据包性质抓包过滤

* 设置SYN标识的TCP数据包：

```
tcp[13]&2==2
```

* 目的不可达的ICMP数据包(type 3):

```
icmp[0]==3
```

*  HTTP GET请求(bytes 'G','E','T' 是十六进制值 47, 45, 54):

```
port 80 and tcp[((tcp[12:1] & 0xf0 ) >> 2  ):4 ] = 0x47455420
```

---
### 针对主机地址和主机名进行显示过滤


* 针对IP地址过滤:

```
ip.addr == 10.0.0.1
```

* IP源地址域:

```
ip.src == 192.168.1.114
```

* 一个网段的IP地址的src/dst:

```
ip.addr== 192.168.1.0/24
```

### 针对端口的显示过滤

* 任意以4000作为源端口号和目的端口号的TCP数据包：

```
tcp.port == 4000
```

* 源端口号:

```
tcp.srcport == 31337
```

### 针对协议进行显示过滤

* 丢弃arp, icmp, dns, 或者其他任何可能产生后部噪声的协议：

```
!(arp or icmp or dns)
```

* 显示路径中所有的重传(有助于跟踪性能低的应用表现和丢包):

```
tcp.analysis.retransmission
```

* ICMP类型域来找到所有的PING包:

```
icmp.type== 8
```

### 针对数据包性质进行显示过滤

* 显示所有的HTTP GET请求：

```
http.request
```

* 显示所有的POST请求:

```
http.request.method == "POST"
```

* 对十六进制值进行过滤:

```
udp contains 33:27:58
```

* TCP头部的序列号域:

```
tcp.seq == 52703261
```


* 长度小于128字节的数据包:

```
frame.len <= 128
```

* 设置SYN表示的TCP数据包:

```
tcp.flags.syn == 1
```

* 设置RST标识的TCP数据包:

```
tcp.flags.rst == 1
```

* 显示所有的TCP重置:

```
tcp.flags.reset == 1
```

* 不设置段bit的IP标识(判断是否有人在尝试ping):

```
ip.flags.df == 0
```


--------------------------------------
#  为了安全使用Wireshak


## 一些侦察提示

### 使用SYN扫描网络

TCP SYN扫描是一种快速稳定的扫描网络中端口和服务的方法。它比其他扫描技术少很多噪声。

基本上，它依赖于三次握手进程来决定目标主机上打开的端口：

1. 攻击向受攻击主机上的一系列端口发送一个TCP SYN数据包。

2. 一旦这个数据包被目标主机接收，它将会有如下响应：

    * **开放端口**: 依赖于一个TCP SYN/ACK数据包（三次）。然后攻击者直到端口是打开的，服务正在监听。

    * **关闭端口，未被过滤**: 攻击者接收RST响应。

    * **被过滤的端口** (通过防火墙，比如): 攻击者不接受任何响应。


### 操作系统指纹

在位置的情况下确定设备上的操作系统类型的技术。

在 **被动指纹** 中，攻击者可以使用从目标发送的数据包中的某些字段来制作隐形指纹。

由于协议的[RFCs](http://en.wikipedia.org/wiki/Request_for_Comments)缺乏特异性，允许了这一情况的发生：虽然TCP，UDP和IP头中包含的各个字段非常具体，但没有为这些字段定义默认值。

比如，以下的几个头部值可以帮助你区分几个操作系统：

* **IP, Initial Time to Live**:
    - Linux, Mac OS：64
    - Windows：128
    - Cisco IOS：255
* **IP, Don't Fragment Flag**:
    - Linux, Mac OS, Windows有设置
    - Cisco IOS无
* **TCP, Max Segment Size**:
    - Windows：1440
    - Mac OS 10, Linux：1460
* **TCP, Window Size**:
    - Linux：2920-5840
    - Cisco IOS：4128
    - Mac OS 10：65535
    - Windows：任意值
* **TCP, StackOK**:
    - Linux, Windowns有设置
    - Cisco IOS, Mac OS 10无

注意：一个很好的使用操作系统指纹的技术是[p0f](http://lcamtuf.coredump.cx/p0f3/).

在 **主动指纹** 中，攻击者主动地向受攻击主机发送构建好的数据包，并以回复来判断操作系统类型。这可以使用[Nmap](http://nmap.org/)来完成。

---


## 一些取证方面的提示

### DNS查询

当用户在线的时候查看用户所做的不同的DNS查询。 一个可能的过滤器是：

```
dsn
```

这将显示出在用户未知的情况下完成的恶意的DNS请求。一个例子是，在访问过的网站中找到隐藏的带有恶意脚本的 **iframe**。

### HTTP GET 头部

查看网络活动中的不同HTTP流：HTML, JavaScript,  image traffic, 302 redirections, non-HTTP streams, Java Archive downloads, 等。一个可能的过滤器是：

```
http
```
你也可以使用这个来查看不同的GET请求：

```
tcp contains "GET"
```

### 使用虚拟机检查DNS泄露

在虚拟机中查看 **statistics --> Endponts**，应该只有一个公共IP地址：这个虚拟机连接的VPN服务器。

---
## ARP缓存污染

### Sniffing

ARP缓存污染允许利用Wireshark接入网线。这可以用于好的目的，也可以用于恶意攻击。

这个方法是这样工作的：网络上的所有设备使用IP地址在第3层相互通信。因为交换机在第2层上工作，通常只能看到被缓存的MAC地址。

当MAC地址不在缓存列表中时，ARP广播询问哪个IP地址拥有某个MAC地址。目的机器通过ARP应答回复数据包及其MAC地址（如上所述）。因此，这个时候，发送计算机具有能够寻址需要与远程计算机通信的信息的数据链路层，然后将信息存储在ARP缓存中。

攻击者可以通过向具有假MAC地址的以太网交换机或路由器发送ARP消息来构造此过程，以拦截另一台计算机的流量。

在Linux，可以使用[arpspoof或Ettercap](http://www.irongeek.com/i.php?page=security/arpspoof)来进行ARP欺骗。例如，如果你的wlan0在192.168.0.10，路由器在192.168.0.1，你可以运行：

```
$ arpspoof -i wlan0 -t 192.168.0.10 192.168.0.1
```

如果你使用Windows，你可以使用[Cain和Abel](http://www.oxid.it/cain.html)来构建ARP缓存污染。


### Denial-of-Service

在需求非常高的网络中，当你重新路由流量时，目标系统发送和接收的所有信息必须首先通过分析系统。这使您的分析系统称为通信过程中的瓶颈，可能会引起[DoS](http://en.wikipedia.org/wiki/Denial-of-service_attack).

你可能可以使用一个[非对称路由](http://www.cisco.com/web/services/news/ts_newsletter/tech/chalktalk/archives/200903.html)的特性来躲避在你的分析系统上传输的所有流量。

---
## 无线嗅探

### The 802.11 频谱

从 **无线局域网** (WLAN) 中捕获流量的特殊之处在于无线频谱是一个 **共享介质** （与有线网络不同，每个客户端都有自己的电缆连接到交换机）。

单个WLAN占用[802.11频谱](http://en.wikipedia.org/wiki/IEEE_802.11)的一部分，允许多个系统在同样的物理介质中运行。在美国，有11个通道可用，WLAN在一段时间内只能运行在一个通道中(嗅探也是一样）。.

但是，一个叫做 **channel hopping** 的技术允许通道间数据的收集和交换。用来完成这个的工具是[kismet](https://www.kismetwireless.net/)，速度可达10通道/秒。



### 无线网卡模式

网卡有四种模式：

* **Managed**: 无线客户端与无线接入点(WAP)直接连接。

* **ad hoc mode**:  设备之间直接连接，与WAP共享功能。

* **Master mode**: 无线网卡与特定的软件合作来使计算机允许其他设备使用WAP。

* **Monitor**: 用来禁止传输和接收数据，并开始监听传输中的数据包。

为了获得Linux中的无线扩展你可以输入：

```
$ iwconfig
```

为了接口（比如eth1）转换为监听模式，你可以输入：

```
$ iwconfig eth1 mode monitor
$ iwconfig eth1 up
```

改变端口通道：

```
$ iwconfig eth` channel 4
```





-------
## 更多相关资料：

- [Wireshark wiki](http://wiki.wireshark.org/)
- [Practical Packet Analysis, ](http://wiki.wireshark.org/)
- [Wireshark plugin  for writing dissectors in Python](https://github.com/ashdnazg/pyreshark)
- [Using Wireshark ti check for DNS Leaks](https://lilithlela.cyberguerrilla.org/?p=76081)
- [Publicly available PCAP files](http://www.netresec.com/?page=PcapFiles)
- [Malware PCAP files](http://contagiodump.blogspot.se/2013/08/deepend-research-list-of-malware-pcaps.html)

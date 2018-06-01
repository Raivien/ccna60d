# 第39天

**开放最短路径优先协议**

**OSPF**

## 第39天任务

- 阅读今天的课文（以下）
- 复习昨天的课文
- 完成今天的实验
- 阅读ICND2记诵指南
- 在网站[subnetting.org](http://subnetting.org/)上学习15分钟

与EIGRP一样，对OSPF的讨论也可以花上几天时间，但这里需要针对那些考试要用到的知识点，进行着重学习。CCNA级别的OSPF知识，尚不足以在大多数网络上涉及与部署该路由技术。

今日将学习以下内容：

- OSPF原理
- DR与BDR
- OSPF的配置
- OSPF故障排除

这一课对应了以下CCNA大纲的要求：

- OSPF的配置与验证（单区，single area）
- 邻居临接（Neighbour adjacencies, 或者叫邻居关系的形成）
- OSPF的各种状态
- 对多区的讨论（discuss multi-area）
- OSPFv2的配置
- 路由器ID
- LSA（链路状态通告）的类型


## 指定与后备指定路由器（Designated and Backup Designated Routers）

如同第12天模块中所指出的，OSPF会在广播与非广播网络类型上选举出指定路由器（DR）与/或非后备指定路由器（BDR）。对后备指定路由器并非这些网络类型上的强制性组件这一点的掌握，是重要的。事实上，仅选出指定路由器，而没有后备指定路由器时，OSPF同样能工作；只是在指定路由器失效时，没有冗余而已，同时网络中的OSPF路由器需要再度进行一遍选举流程，以选出新的指定路由器。

在网段上（广播或非广播网络类型），所有非指定/后备指定路由器，都将与指定路由器与选出的后备指定路由器（若有选出后备指定路由器）建立临接关系，而不会与网段上的其它非指定/后备指定路由器形成临接关系。这些非指定/后备指定路由器，将往全指定路由器多播组地址（the AllDRRouters Multicast group address）`224.0.0.6`发出报文与更新。只有指定路由器与后备指定路由器才会收听发到此组地址的多播报文。随后指定路由器将通告报文给全SPF路由器多播组地址（the AllSPFRouters Multicast group address）`224.0.0.5`。这就令到网段上的所有其它OSPF路由器接收到更新。

对于已选出指定路由器与/或后备指定路由器时报文交换顺序的掌握尤为重要。下面作为一个示例，请设想一个有着4台路由器，分别为`R1`、`R2`、`R3`与`R4`，的广播网络。假定`R4`被选作指定路由器，`R3`被选作后备指定路由器。那么`R1`与`R2`就既不是指定也不是后备指定路由器了，因此在思科OSPF命名法中就被成为`DROther`路由器。此时`R1`上有一个配置改变，随后`R1`就发出一个更新报文到`AllDRRouters`多播组地址`226.0.0.6`。指定路由器`R4`接收到该更新，并发出一个确认报文（acknowledgement），回送到`AllSPFRouters`多播组地址`224.0.0.5`上，`R4`随后使用`AllSPFRouters`多播组地址，将该更新发送给所有其它非指定/后备指定路由器。该更新被其它的`DROther`路由器，也就是`R2`接收到，同时`R2`发出一个确认报文到`AllDRRouters`多播组地址`224.0.0.6`。该过程在下图39.1中进行了演示：

![OSPF指定与后备指定通告](images/3901.png)

*图39.1 -- OSPF的指定与后备指定通告*

> **注意：** 后备指定路由器仅简单地收听发送到`224.0.0.5`（`AllSPFRouters` 多播组地址）与`224.0.0.6`（`AllDRRouters`多播组地址）两个多播组地址的数据包


某台路由器要成为网段的指定路由器或后备指定路由器，其就必须被选中。选举基于以下两点：

- 有着最高的路由器优先级值（the highest router priority value）
- 有着最高的路由器ID（the highest router ID）

默认下所有路由器都有着默认的优先级值`1`。可使用接口配置命令`ip ospf priority <0-255>`对该值进行调整。路由器的优先级值越高，那么其就越有可能被选为该网段的指定路由器。有着第二高优先级的路由器，则将被选为后备指定路由器。如配置了优先级值`0`，则该路由器将不参加到DR/BDR的选举过程。最高路由器优先级与路由器ID原则，仅在参与指定/后备指定路由器选举过程的所有路由器同时加载OSPF进程时才起作用。如这些路由器没有同时加载OSPF进程，则最先完成OSPF进程加载的路由器，将成为网段上的指定路由器。

在确定OSPF路由器ID时，思科IOS将选取已配置的环回接口的最高IP地址。在没有配置环回接口时，IOS软件将使用所有已配置的物理接口的最高IP地址，作为OSPF路由器ID。思科IOS软件还允许管理员使用路由器配置命令`router-id [address]`，来手动指定路由器ID。

重要的是记住在OSPF下，一旦指定路由器与后备指定路由器被选出，那么在进行另一次选举之前，它们都将始终作为指定/后备指定路由器。比如，在某个多路访问网络上已存在一台指定路由器与一台后备指定路由器的情况下，一台有着更高优先级或IP地址的路由器被加入到该同一网段时，既有的指定与后备指定路由器将不会变化。在指定路由器失效时，接过指定路由器的角色的是后备指定路由器，而不是新加入的有着更高优先级或IP地址的路由器。此外，一次新的选举将举行，而那台路由器极有可能被选举为后备指定路由器（Instead, a new election will be held and that router will most likely be elected BDR）。而为了让那台路由器成为指定路由器，就必须将后备路由器移除，或使用`clear ip ospf`命令对OSPF进程进行重置，以强制进行一次新的指定/后备指定路由器选举。一旦完成选举，OSPF像下面这样来使用指定与后备指定路由器：

- 用于减少网段上所要求的临接关系（To reduce the number of adjacencies required on the segment）
- 用于对多路访问网段上的路由器进行通告（To advertise the routers on the Multi-Access segment）
- 用于确保所有更新送达网段上的所有路由器


为了更好地掌握这些基础概念，这里参考下图39.2中的基本OSPF网络拓扑：

![OSPF指定与后备指定路由器基础](images/3902.png)

*图 39.2 -- OSPF 指定与后备指定路由器基础*

在图39.2中，网段上的各台路由器与指定及后备指定路由器之间建立临接关系，但相互之间并不建立临接关系（Referencing Figure 39.2, each router on the segment establishes an adjacency with the DR and the BDR but not with each other）。也就是说，非指定/后备指定路由器之间不会建立临接关系。这一特性阻止网段上的路由器形成相互之间的`N(N-1)`个临接关系，从而降低过多的OSPF数据包在网段上泛滥。

比如在没有网段上的指定/后备指定路由器概念时，各台路由器都需要与网段上的其它路由器建立临接关系。对于图39.2的情形，这将导致网段上的`4(4-1)`，也就是`12`个临接关系。但有了指定/后备指定路由器后，每台路由器只需与这两台路由器，而无需与其它非指定与后备指定路由器，建立临接关系。指定路由器与后备指定路由器之间也会建立临接关系。此特性降低了网段及各台路由器上的临接关系数目，进而降低各台路由器上资源消耗（比如内存与处理器使用）。

对于第二点，OSPF将一条链路，视为两台路由器或两个节点之间的连接。在多路访问网络，比如以太网中，多台路由器可处于同一网段上，就如同图39.2中所演示的那样。在这样的网络中，OSPF使用网络链路状态通告（Network Link State Advertisement, Type 2 LSA, 类型2的链路状态通告），来对多路访问网段上的路由器进行通告。这种链路状态通告是由指定路由器生成，并仅在该区域传播。因为其它非指定/后备指定路由器并不在各自之间建立临接关系，所以此类链路状态通告就令到那些路由器知悉在该多路访问网段上的其它路由器了。

为进一步说明这一点，这里参考图39.2, 假定该网段上的所有路由器都具有默认的OSPF优先级`1`（并同时加载OSPF进程），因为`R4`有着最高的路由器ID而被选为指定路由器。`R3`因为有着第二高的路由器ID而被选为后备指定路由器。因为`R2`与`R1`既不是指定也不是后备指定路由器，因此它们为称为思科命名法中的`DROther`路由器。可在所有路由器上使用`show ip ospf neighbour`命令对此进行验证，如下所示：

```sh
R1#show ip ospf neighbor
Neighbor ID     Pri     State           Dead Time   Address         Interface
2.2.2.2           1     2WAY/DROTHER    00:00:38    192.168.1.2     Ethernet0/0
3.3.3.3           1     FULL/BDR        00:00:39    192.168.1.3     Ethernet0/0
4.4.4.4           1     FULL/DR         00:00:38    192.168.1.4     Ethernet0/0

R2#show ip ospf neighbor
Neighbor ID     Pri     State           Dead Time   Address         Interface
1.1.1.1           1     2WAY/DROTHER    00:00:32    192.168.1.1     FastEthernet0/0
3.3.3.3           1     FULL/BDR        00:00:33    192.168.1.3     FastEthernet0/0
4.4.4.4           1     FULL/DR         00:00:32    192.168.1.4     FastEthernet0/0
R3#show ip ospf neighbor
Neighbor ID     Pri     State           Dead Time   Address         Interface
1.1.1.1           1     FULL/DROTHER    00:00:36    192.168.1.1     FastEthernet0/0
2.2.2.2           1     FULL/DROTHER    00:00:36    192.168.1.2     FastEthernet0/0
4.4.4.4           1     FULL/DR         00:00:35    192.168.1.4     FastEthernet0/0

R4#show ip ospf neighbor
Neighbor ID     Pri     State           Dead Time   Address         Interface
1.1.1.1           1     FULL/DROTHER    00:00:39    192.168.1.1     FastEthernet0/0
2.2.2.2           1     FULL/DROTHER    00:00:39    192.168.1.2     FastEthernet0/0
3.3.3.3           1     FULL/BDR        00:00:30    192.168.1.3     FastEthernet0/0
```

>  **注意：** 那些`DROther`路由器之所以处于`2WAY/DROTHER`状态，是因为它们仅与指定及后备指定路由器交换它们的数据库。那么就因为`DROther`路由器之间没有完整的数据库交换，所以它们绝不会达到OSPF完整临接状态（The DROther routers remain in the `2WAY/DROTHER` state because they exchange their databases only with the DR and BDR routers. Therefore, because there is no full database exchange between the DROther routers, they will never reach the OSPF FULL adjacency state）。

因为`R4`已被选为指定路由器，它就生成网络链路状态通告（the Network LSA），这类链路状态通告，是就该多路访问网段上的其它路由器进行通告的。可在网段上的任意路由器上，使用`show ip ospf database network [link state ID]`命令，或在指定路由器上使用`show ip ospf database network self-originate`命令，对此加以验证。下面演示了在指定路由器（`R4`）上命令`show ip ospf database network self-originate`命名的输出：

```sh
R4#show ip ospf database network self-originate
            OSPF Router with ID (4.4.4.4) (Process ID 4)
                Net Link States (Area 0)
  Routing Bit Set on this LSA
  LS age: 429
  Options: (No TOS-capability, DC)
  LS Type: Network Links
  Link State ID: 192.168.1.4 (address of Designated Router)
  Advertising Router: 4.4.4.4
  LS Seq Number: 80000006
  Checksum: 0x7E08
  Length: 40
  Network Mask: /24
        Attached Router: 4.4.4.4
        Attached Router: 1.1.1.1
        Attached Router: 2.2.2.2
        Attached Router: 3.3.3.3
```

参考上面的输出，指定路由器（`R4`）发起了表示`192.168.1.0/24`子网的类型2（网络）链路状态通告（the Type 2(Network) LSA）。因为该子网上存在多台路由器，所以该`192.168.1.0/24`子网被称作OSPF命名法中的一条传输链路（a transit link in OSPF terminology）。输出中的通告路由器字段（the Advertising Router field）显示了生成此链路状态通告的那台路由器（`4.4.4.4`, `R4`）。网络掩码字段（the Network Mask field）则显示了该传输网络的子网掩码，也就是24位，或`255.255.255.0`。

> 注：OSPF中链路类型（link type）有4种：P2P、Stub、Transit与Virtual link; 网络类型（network type）有两种：传输网络（Transit network）与末梢网络（Stub network）；链路状态通告有六种：Router LSA（一类）、Network LSA（二类）、Network summary LSA（三类）、ASBR summary LSA（四类）、AS external LSA（五类） 与 NSSA LSA（七类）。[参考链接](http://blog.51cto.com/xiaojiejt/1941362)。

所连接路由器字段（the Attached Router field）列出了在该网络网段上所有路由器的路由器ID。这样就令到该网段上的所有路由器，知悉有哪些其它路由器也同样位处该网段上。下面的输出，演示了在`R1`、`R2`与`R3`上的`show ip ospf database network [link state ID]`命令的输出，反映出同样的信息：

```sh
R2#show ip ospf database network
            OSPF Router with ID (2.2.2.2) (Process ID 2)
                Net Link States (Area 0)
  Routing Bit Set on this LSA
  LS age: 923
  Options: (No TOS-capability, DC)
  LS Type: Network Links
  Link State ID: 192.168.1.4 (address of Designated Router)
  Advertising Router: 4.4.4.4
  LS Seq Number: 80000006
  Checksum: 0x7E08
  Length: 40
  Network Mask: /24
        Attached Router: 4.4.4.4
        Attached Router: 1.1.1.1
        Attached Router: 2.2.2.2
        Attached Router: 3.3.3.3


R1#show ip ospf database network
            OSPF Router with ID (1.1.1.1) (Process ID 1)
                Net Link States (Area 0)
  Routing Bit Set on this LSA
  LS age: 951
  Options: (No TOS-capability, DC)
  LS Type: Network Links
  Link State ID: 192.168.1.4 (address of Designated Router)
  Advertising Router: 4.4.4.4
  LS Seq Number: 80000006
  Checksum: 0x7E08
  Length: 40
  Network Mask: /24
        Attached Router: 4.4.4.4
        Attached Router: 1.1.1.1
        Attached Router: 2.2.2.2
        Attached Router: 3.3.3.3
            OSPF Router with ID (4.4.4.4) (Process ID 4)


R3#show ip ospf database network
            OSPF Router with ID (3.3.3.3) (Process ID 3)
                Net Link States (Area 0)
  Routing Bit Set on this LSA
  LS age: 988
  Options: (No TOS-capability, DC)
  LS Type: Network Links
  Link State ID: 192.168.1.4 (address of Designated Router)
  Advertising Router: 4.4.4.4
  LS Seq Number: 80000006
  Checksum: 0x7E08
  Length: 40
  Network Mask: /24
        Attached Router: 4.4.4.4
        Attached Router: 1.1.1.1
        Attached Router: 2.2.2.2
        Attached Router: 3.3.3.3
```

网络链路状态通告的功能及其与其它类型的链路状态通告，特别是与路由器链路通告（类型一，the Router LSA(Type 1)）的关系，将在本模块稍后进行详细介绍。本小节的重点，应放在对指定路由器就多路访问网段上的网络链路状态通告的生成与通告，从而完成对位处该同一网段上的其它路由器的通告，这一过程的理解上。这是因为网段上的路由器仅与指定及后备指定路由器建立临接关系，而相互之间并不建立临接关系。在没有相互之间的临接关系下，路由器就绝不会知道该多路访问网段上的其它非指定/后备指定路由器。

最后，有关指定/后备指定路由器上的第三点，指定/后备指定路由器确保了网段上的所有路由器都有着完整的数据库。非指定/后备指定路由器将更新发送到多播组地址`224.0.0.6`（`AllDRRouter`）。那么指定路由器就通过将该更新发送到多播组地址`224.0.0.5`（`AllSPFRouters`），将这些更新通告给其它非指定/后备指定路由器。下图39.3演示了从`R1`（一台`DROther`）发往指定路由器组地址，涉及图39.2中的那些路由器的一个更新：

![发到指定/后备指定路由器组地址的一个`DROther`更新](images/3903.png)

*图 39.3 -- 发到指定/后备指定路由器组地址的一个`DROther`更新*

`R4`（指定路由器）收到该更新，并接着将相同更新发送到多播组地址`224.0.0.5`（`AllSPFRouters`）。该组地址是由所有OSPF路由器使用的，以确保网段上的所有其它路由器都收到此更新。下图39.4对发自`R4`（指定路由器）的该更新进行了演示：

![到OSPF组地址的指定路由器更新](images/3904.png)

*图 39.4 -- 到OSPF组地址的指定路由器更新*

> **注意**：可以看出这就是来自`R1`的更新，因为图39.3与图39.4中的通告路由器字段（the Advertising Router field）都包含了`R1`的路由器ID（the router ID, RID）, 也就是`1.1.1.1`。


> **注意**：OSPF使用到的其它LSA类型，将在本模块的后面详细介绍。

## 额外的路由器类型（Additional Router Types）

除了多路访问网段上的指定与后备指定路由器外，对OSPF路由器的描述方式，还包括根据它们的位置与在OSPF网络中的作用。在OSPF网络中，通常会发现以下额外的路由器类型：

- 区域边界路由器（Area Border Routers）
- 自治系统边界路由器（Autonomous System Boundary Routers）
- 内部路由器（Internal Routers）
- 骨干路由器（Backbone Routers）


下图39.5演示了一个后两个区域--一个OSPF骨干区域（the OSPF backbone area(`Area 0`)）与一个额外的一般OSPF区域（a additional normal OSPF area(`Area 2`)）, 构成的基本OSPF网络。`R2`有着一个与`R1`的外部边界网关协议邻居关系（an external BGP neighbour relationship）。该图例将用于描述此网络中不同的OSPF路由器类型。

![额外的OSPF路由器类型](images/3905.png)

*图 39.5 -- 额外的OSPF路由器类型*

区域边界路由器（An Area Border Router, ABR），是一台将一个或多个OSPF区域，连接到OSPF骨干的OSPF路由器。这就意味着其必须有一个接口在`Area 0`中，同时有其它接口在某个不同的OSPF区域中。区域边界路由器是所有其归属区域的成员，且它们保有着每个其所归属区域的一个单独链路状态数据库（ABRs are members of all areas to which they belong, and they keep a seperate Link State Database for every area to which they belong）。参考图39.5, `R3`就应被认为是一台区域边界路由器，因为它将`Area 2`连接到了OSPF骨干`Area 0`。

而传统意义上的自治系统边界路由器，则是位处路由域的边沿，且定义了内部与外部网络的边界（An Autonomous System Boundary Router(ASBR), in the traditional sense, resides at the edge of the routing domain and defines the boundary between the internal and the external networks）。参考图39.5, `R2`将被认为是一台自治系统边界路由器。除了注入来自其它协议（比如BGP）的路由信息外，在某台路由器将静态路由或是所连接的子网，注入到OSPF网络时，也可将其划分为自治系统边界路由器。

内部路由器的所有运作接口，都保持在单个的OSPF区域中。基于图39.5中演示的网络拓扑，`R4`将被视为一台内部路由器，因为其仅有的接口，出于单个的OSPF区域中。

骨干路由器是那些有一个接口在OSPF骨干中的路由器。骨干路由器可以包括那些有着仅在OSPF骨干区域的接口的路由器，或者有一个接口在OSPF骨干区域，也有接口在其它区域的路由器（也就是区域边界路由器）。基于图39.5中演示的拓扑，路由器`R2`与`R3`都可被视为骨干路由器。

> **注意**：OSPF的路由器可有多个角色。比如上面的`R2`就同时是一台自治系统边界路由器及骨干路由器，`R3`又同时是一台骨干路由器与区域边界路由器。贯穿本模块，将详细审视这些类型的路由器与其在OSPF域中的角色与功能。

## OSPF数据包类型

OSPF路由器发出的不同类型数据包，包含在这些数据包共有的、24字节的OSPF头部（The different types of packets sent by OSPF routers are contained in the common 24-byte OSPF header）。尽管对该OSPF头部细节的深入，超出了CCNA考试要求的范围，但对该头部中所包含的各个字段，以及它们各自用途的基本掌握，仍然重要。下图39.6对此各种数据包共有的24个八位OSPF头部，进行了演示：

![OSPF协议数据包头部](images/3906.png)

*图 39.6 - OSPF协议数据包头部*

其中的8位版本字段，指出了OSPF的版本。该字段的默认值是`2`。但在开启了OSPFv3时，该字段就被设置为`3`。在第13天时，对OSPFv3进行了详细介绍。

接着的8位类型字段，用于指明该OSPF数据包的类型。五种主要的OSPF数据包类型，将在本课程模块接下来进行介绍，它们是：

- 类型1 = `Hello`数据包
- 类型2 = 数据库描述数据包（Database Description packet）
- 类型3 = 链路状态请求数据包（Link State Request packet）
- 类型4 = 链路状态更新数据包（Link State Update packet）
- 类型5 = 链路状态确认数据包（Link State Acknowledgement packet）

随后的16位数据包长度字段，是用于指明该协议数据包的长度。此长度包括了标准的OSPF头部。

下面的32位路由器ID自动，用于指明发出数据包的路由器的IP地址。在思科IOS设备上，该字段将包含运行OSPF的设备上配置的所有物理接口的最高的IP地址。如在设备上配置了环回接口（Loopback interfaces），那么该字段将包含所有配置的环回接口的最高IP地址。或者在显式地有管理员配置或指定了路由器ID时，该字段也可包含那个手动配置的路由器ID。

>  **注意**: 除非重启了路由器，或者获取IP地址的那个接口被关闭或移除，抑或在路由器上使用了提权的`EXEC`命令`clear ip ospf process`命令重置了OSPF进程，否则在路由器ID被选出后，该路由器ID都不会发生改变。

接下来的32位区域ID（Area ID），用于区分该数据包的OSPF区域（the OSPF area）。数据包只能属于单个OSPF区域。在数据包是通过虚拟链路（a virtual link）接收到的时，那么区域ID就会是OSPF的骨干区域，也就是`Area 0`。本课程模块后面后对虚拟链路进行介绍。

校验和字段是16位长的，它指出了该数据包完整内容，从OSPF头部开始，但排除了64位的认证数据字段的标准IP校验和。如该数据包的长度不是正数个的16位字（16-bit words）长时，则会在进行检验和检查钱，以全`0`字节加以补充。

其后的16位认证类型字段（The 16-bit Authentication(Auth) Type field）指出所使用的认证的类型。该字段仅对OSPFv2有效，且可能包含以下3个代码之一：

- `Code 0` - 意思是空（`0`）认证，也就是没有认证；这是默认选项
- `Code 1` - 表明认证类型是普通文本（the authentication type is plain text）
- `Code 2` - 意思是认证类型为消息摘要算法（MD5, Message Digest Algorithm）

OSPF头部最后的64位认证数据字段，则是在开启了认证时，用于具体的认证信息或数据。重要的是记住 **该字段仅对OSPFv2有效**。在使用的是普通文本认证时，该自动包含了认证密钥（the authentication key）。但在使用的是MD5认证时，该自动就被重新定义为几个其它字段，不过这超出了CCNA考试要求范围。下图39.7显示了线路上捕获到的OSPF数据包的不同字段：

![OSPF数据包头部的线上捕获](images/3907.png)

*图 39.7 - OSPF数据包头部的线上捕获*

在OSPF数据包头部里头，8位的类型字段用于指明OSPF数据包的类型。这里再度说明一下，如下所示的5种OSPF数据包类型：

- 类型1 = `Hello`数据包
- 类型2 = 数据库描述数据包（Database Description packet）
- 类型3 = 链路状态请求数据包（Link State Request packet）
- 类型4 = 链路状态更新数据包（Link State Update packet）
- 类型5 = 链路状态确认数据包（Link State Acknowledgement packet）


### OSPF `Hello` 数据包

`Hello`数据包用于发现其它直接相连的OSPF路由器，以及在OSPF路由器之间建立OSPF临接关系（OSPF adjacencies between OSPF routers）。对于广播及点对点网络，OSPF使用多播来发送`Hello`数据包。这些数据包被投送到`AllSPFRouters`多播组地址`224.0.0.5`。对于非广播链路（比如帧中继），OSPF使用单播（Unicast）来将`Hello`数据包直接发送给那些静态配置的邻居。

> **注意**：默认情况下，所有OSPF数据包（也就是包括多播与单播），都是以IP存活时间（TTL, Time To Live）`1`发送的。这就将这些数据包限制到本地链路。也就是说，无法与距离远于一挑的另一台路由器建立OSPF临接关系。这一点也适用于EIGRP。

OSPF的`Hello`数据包，还在广播链路上用于指定路由器与后备指定路由器的选举。指定路由器仅侦听多播地址`224.0.0.6`（`AllDRRouters`）。本课程模块前面已经介绍了指定与后备指定路由器。下图39.8演示了OSPF`Hello`数据包中所包含的字段：

![OSPF的`Hello`数据包](images/3908.png)

*图 39.8 - OSPF的`Hello`数据包*

其中 **4字节的网络掩码字段**包含了通告OSPF接口的子网掩码（ **The 4-byte Network Mask field** contains the subnet mask of the advertising OSPF interface）。只有在广播介质上，才会检查子网掩码。对于本模块后面会介绍的未编号的点对点接口及虚拟链路，该字段将被设置为`0.0.0.0`。

后面两字节的`Hello`字段，显示`Hello`时间间隔，也就是两个`Hello`数据包之间的秒数，通告路由器要求此字段。其取值范围为`1`到`255`。在广播与点对点介质上的默认值为`10`，在所有其它介质上的默认值为`30`。

随后1字节的选项字段（The 1-byte Options field）是由本地路由器用于通告可选功能（optional capatibilities）。选项字段中的每一位，都表示了不同的功能。深入了解这些位所对应的功能，超出了CCNA考试要求范围。

后面的1字节路由器优先级字段（The 1-byte Router Priority field）包含了本地路由器的优先级。默认该字段的值为`1`。该值用于指定与后备指定路由器的选举。可能的取值范围为`0`到`255`。优先级越高，那么该本地路由器就越有机会成为指定路由器。优先级值`0`就意味着该本地路由器不会参与指定或后备指定路由器的选举。

接下来的4字节指定路由器字段，列出指定路由器的IP地址。在比如某个点对点链路上，或当某台路由器已被显式地配置为不参与此选举时，于尚无指定路由器选出时，就使用值`0.0.0.0`。

其后的4字节后备指定路由器字段，标识出后备路由器，并列出当前后备指定路由器的接口地址。在没有选出后备指定路由器时，就使用值`0.0.0.0`。

最后的（活动）邻居字段（the (Active) Neighbour field）是一个可变长度的字段，显示网段上的所有已接收到`Hello`数据包的OSPF路由器。

### 数据库描述数据包（Database Description Packets）

在各台OSPF路由器对其本地数据库信息进行通告时，于数据库交换期间，就要用到数据库描述数据包。这些数据包通常被称作DBD数据包或DD数据包。第一个DBD数据包用于数据库交换过程的主从选举。DBD数据包还包含了由主路由器选定的初始序列编号（The first DBD packet is used for the Master and Slave election for database exchange. The DBD packet also contains the initial sequence number selected by the Master）。有着较高路由器ID的路由器，成为主路由器并发起数据库同步过程。只有主路由器才能增加DBD数据包中的序列编号。主路由器开启数据库交换，并对从路由器进行信息轮询。数据库交换中的主从选举，是在邻居对的基础上进行的。

明白主从选举过程不同于指定与后备指定路由器的选举过程，尤为重要。通常后错误地假定它们一致（This is commonly incorrectly assumed）。主从选举过程只基于有着最高IP地址的路由器（两台邻居路由器之间）一条原则；但指定与后备指定路由器选举过程，则由IP地址或优先级值两个因素决定。

比如这里假设，两台名为`R1`与`R2`的路由器开始了临接关系建立过程。`R1`有着路由器ID`1.1.1.1`，同时`R2`有着路由器ID`2.2.2.2`。网络管理员将`R1`的OSPF优先级值配置为`255`以确保该路由器被选举为指定路由器。在主从关系确定过程中，`R2`因为有着较高的路由器ID优势，而将被选举为主路由器。但在`R1`上配置的优先级值，导致`R1`被选举为指定路由器。而实际上，在主从选举过程中，作为指定路由器的`R1`就可作为从路由器。

在选出了主从路由器后，本地路由器就通过往对方路由器发送LSA头部，而使用DBD数据包对本地数据库进行概括（After the Master and Slave have been elected, DBD packets are used to summarise the local database by sending LSA headers to the remote router. LSA，Link-State Advertisement, 链路状态通告）。远端路由器分析这些头部，以判断在其自己的LSDB拷贝中是否缺少什么信息。下图39.9中对数据库描述数据包进行了演示：

![OSPF的数据库描述数据包](images/3909.png)

*图 39.9 - OSPF的数据库描述数据包*

在DBD数据包中，两字节的接口MTU字段包含了发出接口的8位二进制的MTU值（the 2-byte Interface MTU field contains the MTU value, in octets, of the outgoing interface）。也就是说，该字段包含了通过相关接口所能发送的最大数据大小（以字节计）。当在虚拟链路上使用接口时，该字段就被设置为值`0x0000`。有了成功建立OSPF的邻居临接关系，所有路由器上的MTU必须一致。如在一台路由器上修改了这个值，就必须在相同子网的所有其它路由器上配置同样的值（或使用`ip ospf mtu-ignore`命令）。

> **注意**：对于EIGRP来说，不必为了成功建立EIGRP的邻居关系，而要求接口MTU一致。

随后的1字节选项字段，包含的是与OSPF`Hello`数据包相同的选项。为简明起见，不再对这些选项进行描述。

其后的数据库描述或标志字段，是一个1字节的、在临接关系形成过程中，提供某台OSPF路由器可否就多个DBD数据包，与邻居进行交换的能力的字段（The Database Description or Flags field is a 1-byte field that provides an OSPF router with the capability to exchange multiple DBD packets with a neighbour during an adjacency formation）。

接着的4字节DBD序列号字段（The 4-byte Sequence Number field），通过使用一个序列号，而用于确保所有DBD数据包在同步过程中，得以接收与处理。主路由器在第一个DBD数据包中，将该字段初始化为一个独特值，其后的每个数据包的序列号都增加`1`。序列号的增加，仅由主路由器进行。

最后的可变长度LSA头部字段（the variable length LSA Header），运送的是描述本地路由器信息的多个LSA头部。每个头部长度为20个8位二进制数，并对数据库中的各个LSA进行唯一地区别。每个DBD数据包可包含多个LSA头部。


### 链路状态请求数据包（Link State Request Packets）

链路状态请求数据包，是由OSPF路由器发送的，用以请求缺失的或过期的数据库信息。这些数据包包含了对所请求的链路状态通告进行独特描述的标识符。单个的链路状态请求数据包可能包含了请求多条链路状态通告的单个的标识符集或多个的标识符集。链路状态请求数据包还用于在数据库交换之后的，对数据库交换期间本地路由器不曾有的那些链路状态通告的请求。下图39.10对OSPF的链路状态请求数据包格式的演示：

![OSPF链路状态请求数据包](images/3910.png)

*图 39.10 - OSPF链路状态请求数据包*

其中的4字节链路状态通告类型字段（The 4-byte Link State Advertisement Type field）包含了所请求的链路状态通告类型。其可包含下列字段之一：

- 类型1 = 路由器链路状态通告
- 类型2 = 网络链路状态通告
- 类型3 = 网络汇总链路状态通告
- 类型4 = 自治系统边界路由器链路状态通告
- 类型5 = 自治系统外部链路状态通告
- 类型6 = 多播链路状态通告
- 类型7 = 次末梢区域外部链路状态通告（Not-So-Stubby Area, NSSA）
- 类型8 = 外部属性链路状态通告（External Attributes Link State Advertisement）
- 类型9 = 本地链路的不透明链路状态通告（Opaque LSA - Link Local, *目前主要用于MPLS多协议标签交换协议）
- 类型10 = 区域的不透明链路状态通告（Opaque LSA - Area, *目前主要用于MPLS多协议标签交换协议）
- 类型11 = 自治系统的不透明链路状态通过（Opaque LSA - Autonomous System, *目前主要用于MPLS多协议标签交换协议）

> **注意**：一些上面列出的链路状态通告将在后面的小节进行讲解。

4字节的链路状态ID字段，编码了特定于LSA的信息。包含在该字段的信息根据LSA的种类而有所不同。最后的4字节通告路由器字段，包含的是最先发起LSA的路由器的路由器ID。

### 链路状态更新数据包（Link State Update Packets）

链路状态更新（LSU）数据包是由路由器用于对链路状态通告进行通告的数据包（advertise Link State Advertisements）。链路状态更新数据包可以是到OSPF邻居的单播，作为从邻居处接收到链路状态请求的回应。然而最为常见的是，它们被可靠地在整个网络中泛洪到`AllSPFRouters`多播组地址`224.0.0.5`，直到所有路由器都有一份数据库的拷贝位置。所泛洪的更新，于随后在链路状态通告的确认数据包中加以确认。如链路状态通告未被确认，就会默认每隔5秒加以重传。下图39.11展示了一个发送给某个邻居的、作为LSR响应的链路状态更新数据包：

![单播的LSU数据包](images/3911.png)

*图 39.11 - 单播的LSU数据包*

下图 39.12 演示了一个可靠地泛洪到多播组地址`224.0.0.5`的LSU：

![多播LSU数据包](images/3912.png)

*图 39.12 - 多播LSU数据包*

链路状态更新数据包由两部分构成。第一部分是4字节的链路状态通告数目字段（the 4-byte Number of LSAs field）。该字段显式了LSU数据包中所运送的LSA条数。第二部分则是一条或多条的链路状态通告。此可变长度字段包含了完整的LSA。每种类型的LSA都有共同的头部格式，与其各自特定的用来描述各自信息的数据字段。一个LSU数据包可包含单一的LSA或多条的LSA。

### 链路状态确认数据包（Link State Acknowledgement Packets）

链路状态确认数据包（LSAck）用于对各条LSA进行确认及作为对LSU数据包的响应。通过显式地使用链路状态确认数据包来对泛洪的数据包加以确认，OSPF所使用的泛洪机制被认为是可靠的。

链路状态确认数据包包含了一般的OSPF头部，以及随后的一个LSA头部清单。此可变长度字段允许本地路由器以单个数据包对多条LSA进行确认。链路状态确认数据包是以多播发送的。在多路访问网络上，如果发送LSAck的是指定或后备指定路由器，那么这些LSAck就被发送到多播组地址`224.0.0.5`（`AllSPFRouters`）。而如果发送LSAck的不是指定或后备指定路由器，那么这些LSAck数据包就被发送到多播组地址`224.0.0.6`（`AllDRRouters`）。下图39.13对LSAck的格式进行了演示：

![链路状态确认数据包](images/3913.png)

*图 39.13 - 链路状态确认数据包*

总之，重要的是记住不同的OSPF数据包类型及它们所包含的信息。这将不仅有助于考试，也可在将OSPF作为一个协议的整个运作进行掌握的过程中有所裨益。

在思科IOS软件中，可使用`show ip ospf traffic`命令来查看OSPF数据包的统计信息。该命令展示了发送及接收道德OSPF数据包的总数，并将这些OSPF数据包细分到单独的OSPF进程，最终又细分到具体进程下开启了OSPF进程的各个接口上。该命令也可用于对OSPF临接关系建立的故障排除，其作为调试用途时，不是处理器占用密集的方式。下面的输出中演示了该命令所打印的信息：

```sh
R4#show ip ospf traffic
OSPF statistics:
  Rcvd: 702 total, 0 checksum errors
        682 hello, 3 database desc, 0 link state req
        12 link state updates, 5 link state acks
  Sent: 1378 total
        1364 hello, 2 database desc, 1 link state req
        5 link state updates, 6 link state acks

            OSPF Router with ID (4.4.4.4) (Process ID 4)
OSPF queue statistics for process ID 4:
                   InputQ     UpdateQ    OutputQ
  Limit            0          200        0
  Drops            0          0          0
  Max delay [msec] 4          0          0
  Max size         2          2          2
    Invalid        0          0          0
    Hello          0          0          1
    DB des         2          2          1
    LS req         0          0          0
    LS upd         0          0          0
    LS ack         0          0          0
  Current size     0          0          0
    Invalid        0          0          0
    Hello          0          0          0
    DB des         0          0          0
    LS req         0          0          0
    LS upd         0          0          0
    LS ack         0          0          0

Interface statistics:
    Interface Serial0/0
OSPF packets received/sent
    Invalid  Hellos   DB-des    LS-req  LS-upd   LS-ack   Total
Rx: 0        683      3         0       12       5        703
Tx: 0        684      2         1       5        6        698
OSPF header errors
  Length 0, Auth Type 0, Checksum 0, Version 0,
  Bad Source 0, No Virtual Link 0, Area Mismatch 0,
  No Sham Link 0, Self Originated 0, Duplicate ID 0,
  Hello 0, MTU Mismatch 0, Nbr Ignored 0,
  LLS 0, Unknown Neighbor 0, Authentication 0,
  TTL Check Fail 0,
OSPF LSA errors
  Type 0, Length 0, Data 0, Checksum 0,

  Interface FastEthernet0/0
OSPF packets received/sent
    Invalid  Hellos   DB-des    LS-req  LS-upd   LS-ack   Total
Rx: 0        0        0         0       0        0        0
Tx: 0        682      0         0       0        0        682
OSPF header errors
  Length 0, Auth Type 0, Checksum 0, Version 0,
  Bad Source 0, No Virtual Link 0, Area Mismatch 0,
  No Sham Link 0, Self Originated 0, Duplicate ID 0,
  Hello 0, MTU Mismatch 0, Nbr Ignored 0,
  LLS 0, Unknown Neighbor 0, Authentication 0,
  TTL Check Fail 0,
OSPF LSA errors
  Type 0, Length 0, Data 0, Checksum 0,

Summary traffic statistics for process ID 4:
  Rcvd: 703 total, 0 errors
        683 hello, 3 database desc, 0 link state req
        12 link state upds, 5 link state acks, 0 invalid
  Sent: 1380 total
        1366 hello, 2 database desc, 1 link state req
        5 link state upds, 6 link state acks, 0 invalid
```

## 临接关系的建立（Establishing Adjacencies）

运行OSPF的路由器在建立临接关系之前，会经历几种状态。在这些状态期间，路由器要交换不同类型的数据包。这些报文交换令到所有路由器建立起临接关系，以具备网络的持久视图。随后对当前网络的变更，就增量更新发送出去。这些状态分别是：`Down`、`Attempt`、`Init`、`2-way`、`Exstart`、`Exchange`、`Loading`以及`Full states`，如下所示：

- `Down`状态就是所有OSPF路由器的开始状态。然而，即便在所指定的路由器死亡间隔，那个接口尚未接收到`Hello`数据包，本地路由器仍可在此状态中显示出一个邻居（However, the local router may also show a neighbour in this state when no Hello packets have been recieved within the specified router dead interval for that interface）。
- `Attempt`状态仅对那些非广播多路访问网络上的OSPF邻居有效。在该状态中，已发出了一个`Hello`数据包，但尚未在死亡间隔中接收到来自静态配置的邻居的信息；但会尽力与该邻居建立临接关系。
- 在OSPF路由器接收到来自邻居的`Hello`数据包，而本地路由器ID并未在接收到的邻居字段（the received Neighbor field）中列出时，就到了`Init`状态。如OSPF`Hello`数据包的参数不匹配，比如各种计时器值等，那么OSPF路由器就再也不会进到此状态之后的状态了。
- `2-way`状态表明OSPF邻居之间的双向通信（各台路由器已看到其它路由器的`Hello`数据包）。在该状态中，本地路由器已接收到一个在邻居字段中有着其自己的路由器ID的`Hello`数据包，同时两台路由器上的`Hello`数据包参数也是一致的。在此状态时，路由器就确定是否与这个邻居成为临接。在多路访问网络上，指定与后备指定路由器在此阶段得以选举出来。
- `Exstart`状态用于数据库同步过程的初始化。本地路由器与其邻居在这个阶段确立何者负责数据库同步过程。在该状态中主从路由器被选举出来，同时在该阶段DBD交换的首个顺序编号有主路由器确定下来。
- `Exchange`状态就是路由器使用DBD数据包对它们的数据库内容进行描述的地方。各个DBD序列被显式的确认，同时一次只允许一个突出的DBD。在此期间，LSR数据包已被发出，以请求LSA的一个新的实例（Each DBD sequence is explicitly acknowledged, and only one outstanding DBD is allowed at a time. During this phase, LSR packets are also sent to request a new instance of the LSA）。在此阶段，`M`（更多）位被用于请求缺失的信息（The `M`(More) bit is used to request missing information during this stage）。在两台路由器都完成了其完整数据库的交换后，它们将把该`M`位设置为`0`。
- 在`Loading`状态，OSPF构造出一个LSR与链路状态重传清单。LSR数据包被发出，以请求某个LSA的较近期的、尚未在`Exchange`过程中接收到的实例。在此阶段，发出的更新被置于链路状态重传清单之上，直到本地路由器接收到确认为止。如本地路由器在阶段又接收到LSR，那么它将以包含了所请求信息的链路状态更新予以响应。
- `Full`状态表明OSPF的邻居们已经完成了它们整个数据库的交换，且都达成一致（也就是它们有着网络的同样视图）。处于该状态的两台路由器就将该临接关系加入到它们的本地数据库，并就此关系在链路状态更新数据包中加以通告。到这里，路由表被计算出来，或在临接关系被重置后被重新计算出来。`Full`正是一台OSPF路由器的正常状态。如果某台路由器被卡在了另一状态，那么就表明临接关系的形成中存在故障。对此的唯一例外就是`2-way`状态，该状态对于其中路由器仅到达指定或后备指定的`Full`状态的广播与非广播多路访问网络，就是所谓的正常状态。其它邻居总是将各自视为`2-way`的。

为了成功建立临接关系，两台路由器上的一些参数必须匹配。包括以下这些参数：

- 接口的MTU值（可被配置为忽略）
- `Hello`与死亡计时器
- 区域ID
- 认证类型与口令
- 末梢区域标志（The Stub Area flag）
- 兼容的网络类型（Compatible network types）

本课程模块将陆续对这些参数进行介绍。如这些参数不匹配，那么OSPF的临接关系将绝不会完整建立。

> **注意**：处理不匹配的参数，还要记住在多路访问网络上，如两台路由器都配置了优先级值`0`，那么临接关系也不会建立。在这类网络上，必须要有指定路由器（The DR must be present on such network types）。


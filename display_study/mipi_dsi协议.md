# MIPI DSI协议知识分享

## MIPI接口简介

MIPI(Mobile Industry Processor Interface)是2003年由ARM、Nokia、ST、TI等公司成立的一个联盟，目的是把手机内部的接口如摄像头、显示屏接口、射频/基带接口等标准化，从而减少手机设计的复杂程度增加设计灵活性。MIPI联盟下面有不同的WorkGroup，分别定义了一系列的手机内部接口标准，比如熟知的摄像头接口CSI、显示接口DSI等。
除了MIPI协议，显示屏的接口协议还包括：VGA、HDMI、DVI、LVDS，详细介绍可参考https://www.jianshu.com/p/df46e4b39428

![图 0](../images/b1343a833a99b2f72ca9e19f40b389cf685ddeeb9a9f0bf4536d05fb5192d909.png)  



## MIPI协议结构分层

![图 1](../images/f31400af2aaa8215b61756ac45b19a8dec06adce7849790a533bd34fe9af1915.png)  

从下往上依次为物理层、通道管理层、协议层、应用层，四层各司其职。下面先简单介绍下四层的作用，后面再重点讲解这四层的工作原理。
1. 物理层：硬件电路控制，将所有的数据转换为电平信号，以输出或者输入。
2. 通道管理层：对数据进行管理，将数据分给给Data通道或者从Data通道接收到后进行重组，以便交给下一层，由于有多个data lane,发送端数据传输分配在这里进行。
3. 协议层：对数据进行封包、检验等。发送端打包数据,包头包尾错误校验数据等。接受端拆包解析出真正的数据,并对数据完整性进行检查。
4. 应用层：command数据或者图像数据就在这一层发送或者接收。我们使用的显示屏DSI接口应属于应用层。

## 物理层（D-PHY）

### 关于DPHY

MIPI物理层主要包括D-PHY、C-PHY、M-PHY，显示模块使用的是D-PHY协议。
D-PHY：采用的是主从结构，即一个lane通道中同一时刻必须存在一个主设备，一个从设备。一般由一个clock 信号线和一到多个data信号线构成，每根lane都传输差分信号，分为Dp线和Dn线。

```c
差分信号：
差分信号介绍：差分信号分为一正一负，两者之间相位差180度，可以抑制共模干扰，还可以提升信号幅度。
差分信号优点：抗干扰能力强，能有效抑制外部的电磁干扰
```
![图 2](../images/d796239ad4b60f8f8d83229f489aac458adc6fcae2d04dbddcceff8e5412ebd7.png)  

### 传输模式
按照D-PHY协议，在整个协议的物理层中，在主机端和从属端之间采用的是同步连接，时钟通道用于传送高速时钟，一个或多个数据通道用于传送低功耗数据信号或高速数据信号。

![图 3](../images/682d355f9257e769f0351832cb1401385d1ebe2a499907800c288e62baf10926.png)  

1. 高速传输模式(Burst Mode)
在高速模式下，输出低摆幅差分信号；当没有数据传输时，data lane处于lp11模式，Dn、Dp都处于高电平状态，当有数据传输时，通过一定的时序进入HS模式，当经过数据传输完毕之后，同样按照一定的时序退出，进入control mode的stop状态。
![图 4](../images/ee9f28d05c6b378871070216816dc9674d1a0f1a758f7a0f4f5754f65650a8d3.png)  
![图 5](../images/d990104e12f55d85a383f569b77bdc030ed32a503439d79d7615cefc67090dc9.png)  

2. 低速传输模式(escape mode) ：发低速command常会使用
![图 6](../images/099fc9be4797425ff265b165b528d2c7c09907711335c792cebb084482a81a39.png)  

1. 高速模式（Burst Mode）
高速传输(HSDT)进入时序:LP-11, LP-01, LP-00,也称之为SoT(Start-of-Transmission)，相应退出高速的时序为EoT(End-of-Transmission)。
![图 7](../images/ca8c646dc8595ae27517488605f30337d081db3cb2addda52861947e580bc6c4.png)  
![图 8](../images/94aa66f0ca58dc127729c6de11a9bfe9541787cabf45a91c58d65250b7cbccc0.png)  

2. Escape mode
Escape mode 工作于LP mode下，能够进入LPDT(Low-Power Data Transmission 低功耗数据传输模式)。在执行Escape Mode Entry时序后,紧接着要发送Entry Command表明使用哪种功能。
![图 9](../images/2a1c7ceafa16216243a7801ca6787bc051cd5c86c5e149c0a3eef7e8e7dce677.png)  
 Escape mode 进入时序：LP11→LP10→LP00→LP01→LP00，退出时序：LP10→LP11
3. 控制模式（Control mode）中转状态
MIPI协议规定，将控制模式的4个不同状态组成不同时序，用来代表着将要进入或者退出某种模式。比如LP11-LP01-LP00序列后，进入高速模式。

```c
总结
High-Speed Mode和Escape Mode之间不可以直接来回切换，必须通过Control Mode进行中转，即High-Speed Mode ↔ Control Mode ↔ Escape Mode
如果没有进行高速发数据，lane通常处于LP11状态，也即Control模式种的Stop状态。
两种模式的退出时序：
• Escape mode request (LP-11→LP-10→LP-00→LP-01→LP-00) 
• Escape mode exit (LP-10→LP-11) 
• High-Speed mode exit (T hs-trail -> T hs-exit)  
• High-Speed mode request (LP-11→LP-01→LP-00) 
```

## 应用层

### DSI协议简介
DSI全称Display Serial Interface。顾名思义，该接口是指用于显示模块的一个串行接口，基于MIPI协议而产生，兼容DPI(显示像素接口，Display Pixel Interface)、DBI(显示总线接口，Display Bus Interface)和DCS(显示命令集，Display Command Set)。

![图 10](../images/432da911aa0e851824aac7e978a2018363394588afab73279c161e606986d6ee.png)  

### 操作模式
DSI支持两种基本的操作模式:command Mode和Video Mode
1. command Mode
Command Mode适用于包含RAM的DDIC模块。一般而言,AP将显示数据发送到DDIC的RAM中,DDIC从RAM中读取数据刷新到屏幕上。
![图 12](../images/e45421997a0ccca645f52089fed9781e62e5c895efb2a9d50544e6229a18c483.png)  
2. Video Mode
Video Mode适用于不包含RAM的DDIC模块,需要实时发送数据。
![图 13](../images/a079eb974359b5a996223382986fd49347a383b5f7af4f48d1bfcb67b9793625.png)  

### 数据传输
1. 发送packet
DSI 将一次发送动作称为一个Transmission，一个Transmission可以包含多个packet。
每写一个寄存器可以称为一个packet，单个packet发送如下：
![图 14](../images/2e2f27b67386acf3398213aa962cc27af50c0219c7180334cfe3e6f889f36ce1.png)  
我们可以将多个packet一起发送,而不用每一个分开发送,以提升效率。合并发送如下:
![图 15](../images/f4a6519035e9501daf1a48eaed97a3d6382cf396b0c8e0649d431f4adf2b8693.png)  
另外，为了提升鲁棒性，DSI又定义了一个EoT Packet，简称EoTp。之前的EoT是从HS mode退出到LP Mode的时序，而EoTp是一个短数据包，接收端在收到EoTp之后即知道发送完成。
2. Packet按照大小分为短包和长包。
短包:Short packets,长度4个字节,包含ECC。短包通常用于发送command。
![图 16](../images/8e596160423ab5a6d3a32039d95d5787659c3915b69d10fb6e5eab899f0eb77c.png)  
长包:Long packets，多用于传输大块的显示数据或参数很多的命令
长包构成如下:
i![图 17](../images/5d2fc0f28f780d64484b49579d5e9c1d93261e31a9aacbfd36455ae7c8f8d6b0.png)  
![图 18](../images/12ada04e75d5089c68412e96f19a3eac0267d709fa14c8b6ca606e6408957ab8.png)  

## 通道管理层(Lane Management)
由于有多个data lane，发送端数据传输分配在这里进行。DSI支持多条data lane,在高速传输时数据分布如下:
![图 19](../images/d0c0b6be6143dfb00ef70d612a808451f63fdaff80af5de814d913f5e800ad80.png)  
![图 20](../images/6540cd2a278aee2081c1f0fd913f65d3323b33cd22352a4d81d9c8de8899bf0e.png)  
高速传输的数据字节数长度是不定的,就存在一个问题,当字节数不能整除data lane数时,数据分布处理如下,以两条data lane举例:
![图 21](../images/576f6f9bd2b4b681480d343b0a833fdaa63f0c149f24fbc6af5dcf9490d1c464.png)  
## DCS指令集
DSI协议声明支持的标准指令集，用于发送 pixel 数据，配置屏端的一系列功能。
DCS 定义了很多很多的 Command，每个 Command 都有 Command Code，有的 Command 带参数，有的不带，DCS 的 List 如下所示：
![图 22](../images/92155beaa4f97861037314b351a99495cebe2ca31e07ad940ef81582a65dbb77.png)  
![图 23](../images/c22625f06a78cb4c830d91e48c91a13481aa1c20e59ae01b37c4f5a26354a339.png)  
![图 24](../images/bd5562d4b043bdb83fb7a0f9ecfa5cd5ebe71982f59508e250f63719236e47a1.png)  

1. 39h 进入idle mode（aod mode）38h 退出idle mode（aod mode）
发送这条命令，使得 Display Module 进入 Idle 模式， 明显可以看到，Idle 模式下，色彩信息被减少。
![图 25](../images/4eee5d031dc884826a041872131a7b1e7324616d4f9650e180f2cdf3f4501b7e.png)  
2. 21h enter_invert_mode 颜色设置中有一个反转，他的规则就是RGB的三个值，分别与255相减取绝对值：rgb(200, 55, 35)的反转颜色为rgb(55, 200, 220)
3. Sleep In (10h)、Sleep Out (11h)
![图 26](../images/101630c7c7b7c5f249a4feb255b6ed8cb1941ef4f5460f10e3d0e5c934042fce.png)  
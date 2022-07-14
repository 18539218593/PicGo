# MIPI CSI-2学习笔记

## 一、CSI-2概述

CSI-2规范定义了发送器和接收器之间的标准数据传输和控制接口。定义了两种高速串行数据传输接口选项（D-phy、C-phy）。

|               | **C-PHY**             | **D-PHY**                                           |
| ------------- | --------------------- | --------------------------------------------------- |
| 时钟模式      | 嵌入时钟              | 同步时钟                                            |
| 信道编码      | 状态编码              | 时钟双沿采样                                        |
| 最小PIN数     | 1lane TX:  3pins (TX) | 1lane DATA  + 1lane CLK:  2pins(DATA)  + 2pins(CLK) |
| 最大PIN数     | 3lanes TX:  9pins(TX) | 4lanes DATA + 1lane CLK:  8pins(DATA)  + 2pins(CLK) |
| 最大速率/Lane | 5.7Gbps               | 2.5Gbps                                             |
| 最大速率      | 17.1Gbps              | 10Gbps                                              |

相机控制接口CCI（Camera Control Interface）的两个物理层选项是一个双向控制接口兼容I2C标准。

注意：从CSI-2 v3.0规范开始，Lane1 D-PHY或C-PHY链路互连相机主机或应用程序处理器(例如,Data1 + / Data1在下图中，或Data1_A / Data1_B / Data1_C如图所示)可以是双向的。对于这种链接，不需要支持物理上独立的CCI。

![image-20220316101143654](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220316101143654.png)

![image-20220316101318574](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220316101318574.png)

## 二、CSI-2层级定义

![image-20220316101556666](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220316101556666.png)

上图定义了CSI-2中通常使用的概念层结构。各层可划分为6层:

* PHY Layer：PHY层指定传输介质（导体），输入/输出电路和时钟机制，即从串行位流中获取“0”“1”信号。规范中的这一部分记录了传输介质的特性，并依据时钟和数据通道之间发信号和产生时钟的关系规定了电器参数。

  信号传输开始（SoT）和传输结束（EoT）的机制被规范化。同样被规范化的还有其他在传输和接收物理层之间能够传输的“out of band”信息（传输层协议使用带外数据（out-of-band，OOB）来发送一些重要的数据，如果通信一方有重要的数据需要通知对方时，协议能够将这些数据快速地发送到对方。为了发送这些数据，协议一般不使用与普通数据相同的通道，而是使用另外的通道。）。位级和字节级同步机制被包含位PHY的一部分。

* Protocol Layer：协议层由其他几个任务明确的层组成。CSI-2协议层允许多数据流共用一个主机处理器端信号接口。协议层指定多数据流怎样被标记和交叉存取，因此每个数据流可以被正确的重建。

  * Pixel/Byte Packing/Unpacking Layer ：CSI-2支持多种像素格式图像应用，包括从6位到24位每个像素的数据格式。在发射端，数据由本层被发送到LLP层（Low Level Protocol）前，本层将应用层传来的数据由像素打包成字节数据；在接收端，执行相反过程，将LLP层发来的数据解包，由字节转成像素，然后才发送到应用层。8bit的像素的数据在本层被传输时不会被改变。

  * Low Level Protocol：LLP指的是SoT与EoT之间的数据包字节流协议，LLP的最小单元为字节。

  * Lane Management.：CSI-2是可扩展的lane，可以提高性能。数据通道的数量不受此规范的限制，可以根据应用程序的带宽要求选择。接口的发送端将发送的数据流中的字节分配（“distributor” function）到一个或多个Lanes上。在接收端，接口从Lanes收集字节，并将它们合并(“merger” function)为重新组合的数据流，恢复原始的流序列。对于C-PHY物理层选项，该层专门分配或收集数据通道之间的字节对((i.e. 16-bits)。基于每个Lane的置乱是一个可选特性，详细说明在后文。

  协议层中的数据被组织成数据包。接口的发送端向要在 Low Level Protocol layer传输的数据追加报头和错误检查信息。在接收端，头在Low Level Protocol layer被剥离，并由接收端相应的逻辑解释。错误检查信息可用于测试输入数据的完整性。

* Application Layer：这一层描述了包含在数据流中的数据的高级编码和解释，超出了本规范的范围。CSI-2规范描述了像素值到字节的映射。

规范的标准段落只与连接的外部部分有关，例如，只和数据和位模式被传输通过的链路有关。所有的内部接口和层次都是纯信息性的。

## 三、摄像头控制接口(CCI)

CCI 是一种用于控制发送器（transmitter）的两线制双向半双工串行接口。 CCI 兼容 I2C 快速模式 (Fm) 或快速模式 Plus (Fm+) [NXP01] 变体，以及 I3C [MIPI03] 接口的单数据速率 (SDR) 或双数据速率 (DDR) 协议。 CCI 应支持高达 400kbps (Fm) 的操作和 7 位从机寻址。 此外，CCI 可以选择支持高达 1Mbps (Fm+)、12.5Mbps (SDR) 或 25Mbps (DDR)。

CCI 可以在有或没有 CSI-2 over C/D-PHY 的情况下使用。 当 CCI 用作 CSI-2 总线的一部分时，CSI-2 接收器应配置为Master，CSI-2 发送器应配置为Slave。 当 CCI 不使用 CSI-2 over C/D-PHY 时，Master应用作Master。 CCI 能够处理总线上的多个Slave。这里通俗的来解释，因为CCI为i2c的子集，i2c是主从协议，所以当使用mipi相机与soc平台链接使用时，soc平台是接受端为主机，而camera为发送端作为Slave。如果我们soc平台仅使用CCI的场景时，soc平台就是Master。

在 CCI (I2C) 中，不支持多主模式。 

在 CCI (I3C) 中，应支持任何 I3C 强制功能和“必需”CCC 命令，并且可能支持任何 I3C 可选功能和命令（例如，多主控、带内中断、热加入）。

注意：不要将CCI术语主从与C-PHY或D-PHY规格中的类似术语混淆，他们没有关系。

通常，在发送端和接收端之间有一个专用的CCI接口。

CCI 是 I2C 或 I3C 协议的子集，包括 I2C 或 I3C 规范中指定的 I2C/I3C 从设备的强制性功能的最小组合。 因此，符合 CCI 规范的发送器也可以连接到系统 I2C 或 I3C 总线。 但是，必须注意不要让 I2C 或 I3C 主设备尝试使用 CCI 主设备或从设备不支持的 I2C 或 I3C 功能。

CCI 发送器可能具有支持 I2C 或 I3C 的附加功能，但这取决于实现。 更多详细信息可在特定器件的数据表中找到。

本规范不试图定义 CCI 主机发送的控制消息的内容。 因此，实现者有责任定义一组控制消息和相应的帧时序以及 CCI 主机在向 CCI 从机发送此类控制消息时必须满足的任何 I2C 或 I3C 延迟要求。

CCI 在 I2C 或 I3C 之上定义了一个额外的数据协议层。

.............................................................关于CCI的具体内容，留着后续学习记录.......................................................

## 四、物理层

CSI-2通道管理层分别与[MIPI01]和[MIPI02]中描述的D-PHY和/或C-PHY物理层进行接口。设备应实现C-PHY 2.0或D-PHY 2.5 物理层中的任何一种，并且可以实现两者。一个实际的约束是，链路两端使用的PHY技术需要匹配:D-PHY发射器不能与C-PHY接收器工作，反之亦然。

### 1. D-PHY物理层选项

用于CSI-2实现的D-PHY物理层通常由多个单向数据通道和一个时钟通道组成。所有实现D-PHY物理层的CSI-2发射器和接收器应支持时钟Lane上的连续时钟行为，并可选地支持非连续时钟行为。

对于连续的时钟行为，时钟通道保持在高速模式，在数据包传输之间产生主动时钟信号。

![img](https://img-blog.csdn.net/20151109203747442)

对于非连续时钟行为，时钟通道在数据包传输之间进入 LP-11 状态。

![img](https://img-blog.csdn.net/20151109203931363)

![image-20220316140703508](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220316140703508.png)

CSI-2发射机对D-PHY物理层的最低要求是：

* Data Lane Module: Unidirectional master, HS-TX, LP-TX and a CIL-MFEN function 

![image-20220316144845800](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220316144845800.png)

* Clock Lane Module: Unidirectional master, HS-TX, LP-TX and a CIL-MCNN function

![image-20220316144455074](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220316144455074.png)

CSI-2接收机对D-PHY物理层的最低要求是：

* Data Lane Module: Unidirectional slave, HS-RX, LP-RX, and a CIL-SFEN function 

![image-20220316144906446](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220316144906446.png)

* Clock Lane Module: Unidirectional slave, HS-RX, LP-RX, and a CIL-SCNN function 

![image-20220316144701084](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220316144701084.png)

所有支持 D-PHY 物理层选项的 CSI-2 实现都应支持所有 D-PHY 数据通道上的前向转义 ULPS。

为了实现更高的数据速率和更多的通道数，[MIPI01] 中描述的物理层在接收数据通道模块中包含了一个独立的去偏移机制。 为便于在接收器处进行纠偏校准，发射器数据通道模块提供了纠偏序列模式。

由于偏斜度校准仅在给定的发射频率下有效:

* 对于初始校准序列，发送器应编程为校准所需的频率。 然后它将发送去偏移校准模式，接收器将自动检测该模式并调整去偏移功能以实现最佳性能。

* 对于任何发送器频率的改变，都应重新进行偏斜校准。

* 一些发射器或接收器可能需要定期重新运行去偏斜校准，建议在垂直或帧消隐周期内进行最佳校准。

* 对于低发射频率或当 [MIPI01] 中描述的接收器与不支持纠偏校准模式的先前版本的发射器配对时，可能会指示接收器绕过纠偏机制。

D-PHY v2.5 物理层 [MIPI01] 提供备用低功耗 (ALP Alternate Low Power) 模式和低压低功耗 (LVLP Low V oltage  Low Power) 信令，其中任何一个都可以选择替换传统的低功耗状态 (LPS Low Power State)。 使用 ALP 模式或 L VLP 信号可以帮助缓解图像传感器和应用处理器的电流泄漏和电气过载问题。 ALP 模式还有助于实现 CSI-2 成像接口通道的更远距离，并且也是第 9.12 节中描述的 CSI-2 统一串行链路 (USL Unified Serial Link) 功能的核心。 USL D-PHY 支持要求在第 7.3.1 节中描述。

### 2. C-PHY物理层选项

CSI-2 实现的 C-PHY 物理层通常由一个或多个单向lane。

CSI-2 发送器通道模块的最低 C-PHY 物理层要求是：

* Unidirectional master, HS-TX, LP-TX and a CIL-MFEN function
* Support for Sync Word insertion during data payload transmission

CSI-2 接收器通道模块的最低 C-PHY 物理层要求是：

* Unidirectional slave, HS-RX, LP-RX, and a CIL-SFEN function 
* Support for Sync Word detection during data payload reception

所有支持C-PHY物理层选项的CSI-2实现应支持所有C-PHY Lanes上的前向逃逸（forward escape  ）ULPS 。

C-PHY 物理层提供备用低功耗 (ALP) 模式和低压低功耗 (L VLP) 信令，其中任何一个都可以选择性地替换传统的低功耗状态 (LPS)。 使用 ALP 模式或 LVLP 信号可以帮助缓解图像传感器和应用处理器的电流泄漏和电气过载问题。 ALP 模式（通过使用高速嵌入式代码取代 LVLP 或传统 LP 信号）还有助于在需要重新驱动器和重新定时器之前实现 CSI-2 成像接口通道的更远距离。 ALP 模式也是第 9.12 节中描述的 CSI-2 统一串行链路 (USL) 功能的核心。 USL C-PHY 支持要求在第 7.3.2 节中描述。

### 3. PHY支持CSI-2统一串行链路(USL)特性

CSI-2 USL 特性，如第 9.12 节所述，要求 D-PHY 和 C-PHY 物理层支持Lane 1 上的双向数据通信，以及分别在第 7.3.1 节和第 7.3.2 节中描述的附加特性 。如果本第 7.3 节与第 7.1 节或第 7.2 节有任何冲突，则以本节为准。 所有 USL 实现的物理层应支持 PHY LP 和/或 L VLP 模式信令，并应支持 ALP 模式信令。

#### 3.1 对USL特性的D-PHY支持要求

CSI-2 USL 实现的 D-PHY 物理层由一个双向数Data Lane（即Data Lane 1）、零个或多个单向Data Lane和一个Clock Lane组成。 所有为 USL 特性实现 D-PHY 物理层的 CSI-2 发送器和接收器都应支持时钟通道上的连续时钟行为，并且可以选择支持非连续时钟行为。

对于连续的时钟行为，时钟通道保持在高速模式，在数据包传输之间产生主动时钟信号。

对于非连续时钟行为，时钟通道可能会在数据包传输之间进入停止状态，这由图像传感器确定并由主机处理器控制。

USL图像传感器的最小D-PHY LP/LVLP模式物理层要求为:

* Clock Lane Module: Unidirectional master, HS-TX, LP-TX, and CIL-MCNN function
* Data Lane 1 Module: Bidirectional master, HS-TX, LP-TX, LP-RX, LP-CD, and CIL-MFAA function; Escape Mode LPDT shall be supported in both the forward and reverse direction 
* Data Lane n Module (for n > 1): Unidirectional master, HS-TX, LP-TX, and CIL-MFEN 772 
   function 773 

USL主机对D-PHY LP/LVLP模式物理层的最低要求为:

* Clock Lane Module: Unidirectional slave, HS-RX, LP-RX, and CIL-SCNN function 775 
* Data Lane 1 Module: Bidirectional slave, HS-RX, LP-TX, LP-RX, LP-CD, and CIL-SFAA function; Escape Mode LPDT shall be supported in both the forward and reverse direction 
* Data Lane n Module (for n > 1): Unidirectional slave, HS-RX, LP-RX, and CIL-SFEN function 

对于使用 D-PHY LP/LVLP 模式实现的 USL，正向逃生模式（forward direction Escape Mode） LPDT 传输应仅使用Data Lane 1，所有反向传输应仅使用Data Lane 1 和 LPDT。 USL 主机应能够接收 LPDT 和高速 (HS) 传输。 请注意，使用 LPDT 进行传输时，传输带宽会大大降低。

USL图像传感器对D-PHY ALP模式物理层的最低要求为:

* Clock Lane Module: Unidirectional master, HS-TX and CIL-MCNN function
* Data Lane 1 Module: Bidirectional master, HS-TX, HS-RX, ALP-ED, and CIL-MREN function
   (CIL-MREE with ALP-ULPS in both forward and reverse direction is recommended) 
* Data Lane n Module (for n > 1): Unidirectional master, HS-TX and CIL-MFEN function

USL主机对D-PHY ALP模式物理层的最低要求为:

* Clock Lane Module: Unidirectional slave, HS-RX, ALP-ED, and CIL-SCNN function
* Data Lane 1 Module: Bidirectional slave, HS-TX, HS-RX, ALP-ED, and CIL-SREN function (CIL-SREE with ALP-ULPS in both forward and reverse direction is recommended) 
* Data Lane n Module (for n > 1): Unidirectional slave, HS-RX, ALP-ED, and CIL-SFEN function 

请注意，D-PHY ALP 模式没有为双向通道模块定义争用检测功能（contention detection function）。 支持 D-PHY 物理层选项的所有 USL 实现应支持所有数据通道上的正向 ULPS。 对于数据通道 1，建议同时支持反向 ULPS 和反向 ALP 唤醒脉冲传输； 更多指导见第 9.12.5.6 节和第 9.12.5.8 节。

#### 3.2 C-PHY对USL特性的支持要求

CSI-2 USL实现的C-PHY物理层由一个双向Lane(即Lane 1)和零个或多个单向Lane组成。

USL图像传感器对C-PHY LP/LVLP模式物理层的最低要求为:

* Lane 1 Module: Bidirectional master, HS-TX, LP-TX, LP-RX, LP-CD, and CIL-MFAA function;
   Escape Mode LPDT shall be supported in both the forward and reverse direction
* Lane n Module (for n > 1): Unidirectional master, HS-TX, LP-TX, and CIL-MFEN function 

对USL主机的最低C-PHY LP/LVLP模式物理层要求为:

* Lane 1 Module: Bidirectional slave, HS-RX, LP-TX, LP-RX, LP-CD, and CIL-SFAA function; 804 
   Escape Mode LPDT shall be supported in both the forward and reverse direction
* Lane n Module (for n > 1): Unidirectional slave, HS-RX, LP-RX, and CIL-SFEN function 

对于使用 C-PHY LP/L VLP 模式实现的 USL，前向 Escape Mode LPDT 传输应仅使用通道 1，所有反向传输应仅使用Lane 1 和 LPDT。 USL 主机应能够接收 LPDT 和高速 (HS) 传输。 请注意，使用 LPDT 进行传输时，传输带宽会大大降低。
USL 图像传感器的最低 C-PHY ALP 模式物理层要求是：

* Lane 1 Module: Bidirectional master, HS-TX, HS-RX, and CIL-MREN function (CIL-MREE 812 
   with ALP-ULPS in both forward and reverse direction is recommended) 
* Lane n Module (for n > 1): Unidirectional master, HS-TX and CIL-MFEN function 814 


USL主机对C-PHY ALP模式物理层的最低要求为:

* Lane 1 Module: Bidirectional slave, HS-TX, HS-RX, and CIL-SREN function (CIL-SREE with 816 
   ALP-ULPS in both forward and reverse direction is recommended) 
* Lane n Module (for n > 1): Unidirectional slave, HS-RX and CIL-SFEN function 

请注意，C-PHY ALP 模式没有为双向通道模块定义争用检测功能。 所有支持 C-PHY 物理层选项的 CSI-2 USL 实现应支持所有通道上的正向 ULPS。 对于Lane 1，建议对反向 ULPS 提供额外的 C-PHY ALP 模式支持； 有关其他指导，请参见第 9.12.5.8 节。

## 五、Multi-Lane Distribution and Merging 

CSI-2 是一种通道可扩展规范。 需要比一个数据通道提供的带宽更多的应用程序，或者那些试图避免高时钟速率的应用程序，可以将数据路径扩展到更多的通道数，并获得峰值总线带宽的近似线性增加。 更高层的数据与串行位或符号流之间的映射被明确定义，以确保主机处理器和使用多个数据通道的外设之间的兼容性。

从概念上讲，在 PHY 和更高功能层之间是一个处理多通道配置的层。 如图 35 和图 36 所示，分别为 D-PHY 和 C-PHY 物理层选项，CSI-2 发送器包含一个通道分布功能 (LDF Lane Distribution Function)，它接受来自低级协议层的数据包字节序列和 将它们分布在 N 个通道中，其中每个通道是物理层逻辑（串行器等）和传输电路的独立单元。 类似地，分别如图 37 和图 38 所示的 D-PHY 和 C-PHY 物理层选项，CSI-2 接收器包含一个通道合并功能 (LMF Lane Distribution Function)，它收集来自 N 个通道的传入字节并合并（merge） 将它们分成完整的数据包传递到接收方低层协议层（LLP）的数据包分解器中。

![image-20220317101431581](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317101431581.png)

![image-20220317101540527](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317101540527.png)

![image-20220317101605926](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317101605926.png)



![image-20220317101712694](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317101712694.png)

通道分配器接收任意字节长度的传输，缓冲 N*b 字节（其中 N = 通道数，对于 D-PHY 或 C-PHY 物理层选项，b = 1 或 2），然后发送组 N * b 字节的并行跨 N 通道，每个通道接收 b 字节。 在发送数据之前，所有通道并行执行 SoT 序列，以向其相应的接收单元指示数据包的第一个字节正在开始。 在 SoT 之后，通道并行发送来自第一个数据包的连续字节组，遵循循环过程。

### 1. Lane Distribution for the D-PHY Physical Layer Option 

在传输结束时，可能会有“额外”字节，因为总字节数可能不是通道数 N 的整数倍。一个或多个通道可能在其他通道之前发送它们的最后一个字节。 通道分配器在并行缓冲最后一组少于 N 个字节，然后发送到 N 个数据通道，将其“有效数据”信号置低到所有通道接下来没有数据为止。 对于发送数据通道数大于所发送的字节数时，未接收到用于传输的字节的通道应保持 LPS 状态。

![image-20220317104544412](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317104544412.png)

![image-20220317104617273](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317104617273.png)

![image-20220317104636317](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317104636317.png)

![image-20220317104656779](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317104656779.png)

### 2. Lane Distribution for the C-PHY Physical Layer Option  

图 45 说明了 N ≥ 1 的 N 通道系统的规范行为：数据包的Byte 1 和 0 作为 16 bit字发送到Lane 1 C-PHY 模块，Byte 3 和 2 发送到Lane 2， Byte 2N-1 和 2N-2 被发送到Lane N，字节 2N+1 和 2N 被发送到通道 1，依此类推。 最后两个字节 B-1 和 B-2 被发送到通道 N，其中 B 是数据包中的总字节数。

对于 N-Lane 发送器，Lane n (1 ≤ n ≤ N) 的 C-PHY 模块应从低层协议层（LLP）生成的 B 字节数据包中发送以下 {Ms byte : ls byte} 字节对序列 : {Byte 2*(k*N+n)-1 : Byte 2*(k*N+n)-2}，对于 k = 0, 1, 2, ..., B/(2N) - 1，其中 Byte 0 是数据包中的第一个字节。 低层协议应保证 B 是 2N 的整数倍。

也就是说，在数据包传输结束时，不应有“额外”字节，因为总字节数总是通道数 N 的偶数倍。通道分配器在发送最后一组 2N 个字节后 与 N 个通道并行，同时向所有通道取消其“有效数据”信号，向每个 C-PHY 通道模块发出信号，表明它可以开始其 EoT 序列。

每个 C-PHY Lane 模块自主运行，但所有 Lane 上的数据包数据传输同时开始和停止。

链路接收端的 N 个 C-PHY 接收器模块并行收集字节对，并将它们发送到 Lane-merging 层。 这重构了传输中的原始字节序列，然后可以将其划分为用于数据包解码器层的单独数据包。

![image-20220317105456349](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317105456349.png)

![image-20220317105519152](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317105519152.png)

![image-20220317105542066](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317105542066.png)

### 3. Multi-Lane Interoperability

当使用多个数据通道时，通道分布和合并层应可通过摄像机控制接口（CCI）重新配置。
“N”个数据通道接收器应与“M”个数据通道发送器连接，当使用多个数据通道时，通过通道分布的 CCI 配置和 CSI-2 发送器和接收器内的合并层。 因此，如果 M<=N，则具有 N 个数据通道的接收器应与具有 M 个数据通道的发送器一起工作。 同样，如果 M>=N，则具有 M 个通道的发送器应与具有 N 个数据通道的接收器一起工作。 发射器通道 1 到 M 应连接到接收器通道 1 到 N。

两种情况：

* 如果 M<=N，则没有性能损失——接收器有足够的数据通道来匹配发送器（图 46 和图 47）。
* 如果 M> N，则可能会损失性能（例如帧速率），因为接收器的数据通道少于发送器（图 48 和图 49）。

请注意，虽然所示示例针对的是 D-PHY 物理层选项，但 C-PHY 物理层选项的处理方式类似，只是没有时钟通道。

![image-20220317110322670](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317110322670.png)

![image-20220317110414941](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317110414941.png)

#### 3.1. C-PHY Lane De-Skew

C-PHY 规范 [MIPI02] 中的 PPI（PHY Protocol Interface  ）定义为每个通道定义了一个 RxWordClkHS，并且没有解决对链路中的所有通道使用公共接收 RxWordClkHS 的问题。 图 50 显示了一种用于对来自弹性缓冲器的数据进行计时的机制，以便将所有 RxDataHS 对齐（De-Skew）到一个 RxWordClkHS。

![image-20220317110820139](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317110820139.png)

## 六、Low Level Protocol

低级别协议(LLP)是一个面向字节的，基于包的协议，支持使用短包和长包格式传输任意数据。为了简单起见，本节中的所有示例都是单独的Lane配置，除非另有说明。

基本的低级别协议特性:

* 任意数据的传输(与有效载荷无关)。
* 8-bit word size。
* 在同一D-PHY链路上支持多达16个交错虚通道，或在同一C-PHY链路上支持多达32 个交错虚通道。
* 用于帧开始、帧结束、行开始和行结束信息的特殊数据包。
* 应用程序特定负载数据的类型、像素深度和格式的描述符。
* 16 bit校验和错误检测代码。
* 6 bit纠错码，用于纠错检测和纠错(仅D-PHY物理层)。

### 1. Low Level Protocol Packet Format

如图 51 所示，为低层协议通信定义了两种数据包结构：长数据包和短数据包。 短包和长包的格式和长度取决于物理层的选择。 对于每个数据包结构，从低功率状态退出后跟传输开始 (SoT) 序列表示数据包的开始。 传输结束 (EoT) 序列后跟低功率状态指示数据包的结束。

![image-20220317141353458](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220317141353458.png)

#### 1.1Low Level Protocol Long Packet Format 

图 52 显示了 D-PHY 物理层选项的低级协议长数据包的结构。 长数据包应由数据类型 0x10 到 0x38 标识。 有关数据类型的说明，请参见表 10。 D-PHY 物理层选项的长数据包应包含三个元素：32 bit数据包头 (PH)、具有可变数量的 8 bit数据字的应用特定数据有效负载和 16 位数据包尾 (PF)。 包头进一步由四个元素组成：一个 8 bit数据标识符（DI）、一个 16 bit字计数字段（WC）、一个 2 bit虚拟通道扩展字段（VCX）和一个 6 bit ECC。 数据包脚注有一个元素，即 16 位bt验和 (CRC)。 有关数据包元素的进一步描述，请参见第 9.2 节至第 9.5 节。

![image-20220318092614317](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318092614317.png)

* 8-bit DATA IDENTIFIER (DI):

包含 2 位虚拟通道 (VC) 和 6 位数据类型 (DT) 信息。VC（位 7:6）是 D-PHY 物理层的 4 位虚拟通道标识符的最低有效两位 选项。 DT（比特 5:0）表示应用特定有效载荷数据的格式/内容。 由应用程序特定层使用。

* 16-bit WORD COUNT (WC):

接收器读取下一个 WC 数据字，与它们的值无关。 接收器不会在有效载荷数据中寻找任何嵌入的同步序列。 接收方使用 WC 值来确定数据包有效负载的结束。

* 6-bit Error Correction Code (ECC) + 2-bit Virtual Channel Extension (VCX):

ECC（bit 5:0）可以纠正数据包标头中的 1 位错误并检测 2 位错误。 VCX（bit 7:6）是 D-PHY 物理层选项的 4 bit虚拟通道标识符的最高有效两位。

图 53 显示了 C-PHY 物理层选项的长数据包结构；它应由四个元素组成：包头（PH）、具有可变数量的 8 位数据字的应用特定数据有效负载、16 位包尾（PF）和零个或多个填充字节（FILLER）。包头是 6N x 16 位长，其中 N 是 C-PHY 物理层通道的数量。如图 53 所示，数据包报头由两个相同的 6N 字节半部分组成，其中每一半部分由以下每个字段的 N 个顺序副本组成：一个 16 位字段，包含五个保留位，一个 3 位虚拟通道扩展(VCX) 字段和 8 位数据标识符 (DI)； 16 位数据包数据字数 (WC)；以及在前四个字节上计算的 16 位数据包头校验和 (PH-CRC)。每个保留位的值应为零。包尾包含一个 16 位校验和 (CRC)，使用与 D-PHY 物理层选项中使用的包头 CRC 和包尾相同的 CRC 多项式在包数据上计算得出。如果需要，在 Packet Footer 之后插入 Packet Filler 字节，以确保 Packet Footer 在 16 位字边界上结束，并且每个 C-PHY 物理层通道传输相同数量的 16 位字（即字节对） .

![image-20220318094005320](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318094005320.png)

* 为什么有很多个相同的下图？

多发送几个副本？

![image-20220318101001718](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318101001718.png)

* Filler0-N

这里应该是CSI对C PHY层要求。

```shell
对于 N-Lane 发送器，Lane n (1 ≤ n ≤ N) 的 C-PHY 模块应从低层协议层生成的 B 字节数据包中发送以下 {ms byte : ls byte} 字节对序列 : {Byte 2*(k*N+n)-1 : Byte 2*(k*N+n)-2}，对于 k = 0, 1, 2, ..., B/(2N) - 1，其中 Byte 0 是数据包中的第一个字节。 低层协议应保证 B 是 2N 的整数倍。
```

如图 54 所示，图 53 中描述的数据包报头结构有效地导致 C-PHY 通道分配器将相同的6个 16 bit字广播到 N 个通道中的每个通道。 另外，每个通道的6个字被分成两个相同的3字组，它们由 [MIPI02] 中描述的强制性 C-PHY 同步字分隔。 同步字由 C-PHY 物理层插入以响应 CSI-2 协议发送器 PPI 命令。

![image-20220318112918727](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318112918727.png)

对于这两种物理层选项，8 位数据标识符字段定义了 2 位虚拟通道 (VC) 和应用程序特定有效载荷数据的数据类型。虚拟通道扩展 (VCX) 字段对于这两个选项也是通用的，但对于 D-PHY 是 2 位字段，对于 C-PHY 是 3 位字段。 VC 和 VCX 字段共同构成 4 位或 5 位虚拟通道标识符字段，它确定与数据包关联的虚拟通道号（参见第 9.3 节）。

对于这两种物理层选项，16 位字数 (WC) 字段定义了数据有效负载中从数据包头的结尾到数据包尾的开头之间的 8 位数据字的数量。字数中不应包括包头、包尾或包填充字节。

对于 D-PHY 物理层选项，6 位纠错码 (ECC) 允许在数据包标头中纠正1 bit错误和检测 2 bit错误。这包括数据标识符、字数和虚拟通道扩展字段值。

C-PHY 物理层选项不使用 ECC 字段，因为 C-PHY 物理链路上的单个符号错误可能会导致接收到的 CSI-2 数据包标头中出现多个位错误，从而导致 ECC 无效。相反，用于 C-PHY 物理层选项的 CSI-2 协议发送器在组成保留、虚拟通道扩展、数据标识符和字计数数据包的四个字节上计算 16 位 CRC头字段，然后传输所有这些字段的多个副本，包括 CRC，以便在发生一个或多个 C-PHY 物理链路错误时由 CSI-2 协议接收器恢复它们。 C-PHY 物理层插入数据包头的多个同步字（如图 54 所示）还通过使 C-PHY 接收器能够从丢失的符号时钟中恢复来促进数据包头数据的恢复；有关 C-PHY 同步字和符号时钟恢复的更多信息，请参见 [MIPI02]。

对于这两种物理层选项，CSI-2 接收器读取数据有效载荷的下一个 WC 8 位数据字，紧跟在数据包头之后。 在读取数据有效负载时，接收器不应寻找任何嵌入的同步代码。 因此，对 8 位有效载荷数据字的值没有限制。 在一般情况下，数据有效载荷的长度应始终是 8 位数据字的倍数。 此外，每种数据类型都可能对数据有效负载的长度施加额外的限制，例如 需要4个byte的倍数。

对于这两种物理层选项，一旦 CSI-2 接收器读取了数据有效负载，它就会读取数据包尾中的 16 位校验和 (CRC)，并将其与自己计算的校验和进行比较，以确定是否发生任何数据有效负载错误 .

填充字节仅由 CSI-2 发送器的低级协议层与 C-PHY 物理层选项一起插入。 任何填充字节的值都应为零。 如果包数据字数 (WC) 为奇数（即 LSB 为“1”），则 CSI-2 发送器应在包尾后插入一个包填充字节，以确保包尾以 16 位字结尾 边界。 CSI-2 发射机还应如果需要，插入额外的填充字节，以确保C-PHY 每个通道传输相同数量的 16 位字。 后面的规则要求填充字节的总数 FC 大于或等于 (WC mod 2) + {{N - (([WC + 2 + (WC mod 2)] / 2) mod N)} mod N} * 2，其中 N 是车道数。 请注意，FC 可能为零。

图 55 说明了通过三个 C-PHY 通道传输的各种长度的数据包所需的最小填充字节数的通道分布。每个数据包所需的填充字节总数范围为 0 到 5，具体取决于数据包数据字数 (WC) 的值。通常，对于 N 通道 C-PHY 系统，每个数据包所需的填充字节的最小数量范围为 0 到 2N-1。

对于 D-PHY 物理层选项，CSI-2 通道分配器功能应将每个字节传递给物理层，然后物理层首先串行传输它的最低有效位。

对于 C-PHY 物理层选项，Lane Distributor 功能应将从 Low Level Protocol 接收到的每对连续字节 2n 和 2n+1（对于 n ≥ 0）分组为一个 16 位字（其最低有效字节为 byte 2n) 然后把这个字传给物理层里的一个模块。 C-PHY 通道模块将每个 16 位字映射成一个 7 symbol字（这里需要注意一下，后面会专门起一小节讲述为什么C PHY是5进制7 symbol编码），然后它首先串行传输最低有效符号。

对于这两种物理层选项，有效载荷数据可以以仅受数据格式要求限制的任何字节顺序呈现给lane分配器功能。多字节协议元素，如字数、校验和和短数据包 16 位数据字段应首先呈现给通道分配器功能最低有效字节。

在 EoT 序列之后，接收器开始寻找下一个 SoT 序列。

![image-20220318132025921](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318132025921.png)

![image-20220318132108714](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318132108714.png)

#### 1.2 Low Level Protocol Short Packet Format 

图 56 和图 57 分别展示了 D-PHY 和 C-PHY 物理层选项的LLP短包结构。对于每个选项，短包结构与相应的LLP长包结构的包头匹配（结构相同），但包头字数（WC）字段应由短包数据字段代替。短数据包应由数据类型 0x00 到 0x0F 标识。有关数据类型的说明，请参见表 10。短数据包应包含只有一个包头； Packet Footer 和 Packet Filler 字节都不应出现。

对于帧同步数据类型，短包数据字段应为帧号。对于线路同步数据类型，短包数据字段应为线路号。有关帧和行同步数据类型的描述，请参见表 13。对于通用短包数据类型，短包数据字段的内容应由用户定义。

对于 D-PHY 物理层选项，纠错码 (ECC) 字段允许在短数据包中纠正单位错误和检测 2 位错误。对于 C-PHY 物理层选项，16 位校验和 (CRC) 允许在短数据包中检测一个或多个位错误，但不支持纠错；后者通过传输多个短包字段的多个副本和通过在所有通道上插入 C-PHY 同步字来促进。

![image-20220318170853748](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318170853748.png)

### 2. Data Identifier (DI) 

数据标识符字节包含虚拟通道 (VC) 和数据类型 (DT) 字段，如图 58 所示。虚拟通道字段包含在数据标识符字节的两个 MS 位中。 数据类型字段包含在数据标识符字节的六个 LS 位中。

![image-20220318193925567](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318193925567.png)

### 3. Virtual Channel Identifier

4 位或 5 位虚拟通道标识符的目的是为在数据流中交错的不同数据流指定单独的逻辑通道提供一种方法。

如图 59 所示，虚拟信道标识符的最低有效两位应从 2 位 VC 字段中复制，最高有效 2 或 3 位应从 VCX 字段中复制。 VCX 字段位于数据包报头中，分别如图 52 和图 53 所示，用于 D-PHY 和 C-PHY 物理层选项。接收器应从传入的数据包头中提取虚拟通道标识符，并将交织的视频数据流解复用到其适当的通道。对于 D-PHY 或 C-PHY 物理层选项，最多支持 N 个数据流，其中 N = 16 或 32；有效的通道标识符是 0 到 N-1。外设中的虚拟通道标识符应该是可编程的，以允许主机处理器控制数据流的解复用方式。

主机处理器从符合先前 CSI-2 规范版本不支持 VCX 字段的外围设备接收数据包应将所有此类数据包中的 VCX 接收值视为零。类似地，符合此 CSI-2 规范版本的外围设备应在传输到符合不支持 VCX 字段的先前版本的主机处理器的所有数据包中将 VCX 字段设置为零。主机处理器和外围设备满足这些要求的方法不在本规范的范围内。

![image-20220318194804523](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318194804523.png)

图60演示了一个使用虚拟通道支持的数据流示例。

![image-20220318194834433](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318194834433.png)

### 4. Data Type (DT)

数据类型值指定有效负载数据的格式和内容。 最多支持 64 种数据类型。

如表 10 所示，有八种不同的数据类型类。在每个类中，最多有九种不同的数据类型定义。 前两类表示短包数据类型。 其余六类表示长数据包数据类型。

有关短数据包数据类型类的详细信息，请参阅第 9.8 节。

有关五种长数据包数据类型类的详细信息，请参阅第 11 节。

​                                                          Table 10 Data Type Classes

| Data    Type | Description                                                  |
| ------------ | ------------------------------------------------------------ |
| 0x00 to 0x07 | Synchronization Short Packet Data Types                      |
| 0x08 to 0x0F | Generic Short Packet Data Types                              |
| 0x10 to 0x17 | Generic Long Packet Data Types                               |
| 0x18 to 0x1F | YUV Data                                                     |
| 0x20 to 0x26 | RGB Data                                                     |
| 0x27 to 0x2F | RAW Data                                                     |
| 0x30 to 0x37 | User Defined Byte-based Data                                 |
| 0x38         | USL Commands (See Section 9.12)                              |
| 0x39 to 0x3E | Reserved for future use                                      |
| 0x3F         | For CSI-2 over C-PHY: Reserved for future use For CSI-2 over D-PHY: Unavailable (0x3F is used for LRTE EPD Spacer) |

### 5. Packet Header Error Correction Code for D-PHY Physical Layer Option

数据标识符、字数和虚拟通道扩展字段的正确解释对数据包结构至关重要。 6-bit Packet Header Error Correction Code (ECC) 允许在后面的字段中纠正单比特错误，并为 D-PHY 物理层选项检测两位错误； ECC 不适用于 C-PHY 物理层选项。 应使用第 9.5.2 节中描述的汉明修改码的 26 位子集。 ECC 解码的错误状态结果可以在接收器的应用层获得。

数据标识符字段 DI[7:0] 应映射到 ECC 输入的 D[7:0]，字数 LS 字节 (WC[7:0]) 到 D[15:8]，字数 MS 字节 (WC[15:8]) 到 D[23:16]，虚拟通道扩展 (VCX) 字段到 D[25:24]。 此映射如图 61 所示，它也用作 ECC 计算示例。

![image-20220318205113045](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220318205113045.png)

### 6. Checksum Generation 

### 7. Packet Spacing

所有 CSI-2 实现都应支持低级协议数据包之间进入和退出低功耗状态 (LPS) 的转换； 但是，如第 9.11 节所述，实现可以选择在数据包之间保持高速状态。 

图 70 说明了使用 LPS 的数据包间隔。 图 70 中所示的数据包间隔不必是 8 位数据字的倍数，因为接收器将在 SoT 序列期间在下一个数据包的数据包标头之前重新同步到正确的字节边界。

![image-20220319104951456](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319104951456.png)

### 8. Synchronization Short Packet Data Type Codes 

​	短包数据类型应仅使用短包格式传输。 有关格式说明，请参见第 9.1.2 节。

![image-20220319105932041](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319105932041.png)

#### 8.1 Frame Synchronization Packets 

每个图像帧应以包含帧开始码的帧开始（FS）包开始。 FS 数据包后面应跟随一个或多个包含图像数据的长数据包和零个或多个包含同步代码的短数据包。 每个图像帧应以包含帧结束码的帧结束（FE）包结束。 有关同步代码数据类型的说明，请参见表 13。

对于 FS 和 FE 同步包，短包数据字段应包含一个 16 位的帧号。 对于与给定帧对应的 FS 和 FE 同步数据包，该帧号应相同。

16 位帧号在使用时应为非零，以区别于帧号无效并保持设置为零的用例。

16位帧号的行为应是以下情况之一:

* 帧号总是zero – frame无效。

* 对于具有相同虚拟通道的每个 FS 数据包，帧号增加 1 或 2，并定期重置为 1； 例如 1、2、1、2、1、2、1、2 或 1、2、3、4、1、2、3、4 或 1、3、5、1、3、5 或 1、2、4， 1, 3, 4。仅当图像帧由于损坏而被屏蔽（即未传输）时，帧号才可以增加 2。 为了适应这种情况，可以根据需要在帧号序列中自由混合 1 或 2 的增量。

#### 8.2 Line Synchronization Packets 

行同步短数据包在每个图像帧的基础上是可选的。 如果图像帧包括行开始（LS）和行结束（LE）行同步短包，其中一个长包具有给定的数据类型和虚拟通道号，则它应包括 LS 和 LE 短包，所有长包都具有相同的数据类型和虚拟通道号 同一图像帧内的数据类型和虚拟通道号。

对于 LS 和 LE 同步数据包，短数据包数据字段应包含一个 16 位的行号。 对于与给定行对应的 LS 和 LE 数据包，该行号应相同。 行号是逻辑行号，不一定等于物理行号。

16 位行号在使用时应为非零，以区别于行号无效并保持设置为零的情况。相同数据类型和虚拟通道内的 16 位行号的行为应为以下之一。

1. 行号始终为零——行号无效。
 要么：
 2. 同一虚拟通道和同一数据类型内的每个 LS 包，行号加一。对于 FS 包之后的第一个 LS 包，行号会定期重置为 1。预期用途是用于逐行扫描（非隔行）视频数据流。行号必须是非零值。
 要么：
 3. 对于同一虚拟通道和同一数据类型内的每个 LS 数据包，行号以大于 1 的相同任意步长值递增。对于 FS 数据包之后的第一个 LS 数据包，行号会定期重置为非零任意起始值。连续帧之间的任意起始值可能不同。预期用途是用于隔行扫描视频数据流。
 图 71 包含在具有像素数据和其他嵌入类型的隔行帧内使用可选 LS/LE 数据包的示例。该图说明了用例：
 1. VC0 DT2 隔行扫描帧，行数加 2。 Frame1 从 1 开始，Frame2 从 2 开始。
 2. VC0 DT1 带行计数的逐行扫描帧。
 3. VC0 DT4 逐行扫描帧，带非操作行计数。
 4. VC0 DT3 无 LS/LE 操作。

![image-20220319113705345](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319113705345.png)

![image-20220319113724632](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319113724632.png)

### 9. Generic Short Packet Data Type Codes

![image-20220319115632257](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319115632257.png)

通用短数据包数据类型的目的是提供一种机制，用于在数据流中包含快门打开/关闭、闪光触发等的时序信息。 通用短数据包中的 16 位用户定义数据字段的目的是将数据类型值和 16 位数据值从发送器传递到接收器中的应用层。 CSI-2 接收器应将数据类型值和相关的 16 位数据值传递给应用层。

### 10. Packet Spacing Examples Using the Low Power State 

本节讨论的数据包由 EoT、LPS、SoT 序列分隔，如 [MIPI01] 中针对 D-PHY 物理层选项和 [MIPI02] 中针对 C-PHY 物理层选项定义的序列。

图 72 和图 73 分别包含由多个数据包和单个数据包组成的数据帧的示例。

请注意，本节图中的 VV ALID、HV ALID 和 DV ALID 信号只是有助于说明帧开始/结束和行开始/结束数据包行为的概念。 VV ALID、HV ALID 和 DV ALID 信号不构成规范的一部分。

![image-20220319120211670](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319120211670.png)



![image-20220319120141802](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319120141802.png)

![image-20220319120655972](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319120655972.png)

一个长包的包尾（或包填充器，如果存在）结束和下一个长包的包头之间的时间段称为行消隐期。

帧 N 中的帧结束数据包和帧 N+1 中的帧开始数据包之间的周期称为帧消隐周期（图 74）。

行消隐期不是固定的，长度可能不同。 接收器应该能够处理由[MIPI01] 或[MIPI02] 中定义的最小包间间隔定义的接近零的行消隐期，视情况而定。 发送器定义了帧消隐期的最短时间。 帧消隐期持续时间应在发送器中进行编程。

应使用帧开始和帧结束数据包。

帧开始和结束数据包间隔的建议（信息性）：

* 帧起始包到第一个数据包的间隔应尽可能接近最小包间隔。
* 最后一个数据包到Frame End 包间隔应尽可能接近最小包间隔

目的是确保帧开始和帧结束数据包准确地表示图像数据帧的开始和结束。 一个有效的例外是当帧开始和帧结束数据包的位置被用于传送像素级准确的垂直同步定时信息时。

帧开始和帧结束数据包的位置可以在帧消隐周期内改变，以便提供像素级准确的垂直同步定时信息。 请参见图 75。

如果需要像素级准确的水平同步时序信息，则应使用 Line Start 和 Line End 数据包来实现。

行开始和行结束数据包的位置（如果存在）可以在行消隐周期内改变，以便提供像素精确的水平同步时序信息。 请参见图 76。

![image-20220319121136759](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319121136759.png)

![image-20220319121155804](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319121155804.png)

### 11. Latency Reduction and Transport Efficiency (LRTE) 

延迟减少和传输效率 (LRTE) 是一项可选的 CSI-2 功能，可促进最佳传输，以支持许多新兴的成像应用。

LRTE 有两个部分，本节将进一步详述：

* 包间延迟减少 (ILR)
* 提高运输效率

#### 11.1 Interpacket Latency Reduction (ILR) 

包间延迟减少。

根据 D-PHY 物理层选项的 [MIPI01] 和 C-PHY 物理层选项的 [MIPI02]，CSI-2 短数据包和长数据包由 EoT、LPS 和 SoT 数据包分隔符分隔。 先进的成像应用、PDAF（相位检测自动对焦）、传感器聚合和机器视觉可以从通过减少这些定界符的开销所产生的有效速度提升中显着受益。

如图 77 所示，包间延迟减少可用于用更高效的包分隔符 (EPD) 信令机制替换传统的 EoT、LPS 和 SoT 包分隔符，从而避免 HS-LPS-HS 转换的需要。 EPD 由 PHY 层和/或协议层元素组成。 PHY 生成的 EPD 元素称为“快速分组定界符”（PDQ）。 协议生成的 EPD 元素称为 Spacer，可以选择在 PDQ 之前。

![image-20220319122537206](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319122537206.png)

如图 77 所示，LRTE 要求在 PHY 高速信令期间在相邻 CSI-2 数据包之间插入 EPD，但不允许在 PHY EoT 之前的 CSI-2 数据包之后插入 EPD。 但是，如本节后面所述，在某些条件下，可以在 PHY EoT 之前的 CSI-2 数据包之后插入 Spacer，但没有 PHY 生成的 PDQ 信令。

##### 11.1.1EPD for C-PHY Physical Layer Option 

简略了解，这写是PHY层硬件选项，详情请见协议手册。

![image-20220319123302936](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319123302936.png)

##### 11.1.2 EPD for D-PHY Physical Layer Option  

![image-20220319123408242](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319123408242.png)

![image-20220319123427384](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319123427384.png)

##### 11.1.3 D-PHY EPD Specifications (for EPD Options 1 and 2) 

图像传感器（发射器）应包括以下两个 16 位寄存器，以促进成像应用的最佳数据包间延迟：

* TX_REG_CSI_EPD_EN_SSP (EPD Enable and Short Packet Spacer) Register 
* TX_REG_CSI_EPD_OP_SLP (EPD Option and Long Packet Spacer) Register 

这两个寄存器的具体信息，请翻阅MIPI CSI-2协议手册查阅并结合image sensor手册。

##### 11.1.4 D-PHY EPD选项2下的鲁棒变长间隔器检测

##### 11.1.5 End-of-Transmission Short Packet (EoTp) 

#### 11.2 同时使用ILR和提高传输效率

在许多成像应用中，EPD 可以与 L VLP 或 ALP 模式信令一起使用，以便从 CSI-2 ILR 和增强的通道传输中受益。

![image-20220319124458737](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319124458737.png)

#### 11.3 LRTE Register Tables 

![image-20220319124735958](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319124735958.png)

![image-20220319124800152](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319124800152.png)

![image-20220319124814447](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319124814447.png)

### 12. Unified Serial Link (USL) 

统一串行链路（USL，参见图 85）是一个可选的 CSI-2 功能，可减少接口线的数量并有助于原生支持更长的覆盖范围。 USL 建立在 LRTE 功能（第 9.11 节）提供的优势之上。

USL 减少了为相机命令接口 (CCI) 使用任何额外的 I2C、I3C 和/或 SPI 互连或用于相机模块控制信号的 GPIO 线的需要。 这是通过使用 CSI-2 封装以及本机支持的 C-PHY 2.0（或更高版本）和 D-PHY v2.5（或更高版本）传输层选项的总线周转 (BTA) 功能来实现的。

MIPI PHY 带宽比 I2C 兼容的 2 线双向控制总线提供的带宽高出许多数量级。 此外，在传输水平行（即水平消隐）和传输帧之间（即垂直消隐）之间存在自然消隐间隔（即空闲周期），可用于更新图像传感器阴影寄存器。

在这个部分：

* 术语 SNS 是指图像传感器，或包含 CMOS 图像传感器和附加互补设备（即非成像设备）的图像传感器模块。
* 术语 APP 是指包含计算机视觉引擎和/或图像处理单元的任意组合的 SoC 应用处理器 (AP)，或主处理器。

从 APP 到 SNS 的反向链路吞吐量不需要高带宽，这大大放宽了 SNS 收发器上接收器实施的时序约束和余量。 通过 MIPI C-PHY/D-PHY 映射所有 CSI-2 事务可以减少线路数量、支持长距离、帮助保护通道实施并降低工程开发成本。

![image-20220319132504637](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319132504637.png)

#### 12.1 USL Technical Overview 

**Lane Usage:** 绝大多数使用 USL 的成像应用预计将映射到具有单个双向lane（lane 1）的管道。但是，如果管道需要多个通道，则第一个通道（lane 1）应为双向，其余通道应为单向并配置为图像传感器的 TX。在多通道管道中，链路中的所有通道都用于 USL_FWD 模式操作，只有第一个通道（通道 1）用于 SL_REV 模式操作。收发器功能始终限于一个通道（lane 1）。 USL 物理层要求详见第 7.3 节。

**USL 命令和事务：**0x38 PH 数据类型（见表 10）和 CSI-2 长数据包格式应用于 USL 命令操作和事务。 “USL 数据包”是数据类型为 0x38 的任何长数据包。

**Virtual Channels:** 为了促进系统平台上的传感器聚合，SNS 应支持具有虚拟通道标识符的 USL 数据包，其中 D-PHY 的虚拟通道标识符范围为 0 到 15，C-PHY 的范围为 0 到 31。 CSI-2 帧开始和帧结束短数据包不得为专门用于 USL 数据包的任何虚拟信道标识符传输。 USL 数据包中使用的虚拟通道标识符允许用于数据类型代码不是0x38 的 CSI-2 数据包（即非 USL 数据包）。 但是，需要一对 CSI-2 帧开始和帧结束短数据包来封装每个图像帧内每个共享虚拟通道标识符的所有非 USL 数据包。

#### 12.1.2 USL Command Payload Constructs 

所有 USL 传输操作都应使用现有的 CSI-2 长数据包 (LgP) 结构。 封装的 USL 有效载荷字段（Size_Bytes、Slave_Address、Sub_Address、Write_data、Read_data、USL_CTL、TSEQ）应将 LSB 数据映射到 Bit0（见图 3）。 所有多字节 USL 有效载荷字段应首先传输最低有效字节。

USL 命令用于使用 CSI-2 LgP 将寄存器读/写请求从 APP 映射到 SNS，并从 SNS 读取完成到 APP。 当 APP 或 SNS 分别生成 USL 命令寄存器 R/W 请求或完成时，LgP 数据包头数据类型字段应设置为值 0x38。 对于所有 C-PHY 或 D-PHY 高速 (HS) 模式传输，封装的 R/W 请求被映射到 USL 数据包的 LgP 数据包数据字段应用特定有效负载中，根据 D-PHY 物理层选项的图 52 ，并根据 C-PHY 物理层选项的图 53。 如第 9.1 节中定义的，应为 C/D-PHY 保留长数据包填充和填充插入。

如第 7.3 节所述，所有 USL 实现都应支持 PHY LP 或 LVLP 模式信令，并应支持 ALP 模式信令。 USL 系统通常在上电之前进行配置，以支持 D-PHY 或 C-P H Y 下的这些信令模式之一。

如果 USL 配置为使用 C-PHY 或 D-PHY LP/LVLP 模式，则使用正向或反向 Escape Mode LPDT 传输的所有 USL 数据包与使用 D-PHY HS 模式传输的 USL 数据包具有相同的格式； 每个字节也首先传输最低有效位。 APP 应能够接收 USL 数据包，既可以作为分布在所有活动通道上的 HS 模式传输，也可以作为仅在通道 1 上驱动的 Escape 模式 LPDT 传输。

在 USL_REV 模式期间，SNS 应支持缓冲来自 APP 的至少 32 个寄存器读取请求（单个和连续）。 在 USL_REV 模式期间，SNS 应支持至少 64 个寄存器写入请求（单个和连续），以及来自 APP 的至少 1024 位写入数据。

如果 USL 配置为使用 C-PHY 或 D-PHY ALP 模式，则在 USL_REV 模式下，APP 可以使用 C-PHY EPD 或 D-PHY EPD 选项 2 将多个 USL 数据包连接到通道 1 上的单个 HS 突发传输 ，如果支持适用的 LRTE EPD 功能（见第 9.11 节）。 SNS HS 模式接收器 LRTE 功能可以从 SNS 数据表中了解，或以其他方式发现。 类似地，在 USL_FWD 模式下，SNS 可以使用后一种 EPD 替代方案之一将多个 USL 和/或非 USL 数据包连接成一个分布在所有活动通道上的单个 HS 突发传输。

如果 USL 配置为使用 C-PHY 或 D-PHY LP/LVLP 模式，则在 USL_FWD 模式下，SNS 可以使用任何 支持的 C-PHY 或 D-PHY LRTE EPD 功能（包括 D-PHY EPD 选项 1）。 在 USL_FWD 或 USL_REV 模式下，LRTE D-PHY EPD 选项 2 可用于连接使用转义模式 LPDT 传输的多个 USL 数据包，但只能在数据包之间使用零个或多个固定长度间隔符，并且不插入 EoTp 短数据包。 使用 LPDT 传输的 USL 数据包的 SNS 间隔生成可由 APP 使用表 18 中的寄存器控制。SNS LP/LVLP 模式接收器 LRTE 功能可从 SNS 数据表中了解，或以其他方式发现。

现有的 LgP 16 位校验和应用于保持所有 USL 转换的传输完整性。 USL 命令事务不需要传统 CCI “S”（开始）、“P”（停止）和“A”（Acks），因为 LgP 传输处理这些功能。

16 位 Size_Bytes 字段包含要写入或读取的数据的连续字节数。 Read_Data 字段包含基于字节的读取数据。

Write_Data字段包含基于字节的写数据。

一个 7 位的 Slave_Address 字段包含要发送或接收命令的设备地址。 例如，CCI (I2C) 从地址例如 0x6C 用于与图像传感器进行通信，并且与 USL 类似，例如相同。 0x6C 将适用。 该字段作为字节的最高有效 7 位传输，其 LSB 类似于 CCI R/W 位。

一个 16 位的 Sub_Address 字段用于指向从设备内部的一个寄存器。 产品特定的成像系统可以根据需要使用 8 位和/或 16 位的 Sub_Address，因为 Sub_Address 是设备特定的并且在平台级别是已知的。 请注意，系统使用单个 USL 链接与多个设备进行通信，例如 通过本地 I2C 总线，可能需要同时使用 8 位和 16 位索引，具体取决于设备。

8 位 USL 控制 (USL_CTL)（表 19）用于确保传输完整性，保证使用 ACK 将命令从 APP 传递到 SNS，并通过支持经常用于成像的连续 R/W 操作来提高传输效率 和视觉固件从APP上传到SNS。

一个 16 位的事务序列 (TSEQ) 字段包含由 SNS 和 APP 生成的唯一非零 USL 命令标识号。 SNS 和 APP 在成功接收到 USL 数据包后生成并清除 Transaction Sequence 字段（如下文所述）。

![image-20220319140055584](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319140055584.png)

![image-20220319140122921](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319140122921.png)

#### 12.3 USL Operation Procedures 

为简化起见，以下示例显示:

* 一个16bit Sub_Address [15:0]

* 包含由 APP 生成的唯一非零 USL 命令标识号的 16 位事务序列 (TSEQ)

##### 12.3.1 APP Initiated USL Transactions 

此示例过程说明 APP 启动四个有效的 USL 命令事务（USL_REV 模式）。

1. ACK 产生

系统使用 TSEQ 来确保 USL 命令的保证交付。 APP 应包含一个 16 位寄存器，用于存储在 USL_FWD 模式期间从 SNS 接收到的最后一个成功的 TSEQ。 进入 USL_REV 模式后，APP 应立即使用以下格式生成两次 ACK：

• LgP Packet Header Data ID = 0x38 

• LgP Packet Data format for ACK generation:

{USL_CTL[7:0], TSEQ[15:0]} 

2. Register Write Request with Write Data 

• LgP Packet Header Data ID = 0x38 

• LgP Packet Data format for all writes: 

{USL_CTL[7:0], TSEQ[15:0], Size_Bytes[15:0], 

Slave_Address[7:1],[W=0], 

 Sub_Address[15:0],

Write_Data [16’d Size_Bytes-1:0]} 

3. Register Read Request 

• LgP Packet Header Data ID = 0x38 

• LgP Packet Data format for all read requests: 

 {USL_CTL[7:0], TSEQ[15:0], Size_Bytes[15:0], 

Slave_Address[7:1],[R=1],

 Sub_Address[15:0]} 

4. Initiate BTA

完成 R/W 寄存器命令后，APP 应生成两次 Initiate BTA 数据包以将链路从 USL_REV 切换到 USL_FWD 模式。

USL_CTL中的Initiate BTA位应该设置为1'b1。

• LgP Packet Header Data ID = 0x38 

• LgP Packet Header Data ID = 0x38 

 {USL_CTL[7:0], TSEQ[15:0]} 

##### 12.3.2 SNS Initiated USL Transactions 

这个示例过程演示了SNS初始化五个有效的USL命令事务(USL_FWD Mode).

1. NAK Generation 

如果图像传感器 (SNS) 从应用处理器 (APP) 接收到无效或非法的 USL 命令请求，则 SNS 应生成否定确认 (NAK)。 默认情况下，在 USL_REV_Mode 期间，SNS 不应在 NAK 之后执行来自 APP 的任何后续 R/W USL 命令请求，除非 APP 启用 USL_CTL 强制命令位。 SNS 应为来自 APP 的第一个非法或失败的 R/W 请求生成否定确认 (NAK)。 SNS 可以选择性地生成由“强制命令”请求产生的附加 NAK。 NAK 应在切换到 USL_FWD_Mode 后立即使用以下格式向 APP 发送两次：

• LgP Packet Header Data ID = 0x38 

• LgP Packet Header Data ID = 0x38 

{USL_CTL[7:0], TSEQ[15:0]} 

2. ACK Generation

产品成像系统使用 TSEQ 来确保从 APP 到 SNS 的 R/W 命令有保证的传递。 图像传感器应包含一个 16 位寄存器，用于存储在 USL_REV 模式期间从 APP 接收到的最后一个成功的 TSEQ。 进入 USL_FWD 模式后，SNS 应在任何 NAK 生成之后立即使用以下格式生成两次 ACK：

• LgP Packet Header Data ID = 0x38 

• LgP Packet Header Data ID = 0x38 

{USL_CTL[7:0], TSEQ[15:0]} 

3. Register Read Completion

• LgP Packet Header Data ID = 0x38 

• LgP Packet Header Data ID = 0x38 

{USL_CTL[7:0], TSEQ[15:0], Size_Bytes[15:0], 

{USL_CTL[7:0], TSEQ[15:0], Size_Bytes[15:0], 

4. Interrupt Generation

在完成 ACK/NAK 生成后，SNS 可以在 USL_FWD 模式下使用 USL_CTL[1:0] 生成带内中断。 强烈建议 SNS 尽早确定优先级并生成中断通知。

以下格式还应用于包括与中断有关的可选信息：

• LgP Packet Header Data ID = 0x38 

• LgP Packet Header Data ID = 0x38 

{USL_CTL[7:0], Slave_Address[15:0], Interrupt_Information[15:0]} 

5. Initiate BTA

在将链路从 USL_FWD 切换到 USL_REV 模式之前，SNS 应生成两次 Initiate BTA 数据包。 USL_CTL 中的 Initiate BTA 位应设置为 1’b1。 TSEQ 应该是来自 REG_USL_ACK_TSEQ 的 16 位值。

• LgP Packet Header Data ID = 0x38 

• LgP Packet Header Data ID = 0x38 

{USL_CTL[7:0], TSEQ[15:0]} 

#### 12.4 监视USL命令传输完整性

本节介绍了使用 TSEQ 和表 20 中定义的两个 SNS 寄存器来监控 USL 命令传输完整性的逐步过程

1. [System Reset] [Power Cycle] 

SNS 应将寄存器 REG_USL_ACK_TSEQ[15:0] 和 REG_USL_NAK_TSEQ[15:0] 重置为值 16'd0。 APP 应重置内部寄存器 APP_REG_USL_ACK_TSEQ[15:0]。

2. [USL_REV Mode]

• 进入USL_FWD 模式后，APP 应使用内部寄存器APP_REG_USL_ACK_TSEQ[15:0] 中的16 位TSEQ 生成两次ACK。 • APP 应生成一个 16 位非零 TSEQ 值，该值随着它发送到 SNS 的每个 USL 命令 R/W 请求而增加。
• SNS 应将最后成功的 USL 命令的 TSEQ 存储到寄存器 REG_USL_TSEQ_ACK[15:0]。
• SNS 应将第一个非法 USL 命令的 TSEQ 存储到寄存器 REG_USL_TSEQ_NAK[15:0]。

3. [USL_REV Mode Exit] 

•APP应将内部寄存器APP_REG_USL_ACK_TSEQ[15:0]重置为16'd0。

4. [USL_FWD Mode] 

• 进入 USL_FWD 模式后，如果在之前的 USL_REV 模式中遇到任何非法操作，SNS 将使用寄存器中的 16 位 TSEQ 生成两次 NAK
REG_USL_TSEQ_NAK[15:0]。
• 接下来，SNS 将使用来自寄存器 REG_USL_TSEQ_ACK[15:0] 的 16 位 TSEQ 生成两次 ACK。
• 如果在 USL_REV 模式期间没有从 APP 成功接收到 USL 命令，则 SNS 应生成 TSEQ 值为 16'd0 的 ACK。
Note：
  SNS 应重置用于在 NAK 之后阻止执行后续命令操作的内部状态。

5. [USL_FWD Mode Exit] 

![image-20220319191420069](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319191420069.png)

#### 12.5 USL上电/复位，SNS配置和模式切换

本节详细介绍了 USL 操作的各个阶段。 请注意，当前 TX 在切换 USL 模式时应始终启动 BTA。

##### 12.5.1 USL Link Modes 

当以下情况出现时，USL链路处于USL反向模式(USL_REV):

• SNS is configured as RX 

• SNS is configured as TX 

• The channel is established to transfer payloads from APP to SNS 

当出现以下情况时，USL链路处于USL转发模式(USL_FWD):

 • SNS is configured as TX 

 • APP is configured as RX 

• The channel is established to transfer payloads from SNS to APP 

##### 12.5.2 USL Power-up / Reset 

上电或复位后，链路以USL_FWD方式出现:

• SNS 最初应将每个PHY 通道（包括时钟通道，如果存在）配置为发送器（TX）。
• APP 最初应将每个PHY 通道（包括时钟通道，如果存在）配置为接收器（RX）。
• 然后，SNS 和APP 应按照适当的PHY 定义的程序初始化它们的Lane 1 PHY；所有其他通道都是单向的，在需要之前应该保持未初始化。
• 如果USL 被配置为使用D-PHY ALP 模式，则时钟通道也应在其初始化后初始化并启动，以促进ALP 模式快速BTA 和数据通道1 上的高速双向数据传输；更多指导见第 9.12.5.6 节。
• 如果USL 配置为使用D-PHY LP/L VLP 模式，则时钟通道的初始化和启动可能会延迟，直到SNS 首次需要使用HS 模式向APP 发送数据包。
请注意，在这种情况下，所有通过数据通道 1 从 APP 到 SNS 的 USL 数据包传输都使用 Escape Mode LPDT，并且不需要时钟通道运行。
• 一旦 SNS 完成内部上电/复位校准，SNS 将使用第 9.12.3.2 节中概述的步骤在通道 1 上启动 BTA。在启动 BTA 之前，SNS 可以选择使用第 9.12.3.2 节中的读取完成格式发送 TX_USL_SNS_BTA_ACK_TIMEOUT[15:0] 的内容。
• BTA 完成后，链路配置为 USL_REV 模式，使 APP 能够使用 Lane 1 配置 SNS。
• 在SNS 配置过程中，APP 可以选择活动Lane 的总数，以便在启动图像流时能够初始化正确数量的单向Lane 并投入使用。

##### 12.5.3 USL SNS Configuration 

当链接处于 USL_REV 模式时，APP 可以使用第 9.12.3.1 节中概述的步骤配置 SNS。 
• 一旦APP 完成一项或多项SNS 配置操作，APP 应使用第9.12.3.1 节中概述的步骤启动BTA。
• BTA 完成后，链路配置为USL_FWD 模式。

##### 12.5.4 USL模式切换SNS配置

进入 USL_FWD 模式后，SNS 使用第 9.12.3.1 节中概述的步骤生成 USL NAK（如果需要），然后生成 USL ACK。 SNS 在内容可用时生成 USL 读取完成，并且在 USL_FWD 模式期间可能与非 USL 有效负载交错。

SNS 应支持表 21 和表 22 中定义的两个 16 位 USL BTA 开关寄存器。APP 应在 USL_REV 模式期间配置这些寄存器。

USL BTA 开关可以在传统摄影和视频应用的垂直消隐期间启动，或者在更高级的视觉应用的预定义 LgP（帧内）之后启动。 CSI-2 成像和视觉系统将需要映射到 PHY 的快速 BTA 持续时间。 图 86 显示了 APP 和 SNS 的 USL_REV 和 USL_FWD 状态图。

当系统上电时，SNS 配置为 RX（接收器），APP 配置为 TX（发送器）。 成像应用允许使用五种 USL 模式，以及图 86 中定义的过渡弧。

USL SNS应支持表25所示的16位操作寄存器。

一个USL SNS应该支持一个或多个16位GPIO寄存器，参见表26。

![image-20220319193418574](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319193418574.png)

![image-20220319193445917](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319193445917.png)

##### 12.5.5 ALP Fast BTA Timeout Support 

由于目标设备 PHY 在接收到来自发起方的“Initiate BTA”USL 命令请求时不提供确认电信号，因此 APP 使用以下 SNS 超时寄存器来缓解潜在的死锁。

![image-20220319193641386](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319193641386.png)

![image-20220319193704942](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319193704942.png)

![image-20220319194128102](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319194128102.png)

![image-20220319194431204](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319194431204.png)

##### 12.5.6 USL Clock Lane Management Under D-PHY ALP Mode (Informative) 

当 USL 配置为使用 [MIPI01] 中所述的 D-PHY ALP 模式时，SNS 和 APP 之间的任何活动数据通信都需要一个运行的高速时钟通道。 此类通信包括通常的前向 SNS 像素流，以及与 USL 数据包传输（DT 代码 0x38）相关的数据通道 1 上的任何双向事务，或由任一低级 PHY 快速 BTA 或 ULPS 进入命令发起 社交网络或应用程序。 当 USL 配置为使用 D-PHY LP 或 L VLP 模式时，时钟通道只需要在 SNS 像素流式传输期间运行。

1.12.5.6.1 System Power-Up and/or Reset (Informative) 

在 SNS ALP 模式 TX 和 RX PHY 初始化之后，SNS 将在切换到 USL_REV 模式之前以某个初始的、与实现相关的频率启动时钟通道。 例如，将此频率设置为等于 SNS 外部参考时钟频率（或其约数）可能是有利的，因为这避免了启动和锁定 PLL 的需要，从而可能节省数百微秒的启动延迟。 必须注意确保所选时钟通道频率至少是 D-PHY 规范要求的最小比特率的两倍，因为 USL 使用的 D-PHY 高速反向数据传输发生在四分之一 正向数据传输的比特率。 例如，2 MHz 时钟通道频率对应于 4 Mbps 的正向比特率和 1 Mbps 的反向比特率。

一旦clock Lane运行，SNS可以使用USL协议切换到USL_REV模式，以便APP根据需要配置SNS CCI寄存器，包括用于设置图像流所需的clock Lane频率的PLL控制寄存器。 APP 采取的最后一个动作是通过写入相应的 CCI 控制寄存器请求启动图像流，然后切换到 USL_FWD 模式； 作为响应，SNS 将停止时钟通道，将所有 PLL 锁定到其配置的频率，重新启动时钟通道，然后才真正开始图像流。

##### 12.5.7 动态时钟控制(信息型)

与传统 CSI-2 非连续时钟模式一样，USL 允许时钟通道在系统应用程序不支持或不立即预期 SNS 和 APP 之间的数据通信期间停止。

Examples of such periods include: 

• 图像流传输期间图像行之间的一个或多个“水平行消隐”(hblank) 周期（假设这些周期足够长，足以包括停止然后重新启动时钟所需的时序开销）。
• 图像流传输期间图像帧之间的全部或部分“垂直帧消隐”(vblank) 周期。
• “硬待机”期间，图像传感器功耗最低，并且不可能访问 CCI 寄存器。 硬备用进入和退出通常使用单独的控制信号（例如，[MIPI04] 中定义的 XSHUTDOWN 信号）进行控制。
• 全部或部分“软待机”期间，图像流暂时停止，以使 APP 能够重新配置有关基本 SNS 操作特性的 CCI 寄存器，例如输出图像分辨率、像素深度、帧速率、比特率、活动通道数、 等等。

停止和重新启动时钟通道可以由 SNS 自动按照 APP 预先理解的约定执行，或者响应于 APP 以特定方式触发 SNS。 例如，当时钟通道在 USL_REV 模式下运行时，APP 可以通过写入 SNS 上的特殊 CCI 控制寄存器来命令 SNS 停止时钟。 相反，当时钟  通道在 USL_REV 模式期间停止（意味着不能访问 CCI 寄存器），APP 可以通过向 SNS 发送一个短的 D-PHY ALP 模式 PHY 到 PHY 唤醒脉冲来请求 SNS 重新启动时钟（如 [ MIPI01]）。

请注意，更改时钟通道频率需要 SNS 停止时钟通道，内部调整频率（这可能涉及重新锁定 PLL），然后根据 D-PHY 规范重新启动时钟通道。 毋庸置疑，后一种操作可能会非常耗时，最好避免从 USL_FWD 模式切换到 USL_REV 模式，然后在相对较短的时间段内（例如 hblank）再次返回。 换句话说，如果 APP 需要在 hblank 期间访问 SNS CCI 寄存器，那么理想的情况是 SNS 和 APP 都可以支持以流式传输像素数据包的全比特率的四分之一的反向传输 ，从而避免在 hblank 期间需要更改时钟通道频率两次。

图 87 说明了图像流 vblank 期间 USL ALP 模式时钟通道管理的示例。从图中时间“A”开始，SNS在发送CSI-2 Frame End短包后自动保持clock Lane运行，并使用USL协议进入USL_REV模式，以使APP开始访问SNS CCI寄存器。此时，APP 也可能希望启动一个内部定时器，在 vblank 周期即将结束时进行警告。在时间“B”，当 APP 至少暂时完成对 CCI 寄存器的访问时，它执行的最后一次访问是写入 TX_USL_ALP_CTRL 寄存器（表 27），例如，它命令 SNS 关闭时钟通道但也期望稍后重新启动它的请求。然后保持 USL_REV 模式（所有 D-PHY 时钟和数据通道都处于停止状态）直到时间“C”，此时 APP 通过发送一个短的 ALP 模式 PHY 到 PHY 唤醒脉冲到社交网络。作为响应，SNS 重新启动时钟通道并使其保持运行，而 APP 根据需要执行更多的 CCI 访问。当在时间“D”完成时，APP 最终切换回 USL_FWD 模式，然后 SNS 必须在下一个图像帧的开头发送 Frame Start 短数据包。

![image-20220319195847151](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319195847151.png)



![image-20220319195904524](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319195904524.png)

##### 12.5.8 USL硬待机模式(信息型)

USL 硬备用进入和退出可以以与传统非 USL 图像传感器基本相同的方式进行控制。 在硬待机模式下，所有通道（包括时钟通道，如果适用）通常处于正向 ULPS 状态，并且 APP 无法访问任何 SNS CCI 寄存器。

SNS 进入硬备用通常由硬件输入信号（例如 XSHUTDOWN）触发，该信号可以相对于图像流异步断言。 每个 Lane 或多或少地立即转换到正向 ULPS 状态； 即，任何进行中的图像流都会简单地终止，并且不会传输 PHY ULPS 进入命令。 但是，APP通常会预先将SNS置于软待机模式，以便彻底停止图像流并将界面置于停止状态或ULPS。 触发硬待机时，SNS 和 APP 上的通道 1 也应自动切换到正向。

从硬备用退出由用于触发硬备用进入的相同硬件输入（例如，XSHUTDOWN）的置低触发，导致每个通道根据 [MIPI01] 中适用的 PHY 初始化程序转换到停止状态和 [ MIPI02]。

##### 12.5.9 USL软待机模式和ULPS进入/退出(信息型)

与传统的非 USL 图像传感器类似，USL 支持 SNS 通过 APP 写入 CCI 控制寄存器进入软待机模式，从而有序地停止图像流。 对于 USL，此 CCI 写入操作必须在 hblank 或 vblank 期间处于 USL_REV 模式时发生。 APP然后使用USL协议切换回USL_FWD模式，以使SNS能够输出所有或部分在控制寄存器被写入时正在进行的图像帧，以Stop中的所有数据通道结束 状态（并且时钟通道仍在 D-PHY ALP 模式情况下运行）。 然后 SNS 切换到 USL_REV 模式以使 APP 可以向 SNS 发出进一步的命令。

一旦流媒体停止，APP 可以选择不同的行动方案。 例如，如果执行时间关键的 SNS 模式更改，则 APP 可以立即更新所需的 SNS CCI 寄存器，通过写入请求重新启动图像流的 CCI 控制寄存器结束该过程，然后使用 USL 协议。 一旦返回 USL_FWD 模式，并且在实际重新启动图像流之前，SNS 可能必须禁用、重新配置和重新锁定一个或多个内部 PLL，如果适用，可能需要 SNS 自动停止和重新启动时钟通道。

另一个可能的行动方案是 APP 在等待来自 APP 的进一步命令时将 SNS 置于扩展睡眠状态。 实现这一点的首选方法是 APP 首先在通道 1 上向 SNS 发送 PHY ULPS 进入命令，然后是 SNS，然后在所有正向数据通道上向 APP 发送 PHY ULPS 进入命令。 首选此方法，因为 Lane 1 在整个 ULPS 期间保持 USL_REV 模式，从而使 APP 可以在需要时通过 Lane 1 向 SNS 发送 ULPS 唤醒信令； 这也是第 7.3 节中建议为 D-PHY 和 C-PHY 通道 1 提供“反向 ULPS”支持的原因。 在 D-PHY ALP 模式下，SNS 会在所有数据通道都已放入 ULPS 后停止时钟通道并将其放入 ULPS。 此时，SNS 可以在内部关闭不必要的电路，同时仍保留内部寄存器状态并保持在 USL_REV 模式。

对于 D-PHY ALP 模式，在软待机模式下的 ULPS 唤醒需要 APP 向 SNS 发送一个长 ALP 唤醒脉冲，如 [MIPI01] 中所述。 一旦 SNS 在数据通道 1 上开始接收后一个脉冲（例如，由 PPI 发出信号），它就可以开始在每个单向数据通道上向 APP 发送一个长的 ALP 唤醒脉冲，从而导致总的往返唤醒 延迟与正向或反向的延迟大致相同。 SNS 还将时钟通道唤醒到停止状态，然后重新启动它以使 APP 能够访问 SNS CCI 寄存器。

对于 C-PHY ALP 模式，在软待机模式下的 ULPS 唤醒需要 APP 向 SNS 发送扩展的 ALP-Pause 唤醒线状态，然后是 [MIPI02] 中所述的 ALP“停止”命令。此时，SNS 可以类似地在每个单向通道上发出 ULPS 唤醒信号。但是，这会导致总的往返唤醒延迟大约是正向或反向延迟的两倍。 APP 执行更多交易（例如，
 CCI 寄存器访问）使用通道 1，而唤醒在其他通道上进行，有效地隐藏了额外的延迟。另一种可能的解决方案是将 C-PHY 通道 1 或所有其他通道放入 ULPS，但不能同时放入。例如，APP 可以通过使用通道 1 写入 CCI 控制寄存器来请求 ULPS；然后，SNS 会将所有单向通道置于 ULPS 中，同时将通道 1 置于停止状态。相反，APP 可以通过仅将 Lane 1 放入 ULPS 来请求 ULPS，而 SNS 将所有单向 Lane 留在 Stop 状态。

### 13. Data Scrambling

数据加扰的目的是通过使用数据随机化技术将链路的信息传输能量分散到可能较大的频带上来减轻 EMI 和 RF 自干扰的影响。 本节中描述的加扰特性是可选的和规范性的：如果 CSI-2 实现包括对加扰的支持，则应按照本节中的描述实现加扰特性。 数据加扰的好处是众所周知的，强烈建议实施这种数据加扰功能，以最大限度地减少系统中的辐射发射。

数据加扰应按通道应用，如图 88 所示。通道分布功能的每个输出应由专用于该通道的单独加扰功能单独加扰，然后再将通道数据发送到 PHY 功能 Tx PPI。

![image-20220319203346422](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319203346422.png)

#### 13.1 CSI-2 Scrambling for D-PHY 

图 89 显示了使用 D-PHY 物理层时两个数据包在两个通道上的突发传输格式。 在传输开始后，传输 HS-ZERO 和 HS-SYNC，包头和数据有效载荷分布在两个通道上。

如果使用 D-PHY 物理层，则每个 Lane 中的加扰器线性反馈移位寄存器 (LFSR) 应在以下任一条件下使用 Lane 种子值进行初始化：

1. 在突发开始时，紧接在 D-PHY 生成的 HS-Sync 之后传输的第一个字节之前发生（适用于 D-PHY EPD 选项 1 和选项 2）。
  2. 在传输可选 D-PHY EPD 选项 1 HS-Idle 时生成的 HS-Sync 之后传输的第一个字节之前。

当使用可选的 D-PHY EPD 选项 2 时，不会在 CSI-2 数据包之间重新初始化扰频器。初始化扰频器时，应使用分配给每个通道的 16 位种子值初始化 LFSR。

![image-20220319204658503](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319204658503.png)

#### 13.2 CSI-2 Scrambling for C-PHY

图 90 显示了使用 C-PHY 物理层时两个数据包在两个通道上的突发传输格式。 在传输开始、前导和同步之后，每个通道上的数据包头被复制两次，每个数据包的数据有效负载分布在两个通道上。 如果使用 C-PHY 物理层，则每个通道中的加扰器 LFSR 应在每个长包头或短包的开头使用分配给每个通道的 16 位种子值之一进行初始化。 每次发送同步字时都会进行此初始化。

![image-20220319204912042](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220319204912042.png)

在某些情况下，图像可能会导致重复传输具有相同或相似长包头和相同像素数据的长包（例如：全暗像素，或全白像素）。 如果加扰器在每个数据包的开头使用相同的种子值初始化，与每个像素行的开头一致，则加扰伪随机序列将以传输相同图像数据行的速率重复。 这会导致发射不那么随机，而是在与传输图像数据行的速率相当的频率处出现峰值。

为了缓解这个问题，每次发送数据包标头时，发送器都会选择不同的种子值。 包头中的同步字对少量数据进行编码，以便发送器可以通知接收器使用哪个起始种子来解扰数据包。 同步字中的少量数据是通过发送 CSI-2 协议发送器选择的同步类型来发送的。 此同步类型值还用于选择加扰器和解扰器中的起始种子。

表 28 显示了 C-PHY 支持的五种可能的同步类型。 同步字值在 C-PHY 规范中进行了规范性规定，并在表 28 中为方便起见重复。 CSI-2 协议仅使用五种可能的同步类型中的前四种，这简化了实现。

![image-20220320121916415](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320121916415.png)

Note:

使用单个种子值时，同步类型 3 是默认的同步字值。

图 91 显示了单通道中的加扰架构。 PRBS 生成的伪随机数作为种子索引，在发送数据包之前从种子列表中选择初始种子值。 该种子索引也应使用 PPI 信号 TxSyncTypeHS0[1:0] 发送到 C-PHY。 TxSyncTypeHS0[2] 始终为零。 TxSyncTypeHS1 [2:0] 类似地用于 32 位数据路径。 C-PHY 确保突发中的第一个数据包以使用同步类型 3 的同步字开始。

![image-20220320122103952](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320122103952.png)

种子列表可以包含一个或四个初始种子值。 发送器和接收器应能够从种子列表中准确选择一个种子值。 当使用单个种子值时，该种子应标识为种子 3，并且发送器应始终发送同步类型 3。发送器和接收器还应具有从四个种子值列表中选择种子值的能力，如 图 91. 当使用四个种子值的列表时，应使用同步类型 0 到同步类型 3 将种子索引值从发送器传送到接收器。

当使用四个种子的列表时，应在发送器中使用伪随机发生器（例如 PRBS）生成两位种子索引。
PRBS 生成器实现的细微差别不会影响发送器和接收器的互操作性，因为接收器响应在发送器中选择的种子索引并使用同步类型传送给接收器。

在接收器处，C-PHY 解码同步字并将 2 位同步类型值传递给 CSI-2 协议逻辑。 CSI-2 协议逻辑使用两位值作为种子索引来选择四个种子值之一来初始化解扰器。 这个概念在图 91 中的单通道图中显示。图 88 显示了使用 PPI 信号来选择使用哪个种子值来初始化加扰器和解扰器。 由于种子选择字段是通过同步字传输的，因此不需要其他机制来协调接收器处特定解扰器初始种子值的选择。

![image-20220320122344316](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320122344316.png)

#### 13.3 Scrambling Details 

长包头、数据有效载荷、长包尾（可能包括填充字节）和短包应加扰。 PHY 生成的不受 CSI-2 协议控制的特殊数据字段不应被加扰。 为清楚起见，表 29 列出了所有未加扰的字段。

![image-20220320122523670](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320122523670.png)

数据加扰器和解扰器伪随机二进制序列 (PRBS) 应使用实现生成多项式的 LFSR 的伽罗瓦形式生成：

(x) = x 16 + x5 + x4 + x3 + 1 

表 30 中的初始 D-PHY 种子值应用于初始化通道 1 到 8 中的 D-PHY 加扰器 LFSR。

![image-20220320122658278](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320122658278.png)

表 31 中的初始 C-PHY 种子值应用于初始化通道 1 到 8 中的 C-PHY 加扰器 LFSR。该表为每个通道号的四个可能的同步类型值中的每一个提供初始种子值。 如果只使用一个同步类型，那么它应该默认为同步类型 3。

![image-20220320122746660](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320122746660.png)

对于需要超过 8 个通道的 D-PHY 和 C-PHY 系统，附件 G 为通道 9 到 32 提供了 24 个额外的种子值，以及为通道 33 和更高通道查找种子值的机制。 对于每个种子值，LSB 对应于扰码器 PRBS 寄存器位 Q0，MSB 对应于位 Q15。

LFSR 应在 G(x) 处为要加扰的每个有效载荷数据字节生成一个八位序列，从其初始种子值开始。 LFSR 将通过为每个后续有效载荷数据字节推进八个比特周期来生成新的 G(x) 比特序列。

加扰应通过八位序列 G(x) 与要加扰的 CSI-2 有效载荷数据的模 2 位加法 (X-OR) 来实现。

实施提示：PRBS 的 8 位值是 PRBS LFSR 寄存器的 Q15:Q8 位在每个第 8 位时钟上的翻转。 设计人员可以选择以并行形式实现 PRBS LFSR，以在单个字节时钟中移动相当于 8 个位置，或者甚至可以将 PRBS LFSR 配置为在单个字时钟中移动 8 个位置的倍数。

对于图 93 中所示的示例，Q[15:8] 被捕获在一个临时寄存器中，然后在再次捕获 Q[15:8] 之前将 PRBS LFSR 移位 8 次。 加扰的执行如下：

 • TxD[7] = PktD[7] ⊕ Q’[8];  
 • TxD[6] = PktD[6] ⊕ Q’[9];  
 • TxD[5] = PktD[5] ⊕ Q’[10];  
 • TxD[4] = PktD[4] ⊕ Q’[11];  
 • TxD[3] = PktD[3] ⊕ Q’[12];  
 • TxD[2] = PktD[2] ⊕ Q’[13];  
 • TxD[1] = PktD[1] ⊕ Q’[14];  
 • TxD[0] = PktD[0] ⊕ Q’[15];

![image-20220320123042456](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320123042456.png)

表 32 说明了 PRBS 寄存器的序列，每次一位，从通道 2 的初始种子值开始。数据加扰序列是输出 G(x)。 当寄存器包含初始种子值时，扰码器输出的第一位是 G(x)（也是图 93 中寄存器的 Q15）输出的值。

![image-20220320123140189](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320123140189.png)

表 33 显示了使用 D-PHY 物理层时通道 2 中 PRBS LFSR 产生的前十个 PRBS 字节输出

![image-20220320123221916](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320123221916.png)

表 34 显示了数据包开头的 PRBS 字输出示例，当使用 C-PHY 物理层时，这些输出由通道 2 中的 PRBS LFSR 生成。

![image-20220320123308187](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320123308187.png)

### 14.Smart Region of Interest (SROI) 

智能感兴趣区域 (SROI) 功能支持矩形感兴趣区域 (ROI) 的自适应传输。 SROI 可用于通过选择性地传输从原始图片（例如人脸或车牌）中提取的一个或多个较小的 ROI 来减少数据带宽。 SROI 在用于计算机视觉的相机以及移动市场以外的机器视觉应用（包括工业、监控和物联网）中都有用例。

除了减少数据带宽，SROI的好处还包括:

• 高帧率/低延迟：数据减少可以提高传感器帧率并减少应用处理器的处理工作量。
• 高分辨率：可以通过捕获必要的信息来提取高分辨率图像，而不会增加传输的数据量。
• 低功耗：处理工作负载的减少可以节省系统功耗。

#### 14.1Overview of SROI Frame Format 

如图 94 所示，每个矩形 ROI 由位置（ROI 矩形左上角像素的 X、Y 坐标）、大小（ROI 矩形的高度和宽度，以像素为单位）和其他元数据（例如，ADC 位 深度；见表 37)。

每个 ROI 都有一个对应的 ROI Information 字段，其中包含位置、大小和其他项目的值（参见第 9.14.5 节）。 ROI 信息值要么由集成到 AP 或图像传感器中的检测功能确定，要么对于某些用例（例如工厂自动化）是预先设置的。 检测 ROI 的方法超出了本规范的范围

ROI 的最大数量也超出了本规范的范围，因为它是特定于实现的。 然而，在 SROI 传输之前，应用处理器和图像传感器应就 ROI 的最大数量达成一致。

SROI帧使用三种数据包类型:

• SROI Short Packet 用于在带有 SROI Long Packet 的图像帧中传输帧开始 (FS) 包和帧结束 (FE) 包。
  如果需要line同步包，则线路开始 (LS) 包和线路结束 (LE) 包也被允许作为 SROI 短包。
  • SROI Embedded Data Packet 用于传输 ROI 信息，详见第 9.14.5 节。
  如果 SROI 嵌入数据包存在于图像帧中，则它应位于图像帧的开头。
  • SROI Long Packet（不包括 SROI Embedded Data Packet）用于传输图像帧的 ROI 图像数据。

SROI 长数据包应使用与传输图像帧的正常、非 SROI 图像行（例如 RAW10）相同的图像数据类型，但也允许使用用户定义的数据类型。 对于 SROI 传输，给定图像帧内的所有 SROI 长数据包不需要具有相同的长度。
当一行原始图像数据中有多个 ROI 时，所有 ROI 数据应合并为一个 SROI Long Packet，区域之间不留空白。 请参见图 94 中的 A(0) 和 D(0) 示例。
由多个 ROI 重叠的任何图像区域，例如图 94 中的 A(n-2) 和 B(0)，应仅传输一次（即，不应传输多次，每个 ROI 一次）。 因此，在计算 SROI Long Packet Word Count 字段的值时，每个重叠图像区域应仅计算一次（即，不应计算多次，每个 ROI 一个）。

术语SROI包是指任何SROI短包、SROI嵌入数据包或SROI长包。

![image-20220320124637033](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320124637033.png)

![image-20220320124717248](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320124717248.png)

#### 14.2 Transmission of SROI Embedded Data Packet 

本节规定图像传感器何时需要在图像帧中发送 SROI 嵌入式数据包 (SEDP)。

这取决于应用处理器 (AP) 是否知道或不知道所有必需的 ROI 信息（参见表 35）。 当图像传感器确实发送 SE DP 时，AP 检测 SROI 数据包的方法取决于 AP 和图像传感器同意使用的 SROI 数据包选项（选项 1 与选项 2，参见第 9.14.3 节）。

• 如果应用处理器已经知道接收 SROI 数据包所需的所有 ROI 信息，则图像传感器不需要传输 SEDP。 然而，图像传感器可以可选地传输SEDP。
如果发送 SEDP 并使用 SROI 数据包选项 1，则 AP 可以通过检查虚拟通道值来检测图像帧中的 SROI 数据包（参见第 9.14.3.1 节）。
• 如果应用处理器还不知道接收 SROI 数据包所需的所有 ROI 信息，则图像传感器应发送 SEDP。
AP 用于检测图像帧中 SROI 数据包的方法取决于正在使用的 SROI 数据包选项：选项 1 的虚拟通道（请参阅第 9.14.3.1 节），或选项 2 的数据类型（请参阅第 9.14.3.2 节） ）。

示例用例在9.14.4节中显示。

![image-20220320125050004](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320125050004.png)

#### 14.3 SROI Packet Detection Options

以下小节详细介绍了将图像帧中的 SROI 数据包与非 SROI 数据包区分开来的两个选项。
在 SROI 传输之前，应用处理器和图像传感器应就将使用两个选项（即选项 1 与选项 2）中的哪一个达成一致。

**SROI Packet Option 1** 

选项 1 通过使用不同的虚拟通道值来区分 SROI 数据包和非 SROI 数据包。
对于选项 1，给定图像帧的所有 SROI 数据包应使用相同的虚拟通道，并且此 SROI 虚拟通道应不同于图像帧的非 SROI 数据包中使用的虚拟通道（参见图 95）。 可以使用虚拟信道交织。
包含 SROI 数据包的图像帧可以使用数据类型交织，但前提是 SROI 长数据包中使用的数据类型与非 SROI 长数据包中使用的数据类型不同（例如，PDAF）。

![image-20220320125358838](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320125358838.png)

**SROI Packet Option 2** 

选项 2 按数据类型区分 SROI 数据包和非 SROI 数据包； 特别是，通过检测图像帧中是否存在 SROI 嵌入式数据包。
对于选项 2，给定图像帧的所有 SROI 数据包应使用非 SROI 数据包使用的相同虚拟通道（参见图 96）。
包含 SROI 数据包的图像帧可以使用数据类型交织，但前提是 SROI 长数据包中使用的数据类型与非 SROI 长数据包中使用的数据类型不同（例如，PDAF）。

![image-20220320125529614](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320125529614.png)

#### 14.4 SROI Use Cases (Informative) 

本节介绍了一些有用的用例来说明SROI特性的使用方式。

##### 14.4.1Use Case 1: ROI Detection by the Application Processor 

如果 AP 负责检测 ROI，或者预先设置了 ROI 信息，则图像传感器不需要传输 SROI Embedded Data Packet。 图 97 显示了一个示例。
在这个用例中，AP 检测 ROI（例如，人脸），然后通过 CCI 将 ROI 的位置和大小发送到图像传感器（参见第 6 节）。 然后，图像传感器仅根据排序的位置和大小划分出 ROI，然后继续为同一图像区域发送 SROI Long Packets。
在这种情况下，不发送 SROI 嵌入式数据包，因为它是不必要的：应用程序处理器已经知道接收 SROI 包所需的所有 ROI 信息（例如，ROI 位置和大小）。

![image-20220320125844458](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320125844458.png)

##### 14.4.2 Use Case 2: ROI Detection by the Image Sensor 

如果图像传感器负责检测 ROI，则图像传感器需要传输 SROI 嵌入式数据包，如图 98 所示。
AP 命令图像传感器获取 ROI（例如，人脸）。 图像传感器检测 ROI 并将其雕刻出来，然后继续跟踪 ROI。
在这种情况下，SROI 嵌入式数据包被（并且必须）传输，因为应用处理器不知道接收 SROI 数据包所需的所有 ROI 信息（例如，位置和大小）。

![image-20220320130020923](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320130020923.png)

#### 14.5 Format of SROI Embedded Data Packet (SEDP) 

对于 SROI 传输，嵌入式数据格式基于 MIPI CCS 规范 [MIPI04] 中的定义。 SROI 帧的嵌入式数据格式代码为 0x0D（1 个字节）。
SROI 嵌入式数据包（参见图 99）由多个 ROI 信息字段组成，每个字段都包含有关一个提取的图像区域的 ROI 信息。每个 ROI 信息字段应以 ROI ID 开头，然后是表 36 定义的 ROI 元素信息字段。表 37 定义了 ROI 元素信息字段中使用的类型 ID 子字段的值的分配和分配。
在嵌入数据行的末尾，应在最后一个 ROI 信息字段的末尾插入 End of Line。
SROI Embedded Data Packet 行的长度不得超过全分辨率图像数据行的长度。在行尾之后，如果 SROI 嵌入式数据包行短于全分辨率图像数据行的长度，则应使用 0x07 字符填充行的其余部分。

![image-20220320130242495](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320130242495.png)

![image-20220320130309234](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320130309234.png)

![image-20220320130351295](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320130351295.png)

![image-20220320130422205](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320130422205.png)

### 15 Packet Data Payload Size Rules

对于 YUV 、 RGB 或 RAW 数据类型，一个长数据包应包含一行图像数据。当数据包在同一虚拟通道内和数据包在同一帧内时，相同数据类型的每个长数据包应具有相同的长度。此规则的一个例外是 YUV420 数据类型，它在第 11.2.2 节中定义。
对于用户定义的基于字节的数据类型、USL 数据类型（代码 0x38）和 SROI 长数据包的数据类型，长数据包可以具有任意长度。数据包之间的间距也可以变化。
所有数据类型的长数据包内有效载荷数据的总大小应为 8 位的倍数。然而，如本规范其他地方定义的数据类型有效载荷数据传输格式也可能对有效载荷大小施加额外的限制。为了满足这些限制，有时可能需要在有效载荷的末尾添加一些“填充”像素，例如，当具有 RAW10 数据类型的数据包包含长度不是四个像素的倍数的图像行时RAW10 传输格式所要求的，如第 11.4.4 节所述。未指定此类填充像素的值。

### 16. Frame Format Examples 

这是一个内容丰富的部分。
本节包含三个示例来说明如何使用 CSI-2 功能。
  • 一般帧格式示例，图 100
  • 数字隔行扫描视频示例，图 101
  • 具有准确同步时序信息的数字隔行扫描视频，图 102

![image-20220320132056157](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320132056157.png)

![image-20220320132120477](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320132120477.png)

![image-20220320132148242](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320132148242.png)

### 17 Data Interleaving 

CSI-2 支持在同一视频数据流中交叉传输不同图像数据格式。
有两种方法可以交错传输不同的图像数据格式：
  • 数据类型
  • 虚拟频道标识符
  可以以任何方式组合前述的交织数据传输方法。

#### 17.1Data Type Interleaving 

数据类型值唯一地定义了该数据包的数据格式。接收方使用数据包头中的数据类型值来解复用包含不同数据格式的数据包，如图 103 所示。注意，图中每个数据包头中的虚拟通道标识符是相同的。
数据包有效载荷数据格式应与数据包头中的数据类型代码一致，如下所示：
 • 对于定义的图像数据类型——0x18 到 0x3F 范围内的任何非保留代码——只有单个相应的 MIPI 定义的数据包有效载荷数据格式应被认为是正确的
 • 保留的图像数据类型——0x18 到0x3F 范围内的任何保留代码——不得使用。对于保留的图像数据类型，不应认为任何数据包有效负载数据格式是正确的
 • 对于通用长包数据类型（代码 0x10 到 0x17）和用户定义的、基于字节的（代码 0x30 – 0x37），任何包有效载荷数据格式都应被认为是正确的
 • 通用长数据包数据类型（代码 0x10 到 0x17）和用户定义的、基于字节的（代码 0x30 – 0x37）不应与满足任何 MIPI 图像数据格式定义的数据包有效负载一起使用
 • 同步短包数据类型（代码 0x00 到 0x07）应仅由标头组成，不应包括有效载荷数据字节
 • 通用短包数据类型（代码 0x08 到 0x0F）应仅包含报头，不应包括有效载荷数据字节
数据格式在第 11 节中进一步定义。

![image-20220320132514255](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320132514255.png)

同一虚拟通道内的所有数据包，独立于数据类型值，共享相同的帧开始/结束和线路开始/结束同步信息。 根据定义，在同一虚拟通道内的帧开始和帧结束数据包之间的所有数据包，与数据类型无关，属于同一帧。
不同数据类型的数据包可以在图 104 所示的数据包级别或图 105 所示的帧级别交错。数据格式在第 11 节中定义

![image-20220320132654234](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320132654234.png)

![image-20220320132720709](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320132720709.png)

#### 17.2 Virtual Channel Identifier Interleaving 

虚拟通道标识符允许单个数据流中的不同数据类型在逻辑上彼此分离。图 106 说明了使用虚拟通道标识符的数据交织。
每个虚拟通道都有自己的帧开始和帧结束数据包（专用于 USL 数据包的虚拟通道除外；参见第 9.12 节）。因此，不同的虚拟通道可能具有不同的帧速率，尽管两个通道的数据速率将保持相同。此外，可以为每个虚拟通道使用数据类型值交织，从而允许虚拟通道内的不同数据类型和第二级数据交织。
因此，接收者应该能够基于虚拟通道标识符和数据类型值的组合来解复用不同的数据包。例如，包含相同数据类型值但在不同虚拟通道上传输的数据包被认为属于图像数据的不同帧（流）。

![image-20220320132852901](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320132852901.png)

## 七、Color Spaces 

本节中的色彩空间定义只是对其他标准的参考。 包含的参考资料仅用于提供信息的目的，而不是为了合规。 使用的色彩空间不限于给出的参考。

### 7.1 RGB Color Space Definition 

在本规范中，缩写 RGB 表示基于 IEC 61966 中 sRGB 定义的 8 位表示的非线性 sRGB 颜色空间。
8 位表示结果为 RGB888。 通过将 8 位值缩放为 5 位（蓝色和红色）和 6 位（绿色）来实现到更常用的 RGB565 格式的转换。 可以通过简单地删除 LSB 或舍入来完成缩放。

### 7.2 YUV Color Space Definition 

在本规范中，缩写 YUV 指的是 ITU-R BT601.4 中定义的 8 位伽马校正 YCBCR 颜色空间。

## 八、Data Formats

本节的目的是为 CSI-2 应用中通常使用的数据格式提供明确的参考。 表 38 总结了这些格式，然后是每种格式的单独定义。 表中未显示的通用数据类型在第 11.1 节中描述。 为简单起见，所有示例都是单通道配置。
CSI-2 应用中最广泛使用的格式通过表 38 中的“主要”名称来区分。CSI-2 的发送器实现应至少支持这些主要格式中的一种。 CSI-2 的接收器实现应支持所有主要格式。
包有效载荷数据格式应与包头中的数据类型值一致。 有关数据类型值的描述，请参见第 9.4 节。

![image-20220320133740647](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320133740647.png)

为清楚起见，本节图中的传输开始和传输结束序列已被省略。
本节的其余部分详细介绍了符合表 38 中列出的每种数据类型的像素序列和其他应用程序数据如何通过 CSI-2 像素到字节封装格式层转换为等效的字节序列，如图 3 所示。
本节中的各种图描述了这些字节序列，如图 107 的顶部所示，其中字节 n 总是在字节 m 之前，因为 n < m。另请注意，即使每个字节都以 LSB 优先顺序显示，但这并不意味着字节本身在输出之前由 Pixel to Byte Packing Formats 层进行位反转。
对于 D-PHY 物理层选项，序列中的每个字节都以 LSB 优先串行传输，而对于 C-PHY 物理层选项，序列中的连续字节对被编码，然后以 LSS 优先串行传输。图 107 说明了单通道系统的这些选项。

![image-20220320133939909](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320133939909.png)

### 8.1 Generic 8-bit Long Packet Data Types 

表39定义了通用的8位长数据包数据类型。

![image-20220320134113536](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320134113536.png)

#### 8.1.1 Null and Blanking Data 

对于空数据类型和空白数据类型，接收器必须忽略数据包有效负载数据的内容。
消隐数据包与空数据包的区别在于它在视频数据流中的重要性。 空包没有意义，而消隐包可以用作例如 ITU-R BT.656 风格视频流中帧之间的消隐线。

#### 8.1.2 Embedded Information

可以在每个图片帧的开头和结尾嵌入包含附加信息的额外行，如图 108 所示。如果存在嵌入信息，则包含嵌入数据的行必须使用数据标识符中的嵌入数据代码 .
在帧的开头可能有零行或多行嵌入数据。 这些行被称为帧头。
在帧的末尾可能有零行或多行嵌入数据。 这些行称为框架页脚。

![image-20220320134452047](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320134452047.png)

#### 8.1.3 Generic Long Packet Data Types 1 Through 4 

这些代码没有特定的定义，并且可以用于例如识别在图像帧内传输的各种类型的供应商特定元数据包。

### 8.2 YUV Image Data 

表 40 定义了本节中描述的 YUV 数据格式的数据类型代码。 YUV420 数据类型传输的行数应为偶数。
YUV420 数据格式分为遗留和非遗留数据格式。 旧的 YUV420 数据格式是为了与现有系统兼容。 非传统 YUV420 数据格式可实现成本更低的实施。

![image-20220320134644429](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320134644429.png)

#### 8.2.1 Legacy YUV420 8-bit

传统 YUV420 8 位数据传输是通过在奇数/偶数行中传输 UYY…/VYY… 序列来执行的。 U 分量在奇数行 (1, 3, 5 ...) 中传输，V 分量在偶数行 (2, 4, 6 ...) 中传输。 该序列如图 109 所示。
表 41 指定了 YUV420 8 位数据包的数据包大小限制。 每个数据包必须是表中值的倍数.

![image-20220320140150041](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140150041.png)

传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 像素到字节的映射如图 110 所示

![image-20220320140209714](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140209714.png)

![image-20220320140233689](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140233689.png)

有一种空间采样选择：

* H.261、H.263和MPEG1空间采样(图111)。

![image-20220320140338465](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140338465.png)

![image-20220320140353643](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140353643.png)

#### 8.2.2 YUV420 8-bit

YUV420 8 位数据传输是通过在奇/偶行中传输 YYYY…/UYVYUYVY… 序列来执行的。 奇数行 (1, 3, 5...) 仅传输亮度分量 (Y)，偶数行 (2, 4, 6...) 传输亮度 (Y) 和色度 (U 和 V) 分量。 偶数行 (UYVY) 的格式与 YUV422 8 位数据格式相同。 数据传输顺序如图 113 所示。
偶数行 (UYVY) 的有效负载数据大小（以字节为单位）是奇数行 (Y) 的有效负载数据大小的两倍。 这是一般 CSI-2 规则的例外，即每行应具有相等的长度。
表 42 指定了 YUV420 8 位数据包的数据包大小限制。 每个数据包必须是表中值的倍数。

![image-20220320140553108](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140553108.png)

传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 像素到字节的映射如图 114 所示。

![image-20220320140633560](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140633560.png)

![image-20220320140708083](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140708083.png)

有两种空间采样选择：

•H.261、H.263和MPEG1空间采样(图115)。

•MPEG2、MPEG4的色度偏移像素采样(CSPS)(图116)。

图117显示了YUV420帧格式。

![image-20220320140835752](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140835752.png)

![image-20220320140851880](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140851880.png)

![image-20220320140905790](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320140905790.png)

#### 8.2.3 YUV420 10-bit  

YUV420 10位数据传输是通过在奇/偶行中传输YYYY…/UYVYUYVY…序列来进行的。 只有亮度分量 (Y) 在奇数行 (1, 3, 5...) 中传输，亮度 (Y) 和色度 (U 和 V) 分量在偶数行 (2, 4, 6...) 中传输。 偶数行 (UYVY) 的格式与 YUV422 –10 位数据格式相同。 该顺序如图 118 所示。
偶数行 (UYVY) 的有效负载数据大小（以字节为单位）是奇数行 (Y) 的有效负载数据大小的两倍。 这是一般 CSI-2 规则的例外，即每行应具有相等的长度。
表 43 指定了 YUV420 10 位数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320141100197](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141100197.png)

传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 像素到字节的映射如图 119 所示。

![image-20220320141120178](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141120178.png)

![image-20220320141137473](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141137473.png)

像素空间采样选项与YUV420 8位数据格式相同。

![image-20220320141209350](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141209350.png)

#### 8.2.4 YUV422 8-bit 

YUV422 8位数据传输是通过传输一个UYVY序列来进行的。 该序列如图 121 所示。
表 44 指定了 YUV422 8 位数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320141324689](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141324689.png)

传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 像素到字节的映射如图 122 所示。

![image-20220320141406884](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141406884.png)

![image-20220320141430291](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141430291.png)

![image-20220320141457161](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141457161.png)

像素空间对齐与 CCIR-656 标准中的相同。 YUV422 的帧格式如图 124 所示。

![image-20220320141536128](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141536128.png)

#### 8.2.5 YUV422 10-bit 

YUV422 10位数据传输是通过传输一个UYVY序列来进行的。 该序列如图 125 所示。
表 45 指定了 YUV422 10 位数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320141721898](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141721898.png)

传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 像素到字节的映射如图 126 所示。

![image-20220320141756872](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141756872.png)

![image-20220320141818643](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141818643.png)

像素空间对齐与 YUV422 8 位数据情况相同。 YUV422 的帧格式如图 127 所示。

![image-20220320141857398](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320141857398.png)

### 8.3 RGB Image Data 

表46定义了本节中描述的RGB数据格式的数据类型代码。

![image-20220320142011416](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142011416.png)

#### 8.3.1 RGB888  

RGB888数据传输是通过传输一个BGR字节序列来进行的。 该序列如图 128 所示。RGB888 帧格式如图 130 所示。
表 47 指定了 RGB888 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320142119109](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142119109.png)

传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 像素到字节的映射如图 129 所示。

![image-20220320142156786](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142156786.png)

![image-20220320142220226](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142220226.png)

![image-20220320142241928](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142241928.png)

#### 8.3.2 RGB666

RGB666 数据传输是通过传输 B0…5、G0…5 和 R0…5（18 位）序列来执行的。 该序列如图 131 所示。RGB666 的帧格式如图 133 所示。
表 48 指定了 RGB666 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320142632320](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142632320.png)

传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 在 RGB666 情况下，一个数据字的长度是 18 位，而不是 8 位。 对 18 位 BGR 字进行逐字翻转； 即不是翻转每个字节（8位），而是翻转每个18位像素值。 如图 132 所示。

![image-20220320142651087](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142651087.png)

![image-20220320142714128](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142714128.png)

![image-20220320142740448](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142740448.png)

#### 8.3.3 RGB565

RGB565 数据传输是通过以 16 位序列传输 B0…B4、G0…G5、R0…R4 来执行的。 该序列如图 134 所示。RGB565 的帧格式如图 136 所示。
表 49 指定了 RGB565 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320142913246](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142913246.png)

传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 在 RGB565 情况下，一个数据字的长度是 16 位，而不是 8 位。 对 16 位 BGR 字进行逐字翻转； 即不是翻转每个字节（8位），而是每两个字节（16位）翻转。 如图 135 所示。

![image-20220320142952230](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320142952230.png)

![image-20220320143006414](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320143006414.png)

![image-20220320143020116](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320143020116.png)

#### 8.3.4 RGB555

RGB555 数据可以通过一些特殊安排通过 CSI-2 总线传输。 应该使 RGB555 数据看起来像 RGB565 数据。 这可以通过将填充位插入绿色分量的 LSB 来实现，如图 137 所示。
帧格式和封装尺寸约束均与 RGB565 情况相同。
传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 在 RGB555 情况下，一个数据字的长度是 16 位，而不是 8 位。 对 16 位 BGR 字进行逐字翻转； 即不是翻转每个字节（8位），而是每两个字节（16位）翻转。 图 137 对此进行了说明。

![image-20220320143134091](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320143134091.png)

#### 8.3.5 RGB444

RGB444 数据可以通过 CSI-2 总线通过一些特殊安排进行传输。 应该使 RGB444 数据看起来像 RGB565 数据。 这可以通过在每个颜色分量的 LSB 中插入填充位来实现，如图 138 所示。
帧格式和封装尺寸约束均与 RGB565 情况相同。
传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。 在 RGB444 情况下，一个数据字的长度是 16 位，而不是 8 位。 对 16 位 BGR 字进行逐字翻转； 即不是翻转每个字节（8位），而是每两个字节（16位）翻转。 如图 138 所示。

![image-20220320143244100](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320143244100.png)

### 8.4 RAW Image Data

RAW 6/7/8/10/12/14/16/20/24 模式用于从图像传感器传输原始图像数据。
其意图是原始图像数据是未经处理的图像数据（即原始拜耳数据）或补色数据，但原始图像数据不限于这些数据类型。
可以传输例如 除了有效像素外，还有遮光像素。 这导致行长度长于每行有效像素的总和的情况。 如果没有另外指定，行长度必须是字（32 位）的倍数。
表 50 定义了本节中描述的 RAW 数据格式的数据类型代码。

![image-20220320143541124](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320143541124.png)

#### 8.4.1 RAW6

6 位原始数据传输是通过 CSI-2 总线传输像素数据来完成的。 该序列如图 139 所示（VGA 情况）。 表 51 指定了 RAW6 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320144718185](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144718185.png)

每个6位像素先发送LSB。这是一般的CSI-2规则的一个例外。

![image-20220320144734787](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144734787.png)

![image-20220320144800957](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144800957.png)

![image-20220320144810418](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144810418.png)

#### 8.4.2 RAW7

7 位原始数据传输是通过在 CSI-2 总线上传输像素数据来完成的。 该序列如图 142 所示（VGA 情况）。 表 52 指定了 RAW7 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320144833834](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144833834.png)

每个7位像素先发送LSB。这是一般的CSI-2规则的一个例外，按字节顺序LSB优先。

![image-20220320144854358](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144854358.png)

![image-20220320144909709](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144909709.png)

![image-20220320144921076](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144921076.png)

#### 8.4.3 RAW8

8 位原始数据传输是通过在 CSI-2 总线上传输像素数据来完成的。 表 53 指定了 RAW8 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320144937820](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144937820.png)

这个序列如图145 (VGA case)所示。

传输的位序遵循一般的CSI-2规则，LSB优先。

![image-20220320144952511](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320144952511.png)

![image-20220320145006320](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145006320.png)

![image-20220320145018007](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145018007.png)



#### 8.4.4 RAW10

10 位原始数据的传输是通过将 10 位像素数据打包成 8 位数据格式来完成的。 表 54 指定了 RAW10 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320145033379](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145033379.png)

图148 (VGA case)说明了这个序列。

传输的位序遵循一般的CSI-2规则:LSB优先。

![image-20220320145048314](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145048314.png)

![image-20220320145102463](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145102463.png)

![image-20220320145112316](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145112316.png)

#### 8.4.5 RAW12

12 位原始数据的传输是通过将 12 位像素数据打包成看起来像 8 位数据格式来完成的。 表 55 指定了 RAW12 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320145125577](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145125577.png)

这个序列如图151 (VGA case)所示。

传输的位序遵循一般的CSI-2规则:LSB优先。

![image-20220320145140169](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145140169.png)

![image-20220320145153184](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145153184.png)

![image-20220320145201673](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145201673.png)



#### 8.4.6 RAW14

14 位原始数据的传输是通过将 14 位像素数据打包成 8 位片来完成的。 每四个像素生成七个字节的数据。 表 56 指定了 RAW14 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320145215961](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145215961.png)

该序列如图154 (VGA case)所示。

P1、P2、P3 和 P4 的 LS 位分布在三个字节中，如图 154 和图 155 所示。对于 P637、P638、P639 和 P640 的 LS 位也是如此。 字节传输期间的位顺序遵循一般的 CSI-2 规则，即 LSB 在前。

Note：
  图 154 已相对于 CSI-2 规范 2.0 版及更早版本中所示的图进行了修改，以便更清楚地与图 155 对应。RAW14 字节打包和传输格式本身相对于早期的 CSI-2 规范版本没有改变 .

![image-20220320145229072](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145229072.png)

![image-20220320145238662](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145238662.png)

![image-20220320145247662](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145247662.png)

#### 8.4.7 RAW16

16 位原始数据的传输是通过将 16 位像素数据打包成 8 位数据格式来完成的。 表 57 指定了 RAW16 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320145301164](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145301164.png)

这个序列如图157 (VGA case)所示。

传输的位序遵循一般的CSI-2规则:LSB优先。

![image-20220320145312868](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145312868.png)

![image-20220320145325174](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145325174.png)

![image-20220320145336765](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145336765.png)

#### 8.4.8 RAW20

20 位原始数据的传输是通过将 20 位像素数据打包成看起来像 10 位数据格式来完成的。 表 58 指定了 RAW20 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320145349112](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145349112.png)

这个序列如图160 (VGA case)所示。

传输的位序遵循一般的CSI-2规则:LSB优先。

![image-20220320145404437](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145404437.png)

![image-20220320145415318](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145415318.png)

![image-20220320145426852](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145426852.png)

#### 8.4.9 RAW24

24 位原始数据的传输是通过将 24 位像素数据打包成看起来像 12 位数据格式来完成的。 表 59 指定了 RAW24 数据包的数据包大小限制。 每个数据包的长度必须是表中值的倍数。

![image-20220320145439688](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145439688.png)

图163 (VGA case)说明了这个序列。

传输的位序遵循一般的CSI-2规则:LSB优先。

![image-20220320145449229](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145449229.png)

![image-20220320145459943](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145459943.png)

![image-20220320145516527](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145516527.png)

### 8.5 User Defined Data Formats

用户定义的数据类型值应用于通过 CSI-2 总线传输任意数据，例如 JPEG 和 MPEG4 数据。 数据应被打包，以便数据长度可被 8 位整除。 如果需要数据填充，则应在数据呈现给 CSI-2 协议接口之前添加填充。
传输中的位顺序遵循一般的 CSI-2 规则，LSB 在前。

![image-20220320145638684](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145638684.png)

以比特为单位的分组数据大小应能被 8 整除，即应传输整数字节。
对于用户定义的数据：
  • 帧作为任意大小的数据包序列传输。
  • 数据包大小可能因数据包而异。
  • 数据包之间的数据包间隔可能会有所不同。

![image-20220320145722781](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145722781.png)

有8种不同的User Defined数据类型代码，如表60所示。

![image-20220320145824746](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320145824746.png)

## 九、Recommended Memory Storage

本节内容丰富。
CSI-2 数据协议需要连接到 CSI 发送器的接收器的某些行为。 以下部分描述了应如何在接收器内部存储不同的数据格式。 虽然内容丰富，但提供此部分是为了通过建议不同接收器之间的通用数据存储格式来简化应用软件开发。

### 9.1 General/Arbitrary Data Reception 

在一般情况下，对于任意数据，传输的有效载荷数据的第一个字节映射 32 位存储器字的 LS 字节，传输的有效载荷数据的第四字节映射到 32 位存储器字的 MS 字节。
图 169 显示了通用 CSI-2 字节到 32 位内存字的映射规则。

![image-20220320150237961](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320150237961.png)

### 9.2 RGB888 Data Reception

RGB888数据格式字节到32位内存的字映射遵循通用的CSI-2规则。

![image-20220320151113863](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151113863.png)

### 9.3 RGB666 Data Reception

![image-20220320151127252](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151127252.png)

### 9.4 RGB565 Data Reception

![image-20220320151141579](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151141579.png)

### 9.5 RGB555 Data Reception

 ![image-20220320151157756](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151157756.png)

### 9.6 RGB444 Data Reception

RGB444数据格式字节到32位内存字映射有一个特殊的转换，如图174所示。

![image-20220320151217858](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151217858.png)

### 9.7 YVU422 8-bit Data Reception

YUV422 8 位数据格式字节到 32 位内存字的映射不遵循通用 CSI-2 规则。
对于 YUV422 8 位数据格式，传输的有效载荷数据的第一个字节映射 32 位内存字的 MS 字节，传输的有效载荷数据的第四个字节映射到 32 位内存字的 LS 字节。

![image-20220320151330054](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151330054.png)

### 9.8 YVU422 10-bit Data Reception

YUV422的10位数据格式字节到32位内存的字映射遵循通用的CSI-2规则。

![image-20220320151347230](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151347230.png)

### 9.9 YUV420 8-bit (Legacy) Data Reception 

YUV420 8 位（传统）数据格式字节到 32 位内存字映射不遵循通用 CSI-2 规则。
对于 YUV422 8 位（传统）数据格式，传输的有效载荷数据的第一个字节映射 32 位内存字的 MS 字节，传输的有效载荷数据的第四个字节映射到 32 位内存字的 LS 字节。

![image-20220320151506729](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151506729.png)

![image-20220320151522306](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151522306.png)

### 9.10 YVU420 8-bit Reception

YUV420的8位数据格式字节到32位内存的字映射遵循通用的CSI-2规则。

![image-20220320151543989](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151543989.png)

![image-20220320151554634](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151554634.png)

### 9.11 YVU420 10-bitData Reception

YUV420的10位数据格式字节到32位内存的字映射遵循通用的CSI-2规则。

![image-20220320151654201](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151654201.png)

![image-20220320151706280](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151706280.png)

### 9.12 RAW6 Data Reception

![image-20220320151719690](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151719690.png)

### 9.13 RAW7 Data Reception

![image-20220320151731736](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151731736.png)

### 9.14 RAW8 Data Reception

字节到32位内存字映射的RAW8数据格式遵循通用的CSI-2规则。

![image-20220320151748204](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151748204.png)

### 9.15 RAW10 Data Reception

RA W10数据格式字节到32位内存的字映射遵循通用的CSI-2规则。

![image-20220320151943090](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151943090.png)

### 9.16 RAW12 Data Reception

RA W12数据格式字节到32位内存的字映射遵循通用的CSI-2规则。

![image-20220320151957324](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320151957324.png)

### 9.17 RAW14 Data Reception

![image-20220320152010855](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320152010855.png)

### 9.18 RAW16 Data Reception

RA W16数据格式字节到32位内存的字映射遵循通用的CSI-2规则。

![image-20220320152023778](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320152023778.png)

### 9.19 RAW20 Data Reception

AW20数据格式字节到32位内存的字映射遵循通用的CSI-2规则。

![image-20220320152036181](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320152036181.png)

### 9.20 RAW24 Data Reception

AW24数据格式字节到32位内存的字映射遵循通用的CSI-2规则。

![image-20220320152046517](C:\Users\zhilu.zhang\AppData\Roaming\Typora\typora-user-images\image-20220320152046517.png)

## 十、补充部分。

### 1、关于C-PHY编码方式

#### 10.1 信号采样的发展方式

##### 1 信号加时钟

最传统的使用时钟和信号线传输，都是采用时钟信号在高电平或者是上升沿的时候采样。因此传输的最大速率是时钟频率相同的。 X bps = X hz.

下图是基于上升沿的采样例子，早期camera的并口（CPI）就是基于这样的一个采样形式。一般都是在下降沿之后改变信号上升沿时进行点平的判断。下图中红色的线的位置是采样的时机。

![传统采样方式1](E:\截图\传统采样方式1.jpg)

另外的种就是基于时钟在高电平时候的采样形式，常用的I2C信号就是基于这么一种形式。下图中红色实线为采样时机。

![传统采样方式2](E:\截图\传统采样方式2.jpg)

##### 2 D-Phy使用的 DDR同步的上升下降沿

但是随着时钟频率的提升，信号也变得容易被干扰。导致采样的时候出现错误。因此后来出现了差分信号。即是用一对信号线代表一个信号。两个信号（时钟）线的差用来代表信号的状态。当P线为1，N线为0的时候为信号1，当P线为0，N线为1的时候为信号0. 每组信号线一般在文献中被称为Lane，与 line 所区别。使用差分信号的好处是能够通过两条线的差抵抗其它信号的干扰。当有一个正向或者负向干扰的时候两者的差，并不会变。

除了差分信号还使用了DDR的采样方法，一个时钟周期采样两次的方式进行采样。上升沿一次下降沿一次，这样的好处是相同的时钟周期中信号的传输速度增加了一倍及Y bps =2 * Xhz。在相同频率下传输速率增大了一倍，减少了对外的干扰。

下图中蓝色虚线部分是采样时机。

![传统采样方式3](E:\截图\传统采样方式3.jpg)

##### 3 M-Phy和 C-Phy使用的两种嵌入式时钟

先介绍M-Phy ，M-Phy 的嵌入式使用不同宽度的高低电平作为一组信号的周期，以高低点平的宽度来判断数据为0还是为1. 因为每次都为一个相同的时间周期。因此信号本身也可以理解为时钟信号就不需要额外的时钟信号，这也就是M-Phy 的嵌入式使用的做法。在M-Phy中最复杂的还是其使用的8b10b信道编码，其使用的是IBM01《A DC-Balanced, Partitioned-Block, 8B/10B Transmission Code》这本身是用在ADSL 传输协议上的一种编码，大家有兴趣可以自己翻下。使用这个编码也是M-Phy 能够传输这么高速率数据的主要原因。

![传统采样方式4](E:\截图\传统采样方式4.jpg)

而C-Phy 采用了另外一种方式嵌入时钟。C-Phy 有三根信号线，不像M-Phy 和D-Phy ， C-Phy 并不是严格意义上的差分信号线。而是更像我们的工业用电一样，它使用三根信号之间的差作为信号判断。其三根信号之间必然有一根在 3/4V 一根在1/2 V 一根在1/4V。三根线在同一时刻的状态一定不懂，因此其有六个不同的状态。协议中使用+x，-x, +y，-y, +z，-z 代表。

![传统采样方式5](E:\截图\传统采样方式5.jpg)

下图中红色虚线为采样时机。

![传统采样方式6](E:\截图\传统采样方式6.jpg)

C-Phy 的时钟就是靠这六种状态机之间的转换形成的。这中间有三点很重要：

* 1 C-Phy 每次传输周期三根线的状态必须发生变化，即状态机的切换代表一次传输周期。在协议中状态的切换速度记作 sym/s

* 2 每次传输符号所代表的意义是由上次状态到这次状态的切换所代表的。也就是说两次状态的变化才代表了到底传输了什么符号。

* 3 6种状态每次由一种状态变换到另外一种状态最多能有五种不同的可能性。因此这个编码是个5进制编码。

![传统采样方式7](E:\截图\传统采样方式7.jpg)

C-Phy 的最大传输速率是2.5Gsymbols/s，如果按照每个符号能代表5个数的话。按照bp/s计算的最大传输速率应该为2.5G*log2(5)约为 2.5G*2.32.但是Spec上却是使用的2.5G*2.28这是为什么呢？是因为在这中间C-Phy 又做了一个16bit to7symbols的编码过程。16/7 约为2.28左右。16bit to 7symbols编码解码过程如下。

![传统采样方式8](E:\截图\传统采样方式8.jpg)

##### 4 本小结引用

[Camera接口之MIPI D-Phy，M-Phy，C-Phy信号采样](https://zhuanlan.zhihu.com/p/37373801)
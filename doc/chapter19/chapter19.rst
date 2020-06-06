1. DMA—直接存储区访问

本章参考资料：《IMXRT1050RM》（参考手册）。

学习本章时，配合《IMXRT1050RM》第22章Enhanced Direct Memory Access
(eDMA)和第21章Direct Memory Access Multiplexer
(DMAMUX)一起阅读，效果会更佳，特别是涉及到寄存器说明的部分。

特别说明，本书内容是以RT1050系列控制器资源讲解。

1. DMA简介

DMA(Direct Memory
Access,直接存储区访问)为实现数据高速在外设寄存器与存储器之间或者存储器与存储器之间传输提供了高效的方法。之所以称之为高效，是因为DMA传输实现高速数据移动过程几乎不占用CPU。从硬件层次上来说，DMA控制器是独立于Cortex-M4内核的，有点类似GPIO、USART外设一般，只是DMA的功能是可以快速移动内存数据。

RT1052的DMA功能齐全，工作模式众多，配合DMA多路复用模块(DMAMUX)可以更加灵活、高效的使用DAM资源，适合不同编程环境要求。RT1052的DMA支持外设到存储器传输、存储器到外设传输和存储器到存储器传输三种传输模式。这里的外设一般指外设的数据寄存器，比如ADC、SPI、I2C、DCMI等等外设的数据寄存器，存储器一般是指片内SRAM、外部存储器、片内Flash等等。

外设到存储器传输就是把外设数据寄存器内容转移到指定的内存空间。比如进行ADC采集时我们可以利用DMA传输把AD转换数据转移到我们定义的存储区中，这样对于多通道采集、采样频率高、连续输出数据的AD采集是非常高效的处理方法。

存储区到外设传输就是把特定存储区内容转移至外设的数据寄存器中，这种多用于外设的发送通信。

存储器到存储器传输就是把一个指定的存储区内容拷贝到另一个存储区空间。功能类似于C语言内存拷贝函数memcpy，利用DMA传输可以达到更高的传输效率，特别是DMA传输几乎不占用CPU的，可以节省很多CPU资源。

1. DMA功能框图

RT1052的DMA模块分为两个主要模块: eDMA驱动模块(eDMA
engine)和传输控制描述符TCD(transfer-control
descriptor)。eDMA驱动模块分为4部分，如图中标号1到4所示，传输控制描述符分为两部分，如图中标号⑤和⑥处所示。

eDMA驱动模块和TCD的关系可以简单理解为执行任务者和任务清单之间的关系。eDMA驱动模块如同执行任务者，他根据任务清单选择需要执行的任务，并将任务的执行过程和执行结果记录到任务清单。对应到DMA数据传输，TCD就是一块最多只能记录32个传输任务的存储单元，eDMA驱动模块从TCD中读取传输任务，在执行过程中更新传输进度到TCD，执行完成后将源地址、目的地址等一些信息写回TCD。

|image0|

图 19‑1 DMA框图

1. ①地址路径（Address path）

该模块主要完成通道的仲裁和地址的计算。RT1052的DMA拥有32个通道，同一时间只能有一个DAM通道运行。如果有个DAM通道正在运行运行此时另外一个DMA通道被激活RT1052根据控制寄存器DCHPRIn[ECP]的值选择不同的处理方式，当DCHPRIn[ECP]
=
1，工作在抢占模式，DMA根据预选设定好的通道优先级选择优先级最高的执行。当DCHPRIn[ECP]
= 0，工作在轮转模式，DMA根据通道编号从高到低依次执行。

DMA完成一次基本的数据读、写之后根据原地址偏移寄存器（TCDa_SOFF）和目的地址偏移寄存器（TCDa_DOFF）的设定值自动计算下一次数据传输的源地址和目的地址。原地址偏移寄存器和目的地址偏移寄存器是有符号的寄存器，根据符号决定一次基本传输之后地址增加还是减少。

RT1052的DMA传输分为次循环和主循环，次循环用于指定一次DMA请求传输的数据量（单位为字节），主循环用于指定次循环的次数。

每当完成基本数据传输或完成一个次循环或者完成主循环都能够调整DAM的读写地址，这些工作是由该模块完成的。

1. ②数据路径（Data path）

该模块实现总线主读写数据路径。它包括一个数据缓冲区和多路复用逻辑（multiplex
logic）以满足不同的数据对齐要求。内部读数据总线是主要输入，内部写数据总线是主要输出。

地址和数据路径模块直接支持两级流水线内部总线。地址路径模块表示总线管道的第一阶段(地址阶段)，而数据路径模块实现管道的第二阶段(数据阶段)。

1. ③编程模型与通道仲裁（Program model/channel arbitration）

此块实现eDMA编程模型的第一部分以及通道仲裁逻辑。编程模型寄存器连接到内部外围总线。eDMA外围请求输入和中断请求输出也连接到这个块。

1. ④控制模块（Control）

此块提供eDMA引擎的所有控制功能。对于源和目标大小相等的数据传输，eDMA引擎执行一系列源读/目标写操作，直到次循环字节数中指定的字节数移动为止。对于大小不相等的描述符，对于较大的引用，需要多次访问较小的数据。例如，如果源大小引用16位数据，而目标是32位数据，则执行两次读取，然后执行一次32位写入。

1. ⑤存储阵列(Memory array)

TCD由存储阵列和存储控制器组成。存储阵列可以简单理解为一块存储空间，用于保存32个DAM通道的传输信息、配置信息以及状态信息。存储阵列存储结构如图
19‑2所示。

|image1|

图 19‑2TCD存储阵列结构

从图
19‑2不难看出每个通道拥有11个传输描述寄存器，一些传输描述符我们可能会感到陌生，不要着急，我们稍后讲解eDMA基本概念和工作过程时会详细介绍。首先简单了解这些传输描述寄存器的作用，各个寄存器的作用如表格
19‑1所示。

注释：在图
19‑2中如果仔细数，可能会认为每个通道拥有18或16个描述寄存器。其实不是的。因为根据配置不同次循环传输数据量寄存器、主循环计数和通道连接寄存器、和主循环起始计数值寄存器的略有差异，如果查看《IMXRT1050RM》（参考手册）会发现这些名字不同的寄存器的基址与地址偏移是相同的，所以是一个寄存器。

表格 19‑1传输描述寄存器

+-----------------------------------+-----------------------------------+
| 寄存器名                          | 作用                              |
+===================================+===================================+
| TCD0_SADDR                        | 源地址寄存器，保存数据源的起始地址 |
+-----------------------------------+-----------------------------------+
| TCD0_SOFF                         | 源地址偏移，完成一次基本读写后源地址的变化量，(有符号类型) |
+-----------------------------------+-----------------------------------+
| TCD0_ATTR                         | 传输属性寄存器，主要用于设置源地址和目的地址的数据宽度 |
+-----------------------------------+-----------------------------------+
| TCD0_NBYTES_MLNO                  | 次循环传输数据量寄存器，根据次循环偏移使用情况分为禁止次循环偏移、 |
|                                   | 开启但没有使用次循环偏移、开启并使用了次循环偏移。这三种情况对应同 |
| TCD0_NBYTES_MLOFFNO               | 一个寄存器，但是寄存器各个位的含义 |
|                                   |                                   |
| TCD0_NBYTES_MLOFFYES              |                                   |
|                                   | 不同                              |
+-----------------------------------+-----------------------------------+
| TCD0_SLAST                        | 源地址最终调整寄存器，当主循环完成后，可通过设置该寄存器修改最终的 |
|                                   | 源地址                            |
+-----------------------------------+-----------------------------------+
| TCD0_DADDR                        | 目的地址寄存器，保存写入区域的起始地址 |
+-----------------------------------+-----------------------------------+
| TCD0_DOFF                         | 目的地址偏移寄存器，完成一次基本读写后目的地址的变化量，(有符号类 |
|                                   | 型)                               |
+-----------------------------------+-----------------------------------+
| TCD0_CITER_ELINKNO                | 主循环计数和通道连接寄存器，同次循环传输数据量寄存器类似，根据配置 |
|                                   | 不同，寄存器的各个位含义不同。    |
| TCD0_CITER_ELINKYES               |                                   |
+-----------------------------------+-----------------------------------+
| TCD0_DLASTSGA                     | 目的地址最终调整寄存器，当主循环完成后，可通过设置该寄存器修改最终 |
|                                   | 的目的地址                        |
+-----------------------------------+-----------------------------------+
| TCD0_CSR                          | TCD控制和状态寄存器。             |
+-----------------------------------+-----------------------------------+
| TCD0_BITER_ELINKNO                | 主循环起始计数值寄存器，同主循环计数和通道连接寄存器类，根据配置不 |
|                                   | 同，寄存器的各个位含义不同。特别注意，初始化时 |
| TCD0_BITER_ELINKYE                |                                   |
|                                   |                                   |
|                                   | 要保证与主循环起始计数值寄存器的值相等。 |
+-----------------------------------+-----------------------------------+

1. ⑥存储控制器(Memory controller)

存储控制器用于管理来自eDMA引擎的访问以及来自内部外围总线的访问。在同时访问的情况下，eDMA引擎优先级更高，外围总线的访问被停止。

1. eDAM传输基本概念

   1. 数据宽度

DMA的源数据宽度与目的数据宽度可以不同。数据宽度的设置是通过TCDa_ATTR寄存器设置的。当源数据宽度与目的数据宽度相同时，执行一次读取执行一次写入。当源数据宽度小于目的数据宽度，例如源数据宽度为16位，目的数据宽度为32位，则DMA执行两次读取执行一次写入。

1. 次循环与主循环

次循环和主循环可以用于控制传输的数据量，次循环用于设置一次DMA传输请求传输的数据量单位为字节，主循环用于设置执行多少个次循环之后停止该DMA通道的传输。

根据是否使用次循环映射，次循环设置寄存器有三个选择如下所示

-  CR[EMLM] =
   0，禁用次循环映射。这种情况下TCD_NBYTES_MLNO寄存器的0到31位（共32位）用于指定一个次循环（即一个DMA传输请求）传输的数据量（和选择的源数据宽度和目的数据宽度无关）。当一个次循环执行完成之后使用源地址寄存器（TCDa_SADDR）和目的地址寄存器（TCDa_DADDR）重新初始化源地址和目的地址，所以当执行下次小循环时将接着上次结束时的地址继续执行。

-  CR[EMLM] = 1，启用次循环映射，TCDa_NBYTES_MLOFFNO[SMLOE] =
   0，TCDa_NBYTES_MLOFFNO[DMLOE] =
   0，禁用源地址和目的地址次循环偏移。这种情况下使用TCDa_NBYTES_MLOFFNO[NBYTES]寄存器指定一个次循环传输的数据量。

-  CR[EMLM] = 1，启用次循环映射，TCDa_NBYTES_MLOFFYES[SMLOE] =
   1或者TCDa_NBYTES_MLOFFYES[DMLOE] =
   1，启用源地址或目的地址次循环偏移。TCDa_NBYTES_MLOFFYES[NBYTES]
   寄存器指定一个次循环传输的数据量（和选择的源数据宽度和目的数据宽度无关），
   当一个次循环执行完成之后，当前地址加上偏移寄存器TCDa_NBYTES_MLOFFYES[MLOFF]设定的原地址或目的地址偏移值，作为下次传输的起始地址

    主循环与次循环的关系如图 19‑3所示。

    |image2|

图 19‑3主循环与次循环

根据是否使用通道连接主计数值得设置选择不同的寄存器，分为如下两种情况

-  禁止次循环通道连接。TCDa_BITER_ELINKNO[ELINK] =
   0表示禁止通道连接，此时TCDa_BITER_ELINKNO[BITER]用于指定主循环计数值，如果设置只执行一次则该寄存应该设为1，注意:
   TCDa_BITER_ELINKNO[BITER]寄存器与相对应的TCDa_CITER_ELINKNO[BITER]寄存器的值应该。

-  使能通道连接。TCDa_BITER_ELINKYES[ELINK] = 1
   表示启用通道连接。TCDa_BITER_ELINKYES[BITER]用于指定次循环执行次数，TCDa_BITER_ELINKYES[LINKCH]
   = n
   表示当前通道与通道n连接，当一个次循环执行结束后DAM自动设置TCDn_CSR[START]寄存器触发一次通道n的DMA传输。

注意：在初始化时TCDa_BITER_ELINKYES寄存器与TCDa_BITER_ELINKYES寄存器的设置要相同，这两个寄存器分别表示启始主循环次数和当前主循环次数。同样如果禁止通道连接TCDa_CITER_ELINKNO寄存器与TCDa_BITER_ELINKNO寄存器的设置要相同。

1. 源地址和目标地址的设置及地址偏移设置

TCDa_SADDR寄存器与TCDa_DADDR寄存器分别用于设置DMA源始地址与目的启始地址。

TCDa_SOFF与TCDa_DOFF寄存器分别用于设置DMA执行一次读写操作之后原地址和目的地址的偏移值。当完成一个次循环之后DMA重新使用源起始地址寄存器（TCDa_SADDR）和目的启始寄存器（TCDa_DADDR）初始化DMA当前读写地址。如果启用了次循环映射还会添加次循环映射指定的偏移值。

当主循环计时结束之后能够使用TCDa_SLAST寄存器与TCDa_DLASTSGA最终调整DMA的读写地址。

1. 通道组优先级与通道优先级

T1052拥有32个DMA通道，这32个通道被分为两个通道组，通道组0包含通道0到15，通道组1包含通道16到31。

CR[ERCA] 寄存 控制是否使用固定优先级模式，只有CR[ERCA] =
0时优先级的设置才有效。CR[GRP0PRI]寄存器与CR[GRP1PRI]寄存器分别用于设置通道组0与通道组1的优先级。初始状态下CR[GRP0PRI]
= 0，CR[GRP1PRI] =
1。优先级数值越大对应的优先级越高。默认情况下，DMA通道组1的所有通道的优先级高于DMA通道组0的所有通道的优先级。

DCHPRIn(n取0到31)，每一个DMA通道有各自的通道优先级设置寄存器。DCHPRIn[CHPRI]寄存器用于设置通道优先级。对于通道组0，默认情况下通道优先级与通道号减1。对于通道1，默认情况下通道优先级等于通道号减16。DCHPRIn[ECP]寄存器配置该通道是否允许被更改优先级的通道打断，默认情况下是允许。DCHPRIn[DPA]寄存器配置该通道是否能够打断较低优先级的通道，默认情况下是能够打断的。

如果我们使用通道固定优先级（CR[ERCA] =
0）并且优先级设置保持默认则通道的优先与通道编对应，通道编号越大优先级越高。

1. DAM请求

RT1052大多数外设能够申请DMA传输请求，在RT1052官方的SDK库中定义了114个DMA请求源，部分代码如代码清单
19‑1

代码清单 19‑1DMA请求源（MINXRT1052.h）

1 typedef enum \_dma_request_source

2 {

3 kDmaRequestMuxFlexIO1Request0Request1 = 0|0x100U, /**< FlexIO1 \*/

4 kDmaRequestMuxFlexIO2Request0Request1 = 1|0x100U, /**< FlexIO2 \*/

5 kDmaRequestMuxLPUART1Tx = 2|0x100U, /**< LPUART1 Transmit \*/

6 kDmaRequestMuxLPUART1Rx = 3|0x100U, /**< LPUART1 Receive \*/

7 kDmaRequestMuxLPUART3Tx = 4|0x100U, /**< LPUART3 Transmit \*/

8 kDmaRequestMuxLPUART3Rx = 5|0x100U, /**< LPUART3 Receive \*/

9 kDmaRequestMuxLPUART5Tx = 6|0x100U, /**< LPUART5 Transmit \*/

10 kDmaRequestMuxLPUART5Rx = 7|0x100U, /**< LPUART5 Receive \*/

11 kDmaRequestMuxLPUART7Tx = 8|0x100U, /**< LPUART7 Transmit \*/

12 kDmaRequestMuxLPUART7Rx = 9|0x100U, /**< LPUART7 Receive \*/

13 kDmaRequestMuxCSI = 12|0x100U, /**< CSI \*/

14 . . .

15 . . .

16 . . .

17 } dma_request_source_t;

借助DMAMUX模块
(后面将会讲到)，每一个DMA通道可以选择任意一个DAM触发源作为DMA的触发信号。这样可以极大的提高DMA使用的灵活性与使用效率。

1. 通道错误与中断

错误状态寄存器（ES）列出了所有可能的错误状态。当发生通道错误时每个通道可以独立配置处理方式，可以选择忽略错误也可以选择产生错误中断。错误中断使能寄存器（EEI）是一个32位寄存器，每一位控制一个通道，我们可以直接修改该寄存设置通道发生错误时是否产生中断，也可以通过清除错误中断使能寄存器（SEEI）禁止错误中断寄存器（CEEI）设置单个通道。

1. eDMA基本工作流程

在讲解eDMA之前我们以执行者与任务清单为例简单介绍了eDMA工作流程。这一小节结合eDMA功能框图详细介绍eDAM从请求到传输结束的全过程。

1. 激活eDMA传输通道

激活eDMA传输通道的数据流如图 19‑4所示。

|image3|

图 19‑4激活通道流程

1. ①eDMA外部传输请求

eDMA传输通道被激活的前提是有eDMA传输请求或者寄存器TCDn_CSR[START]被置1。两种方式eDMA的激活流程是相同的。

1. ② 进行优先级仲裁

eDMA请求通过控制模块之后，进入程序模型和通道总裁模块，根据CR[ERCA] 寄存
器的配置，仲裁使用使用固定优先级或循环算法。如果使用固定优先级则高优先级的通道可以打断低优先级优先得到处理，低优先级只能等待高优先级执行结束。使用循环算法情况下eDMA会根据通道号从大到小依次执行，但是如果一个传输通道正在执行此时另外一个通道号更大的请求不会打断当前的传输。

1. ③将eDMA请求通道编号转化为TCD地址

仲裁通过后，eDMA通道编号经过地址路径模块转化为地址，用于访问TCDn的本地内存。

1. ④读取并加载地址路径

第③部分已经得到eDMA传输通道的传输描述符的地址，该部分的作用是将传输描述符加载到eDMA引擎。至此，一切准备就绪，可以开始DMA传输。

1. 进行数据传输

eDMA从TCD获取传输信息之后即可开始DMA传输，传输工作是由硬件自动完成的，程序员无需过多关心传输的具体过程。数据传输过程如图
19‑5所示。

|image4|

图 19‑5DMA传输过程

1. ①地址计算

DMA传输过程中大多数情况下需要不断的移动读写地址，这部分工作由地址路径模块完成的，它根据从TCD读取得到的配置参数完成地址的计算。

1. ②数据路径

数据路径模块根据地址路径模块提供的地址信息执行源读取，并且读取的数据被暂存在数据路径模块中，当达到目标写数据宽度后再执行写入操作。

1. ③控制模块

如果源地址与目的地址数据宽度不同，则控制模块根据数据宽度的差异控制数据路径模块。例如源数据宽度为16位目的数据宽度为32位，则数据路径模块会执行两次读操作之后执行一次写操作。当传输完成后控制模块向外发出传输完成标志。

1. 更新传输控制描述符(TCD)

一个次循环传输完成后，执行数据传输带额最后阶段，更新传输控制描述符(TCD)。如图
19‑6所示。

|image5|

图 19‑6更新传输控制描述符(TCD)

一个次循环完成后需要更新TCD中的某些寄存器，如TCDn_SADDR、TCDn_DADDR、TCDn_CITER_ELINKNO。如果主循环计数完成则还要处理其他任务，例如可选的中断请求、最终调整源地址和目标地址、重新加载TCDn_BITER_ELINKNO和TCDn_CITER_ELINKNO寄存器等。

1. DMAMUX简介及使用方法

DMA多路复用器(DMAMUX)将DMA源(称为槽)路由到32个DMA通道中的任何一个。如图
19‑7所示。

|image6|

图 19‑7DMAMUX功能框图

DMAMUX的主要目的是使用户方便灵活的使用DMA。DMAMUX为每个DMA通道提供了一个通道配置寄存器CHCFGa（a取0到31）如

|image7|

图 19‑8CHCFGa寄存器

通过这些寄存器可以独立的设置每个通道的DMA触发源、工作模式等。CHCFGa[SOURCE]用于指定通道的DMA触发入源，RT1052的SDK库中列出了114个DMA输入源如代码清单
19‑1所示。CHCFGa[ENBL]、CHCFGa[ENBL]、CHCFGa[A_ON]寄存器用于设置DMA的工作方式，如表格
19‑2所示。

表格 19‑2DMAMUX工作模式配置

+-------------+-------------+-------------+-------------+-------------+
| ENBL        | TRIG        | A_ON        | 功能        | 模式        |
+=============+=============+=============+=============+=============+
| 0           | x           | x           | 禁止DMA通道 | 禁用模式(Disabl |
|             |             |             |             | ed          |
|             |             |             |             | Mode)       |
+-------------+-------------+-------------+-------------+-------------+
| 1           | 0           | 0           | 启用DMA通道不使用触 | 正常模式(Normal |
|             |             |             | 发功能      |             |
|             |             |             |             | Mode)       |
+-------------+-------------+-------------+-------------+-------------+
| 1           | 1           | 0           | 启用DMA并且使用触发 | 周期触发模式 |
|             |             |             | 功能        |             |
|             |             |             |             | (Periodic   |
|             |             |             |             | Trigger     |
|             |             |             |             | Mode)       |
+-------------+-------------+-------------+-------------+-------------+
| 1           | 0           | 1           | DMA通道一直运行 | 始终运行模式 |
|             |             |             |             |             |
|             |             |             |             | (Always On  |
|             |             |             |             | Mode)       |
+-------------+-------------+-------------+-------------+-------------+
| 1           | 1           | 1           | DMA一直运行并且使用 | 等待触发模式 |
|             |             |             | 触发        |             |
|             |             |             |             | (Always On  |
|             |             |             |             | TriggerMode |
|             |             |             |             | )           |
+-------------+-------------+-------------+-------------+-------------+

下面简要讲解这几种工作模式

-  禁用模式(Disabled Mode)，禁用该DMA通道

-  正常模式(Normal
   Mode)，正常模式是最常用的一种工作模式，适合于所有DMA通道。
   CHCFGa[SOURCE]寄存指定了DMA传输请求源（Source），DAM通道每收到一传输请求信号执行一次传输，待传输完成之后（次循环与主循环执行结束）自动停止。

-  周期触发模式(Periodic Trigger
   Mode)，该模式只适合DMA的前4个通道（0到3）该模式下DMA工作过程如图
   19‑9所示。

    |image8|

图 19‑9周期触发模式工作过程

从图中可以看出，在周期触发模式下只有当有外部请求时周期性触发信号才能触发DMA请求。

-  始终运行模式(Always On
   Mode)，在始终运行模式下DMA通道不断的执行从源地址传输数据到目的地址，一次传输执行完成之后不会停止，循环执行。

-  等待触发模式 (Always On
   TriggerMode)，与周期触发模式对比，该模式相当于外部请求信号一直存在，只要产生周期触发信号就会产生DMA请求。

   1. DMA初始化结构体详解

RT1052的SDK库为DMA的初始化建立了两个初始化结构体，edma_config_t用于配置DMA的工作方式，
edma_transfer_config_t 用于配置DMA传输设置。
编程时我们只需要修改这两个结构体提供的配置选项即可。

1. DMA配置结构体

代码清单 19‑2edma_config_t初始化结构体(fls_edma.h)

1 typedef struct \_edma_config

2 {

3 bool enableContinuousLinkMode; /*是否开启次循环连接模式*/

4 bool enableHaltOnError; /*是否允许错误停止模式*/

5 bool
enableRoundRobinArbitration;/*选择使用固定优先级模式或轮询通道仲裁模式*/

6 bool enableDebugMode; /*是否使能Debug模式*/

7 } edma_config_t;

-  次循环连接的作用是当该通道的一个次循环执行结束后自动切换到连接的通道执行。可以配置连接到自身，这样该通道一个次循环执行结束之后自动开启下一次循环。enableContinuousLinkMode
   = 1,开启次循环通道连接。enableContinuousLinkMode =
   0，关闭次循环通道连接。

-  如果在DMA传输过程中发生错误，我们可以通过enableHaltOnError配置项设置如何处理DMA错误，enableHaltOnError
   = 1
   如果一个通道发生则忽略所有通道的DMA传输请求，直到错误标志位被清除。enableHaltOnError
   = 0 忽略错误。

-  DMA拥有32个通道，同一时间只能有一个通道传输数据，当多个通道请求传输数据时根据enableRoundRobinArbitration配置选项决定使用固定优先级模式还是根据通道号从大到小依次执行。enableRoundRobinArbitration
   =
   0，根据预先设定的通道优先级选择当前执行的通道，enableRoundRobinArbitration
   = 1,忽略通达优先级，根据通道编号从大到小依次执行。

-  enableDebugMode，设置是否使能Debug模式。enableDebugMode =
   0，在Debug模式下，DMA正常运行不受影响。enableDebugMode =
   1，DMA延时新通道的启动，允许当前执行通道执行完成，当退出Debug模式后通道恢复正常运行。

   1. DMA 传输配置结构体edma_transfer_config_t

代码清单 19‑3edma_transfer_config_t初始化结构体(fsl_edma.h)

1 typedef struct \_edma_transfer_config

2 {

3 uint32_t srcAddr; /*源数据地址*/

4 uint32_t destAddr; /*目的数据地址 \*/

5 edma_transfer_size_t srcTransferSize; /*源数据宽度*/

6 edma_transfer_size_t destTransferSize; /*目的数据宽度*/

7 int16_t srcOffset; /*源地址偏移*/

8 int16_t destOffset; /*目的地址偏移*/

9 uint32_t minorLoopBytes; /*次循环，传输字节数*/

10 uint32_t majorLoopCounts; /*主循环，循环计数值（循环次数）*/

11 } edma_transfer_config_t;

-  srcAddr，源数据地址。用于设置源地址的起始地址。

-  destAddr，目的数据地址。用于设置目的地址的起始地址。

-  srcTransferSize，源数据的宽度。该配置项是一个枚举类型，多种数据宽度可选，如代码清单
   19‑4

代码清单 19‑4数据宽度定义(fsl_edma.h)

1 typedef enum \_edma_transfer_size

2 {

3 kEDMA_TransferSize1Bytes = 0x0U, /\* 一次传输1个字节 \*/

4 kEDMA_TransferSize2Bytes = 0x1U, /\* 一次传输2个字节 \*/

5 kEDMA_TransferSize4Bytes = 0x2U, /\* 一次传输4个字节*/

6 kEDMA_TransferSize8Bytes = 0x3U, /\* 一次传输8个字节*/

7 kEDMA_TransferSize16Bytes = 0x4U, /*一次传输16个字节*/

8 kEDMA_TransferSize32Bytes = 0x5U, /*一次传输32个字节*/

9 } edma_transfer_size_t;

我们根据实际需要选择数据宽度即可。如果选择数据宽度为32字节则DMA数据总线要传输8次（32位总线每次最多传输4字节）才能完成一次传输，在此期间即使使用了固定通道优先级，高优先级通道也不能打断该通道的传输。一次传输是原子性的，不能被打断。

-  destTransferSize，目的地址宽度。类似于源数据宽度。

-  srcOffset，源地址偏移。当DMA完成一次传输，该寄存器用于设置下一个读地址与当前读地址的偏移。该寄存器是有符号的，设置位正值，表示地址增加。为负值，表示地址减少。单位为字节。

-  destOffset，目的地址偏移。类似于源地址偏移。

-  minorLoopBytes，设置一个次循环传输的字节数。

-  majorLoopCounts，设置主循环计数值。用于设置次循环执行次数。当主循环执行完成，如果开启了中断则会触发DMA传输完成中断。

   1. DMA传输句柄edma_handle_t

代码清单 19‑5传输句柄edma_handle_t (fsl_edma.h)

1 typedef struct \_edma_handle

2 {

3 edma_callback callback; /*主循环完成回调函数 \*/

4 void \*userData; /*回调函数参数 \*/

5 DMA_Type \*base; /*eDMA基地址 \*/

6 edma_tcd_t \*tcdPool; /*指向 TCDs 的指针*/

7 uint8_t channel; /*eDMA 通道编号 \*/

8 /*第一个TCD的索引号. 该编号指定的TCD将会被加载到eDMA驱动器*/

9 volatile int8_t header;

10
/*最后一个TCD的索引号.该编号指定的TCD将会从eEMA驱动器保存到TCD存储结构
\*/

11 volatile int8_t tail;

12 volatile int8_t tcdUsed; /*已经使用的 TCD 槽数量. \*/

13 volatile int8_t tcdSize; /*在队列中TCD槽总数*/

14 uint8_t flags; /*当前eDMA通道状态 \*/

15 } edma_handle_t;

edma_handle_t传输句柄结构比较复杂，使用过程中会结合一些结构体直接操作eDMA相关寄存。但是如果我们使用RT1052官方提供的相关函数则使用起来非常简单，这里只对该句柄做简单介绍，如果想深入了解传输句柄可以参考fsl_edma.c/h相关代码。

-  Callback
   指定主循环完成的回调函数，该变量是edma_callback类型的函数指针，函数原型为void
   (*edma_callback)(struct \_edma_handle \*handle, void \*userData, bool
   transferDone, uint32_t tcds)。

-  userData 回调函数参数。

-  Base 指定DMA基址，

-  tcdPool ，指向TCD的指针，TCD是Transfer Control
   Descriptor的缩写，即传输控制描述符。tcdPool是edma_tcd_t类型的结构体指针，edma_tcd_t结构体如代码清单
   19‑6

代码清单 19‑6edma_tcd_t结构体(fsl_edma.h)

1 typedef struct \_edma_tcd

2 {

3 \__IO uint32_t SADDR; /*SADDR 寄存器,用于设置源地址 \*/

4 \__IO uint16_t SOFF; /*SOFF 寄存器, 每次传输之后的源地址偏移量 \*/

5 \__IO uint16_t ATTR; /*ATTR 寄存器, 源和目的传输的数据宽度*/

6 \__IO uint32_t NBYTES; /*Nbytes 寄存器, 用于设置次循环传输字节数 \*/

7 \__IO uint32_t SLAST; /*SLAST 寄存器 \*/

8 \__IO uint32_t DADDR; /*DADDR 寄存器, 用于设置目的地址 \*/

9 \__IO uint16_t DOFF; /*DOFF 寄存器, 每次传输之后目的地址偏移量 \*/

10 \__IO uint16_t CITER; /*CITER 寄存器, 用于保存次循环未完成的字节数*/

11 /*DLASTSGA 寄存器, 在scatter-gather 模式下用于保存下一个eDMA的TCD \*/

12 \__IO uint32_t DLAST_SGA;

13 \__IO uint16_t CSR; /*CSR 寄存器, 用于保存 TCD 状态 \*/

14 \__IO uint16_t BITER; /*BITER 寄存器, 次循环计数值. \*/

15 } edma_tcd_t;

edma_tcd_t结构体定义了eDMA传输控制寄存器。我们通过传输配置结构体（edma_transfer_config_t）以及其他方式设置的配置参数通过调用EDMA_SubmitTransfer（）
函数初始化这些寄存器。

-  channel，DMA通道号。

   1. DMA存储器到存储器模式实验

DMA工作模式多样，具体如何使用需要配合实际传输条件具体分析。接下来我们通过两个实验详细讲解DMA不同模式下的使用配置，加深我们对DMA功能的理解。

DMA运行高效，使用方便，在很多测试实验都会用到，这里先详解存储器到存储器和存储器到外设这两种模式，其他功能模式在其他章节会有很多使用到的情况，也会有相关的分析。

存储器到存储器模式可以实现数据在两个内存的快速拷贝。我们先定义一个静态的源数据，然后使用DMA传输把源数据拷贝到目标地址上，最后对比源数据和目标地址的数据，看看是否传输准确。

1. 硬件设计

DMA存储器到存储器实验不需要其他硬件要求，只用到RGB彩色灯用于指示程序状态，关于RGB彩色灯电路可以参考GPIO章节。

1. 软件设计

这里只讲解核心的部分代码，有些变量的设置，头文件的包含等并没有涉及到，完整的代码请参考本章配套的工程。这个实验代码比较简单，主要程序代码都在main.c文件中。

1. 编程要点

1) 初始化DMAMUX；

2) 设置DMA通道的工作模式并使能通道

3) 设置传输配置结构体；

4) 创建传输句柄，提交DMA传输请求；

5) 编写传输完成回调函数。

   1. 代码分析

**DMA宏定义及相关变量定义**

代码清单 19‑7DMA 相关宏及变量定义(bsp_dma.h)

1
/*------------------------第一部分，宏定义--------------------------------*/

2 #define EXAMPLE_DMAMUX DMAMUX //DMAMUX 基址

3 #define EXAMPLE_DMA DMA0 //DMA基址

4 #define eDAM_Channel 0 //DMA通道

5 #define BUFF_LENGTH 4U //输出缓冲区长度

6

7
/*------------------------第二部分，变量定义----------------------------*/

8 edma_handle_t g_EDMA_Handle; //定义eDAM传输句柄

9 volatile bool g_Transfer_Done = false;//定义传输完成标志

10 uint32_t srcAddr[BUFF_LENGTH] = {0x01, 0x02, 0x03,
0x04};//源数据缓冲区

11 uint32_t destAddr[BUFF_LENGTH] = {0x00, 0x00, 0x00,
0x00};//目的数据缓冲区

根据注释不难看出各个变量和宏定义的作用。第一部分定义了本工程使用的一些宏定义，其中eDAM_Channel定义了使用的通道号，因为本程序中使用了DMAMUX所以可以选择任意通道（0到31），BUFF_LENGTH定义缓冲区长度，即第二部分
定义的数组的长度，本程序使用两个数组，分别作为源和目的。使用DMA将数组srcAddr[BUFF_LENGTH]的内容搬移到数组destAddr[BUFF_LENGTH]。最终打印数组destAddr[BUFF_LENGTH]的内容，验证DMA传输是否正确。

g_EDMA_Handle 是eDMA传输句柄，是DMA初始化的核心，

**DMAMUX设置**

代码清单 19‑8初始化DMAMUX通道(bsp_dma.c)

1 /\* 初始化DMAMUX \*/

2 DMAMUX_Init(EXAMPLE_DMAMUX);

3 /*设置DMA 通道一直处于活动状态*/

4 DMAMUX_EnableAlwaysOn(EXAMPLE_DMAMUX, eDAM_Channel, true);

5 /*使能通道*/

6 DMAMUX_EnableChannel(EXAMPLE_DMAMUX, eDAM_Channel);

DMAMUX模块用于实现32个DAM通道与一百多个DMA请求源之间的连接。默认情况下这些通道都是关闭的，使用之前要开启。开启通道有两种方式，一种使用代码清单
19‑8函数DMAMUX_EnableAlwaysOn（）初始化为“一直活动模式”在该模式下我们随时能够使用该通道传输数据。另一种使用DMAMUX_SetSource（）设置和通道关联的触发源，只有DMA通道收到有效触发信号时才执行输出传输。

**初始化DMA**

代码清单 19‑9DMA工作模式及传输设置(bsp_dma.c)

1
/*-----------------------------第一部分-------------------------------*/

2 /*获取默认配置*/

3 EDMA_GetDefaultConfig(&userConfig);

4 /*初始化eDMA*/

5 EDMA_Init(EXAMPLE_DMA,&userConfig);

6

7
/*------------------------------第二部分------------------------------*/

8 /*初始化传输配置结构体*/

9 transferConfig.srcAddr = (uint32_t)srcAddr; //源地址

10 transferConfig.srcOffset = 4; //源地址偏移

11 transferConfig.srcTransferSize =
kEDMA_TransferSize4Bytes;//源数据宽度

12

13 transferConfig.destAddr = (uint32_t)destAddr;//目的地址

14 transferConfig.destOffset = 4; //目的地址偏移

15 transferConfig.destTransferSize =
kEDMA_TransferSize4Bytes;//目的数据宽度

16

17 transferConfig.minorLoopBytes = 16; //次循环传输字节数

18 transferConfig.majorLoopCounts = 1; //主循环计数值

-  第一部分，设置DMA的工作模式，适用于所有的通道，函数EDMA_GetDefaultConfig
   用于获取默认的配置，参数userConfig是edma_config_t类型的结构体，用于保存默认的配置参数。默认选项如代码清单
   19‑10所示。

代码清单 19‑10edma_config_t默认配置(fsl_edma.c)

1 enableRoundRobinArbitration = false; //使用固定优先级模式，

2 enableHaltOnError = true; //使能错误停止

3 enableContinuousLinkMode = false; //不使用次循环通道连接

4 enableDebugMode = false; //禁止DEBUG模式（在Debug 模式下DMA正常运行）

获得默认配置之后，如果不需要修改则调用EDMA_Init（）函数即可完成初始化。

-  第二部分，初始化传输配置结构体，有关传输配置结构体详细介绍请参考代码清单
   19‑3edma_transfer_config_t初始化结构体。这里我们设置源地址和目的地址的数据宽度为4字节，源地址偏移和目的地址偏移设置为4。次循环传输数据量为16字节，主循环计数值为1。

**初始化传输句柄**

代码清单 19‑11传输句柄初始化(bsp_dma.c)

1 /*创建eDMA句柄*/

2 EDMA_CreateHandle(&g_EDMA_Handle, EXAMPLE_DMA, eDAM_Channel);

3 /*设置传输完成回调函数*/

4 EDMA_SetCallback(&g_EDMA_Handle, EDMA_Callback, NULL);

5 /*提交eDAM传输请求*/

6 EDMA_SubmitTransfer(&g_EDMA_Handle, &transferConfig);

7 /*启动传输*/

8 EDMA_StartTransfer(&g_EDMA_Handle);

RT1052
DAM的初始化最终是通过传输句柄完成的，传输句柄中保存了一个DMA传输通道的所有信息。传输句柄的实现比较复杂，有兴趣的话可以仔细研究这些有关传输句柄的函数特别是EDMA_SubmitTransfer函数。我们这里只简要讲解各个函数的作用。

-  函数EDMA_CreateHandle，用于创建一个传输句柄。该函数首先将传输句柄的所有内容清零，然后根据传入的DMA基址和通道号初始化传输句柄。

-  函数EDMA_SetCallback，初始化传输句柄的回调函数和回调函数参数。当DAM拥有一个可选的主循环执行结束中断，如果开启了中断，主循环执行结束后会跳转到回调函数中执行。在创建任务句柄时默认开启了传输完成中断，在传输完成中断服务函数中调用了相应的回调函数，并且在RT1052的官方库中已经实现了中断服务函数。

-  函数EDMA_SubmitTransfer，提交DMA传输请求。传输配置结构体transferConfig保存了DMA通道的配置信息，通过该函数将这些配置信息保存到通道对应的TCD。

-  函数EDMA_StartTransfer，传输句柄设置完成之后调用该函数即可启动DMA传输。

**回调函数**

代码清单 19‑12DMA传输完成回调函数(bsp_dma.c)

1 void EDMA_Callback(edma_handle_t \*handle,\\

2 void \*param, \\

3 bool transferDone, \\

4 uint32_t tcds)

5 {

6 if (transferDone)

7 {

8 g_Transfer_Done = true;

9 }

10 }

在回调函数中我们只是根据回调函数参数transferDone
判断传输是否完成，如果传输完成将全局变量g_Transfer_Done设置为“true”。

**主函数**

代码清单 19‑13 存储器到存储器模式主函数(main.c)

1 int main(void)

2 {

3 uint32_t i = 0;//用于for循环

4 /*-----------此处省略系统初始化和打系统时钟相关代码------*/

5

6 /*----------------------------第一部分-------------------*/

7 /\* 初始化LED引脚 \*/

8 LED_GPIO_Config();

9 DMA_Config();

10 while(1)

11 {

12 /*--------------------------第二部分-------------------*/

13 while (g_Transfer_Done != true)

14 {

15 //等待传输完成

16 }

17 /*--------------------------第三部分-------------------*/

18 /\* 打印目的缓存区内容 \*/

19 PRINTF("\r\n eDAM 存储器到存储器传输完成\r\n");

20 PRINTF("目的地址是数据为:\r\n");

21 for (i = 0; i < BUFF_LENGTH; i++)

22 {

23 PRINTF("%d\t", destAddr[i]);

24 }

25 while(1)

26 {

27 ;

28 }

29 }

30 }

代码第一部分调用DMA_Config();函数初始化DMA，并且开启了DMA传输。代码第二部分，等待传输完成。g_Transfer_Done是在bsp_dma.c文件中定义的一个全局变量，用于向main函数传递当前DMA传输状态。第三部分，输出目的地址的内容。

1. 下载验证

确保开发板供电正常，编译程序并下载。打开串口调试助手，观察开发板数据信息。正常情况下输出内容与源地址缓冲区设定的值相同。

1. DMA存储器到外设模式实验

DMA存储器到外设传输模式非常方便把存储器数据传输外设数据寄存器中，这在STM32芯片向其他目标主机，比如电脑、另外一块开发板或者功能芯片，发送数据是非常有用的。RS-232串口通信是我们常用开发板与PC端通信的方法。我们可以使用DMA传输把指定的存储器数据转移到USART数据寄存器内，并发送至PC端，在串口调试助手显示。

1. 硬件设计

存储器到外设模式使用到UART1功能，具体电路设置参考UART章节，无需其他硬件设计。

1. 软件设计

这里只讲解核心的部分代码，有些变量的设置，头文件的包含等并没有涉及到，完整的代码请参考本章配套的工程。我们编写两个串口驱动文件bsp_usart_dma.c和bsp_usart_dma.h，有关串口和DMA的宏定义以及驱动函数都在里边。

1. 编程要点

1) 配置UART通信功能；

2) 设置DMA工作模式，设置DMAMUX；

3) 创建DMA传输句柄、UART DAM句柄；

4) 编写UART传输完成回调函数；

5) 编写主函，实现接收、发送功能。

   1. 代码分析

**DMA相关宏定义**

在程序中一般使用宏重命令使用的外设，这样的好处是移植代码时只需要在头文件中修改宏定义的值即可。如代码清单
19‑14所示。

代码清单 19‑14宏定义(bsp_dam_uart.h)

1 /***********************此处省略串口GPIO相关宏定义*****************/

2

3 /*定义本程序使用的串口*/

4 #define DEMO_LPUART LPUART1

5 /*UART时钟频率*/

6 #define DEMO_LPUART_CLK_FREQ BOARD_DebugConsoleSrcFreq()

7 #define LPUART_TX_DMA_CHANNEL 0U //UART发送使用的DMA通道号

8 #define LPUART_RX_DMA_CHANNEL 1U //UART接收使用的DMA通道号

9 #define LPUART_TX_DMA_REQUEST
kDmaRequestMuxLPUART1Tx//定义串口DMA发送请求源10#define
LPUART_RX_DMA_REQUEST kDmaRequestMuxLPUART1Rx//定义串口DMA接收请求源

11 /*定义所使用的DMA多路复用模块(DMAMUX)*/

12 #define EXAMPLE_LPUART_DMAMUX_BASEADDR DMAMUX

13 #define EXAMPLE_LPUART_DMA_BASEADDR DMA0 //定义使用的DAM

14 #define ECHO_BUFFER_LENGTH 8 //UART接收和发送数据缓冲区长度

为节省篇幅，这里省略了串口GPIO相关的宏定义，详细请参考本章配套程序。结合宏定义的名字和注释不难看出这些宏定义的作用，这里不再赘述。需要说明一下两点。

1. 本程序选择的是UART1，我们都知道，UART1是系统串口，在main函数的开始部分已经完成了初始化，这里再次将其初始化为使用DMA传输。为方便移植，本章配套程序添加了GPIO初始化部分，如果在实际应用中需要使用其他串口只需要修改引脚宏定义和串口号宏定义即可。

2. 串口的发送和接收使用了两个不同的DMA通道，两个通道都要初始化。

**初始化串口**

代码清单 19‑15UART初始化(bsp_dma_uart.c)

1 void UART_Init(void)

2 {

3 lpuart_config_t lpuartConfig;//定义LUART初始化结构体

4

5 /**********************第一部分***********************/

6 /*初始化UART引脚*/

7 UART_GPIO_Init();

8 /\* LPUART.默认配置 \*/

9

10 /*********************第二部分***********************/

11 /\*

12 \* lpuartConfig.baudRate_Bps = 115200U;

13 \* lpuartConfig.parityMode = kLPUART_ParityDisabled;

14 \* lpuartConfig.stopBitCount = kLPUART_OneStopBit;

15 \* lpuartConfig.txFifoWatermark = 0;

16 \* lpuartConfig.rxFifoWatermark = 0;

17 \* lpuartConfig.enableTx = false;

18 \* lpuartConfig.enableRx = false;

19 \*/

20 LPUART_GetDefaultConfig(&lpuartConfig);

21 lpuartConfig.baudRate_Bps = BOARD_DEBUG_UART_BAUDRATE;

22 lpuartConfig.enableTx = true;

23 lpuartConfig.enableRx = true;

24

25 /*************************第三部分*******************/

26 LPUART_Init(DEMO_LPUART, &lpuartConfig, DEMO_LPUART_CLK_FREQ);

27 }

UART初始化与之前的串口收发初始化类似。第一部分，初始化UART使用的外部引脚。第二部分，获取UART的默认配置并在默认配置基础上修改配置项。第三部分，调用LPUART_Init函数完成初始化。

**串口DMA传输句柄**

串口DMA传输句柄实际是一个结构体，这个结构体经过初始化后会保存串口DMA传输所需的配置信息和状态信息，如代码清单
19‑16所示。

代码清单 19‑16LPUART eDMA结构体(fsl_lpuart_edma.h)

1 struct \_lpuart_edma_handle

2 {

3 lpuart_edma_transfer_callback_t callback; /*回调函数*/

4 void \*userData; /*回调函数参数*/

5 size_t rxDataSizeAll; /*接收数据量 \*/

6 size_t txDataSizeAll; /*发送数据量 \*/

7

8 edma_handle_t \*txEdmaHandle; /\* 串口发送使用的eDMA句柄 \*/

9 edma_handle_t \*rxEdmaHandle; /\* 串口接收使用的eDMA句柄 \*/

10

11 uint8_t nbytes; /*最初配置的eDMA次循环传输计数 \*/

12

13 volatile uint8_t txState; /*发送状态 \*/

14 volatile uint8_t rxState; /*接收状态*/

15 };

    \_lpuart_edma_handle的结构体成员介绍如下：

-  callback，指定DMA传输完成回调函数。这时一个lpuart_edma_transfer_callback_t类型的函数指针，函数原型如代码清单
   19‑17所示。

代码清单 19‑17lpuart_edma_transfer_callback_t函数指针(fsl_lpuart_edma.h)

1 typedef void (*lpuart_edma_transfer_callback_t)(LPUART_Type \*base,

2 lpuart_edma_handle_t \*handle,

3 status_t status,

4 void \*userData);

lpuart_edma_transfer_callback_t函数指针的入口参数介绍如下：

1. base，指定使用的那个LPUART，RT1052共有8个LPUART。

2. handle，这是一个串口DMA传输句柄指针，即_lpuart_edma_handl类型结构体指针。

3. status，用于保存状态和错误返回值。

4. userData，指定回调函数参数。

    在回调函数中我们可以通过这些函数参数得知所使用的串口、串口控制句柄等参数，进而可以在回调函数中对串口执行相应操作。回调函数是通过中断实现的，在中断服务函数中直接或间接调用了回调函数。

-  userData，指定回调函数参数。如果需要我么可以通过该参数向回调函数中传入其他信息，如果不需要设置位NULL即可。

-  rxDataSizeAll，保存串口接收的数据量，单位为字节。

-  txDataSizeAll，保存串口发送的数据量，单位为字节。

-  txEdmaHandle，指定串口发送DMA的DMA传输句柄。串口发送和接收拥有各自的DMA传输通道，所以要分别指定发送和接收的DMA传输句柄。

-  rxEdmaHandle，指定串口接收DMA的DMA传输句柄。

-  nbytes，eDMA次循环计数值。

-  txState，用于记录发送状态。

-  rxState，用于记录接收状态。

**DMA初始化**

DMA初始化的内容较多，首先要初始化DMAMUX，之后初始化DMA。DMA的初始化涉及之前讲过的DMA传输句柄和本小节将新引入的概念串口
DMA传输句柄。串口DMA初始化代码如代码清单 19‑18所示。

代码清单 19‑18DMA初始化代码(bsp_dma_uart.c)

1 /*****************************第一部分*****************************/

2 extern lpuart_edma_handle_t g_lpuartEdmaHandle;

3 extern edma_handle_t g_lpuartTxEdmaHandle;

4 extern edma_handle_t g_lpuartRxEdmaHandle;

5

6 void UART_DMA_Init(void)

7 {

8 edma_config_t config;//定义eDMA初始化结构体

9

10 /***************************第二部分****************************/

11 /*初始化DMAMUX \*/

12 DMAMUX_Init(EXAMPLE_LPUART_DMAMUX_BASEADDR);

13 /\* 为LPUART设置DMA传输通道*/

14 DMAMUX_SetSource(EXAMPLE_LPUART_DMAMUX_BASEADDR, \\

15 LPUART_TX_DMA_CHANNEL, LPUART_TX_DMA_REQUEST);

16 DMAMUX_SetSource(EXAMPLE_LPUART_DMAMUX_BASEADDR, \\

17 LPUART_RX_DMA_CHANNEL, LPUART_RX_DMA_REQUEST);

18DMAMUX_EnableChannel(EXAMPLE_LPUART_DMAMUX_BASEADDR,LPUART_TX_DMA_CHANNEL);

19DMAMUX_EnableChannel(EXAMPLE_LPUART_DMAMUX_BASEADDR,LPUART_RX_DMA_CHANNEL);

20

21 /*************************第三部分*****************************/

22 /\* 初始化DMA \*/

23 EDMA_GetDefaultConfig(&config);

24 EDMA_Init(EXAMPLE_LPUART_DMA_BASEADDR, &config);

25 /*创建eDMA传句柄*/

26 EDMA_CreateHandle(&g_lpuartTxEdmaHandle,
EXAMPLE_LPUART_DMA_BASEADDR,\\

27 LPUART_TX_DMA_CHANNEL);

28 EDMA_CreateHandle(&g_lpuartRxEdmaHandle,
EXAMPLE_LPUART_DMA_BASEADDR,\\

29 LPUART_RX_DMA_CHANNEL);

30

31 /***********************第四部分******************************/

32 /\* 初始化 LPUART DMA 句柄 \*/

33 LPUART_TransferCreateHandleEDMA(DEMO_LPUART,\\

34 &g_lpuartEdmaHandle,\\

35 LPUART_UserCallback,\\

36 NULL, \\

37 &g_lpuartTxEdmaHandle,

38 &g_lpuartRxEdmaHandle);

39 }

下面简要讲解各个部分代码，如下所示：

-  第一部分，定义DMA传输句柄和LPUART
   eDMA句柄。这些句柄定义在该章节对应的工程的main.c文件，使用extern
   扩展到该文件。有关DMA传输句柄在19.6.3
   DMA传输句柄edma_handle_t章节有过介绍，这里不再赘述。

-  第二部分，初始化DMAMUX， 有关DAMMUX的介绍请参考19.5
   DMAMUX简介及使用方法章节。函数DMAMUX_Init用于开启DMAMUX的时钟。函数DMAMUX_SetSource为DMA传输通道设置触发源。函数DMAMUX_EnableChannel使能DMA传输。

-  第三部分，初始化eDMA。eDMA的初始化与19.7
   DMA存储器到存储器模式实验初始化类似，差别是这里没有配置传输配置结构体，也没有提交传输请求。这部分工作不是不需要，而是放在了函数LPUART_TransferCreateHandleEDMA里面完成。如第四部分所示。

-  第四部分，初始化串口DMA传输句柄。函数LPUART_TransferCreateHandleEDMA用于实现串口DMA传输句柄的初始化，实际工作就是初始化串口DMA传输句柄配置项。函数原型如代码清单
   19‑19所示。

代码清单
19‑19LPUART_TransferCreateHandleEDMA串口句柄初始化函数(fsl_lpuart_edma.c)

1 void LPUART_TransferCreateHandleEDMA(LPUART_Type \*base,

2 lpuart_edma_handle_t \*handle,

3 lpuart_edma_transfer_callback_t callback,

4 void \*userData,

5 edma_handle_t \*txEdmaHandle,

6 edma_handle_t \*rxEdmaHandle)

7 {

8 assert(handle);

9 /*************************第一部分**************************/

10 uint32_t instance = LPUART_GetInstance(base);

11

12 s_edmaPrivateHandle[instance].base = base;

13 s_edmaPrivateHandle[instance].handle = handle;

14

15 /*************************第二部分*************************/

16 memset(handle, 0, sizeof(*handle));

17

18 handle->rxState = kLPUART_RxIdle;

19 handle->txState = kLPUART_TxIdle;

20

21 handle->rxEdmaHandle = rxEdmaHandle;

22 handle->txEdmaHandle = txEdmaHandle;

23

24 handle->callback = callback;

25 handle->userData = userData;

26

27 /*************************第三部分***************************/

28 #if defined(FSL_FEATURE_LPUART_HAS_FIFO) &&
FSL_FEATURE_LPUART_HAS_FIFO

29 /\* Note:

30 Take care of the RX FIFO, EDMA request only assert when received
bytes

31 equal or more than RX water mark, there is potential issue if RX
water

32 mark larger than 1.

33 For example, if RX FIFO water mark is 2, upper layer needs 5 bytes
and

34 5 bytes are received. the last byte will be saved in FIFO but not
trigger

35 EDMA transfer because the water mark is 2.*/

36 if (rxEdmaHandle)

37 {

38 base->WATER &= (~LPUART_WATER_RXWATER_MASK);

39 }

40 #endif

41

42 /******************************第四部分*********************/

43 /\* Configure TX. \*/

44 if (txEdmaHandle)

45 {

46 EDMA_SetCallback(handle->txEdmaHandle, LPUART_SendEDMACallback,\\

47 &s_edmaPrivateHandle[instance]);

48 }

49 /\* Configure RX. \*/

50 if (rxEdmaHandle)

51 {

52 EDMA_SetCallback(handle->rxEdmaHandle, LPUART_ReceiveEDMACallback,\\

53 &s_edmaPrivateHandle[instance]);

54 }

55 }

    各部分代码讲解如下：

1. 第一部分，指定使用的串口号和该串口号对应的串口DMA传输句柄。函数LPUART_GetInstance根据串口基址获取串口号。s_edmaPrivateHandle[]是lpuart_edma_private_handle_t类型的结构体数组，长度为8，依次保存8个串口的串口编号和串口DMA传输句柄。

2. 第二部分，设置串口DMA传输句柄的初始值。对比串口DMA传输句柄可以发现，第二部分设置了串口初始状态为接收和发送空闲、指定了接收和发送的DMA传输句柄、指定回调函数、指定回调函数参数。

3. 第三部分，如果使用了接收、发送FIFO，设置接收水印值。我们向发送FIFO内写入数据，串口模块会自动发送FIFO里面的内容，直到发送FIFO位空。接收FIFO比较特殊，只有接收FIFO内的数据大于或等于水印值才能触发DAM传输，例如接收FIFO水印值设置为8，我们通过串口一次发送10个字节，串口收到后触发一次DMA传输，将前8个字节发送出去，剩余的两个字节保存在FIFO中不能触发DMA传输。

4. 第四部分，设置串口接收和发送DMA传输句柄的回调函数

**回调函数**

当DMA传输完成之后会触发相应中断，在中断服务函数中直接或间接调用回调函数，在回调函数中我们可以通过检查传输标志位确定串口的当前状态。由于在中断服务函数中调用的回调函数，所以要像对待中断服务函数那样对待回调函数，比如不要该函数中进行耗时的操作、阻塞型操作等。

串口 DMA传输回调函数调用关系如图 19‑10所示：

|image9|

图 19‑10回调函数调用关系

从图
19‑10可以看出一个串口如果用DMA进行数据收发，会使用到三个回调函数，串口接收DMA传输完成回调函数和串口发送DMA传输完成回调函数（标号①处）分别在各自的DMA传输完成的中断服务函数中调用。这两个回调函数定义在fsl_lpuart_edma.c文件，由NXP官方编写，一般情况下我们无需修改。

串口DMA传输完成回调函数（标号②处）是自行定义的函数，并在初始化LPUART DMA
句柄是作为参数传递到LPUART DMA 句柄。函数原型如代码清单 19‑20所示。

代码清单 19‑20串口DMA传输完成回调函数(fsl_lpuart_edma.c)

1 /*****************************第一部分***********************/

2 extern volatile bool rxBufferEmpty;

3 extern volatile bool txBufferFull;

4 extern volatile bool txOnGoing;

5 extern volatile bool rxOnGoing;

6

7 /*****************************第二部分**********************/

8 /\* LPUART 回调函数 \*/

9 void LPUART_UserCallback(LPUART_Type \*base, \\

10 lpuart_edma_handle_t \*handle, status_t status, void \*userData)

11 {

12 userData = userData;

13

14 /*************************第三部分********************/

15 if (kStatus_LPUART_TxIdle == status)

16 {

17 txBufferFull = false;

18 txOnGoing = false;

19 }

20

21 if (kStatus_LPUART_RxIdle == status)

22 {

23 rxBufferEmpty = false;

24 rxOnGoing = false;

25 }

26 }

各部分代码简要讲解如下：

-  第一部分，定义发送、接收状态标志位和接收、发送缓冲器空或满标志位。这些变量定义在实验对应工程的main函数中，使用extern扩展到该文件。

-  第二部分，这部分代码最为第二部分单独列出主要是想强调定义回调函数时函数名与函数内容是自行定义的，但是函数参数要可函数指针lpuart_edma_transfer_callback_t一致。

-  第三部分，根据串口状态信息设置串口状态标志位，如果串口发送空闲，则设置发送缓冲区满标志txBufferFull为false，设置正在发送标志txOnGoing为false。如果串口接收空闲，则设置发送缓冲区空标志rxBufferEmpty为false，正在接收标志rxOnGoing为false。

**主函数**

代码清单 19‑21 存储器到外设模式主函数(main.c)

1 /******************************第一部分**************************/

2 /*设置TX和RX数据存储区*/

3 AT_NONCACHEABLE_SECTION_INIT(uint8_t
g_txBuffer[ECHO_BUFFER_LENGTH])={0};

4 AT_NONCACHEABLE_SECTION_INIT(uint8_t
g_rxBuffer[ECHO_BUFFER_LENGTH])={0};

5

6 /******************************第二部分************************/

7 /*定义UART传输状态标志*/

8 volatile bool rxBufferEmpty = true; //接收缓冲区空

9 volatile bool txBufferFull = false; //发送缓冲区满

10 volatile bool txOnGoing = false; //正在执行发送

11 volatile bool rxOnGoing = false; //正在执行接收

12

13 /******************************第三部分************************/

14 lpuart_edma_handle_t g_lpuartEdmaHandle; //串口DMA传输句柄

15 edma_handle_t g_lpuartTxEdmaHandle; //串口DMA发送句柄

16 edma_handle_t g_lpuartRxEdmaHandle; //串口DMA接收句柄

17

18 /*****************************第四部分*************************/

19 /*设置系统启动提示信息*/

20 AT_NONCACHEABLE_SECTION_INIT(uint8_t g_tipString[]) =

21 "LPUART EDMA example\r\nSend back received \\

22 data\r\nEcho every 8 characters\r\n";

23

24 /\* 主函数*/

25 int main(void)

26 {

27

28 /**************************第五部分*************************/

29 /*定义传输结构体*/

30 lpuart_transfer_t xfer; //定义提示信息传输结构体

31 lpuart_transfer_t sendXfer; //定义发送传输结构体

32 lpuart_transfer_t receiveXfer; //定义接收传输结构体

33

34
/****************此处省略系统初始化和系统时钟打印相关代码*************/

35

36

37

38 /**************************第六部分*************************/

39 UART_Init(); //初始化串口

40 UART_DMA_Init(); //初始化串口DMA传输使用的DMA

41

42 /**************************第七部分************************/

43 /\* 发送提示信息 \*/

44 xfer.data = g_tipString;

45 xfer.dataSize = sizeof(g_tipString) - 1;

46 txOnGoing = true;

47 LPUART_SendEDMA(DEMO_LPUART, &g_lpuartEdmaHandle, &xfer);

48 /\* 等待发送完成 \*/

49 while (txOnGoing)

50 {

51

52 }

53

54 /************************第八部分************************/

55 /\* 设置UART发送和接收传输结构体 \*/

56 sendXfer.data = g_txBuffer;

57 sendXfer.dataSize = ECHO_BUFFER_LENGTH;

58 receiveXfer.data = g_rxBuffer;

59 receiveXfer.dataSize = ECHO_BUFFER_LENGTH;

60

61 /************************第九部分*************************/

62 /*轮询检测串口当前状态，接收到数据后立即发送出去*/

63 while (1)

64 {

65 /\*
如果接收空闲并且接收缓冲区为空，表示当前串口空闲此时等待接收数据*/

66 if ((!rxOnGoing) && rxBufferEmpty)

67 {

68 rxOnGoing = true;

69 LPUART_ReceiveEDMA(DEMO_LPUART, &g_lpuartEdmaHandle, &receiveXfer);

70 }

71

72 /\* 如果发送空闲并且发送缓冲器满，此时应当开始发送数据*/

73 if ((!txOnGoing) && txBufferFull)

74 {

75 txOnGoing = true;

76 LPUART_SendEDMA(DEMO_LPUART, &g_lpuartEdmaHandle, &sendXfer);

77 }

78

79 /\*
如果发送缓冲区空并且接收缓冲区满，此时应将接收缓冲区的内容拷贝到发送缓冲区

80 \*/

81 if ((!rxBufferEmpty) && (!txBufferFull))

82 {

83 memcpy(g_txBuffer, g_rxBuffer, ECHO_BUFFER_LENGTH);

84 rxBufferEmpty = true;

85 txBufferFull = true;

86 }

87 }

88 }

在main.c文件夹下定义了大量的全局变量用于保存发送、接收信息以及串口当前状态信息等。在main函数中将会使用这些全局变量以实现LPUART
使用DMA方式进行数据传输。各部分代码讲解如下所示：

-  第一部分，定义串口接收、发送数据缓冲区，或者说数据存储区。数组g_txBuffer[]保存有将要发送的数据，数组g_rxBuffer[]保存有接收到的数据。初始化时将他们初始化为0。

-  第二部分，定义串口传输状态标志。包括接收、发送执行状态标志rxOnGoing、txOnGoing。以及接收、发送数据缓冲区状态表示rxBufferEmpty、txBufferFull。

-  第三部分，定义串口DMA传输句柄与DMA发送、接收句柄。

-  第四部分，定义系统启动时的提示信息，这是一个字符数组，用于保存系统启动后输出的提示信息。

-  第五部分，定义串口数据传输结构体，使用
   串口发送数据时我们要知道发送数据的地址和发送数据的长度，串口数据传输结构体的作用就是记录发送数据的起始地址和数据长度。结构体如代码清单
   19‑22所示。

代码清单 19‑22lpuart_transfer_t串口数据传输结构体(fsl_lpuart.h)

1 typedef struct \_lpuart_transfer

2 {

3 uint8_t \*data; /\* 将要发送数据的起始地址*/

4 size_t dataSize; /*将要发送的数据量，单位（字节） \*/

5 } lpuart_transfer_t;

-  第六部分，调用串口初始化函数和串口DMA初始化函数初始化串口和串口使用的DMA。

-  第七部分，初始化提示信息结构体并使用DMA发送提示信息。第七部分完整展现使用DMA发送一组数据的过程，如图
   19‑11所示。

|image10|

图 19‑11串口DMA发送流程

从图 19‑11可以看出使用串口DMA发送流程大致分为三步，如下所示：

1. 初始化传输结构体，该步骤实际工作就是将要发送的数据信息（数据起始地址和数据长度）保存到一个结构体内，方便使用。

2. 调用LPUART_SendEDMA函数启动发送，函数声明如代码清单 19‑23所示。

代码清单 19‑23LPUART_SendEDMA函数(fsl_lpuart_dema.h)

1 status_t LPUART_SendEDMA(LPUART_Type \*base,\\

2 lpuart_edma_handle_t \*handle,\\

3 lpuart_transfer_t \*xfer);

    LPUART_SendEDMA函数共有三个参数，base指定用于指定使用哪一个串口。handle用于指定串口DMA传输句柄，在第三部分代码定义了串口DMA传输句柄，并且在初始化DMA时已经完成了初始化。xfer用于指定传输结构体。

1. 在while(1)中等待传输完成，DMA传输完成后会执行回调函数，在回调函数中设置传输标志txOnGoing位false。

-  第八部分，初始化接收缓冲区和发送缓冲区的传输结构体。

-  第九部分，在while(1)死循环中不断检测串口状态，如果收到数据则将数据发送回去。执行流程如图
   19‑12所示。

|image11|

图 19‑12主循环执行流程

结合图
19‑12，主循环由三个if判断语句组成，根据当前UART传输状态标志执行不同的操作，结合源码和图
19‑12很容易理解，这里不再赘述。需要说明的有两点，如图
19‑12标号①和②处所示。

1. 由于使用了DMA，所以数据的接收与发送与CPU是异步进行的，CPU发送传输命令后DMA自动执行发送或接受，数据传输过程几乎不占用CPU，此时CPU继续向下执行。如果没有使用DMA则CPU会一直执行数据的发送直到中断发生或者发送完成。

2. DMA传输完成后（包括接收和发送）会触发中断，进而执行中断服务函数，在中断服务函数中调用的回调函数，所以当接收或发送完成后会更新串口当前状态。

   1. 下载验证

保证开发板相关硬件连接正确，用USB线连接开发板“USB TO
UART”接口跟电脑，在电脑端打开串口调试助手，把编译好的程序下载到开发板。程序运行后在串口调试助手输出“LPUART
EDMA example Send back received data, Echo every 8
characters.”提示信息。

单片机接收内容超过8个字节后会输出接收到的内容，最后不足8个字节的内容不输出。

.. |image0| image:: F:\文档\RT1052/media/image1.png
   :width: 5.76597in
   :height: 4.02569in
.. |image1| image:: F:\文档\RT1052/media/image2.png
   :width: 5.76597in
   :height: 5.77917in
.. |image2| image:: F:\文档\RT1052/media/image3.png
   :width: 5.49375in
   :height: 3.57153in
.. |image3| image:: F:\文档\RT1052/media/image4.png
   :width: 5.76597in
   :height: 4.46736in
.. |image4| image:: F:\文档\RT1052/media/image5.png
   :width: 5.61042in
   :height: 4.36389in
.. |image5| image:: F:\文档\RT1052/media/image6.png
   :width: 5.76597in
   :height: 4.44167in
.. |image6| image:: F:\文档\RT1052/media/image7.png
   :width: 5.76597in
   :height: 4.27292in
.. |image7| image:: F:\文档\RT1052/media/image8.png
   :width: 5.76597in
   :height: 1.57153in
.. |image8| image:: F:\文档\RT1052/media/image9.png
   :width: 5.76597in
   :height: 1.09097in
.. |image9| image:: F:\文档\RT1052/media/image10.png
   :width: 5.76597in
   :height: 4.71458in
.. |image10| image:: F:\文档\RT1052/media/image11.png
   :width: 4.90903in
   :height: 4.41528in
.. |image11| image:: F:\文档\RT1052/media/image12.png
   :width: 5.76597in
   :height: 5.27292in

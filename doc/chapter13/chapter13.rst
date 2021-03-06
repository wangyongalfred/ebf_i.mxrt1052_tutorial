RT1052中断应用概览
------------------

本章参考资料《cortex_m7_trm》(Cortex-M7技术参考手册)第七章Nested
Vectored Interrupt Controller

RT1052中断非常强大，每个外设都可以产生中断，所以中断的讲解放在哪一个外设里面去讲都不合适，这里单独抽出一章来做一个总结性的介绍，这样在其他章节涉及到中断部分的知识我们就不用费很大的篇幅去讲解，只要示意性带过即可。

本章如无特别说明，异常就是中断，中断就是异常，请不要刻意钻牛角尖较劲。

异常类型
~~~~~~~~

RT1052在内核水平上搭载了一个异常响应系统，
支持为数众多的系统异常和外部中断。其中系统异常有10个，外部中断有160个。除了个别异常的优先级被定死外，其它异常的优先级都是可编程的。有关具体的系统异常和外部中断可在SDK库文件MINXRT1052.h这个头文件查询到，在IRQn_Type这个枚举类型里面包含了RT1052全部的异常声明。

NVIC简介
~~~~~~~~

在讲如何配置中断优先级之前，我们需要先了解下NVIC。NVIC是嵌套向量中断控制器，控制着整个芯片中断相关的功能，它跟内核紧密耦合，是内核里面的一个外设。但是各个芯片厂商在设计芯片的时候会对Cortex-M7内核里面的NVIC进行裁剪，把不需要的部分去掉，所以说RT1052的NVIC是Cortex-M7的NVIC的一个子集。

NVIC中涉及的一些概念
^^^^^^^^^^^^^^^^^^^^

NVIC涉及很多的专业术语或者说是概念，比如中断请求，中断号、中断优先级，中断优先级分组（抢占优先级和子优先级）、中断服务函数。下面简单介绍这些概念：

-  中断请求，RT1052的大多数外设都能够发送中断请求，比如一个引脚由低电平变为高电平、串口接收到数据、定时器计时完成等都可以向CPU发送中断请求。

-  中断号，CPU通过中断号来区分不同的中断。每个中断请求拥有一个固定的标号，这个标号就是中断号，不同的中断请求可以拥有相同的中断号。比如RT1052拥有151个中断号，但是可以接收240个中断请求。

-  中断优先级，在RT1052中每个中断号拥有一个优先级，优先级的大小可以通过软件设定。优先级数值越小，优先级越高。

-  中断优先级分组，在NVIC中将优先级分位两部分，分别为抢占优先级和子优先级。抢占优先级顾名思义，具有抢占能力，如果一个中断正在执行，另外一个抢占优先级更高的中断发生了则打断底优先级的中断。如果两个中断请求的抢占优先级相同，子优先级不同则高优先级不能打断低优先级，但是如果两个中断请求同时发生并且抢占优先级相同，则根据子优先级决定谁先执行。

-  中断服务函数，每一个中断号对应一个函数，我们把这种对应关系叫做中断向量表，这个函数称为中断服务函数。当中断的到相应时CPU就会执行中断服务函数里面的内容。

NVIC的SDK库支持
^^^^^^^^^^^^^^^

NVIC为内核外设，涉及到许多Cortex-M7的内核知识，我们在这里不过多的讲解。为了方便使用，芯片厂商将这些涉及内核寄存器的操作封装为函数，我们只需要调用这些函数即可使用NVIC。有关NVIC的内容定义在SDK的core_cm7.h中

表格 13‑1符合CMSIS标准的NVIC库函数

+-----------------------------------+----------------------------------------------------------------+
|            NVIC库函数             |                              描述                              |
+===================================+================================================================+
| void                              | 设置中断优先级分组                                             |
| \__NVIC_SetPriorityGrouping(uint3 |                                                                |
| 2_t                               |                                                                |
| PriorityGroup)                    |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| uint32_t                          | 读取当前优先级分组                                             |
| \__NVIC_GetPriorityGrouping(void) |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| Void \_\_NVIC_EnableIRQ(IRQn_Type | 使能中断                                                       |
| IRQn)                             |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| uint32_t                          | 查看中断是否使能                                               |
| \__NVIC_GetEnableIRQ(IRQn_Type    |                                                                |
| IRQn)                             |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| void \__NVIC_DisableIRQ(IRQn_Type | 禁止中断                                                       |
| IRQn)                             |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| uint32_t                          | 查看中断是否被挂起                                             |
| \__NVIC_GetPendingIRQ(IRQn_Type   |                                                                |
| IRQn)                             |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| Void                              | 挂起中断                                                       |
| \__NVIC_SetPendingIRQ(IRQn_Type   |                                                                |
| IRQn)                             |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| void                              | 清除中断挂起标志位                                             |
| \__NVIC_ClearPendingIRQ(IRQn_Type |                                                                |
| IRQn)                             |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| uint32_t                          | 读取当前正在运行的中断                                         |
| \__NVIC_GetActive(IRQn_Type IRQn) |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| void                              | 设置中断优先级                                                 |
| \__NVIC_SetPriority(IRQn_Type     |                                                                |
| IRQn, uint32_t priority)          |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| uint32_t                          | 查看一个中断的优先级                                           |
| \__NVIC_GetPriority(IRQn_Type     |                                                                |
| IRQn)                             |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| uint32_t NVIC_EncodePriority      | 将中断优先级分组、抢占优先级、从优先级转化为无符号32优先级编码 |
| (uint32_t PriorityGroup, uint32_t |                                                                |
| PreemptPriority, uint32_t         |                                                                |
| SubPriority)                      |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| void NVIC_DecodePriority          | 将无符号32位优先级编码转换为优先级分组、抢占优先级、子优先级   |
| (uint32_t Priority, uint32_t      |                                                                |
| PriorityGroup, uint32_t\* const   |                                                                |
| pPreemptPriority, uint32_t\*      |                                                                |
| const pSubPriority)               |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| void \__NVIC_SetVector(IRQn_Type  | 设置中断向量（中断跳转地址）                                   |
| IRQn, uint32_t vector)            |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| uint32_t                          | 获得中断向量（中断跳转地址）                                   |
| \__NVIC_GetVector(IRQn_Type IRQn) |                                                                |
+-----------------------------------+----------------------------------------------------------------+
| void \__NVIC_SystemReset(void)    | 发送软件复位请求。                                             |
+-----------------------------------+----------------------------------------------------------------+

SDK为NVIC提供了这些库函数，涵盖了NVIC几乎所有功能，实际编程过程中常用的函数只有几个。而且这些函数名经过宏定义之后非常容易理解，我们能够通过函数名直到函数的功能，完全不需要记忆。

.. code-block:: c
   :name: 代码清单 13‑1中断函数宏定义（core_cm7.h）
   :caption: 代码清单 13‑1中断函数宏定义（core_cm7.h）
   :linenos:

   #define NVIC_SetPriorityGrouping    __NVIC_SetPriorityGrouping
   #define NVIC_GetPriorityGrouping    __NVIC_GetPriorityGrouping
   #define NVIC_EnableIRQ              __NVIC_EnableIRQ
   #define NVIC_GetEnableIRQ           __NVIC_GetEnableIRQ
   #define NVIC_DisableIRQ             __NVIC_DisableIRQ
   #define NVIC_GetPendingIRQ          __NVIC_GetPendingIRQ
   #define NVIC_SetPendingIRQ          __NVIC_SetPendingIRQ
   #define NVIC_ClearPendingIRQ        __NVIC_ClearPendingIRQ
   #define NVIC_GetActive              __NVIC_GetActive
   #define NVIC_SetPriority            __NVIC_SetPriority
   #define NVIC_GetPriority            __NVIC_GetPriority
   #define NVIC_SystemReset            __NVIC_SystemReset

定义这些宏时为了提高软件的兼容性，方便移植。

.. code-block:: c
   :name: 代码清单 13‑2中断号（MIMXRT1052.h）
   :caption: 代码清单 13‑2中断号（MIMXRT1052.h）
   :linenos:

   typedef enum IRQn {
   
      CTI0_ERROR_IRQn              = 17,
      LPUART6_IRQn                 = 25,
      LPUART7_IRQn                 = 26,
      LPUART8_IRQn                 = 27,
      LPI2C1_IRQn                  = 28,
      LPI2C2_IRQn                  = 29,
      LPI2C3_IRQn                  = 30,
      ..        ..          ..
      ..        ..          ..
      ..        ..          ..
   
      PWM4_3_IRQn                  = 150,
      PWM4_FAULT_IRQn              = 151,
      Reserved168_IRQn             = 152,
      Reserved169_IRQn             = 153,
      Reserved170_IRQn             = 154,
      Reserved171_IRQn             = 155,
      Reserved172_IRQn             = 156,
      Reserved173_IRQn             = 157,
      SJC_ARM_DEBUG_IRQn           = 158,
      NMI_WAKEUP_IRQn              = 159
   } IRQn_Type

这个枚举类型列出了所有RT1052支持的中断号，这里只列出了一小部分，详细请查看MIMXRT1052.h文件。

有了SDK库函数和中断号我们就可以操作中断了，在使用到中断时我们将会具体讲解使用方法

优先级的定义
~~~~~~~~~~~~

优先级定义
^^^^^^^^^^

在NVIC
有一个专门的寄存器：应用程序中断和复位控制寄存器AIRCR（详细请参考《armv7m_arm》参考手册第B3.2.6章节）。AIRCR[PRIGROUP]用来配置外部中断的优先级分组，宽度为3bit，原则上可以配置为0到7，RT1052对中断控制进行了删减，在RT1052硬件平台只能设置为4到7。

表格 13‑2RT1052使用4bit表达优先级

+----------------+-----------------+------+------+------+------+------+------+
|      bit7      |      bit6       | bit5 | bit4 | bit3 | bit2 | bit1 | bit0 |
+================+=================+======+======+======+======+======+======+
| 用于表达优先级 | 未使用，读回为0 |      |      |      |      |      |      |
+----------------+-----------------+------+------+------+------+------+------+

用于表达优先级的这4bit，又被分组成抢占优先级和子优先级。如果有多个中断同时响应，抢占优先级高的就会
抢占
抢占优先级低的优先得到执行，如果抢占优先级相同，就比较子优先级。如果抢占优先级和子优先级都相同的话，就比较他们的硬件中断编号，编号越小，优先级越高。

优先级分组
^^^^^^^^^^

优先级的分组由内核外设SCB的应用程序中断及复位控制寄存器AIRCR的PRIGROUP[10:8]位决定，RT1052分为了4组，具体如下：主优先级=抢占优先级

+---------+--------------+------------+------------+----------+----------+
| PRIGRO  | 中断优先级值 |    级数    |            |          |          |
| UP[2:0] |  PRI_N[7:4]  |            |            |          |          |
|         |              |            |            |          |          |
+=========+==============+============+============+==========+==========+
|         | 二进制点     | 主优先级位 | 子优先级位 | 主优先级 | 子优先级 |
|         |              |            |            |          |          |
+---------+--------------+------------+------------+----------+----------+
| 0b 100  | 0b xxx.y     | [7:5]      | [4]        | 8        | 2        |
+---------+--------------+------------+------------+----------+----------+
| 0b 101  | 0b xx.yy     | [7:6]      | [5:4]      | 4        | 4        |
+---------+--------------+------------+------------+----------+----------+
| 0b 110  | 0b x.yyy     | [7]        | [6:4]      | 2        | 8        |
+---------+--------------+------------+------------+----------+----------+
| 0b 111  | 0b .yyyy     | None       | [7:4]      | None     | 16       |
+---------+--------------+------------+------------+----------+----------+

设置优先级分组可调用库函数NVIC_SetPriorityGrouping实现，有关NVIC中断相关的库函数都在库文件core_cm7.h。

表格 13‑3 优先级分组真值表

+------------+----------+----------+------------------+
| 优先级分组 | 主优先级 | 子优先级 | 描述             |
+============+==========+==========+==================+
| 4          | 0-7      | 0-1      | 主-3bit，子-1bit |
+------------+----------+----------+------------------+
| 5          | 0-3      | 0-3      | 主-2bit，子-2bit |
+------------+----------+----------+------------------+
| 6          | 0-1      | 0-7      | 主-1bit，子-3bit |
+------------+----------+----------+------------------+
| 7          | -        | 0-15     | 主-0bit，子-4bit |
+------------+----------+----------+------------------+

中断编程
~~~~~~~~

在配置每个中断的时候一般有4个编程要点：

1. 使用NVIC_SetPriorityGrouping(uint32_t
   PriorityGroup)函数配置中断优先级分组。PriorityGroup只能取4到7。

2. 使用uint32_t NVIC_EncodePriority (uint32_t PriorityGroup, uint32_t
   PreemptPriority, uint32_t
   SubPriority)函数配置具体外设中断通道的抢占优先级和子优先级。这个函数不会真正的实现设置，只是将中断优先级分组、抢占优先级、子优先级转化为32位中断优先级编码，供其他函数使用

3. 使用NVIC_SetPriority(IRQn_Type IRQn, uint32_t
   priority)函数设置中断编号的优先级，priority的值是函数NVIC_EncodePriority的返回值。

4. 使用NVIC_EnableIRQ(IRQn_Type IRQn)函数使能中断请求。

5. 编写中断服务函数

在启动文件startup_MIMXRT1052.s中我们预先为每个中断都写了一个中断服务函数，只是这些中断函数都是为空，为的只是初始化中断向量表。实际的中断服务函数都需要我们重新编写。

关于中断服务函数的函数名必须跟启动文件里面预先设置的一样，如果写错，系统就在中断向量表中找不到中断服务函数的入口，直接跳转到启动文件里面预先写好的空函数，并且在里面无限循环，实现不了中断。

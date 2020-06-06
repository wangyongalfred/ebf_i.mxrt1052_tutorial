GPIO输入—按键中断检测
---------------------

按键检测使用到GPIO外设的基本输入功能，本章中不再赘述GPIO外设的概念，如您忘记了，可重读前面“7.2
GPIO框图剖析”小节，RT1052标准库中GPIO初始化结构体gpio_pin_config_t的定义与“8.3.5
定义引脚模式的枚举类型”小节中讲解的相同。

GPIO中断简介
~~~~~~~~~~~~

RT1052拥有5组GPIO，每组GPIO拥有32个GPIO输入输出引脚，每个输入输出引脚都能够触发中断。RT1052并没有为每个输入输出引脚分配一个中断号，而是为每组GPIO分配两个中断编号，其中低16
个输入输出引脚（GPIOx_n，x取1到5，n取0到15）共用一个中断编号，高16个输入输出引脚使用另外一个中断编号。

每组GPIO拥有各自的中断相关寄存器，包括一个中断屏蔽寄存器（GPIOx_IMR），一个中断状态寄存（GPIOx_ISR），两个中断配置寄存器（GPIOx_ICR1、GPIOx_ICR2）。通过配置这些寄存器我们可以灵活的设置每一个输入输出引脚是否使用中断、中断触发条件、以及当前中断状态。下面依次讲解各个寄存，理解了这几个寄存器就基本上掌握了GPIO中断，比较简单。

-  中断屏蔽寄存器（GPIOx_IMR），中断屏蔽寄存用于配置输入输出引脚是否使用中断，该寄存器是32位寄存，0到31位依次控制输入输出引脚0到31。系统复位后
   GPIOx_IMR = 0，表示禁止所有中断，如果要使用中断必须设置相应位为1。

-  中断状态寄存器（GPIOx_ISR），该寄存器是32位寄存器，0到31位依次标记0到31个输入输出引脚的中断状态。如果发生了中断则相应的中断标志位被硬件自动置1，软件写入1清除中断标志位。

-  中断配置寄存器（GPIOx_ICR），每组GPIO拥有两个中断配置寄存器（32位），低16个引脚使用GPIOx_ICR1，高16个引脚使用GPIOx_ICR2。每个输入/输出引脚占用两位，可配置为低电平触发中断、高电平触发中断、上升沿触发中断、下降沿触发中断。我们根据外部电路和实际需要配置为不同的触发模式。

硬件设计
~~~~~~~~

按键中断检测使用的硬件与第12章
GPIO输入—按键查询检测使用的硬件相同，详细请参考第12章 硬件设计章节。

软件设计
~~~~~~~~

同LED的工程，为了使工程更加有条理，我们把按键相关的代码独立分开存储，方便以后移植。在“工程模板”之上新建“bsp_key_it.c”及“bsp_key_it.h”文件，这些文件也可根据您的喜好命名，这些文件不属于RT1052标准库的内容，是由我们自己根据应用需要编写的。

编程要点
^^^^^^^^

-  定义按键的相关引脚；

-  配置引脚的MUX复用模式及PAD属性配置；

-  初始化GPIO目标引脚为输入模式，设置中断触发模式并开启中断。

-  设置中断优先级、使能中断。

-  编写简单测试程序，检测按键的状态，实现按键控制LED灯。

代码分析
^^^^^^^^

按键引脚宏定义
''''''''''''''

同样，在编写按键驱动时，也要考虑更改硬件环境的情况。我们把按键检测引脚相关的宏定义到
“bsp_key_it.h”文件中，具体见代码清单 11‑1。

.. code-block:: c
   :name: 代码清单 14‑1 按键检测引脚相关的宏（bsp_key_it.h文件）
   :caption: 代码清单 14‑1 按键检测引脚相关的宏（bsp_key_it.h文件）
   :linenos:

   //WAUP按键
   #define CORE_BOARD_WAUP_KEY_GPIO          GPIO5
   #define CORE_BOARD_WAUP_KEY_GPIO_PIN      (0U)
   #define CORE_BOARD_WAUP_KEY_IOMUXC        IOMUXC_SNVS_WAKEUP_GPIO5_IO00
   #define CORE_BOARD_WAUP_KEY_NAME          "CORE_BORE_WAUP_KEY"
   #define CORE_BOARD_WAUP_KEY_ID            0
   //中断相关，IRQ中断号及IRQHandler中断服务函数
   #define CORE_BOARD_WAUP_KEY_IRQ           GPIO5_Combined_0_15_IRQn
   #define CORE_BOARD_WAUP_KEY_IRQHandler    GPIO5_Combined_0_15_IRQHandler
   
   //MODE按键
   #define CORE_BOARD_MODE_KEY_GPIO          GPIO1
   #define CORE_BOARD_MODE_KEY_GPIO_PIN      (5U)
   #define CORE_BOARD_MODE_KEY_IOMUXC        IOMUXC_GPIO_AD_B0_05_GPIO1_IO05
   #define CORE_BOARD_MODE_KEY_NAME          "CORE_BORE_MODE_KEY"
   #define CORE_BOARD_MODE_KEY_ID            1
   //中断相关，IRQ中断号及IRQHandler中断服务函数
   #define CORE_BOARD_MODE_KEY_IRQ           GPIO1_Combined_0_15_IRQn
   #define CORE_BOARD_MODE_KEY_IRQHandler    GPIO1_Combined_0_15_IRQHandler

以上代码根据按键的硬件连接，把检测按键输入的GPIO端口、引脚号以及IOMUXC复用配置封装起来了。其中CORE_BOARD_xxx_NAME宏定义的是字符串，用来表示按键的名字，方便调试时通过串口输出。

初始化引脚复用功能
''''''''''''''''''

RT1052的官方SDK库已经为每个引脚定义了所有可用的复用功能，不需要用户配置。我们只需要选择配置选项然后调用初始化函数即可。

.. code-block:: c
   :name: 代码清单 14‑2初始化复用功能代码（bsp_key_it.c）
   :caption: 代码清单 14‑2初始化复用功能代码（bsp_key_it.c）
   :linenos:

   /*********************************第一部分*************************/
   #define CORE_BOARD_WAUP_KEY_IOMUXC        IOMUXC_SNVS_WAKEUP_GPIO5_IO00
   #define IOMUXC_SNVS_WAKEUP_GPIO5_IO00 0x400A8000U, 0x5U, 0, 0, 0x400A8018U
   
   /*********************************第二部分*************************/
   static void Key_IOMUXC_MUX_Config(void)
   {
      /* 设置按键引脚的复用模式为GPIO，不使用SION功能 */
      IOMUXC_SetPinMux(CORE_BOARD_WAUP_KEY_IOMUXC, 0U);
      IOMUXC_SetPinMux(CORE_BOARD_MODE_KEY_IOMUXC, 0U); 
   }

第一部分，定义了一些宏定义，这些宏定义并不在bsp_key_it.c文件中，放在这里只是为了方便理解程序。第一个宏定义是我们自定义的，只是为IOMUXC_SNVS_WAKEUP_GPIO5_IO00起了个别名，方便代码的移植。第二个宏定义是SDK官方定义的引脚复用功能配置。在配置引脚复用功能时根据需要选择即可。

第二部分，根据选择的复用功能使用IOMUXC_SetPinMux（）函数初始化引脚。

初始化引脚PAD属性
'''''''''''''''''

.. code-block:: c
   :name: 代码清单 14‑3中断检引脚PAD属性设置（bsp_key_it.c）
   :caption: 代码清单 14‑3中断检引脚PAD属性设置（bsp_key_it.c）
   :linenos:

   static void Key_IOMUXC_PAD_Config(void)
   {
      /* 设置按键引脚属性功能 */    
      IOMUXC_SetPinConfig(CORE_BOARD_WAUP_KEY_IOMUXC, KEY_PAD_CONFIG_DATA); 
      IOMUXC_SetPinConfig(CORE_BOARD_MODE_KEY_IOMUXC, KEY_PAD_CONFIG_DATA); 
   }

引脚PAD属性的设置依然是通过宏定义和配置函数来实现的，IOMUXC_SetPinConfig()是PAD属性设置函数，CORE_BOARD_WAUP_KEY_IOMUXC是引脚复用功能宏定义，KEY_PAD_CONFIG_DAT是引脚PAD配置宏定义如代码清单
14‑4。

.. code-block:: c
   :name: 代码清单 14‑4PAD参数宏定义(bsp_key_it.h)
   :caption: 代码清单 14‑4PAD参数宏定义(bsp_key_it.h)
   :linenos:

   #define KEY_PAD_CONFIG_DATA            (SRE_0_SLOW_SLEW_RATE| \
                                          DSE_0_OUTPUT_DRIVER_DISABLED| \
                                          SPEED_2_MEDIUM_100MHz| \
                                          ODE_0_OPEN_DRAIN_DISABLED| \
                                          PKE_1_PULL_KEEPER_ENABLED| \
                                          PUE_1_PULL_SELECTED| \
                                          PUS_3_22K_OHM_PULL_UP| \
                                          HYS_1_HYSTERESIS_ENABLED)   
      /* 配置说明 : */
      /* 转换速率: 转换速率慢
         驱动强度: 关闭
         速度配置 : medium(100MHz)
         开漏配置: 关闭 
         拉/保持器配置: 使能
         拉/保持器选择: 上下拉
         上拉/下拉选择: 22K欧姆上拉
         滞回器配置: 开启 （仅输入时有效，施密特触发器，使能后可以过滤输入噪声）*/

在每个使用到外部引脚的程序中一般会定义一到多个类似于这样的宏定义，不同外设对引脚PAD属性要求不同，根据需要修改这些配置参数即可。

初始化GPIO模式
''''''''''''''

.. code-block:: c
   :name: 代码清单 14‑5GPIO工作模式设置(bsp_key_it.c)
   :caption: 代码清单 14‑5GPIO工作模式设置(bsp_key_it.c)
   :linenos:

   static void Key_GPIO_Mode_Config(void)
   {     
      /* 配置为输入模式，低电平中断，后面通过GPIO_PinInit函数加载配置 */
      gpio_pin_config_t key_config;
      
      /** 核心板的按键，GPIO配置 **/       
      key_config.direction = kGPIO_DigitalInput;    //输入模式
      key_config.outputLogic =  1;                  //默认高电平（输入模式时无效）
      key_config.interruptMode = kGPIO_IntLowLevel; //低电平触发中断
      
      /* 初始化 KEY GPIO. */
      GPIO_PinInit(CORE_BOARD_WAUP_KEY_GPIO,\
                  CORE_BOARD_WAUP_KEY_GPIO_PIN, &key_config);
      GPIO_PinInit(CORE_BOARD_MODE_KEY_GPIO,\
                  CORE_BOARD_MODE_KEY_GPIO_PIN, &key_config);
   }

GPIO模式包括包括GPIO的方向（输入或输出），GPIO默认电平（只有设置位输出时设置才有效）以及是否开启中断。这些配置是通过gpio_pin_config_t结构体定义的。在该实验中按键按下后引脚为低电平，所以设置设置为低电平触发中断。

在初始化函数GPIO_PinInit（）中CORE_BOARD_WAUP_KEY_GPIO和CORE_BOARD_WAUP_KEY_GPIO_PIN分别用于设置GPIO组和GPIO引脚号。key_config是GPIO模式配置结构体，我们设置的配置参数保存在这里。

GPIO中断配置
''''''''''''

.. code-block:: c
   :name: 代码清单 14‑6GPIO中断初始化(bsp_key_it.c)
   :caption: 代码清单 14‑6GPIO中断初始化(bsp_key_it.c)
   :linenos:

   static void Key_Interrupt_Config(void)   
   {
      /***************************第一部分***************************/
      /* 开启GPIO引脚的中断 */
      GPIO_PortEnableInterrupts(CORE_BOARD_WAUP_KEY_GPIO,\
                                 1U << CORE_BOARD_WAUP_KEY_GPIO_PIN);                    
      GPIO_PortEnableInterrupts(CORE_BOARD_MODE_KEY_GPIO,\
                                 1U << CORE_BOARD_MODE_KEY_GPIO_PIN); 
      
      /**************************第二部分***************************/
      /*设置中断优先级,*/
      set_IRQn_Priority(CORE_BOARD_WAUP_KEY_IRQ,\
                           Group4_PreemptPriority_6, Group4_SubPriority_0);
      set_IRQn_Priority(CORE_BOARD_MODE_KEY_IRQ,\
                           Group4_PreemptPriority_6, Group4_SubPriority_1);
      
      /*************************第三部分****************************/
      /* 使能中断 */
      EnableIRQ(CORE_BOARD_WAUP_KEY_IRQ);
      EnableIRQ(CORE_BOARD_MODE_KEY_IRQ);
   }

-  第一部分，开启GPIO引脚的中断。在14.1
   GPIO中断简介这一章节介绍了GPIOx_IMR寄存器用于设置是否开启GPIO中断，GPIO_PortEnableInterrupts函数实际就是在该寄存器对应的使能位置1，函数原型如代码清单
   14‑7。

.. code-block:: c
   :name: 代码清单 14‑7GPIO_PortEnableInterrupts函数原型(fsl_gpio.h)
   :caption: 代码清单 14‑7GPIO_PortEnableInterrupts函数原型(fsl_gpio.h)
   :linenos:

   static inline void GPIO_PortEnableInterrupts(GPIO_Type *base,uint32_t mask)
   {
      base->IMR |= mask;
   }

-  第二部分，设置中断优先级。第13章
   RT1052中断应用概览我们介绍了两个自定义的有关优先级设定的函数，分别用于设置优先级分组和设置优先级。在主函数中已经设定了中断优先级分组，在这里只需要设定中断优先级即可。

-  第三部分，使能中断。在第一部分代码开启了GPIO引脚中断功能，GPIO已经可以发送中断请求。这部分代码用于使能中断，只有使能了中断，中断请求才能够被CPU接收到。

检测按键的状态
''''''''''''''

初始化按键后，我们只需要在中断服务函数中更新按键状态就可以了。

.. code-block:: c
   :name: 代码清单 14‑8 检测按键的状态(bsp_key.c文件)
   :caption: 代码清单 14‑8 检测按键的状态(bsp_key.c文件)
   :linenos:

   /********************中断服务函数**************************/
   /**
   * @brief  GPIO 输入中断服务函数
   *         CORE_BOARD_WAUP_KEY_IRQHandler只是一个宏，
   *         在本例中它指代函数名GPIO5_Combined_0_15_IRQHandler，
   *         中断服务函数名是固定的，可以在启动文件中找到。
   * @param  中断服务函数不能有输入参数
   * @note   中断函数一般只使用标志位进行指示，完成后尽快退出，
   *         具体操作或延时尽量不放在中断服务函数中
   * @retval 中断服务函数不能有返回值
   */
   void CORE_BOARD_WAUP_KEY_IRQHandler(void)
   {
      /* 清除中断标志位 */
      GPIO_PortClearInterruptFlags(CORE_BOARD_WAUP_KEY_GPIO,
                                    1U << CORE_BOARD_WAUP_KEY_GPIO_PIN);

      /* 设置按键中断标志 */
      g_KeyDown[CORE_BOARD_WAUP_KEY_ID] = true;

      /* 以下部分是为 ARM 的勘误838869添加的,
         该错误影响 Cortex-M4, Cortex-M4F内核，
         立即存储覆盖重叠异常，导致返回操作可能会指向错误的中断
         CM7不受影响，此处保留该代码
      */

      /* 原注释：Add for ARM errata 838869, affects Cortex-M4,
         Cortex-M4F Store immediate overlapping
         exception return operation might vector to incorrect interrupt */
   #if defined __CORTEX_M && (__CORTEX_M == 4U)
      __DSB();
   #endif
   }

   /**
   * @brief  GPIO 输入中断服务函数
   *         CORE_BOARD_MODE_KEY_IRQHandler只是一个宏，
   *         在本例中它指代函数名GPIO1_Combined_0_15_IRQHandler，
   *         中断服务函数名是固定的，可以在启动文件中找到。
   * @param  中断服务函数不能有输入参数
   * @note   中断函数一般只使用标志位进行指示，完成后尽快退出，
   *         具体操作或延时尽量不放在中断服务函数中
   * @retval 中断服务函数不能有返回值
   */
   void CORE_BOARD_MODE_KEY_IRQHandler(void)
   {
      /* 清除中断标志位 */
      GPIO_PortClearInterruptFlags(CORE_BOARD_MODE_KEY_GPIO,
                                    1U << CORE_BOARD_MODE_KEY_GPIO_PIN);

      /* 设置按键中断标志 */
      g_KeyDown[CORE_BOARD_MODE_KEY_ID] = true;

      /* 以下部分是为 ARM 的勘误838869添加的,
         该错误影响 Cortex-M4, Cortex-M4F内核，
         立即存储覆盖重叠异常，导致返回操作可能会指向错误的中断
         CM7不受影响，此处保留该代码
      */

      /* 原注释：Add for ARM errata 838869, affects Cortex-M4,
         Cortex-M4F Store immediate overlapping
         exception return operation might vector to incorrect interrupt */
   #if defined __CORTEX_M && (__CORTEX_M == 4U)
      __DSB();
   #endif
   }

当中断发生时，对应的中断服务函数就会被执行，我们可以在中断服务函数实现一些控制。

在这里我们定义了两个中断服务函数，因为我们使用到了两中断编号，在中断服务函数中我们只是清除中断标志位并且更新按键状态。

主函数
''''''

接下来我们使用主函数编写按键检测流程，见代码清单 12‑4。

.. code-block:: c
   :name: 代码清单 14‑9 按键检测主函数（main.c文件）
   :caption: 代码清单 14‑9 按键检测主函数（main.c文件）
   :linenos:

   int main(void)
   {
      /* 初始化内存管理单元 */
      BOARD_ConfigMPU();
      /* 初始化开发板引脚 */
      BOARD_InitPins();
      /* 初始化开发板时钟 */
      BOARD_BootClockRUN();
      /* 初始化调试串口 */
      BOARD_InitDebugConsole();
      /*设置中断优先级分组*/
      Set_NVIC_PriorityGroup(Group_4); 
      
      /* 打印系统时钟 */
      PRINTF("\r\n");
      PRINTF("*****欢迎使用 野火i.MX RT1052 开发板*****\r\n");
      PRINTF("CPU:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_CpuClk));
      PRINTF("AHB:             %d Hz\r\n", CLOCK_GetFreq(kCLOCK_AhbClk));
      PRINTF("SEMC:            %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SemcClk));
      PRINTF("SYSPLL:          %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllClk));
      PRINTF("SYSPLLPFD0:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd0Clk));
      PRINTF("SYSPLLPFD1:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd1Clk));
      PRINTF("SYSPLLPFD2:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd2Clk));
      PRINTF("SYSPLLPFD3:      %d Hz\r\n", CLOCK_GetFreq(kCLOCK_SysPllPfd3Clk));  
      
      PRINTF("GPIO输入—按键中断实验\r\n");
      
      /* 初始化LED引脚 */
      LED_GPIO_Config();
      
      /* 初始化KEY引脚 */
      Key_IT_GPIO_Config();
      
      /* 等待按键中断来临，当按键按下就执行按键中断服务函数，
         按键中断服务函数在bsp_key_it.c实现。
         中断服务函数会对全局变量g_KeyDown设置为true。
      */ 
   
      while(1)
      {   
         /* WAUP按键的标志 */
         /* 若g_KeyDown为true表明按键被按下 */
         if(g_KeyDown[CORE_BOARD_WAUP_KEY_ID])
         {
            /* 稍微延时 */
            delay(100);
            /* 等待至按键被释放 （高电平）*/
            if(1 == GPIO_PinRead(CORE_BOARD_WAUP_KEY_GPIO,\
                              CORE_BOARD_WAUP_KEY_GPIO_PIN))
            {
                  /* 翻转LED灯，串口输出信息 */
                  CORE_BOARD_LED_TOGGLE;
                  PRINTF("检测到 %s 按键操作\r\n", CORE_BOARD_WAUP_KEY_NAME);
            }
            /* 重新设置标志位 */
            g_KeyDown[CORE_BOARD_WAUP_KEY_ID] = false; 
         }
         
         /* MODE按键的标志 */
         /* 若g_KeyDown为true表明按键被按下 */
         if(g_KeyDown[CORE_BOARD_MODE_KEY_ID])
         {
            /* 稍微延时 */
            delay(100);
            /* 等待至按键被释放 （高电平）*/
            if(1 == GPIO_PinRead(CORE_BOARD_MODE_KEY_GPIO,\
                              CORE_BOARD_MODE_KEY_GPIO_PIN))
            {
               /* 翻转LED灯，串口输出信息 */
               CORE_BOARD_LED_TOGGLE;
               PRINTF("检测到 %s 按键操作\r\n", CORE_BOARD_MODE_KEY_NAME);
            }
            /* 重新设置标志位 */
            g_KeyDown[CORE_BOARD_MODE_KEY_ID] = false; 
         }
      }     
   }

代码中初始化LED灯及按键后，在while函数里不断判断按键状态，当检测到有按键按下，等待按键被释放，之后翻转RGB灯的状态，通过串口输出按下的按键名字，最后清除按键按下标志位。

下载验证
~~~~~~~~

把编译好的程序下载到开发板并复位，按下按键可以控制LED灯亮、灭状态。打开串口调试助手，当按键松开后可以看到串口调试助手打印的按键信息。

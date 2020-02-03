.. vim: syntax=rst

中断管理
----

本章所述中断管理主要是针对中断处理程序的管理。

异常与中断的基本概念
~~~~~~~~~~

异常是导致处理器脱离正常运行转向执行特殊代码的任何事件，如果不及时进行处理，轻则系统出错，重则会导致系统毁灭性瘫痪。所以正确地处理异常，避免错误的发生是提高软件鲁棒性（稳定性）非常重要的一环，对于实时系统更是如此。

异常是指任何打断处理器正常执行，并且迫使处理器进入一个由有特权的特殊指令执行的事件。异常通常可以分成两类：同步异常和异步异常。由内部事件（像处理器指令运行产生的事件）引起的异常称为同步异常，例如：被零除的算术运算引发一个异常；在某些处理器体系结构中，对于确定的数据尺寸必须从内存的偶数地址进行读和写操
作，而从奇数内存地址的读或写操作将引起存储器存取一个错误事件并引起一个异常（称为校准异常）。

异步异常主要是指由于外部异常源产生的异常，是一个由外部硬件装置产生的事件引起的异步异常。同步异常不同于异步异常的地方是事件的来源，同步异常事件是由于执行某些指令而从处理器内部产生的，而异步异常事件的来源是外部硬件。如：按下设备某个按钮产生的事件。同步异常与异步异常的区别还在于，同步异常触发后，系统必
须立刻进行处理而不能够依然执行原有的程序指令步骤；而异步异常则可以延缓处理甚至是忽略，如：按键中断，虽然中断异常触发了，但是系统可以忽略它继续运行（同样也忽略了相应的按键事件）。

中断，中断属于异步异常，所谓中断是指CPU正在处理某件事情的时候，外部发生了某一事件，请求CPU迅速处理，CPU暂停当前的工作，转而去处理所发生的事件，处理完后再回到原先被打断的地方，继续原来的工作，这样的过程称为中断。

中断能打断任务的运行，无论该任务具有什么样的优先级，因此中断一般用于处理比较紧急的事件，而且只做简单处理，如标记该事件，在使用LiteOS系统时，一般建议使用信号量或事件标志中断的发生，并通知对应的处理任务，在任务中处理。

通过中断机制，在外设不需要CPU介入时，CPU可以执行其他任务，而当外设需要CPU时通过产生中断信号使CPU立即停止当前处理，并响应中断请求。这样可以避免CPU把大量时间耗费在等待、查询外设状态的操作上，因此将大大提高系统实时性以及执行效率。

此处读者要知道一点，LiteOS源码中有多处临界段的地方， 临界段虽然保护了关键代码的执行不被打断， 但也会影响系统的实时。比如：某个时刻有一个任务在运行中将中断屏蔽，即进入到了临界区中，这个时候如果有一个紧急的中断事件被触发，该中断就无法及时得到响应，必须在中断开启才可以得到响应，
如果屏蔽中断的时间超过了紧急中断能够容忍的限度， 危害是可想而知的。所以，操作系统的中断在某些时候会有适当的延迟响应中断，即使是调用中断屏蔽函数进入临界区的时候，也需快进快出。

LiteOS接管中断版本的中断支持：

1. 中断初始化。

2. 中断创建。

3. 中断删除。

4. 中断使能。

5. 中断屏蔽。

6. 中断共享。

LiteOS非接管接管中断版本的中断支持：

1. 中断使能。

2. 中断屏蔽。

中断的介绍
^^^^^

与中断相关的硬件可以划分为三类：外设、中断控制器、CPU本身。

外设：当外设需要请求CPU时，产生一个中断信号，该信号连接至中断控制器。

中断控制器：中断控制器是CPU众多外设中的一个，它一方面接收其他外设中断信号的输入，另一方面，它会发出中断信号给CPU。可以通过对中断控制器编程实现对中断源的优先级、触发方式、打开和关闭中断源等操作。在Cortex-M系列控制器中常用的中断控制器是内嵌向量中断控制器（Nested Vectored
Interrupt Controller，NVIC）。

CPU：CPU会响应中断源的请求，中断当前正在执行的任务，转而执行中断处理程序。NVIC最多支持240个中断，每个中断最多256个优先级。

和中断相关的名词解释
^^^^^^^^^^

中断号：每个中断请求信号都会有特定的标志，使得计算机能够判断是哪个设备提出的中断请求，这个标志就是中断号。

中断请求：“紧急事件”需向CPU提出申请，要求CPU暂停当前执行的任务，转而处理该“紧急事件”，这一申请过程称为中断请求。

中断优先级：为使系统能够及时响应并处理所有中断，系统根据中断时间的重要性和紧迫程度，将中断源分为若干个级别，称作中断优先级。

中断处理程序：当外设产生中断请求后，CPU暂停当前的任务，转而响应中断申请，即执行中断处理程序。

中断触发：中断源发出并送给CPU控制信号，将中断触发器置“1”，表明该中断源产生了中断，要求CPU去响应该中断，CPU暂停当前任务，执行相应的中断处理程序。

中断触发类型：外部中断申请通过一个物理信号发送到NVIC，可以是电平触发或边沿触发。

中断向量：中断服务程序的入口地址。

中断向量表：存储中断向量的存储区，中断向量与中断号对应，中断向量在中断向量表中按照中断号顺序存储。

临界段：代码的临界段也称为临界区，一旦这部分代码开始执行，则不允许任何中断打断。为确保临界段代码的执行不被中断，在进入临界段之前须关中断，而临界段代码执行完毕后，要立即开中断。

中断的应用场景
~~~~~~~

举个例子：假如读者正在给朋友写信，电话铃响了，这时读者放下手中的笔去接电话，通话完毕再继续写信。这个例子就表现了中断及其处理的过程：电话铃声使读者暂时中止当前的写信，而去处理更为急需处理的事情——接电话，当把急需处理的事情处理完毕之后，再继续写信。在这个例子中，电话铃声就可以称为“中断请求”；读者暂
停写信去接电话就叫作“中断响应”；接电话的过程就是“中断处理”。由此可以看出，在计算机执行程序的过程中，由于出现某个特殊情况（或称为“事件”），使得系统暂时中止当前运行的程序，而转去执行处理这一特殊事件的程序，处理完毕之后再回到原来程序的中断点继续运行，而这个过程就被称为中断。

本书再举一个例子来说明中断的作用：假设有一个朋友来拜访读者，但是由于读者不知朋友何时到达，读者只能在门口等待，也就无法做其他事情；但如果在门口装一个门铃，读者就不必在门口等待，可以在家里去做其他的工作，当朋友到来后按门铃通知，读者这时才停止手中的工作去开门，这就避免了不必要的等待。同理CPU也是如此
，在中断未到来时，CPU可以去处理其他事情，当中断到来时CPU再去响应中断并完成处理，这样子CPU的处理将更加高效。

中断的运作机制
~~~~~~~

当中断产生时，处理机将按如下的顺序执行。

1. 保存当前处理机状态信息。

2. 载入异常或中断处理函数到PC寄存器。

3. 把控制权转交给处理函数并开始执行。

4. 当处理函数执行完成时，恢复处理器状态信息。

5. 从异常或中断中返回到前一个程序执行点。

中断使得CPU可以在事件发生时才给予处理，而不必让CPU时刻查询是否有相应的事件发生。通过两条特殊指令：关中断和开中断可以让处理器不响应或响应中断，在关闭中断期间，通常处理器会把新产生的中断挂起，当中断打开时立刻进行响应，所以会有适当的延时响应中断，故用户在进入临界区的时候应快进快出。

中断发生的环境有两种情况：在任务的上下文中，在中断服务函数处理上下文中。

1. 任务在工作的时候，如果此时发生了一个中断，无论任务的优先级是多高，都会打断当前任务的执行，从而转到对应的中断服务函数中执行，其过程如图11‑1所示。

图11‑1\ **(1)、(3)**\ ：在任务运行的时候发生了中断，那么中断会打断任务的运行，操作系统将先保存当前任务的上下文环境，转而去处理中断服务函数。

图11‑1\ **(2)、(4)**\ ：当且仅当中断服务函数处理完的时候才恢复任务的上下文环境，继续运行任务。

|interr002|

图11‑1中断发生在任务上下文

2. 在执行中断服务例程的过程中，如果有更高优先级的中断源触发中断，由于当前处于中断处理上下文环境中，根据不同的处理器构架可能有不同的处理方式，如：新的中断等待挂起直到当前中断处理离开后再行响应；或新的高优先级中断打断当前中断处理过程，而去直接响应这个更高优先级的新中断源，后者可以称之为中断嵌套。Lite
   OS允许中断嵌套，即在一个中断服务函数期间，处理器可以响应另外一个优先级更高的中断，过程如图11‑2所示。

图11‑2\ **(1)**\ ：当中断1的服务函数在处理的时候发生了中断2，由于中断2的优先级比中断1更高，所以发生了中断嵌套，那么操作系统将先保存当前中断服务函数的上下文环境，并且转向处理中断2，当且仅当中断2执行完的时候图11‑2\ **(2)**\ ，才能继续执行中断1。

|interr003|

图11‑2中断嵌套发生

中断延迟的基本概念
~~~~~~~~~

即使操作系统的响应很快了，但对于中断的处理仍然存在着中断延迟响应的问题，称之为中断延迟（ Interrupt Latency ） 。

中断延迟是指从硬件中断发生到开始执行中断处理程序第一条指令之间的这段时间。也就是：系统接收到中断信号到操作系统作出响应，并完成换到转入中断服务程序的时间。也可以简单地理解为：（外部）硬件发生中断，到系统执行中断服务子程序（ISR）的第一条指令的时间。

中断的处理过程是：外界硬件发生了中断后，CPU到中断处理器读取中断向量，并且查找中断向量表，找到对应的中断服务子程序（ISR）的首地址，然后跳转到对应的ISR去做相应处理。这部分时间，本书称之为：识别中断时间。

在允许中断嵌套的实时操作系统中，中断也是基于优先级的，允许高优先级中断抢断正在处理的低优先级中断，所以，如果当前正在处理更高优先级的中断，即使此时有低优先级的中断，也系统不会立刻响应，而是等到高优先级的中断处理完之后，才会响应。而在不支持中断嵌套的情况下（如相同的子优先级中断），即中断是不允许抢占的
，如果当前系统正在处理一个中断，而此时另一个中断到来了，系统也是不会立即响应的，而只是等处理完当前的中断之后，才会处理后来的中断。这部分时间，本书称之为：等待中断打开时间。

在操作系统中，很多时候会主动进入临界段，系统不允许当前状态被中断打断，故而在临界区发生的中断会被挂起，直到退出临界段时候打开中断。这部分时间，本书称之为：关闭中断时间。

中断延迟可以定义为，从中断开始的时刻到中断服务例程开始执行的时刻之间的时间段。中断延迟 = 识别中断时间 + [等待中断打开时间] + [关闭中断时间]。

注意：“[ ]”的时间是不一定都存在的，此处为最大可能的中断延迟时间。

中断的使用讲解
~~~~~~~

接管中断版本的移植
^^^^^^^^^

按照第2章 的内容进行移植，移植的版本为接管中断版本。

接管中断版本的常用函数讲解
^^^^^^^^^^^^^

创建硬件中断函数LOS_HwiCreate()
'''''''''''''''''''''''

既然LiteOS接管了中断，那么关于中断的注册创建那也是由LiteOS管理，系统要知道当前创建了什么中断，如果没有创建中断就使用了中断的话，那么往往会发生致命的错误。所以LiteOS提供了创建硬件中断的函数LOS_HwiCreate()，其源码如代码清单 11‑1所示。

代码清单 11‑1创建硬件中断函数LOS_HwiCreate()源码

1 LITE_OS_SEC_TEXT_INIT UINT32 LOS_HwiCreate(HWI_HANDLE_T uwHwiNum, **(1)**

2 HWI_PRIOR_T usHwiPrio, **(2)**

3 HWI_MODE_T usMode, **(3)**

4 HWI_PROC_FUNC pfnHandler, **(4)**

5 HWI_ARG_T uwArg ) **(5)**

6 {

7 UINTPTR uvIntSave;

8

9 if (NULL == pfnHandler) { **(6)**

10 return OS_ERRNO_HWI_PROC_FUNC_NULL;

11 }

12

13 if (uwHwiNum >= OS_HWI_MAX_NUM) { **(7)**

14 return OS_ERRNO_HWI_NUM_INVALID;

15 }

16

17 if (m_pstHwiForm[uwHwiNum + OS_SYS_VECTOR_CNT] !=

18 (HWI_PROC_FUNC)osHwiDefaultHandler) { **(8)**

19 return OS_ERRNO_HWI_ALREADY_CREATED;

20 }

21

22 if ((usHwiPrio > OS_HWI_PRIO_LOWEST) \|\|

23 (usHwiPrio < OS_HWI_PRIO_HIGHEST)) { **(9)**

24 return OS_ERRNO_HWI_PRIO_INVALID;

25 }

26

27 uvIntSave = LOS_IntLock();

28 #if (OS_HWI_WITH_ARG == YES)

29 osSetVector(uwHwiNum, pfnHandler, uwArg);

30 #else

31 osSetVector(uwHwiNum, pfnHandler); **(10)**

32 #endif

33 NVIC_EnableIRQ((IRQn_Type)uwHwiNum); **(11)**

34 NVIC_SetPriority((IRQn_Type)uwHwiNum, usHwiPrio); **(12)**

35

36 LOS_IntRestore(uvIntSave);

37

38 return LOS_OK;

39

40 }

代码清单 11‑1\ **(1)**\ ：uwHwiNum是硬件的中断向量号，可以在stm32fxxx.h找得到，比如霸道开发板的可以在stm32f10x.h中找到相应的中断向量号，如代码清单 11‑2所示。

代码清单 11‑2 stm32f10x.h中断向量号（部分）

1 /*\*

2 \* @brief STM32F10x中断号定义，根据所选平台选择

3 \*

4 \*/

5 typedef enum IRQn {

6 /\* Cortex-M3处理器异常号 \/

7 NonMaskableInt_IRQn = -14,

8 MemoryManagement_IRQn = -12,

9 BusFault_IRQn = -11,

10 UsageFault_IRQn = -10,

11 SVCall_IRQn = -5,

12 DebugMonitor_IRQn = -4,

13 PendSV_IRQn = -2,

14 SysTick_IRQn = -1,

15

16 /\* STM32特定的中断号 \/

17 WWDG_IRQn = 0,

18 PVD_IRQn = 1,

19 TAMPER_IRQn = 2,

20 RTC_IRQn = 3,

21 FLASH_IRQn = 4,

22 RCC_IRQn = 5,

23 EXTI0_IRQn = 6,

24 EXTI1_IRQn = 7,

25 EXTI2_IRQn = 8,

26 EXTI3_IRQn = 9,

27 EXTI4_IRQn = 10,

28 DMA1_Channel1_IRQn = 11,

29 DMA1_Channel2_IRQn = 12,

30 DMA1_Channel3_IRQn = 13,

31 DMA1_Channel4_IRQn = 14,

32 DMA1_Channel5_IRQn = 15,

33 DMA1_Channel6_IRQn = 16,

34 DMA1_Channel7_IRQn = 17,

代码清单 11‑1\ **(2)**\ ：usHwiPrio是硬件中断优先级。

代码清单 11‑1\ **(3)**\ ：usMode是硬件中断模式。

代码清单 11‑1\ **(4)**\ ：pfnHandler是触发硬件中断时使用的中断处理程序。即中断服务函数，需要用户自己编写并且声明，在创建注册硬件中断的时候将函数指针传入。

代码清单 11‑1\ **(5)**\ ：uwArg中断服务函数的输入参数。

代码清单 11‑1\ **(6)**\ ：判断用户是否实现中断服务函数，如果中断服务函数指针为NULL，则返回错误代码。

代码清单 11‑1\ **(7)**\ ：如果中断向量号大于OS_HWI_MAX_NUM（Cortex-m3， Cortex-m4，Cortex-m7内核的最大中断向量号默认为 240），则返回错误代码。

代码清单 11‑1\ **(8)**\ ：根据向量号判断当前的中断是否已经注册，如果是则无需重复注册，返回错误代码。

代码清单 11‑1\ **(9)**\ ：判断中断的优先级是否有效，默认范围是OS_HWI_PRIO_HIGHEST（0）~ OS_HWI_PRIO_LOWEST（7），数值越低，优先级越大。

代码清单 11‑1\ **(10)**\ ：根据中断向量号与中断服务函数用来设置中断向量表，形成映射关系，该宏定义如代码清单 11‑3所示。

代码清单 11‑3 osSetVector宏定义

1 #define osSetVector(uwNum, pfnVector) \\

2 m_pstHwiForm[uwNum + OS_SYS_VECTOR_CNT] = osInterrupt;\\

3 m_pstHwiSlaveForm[uwNum + OS_SYS_VECTOR_CNT] = pfnVector;

4 #endif

代码清单 11‑1\ **(11)**\ ：根据中断向量号使能中断，通过设置NVIC寄存器使能对应的中断。

代码清单 11‑1\ **(12)**\ ：设置中断的优先级，根据传递进来的中断向量号与优先级配置对应的优先级。

创建硬件中断的函数使用实例如代码清单 11‑4所示。

代码清单 11‑4创建硬件中断函数LOS_HwiCreate()实例

1 uvIntSave = LOS_IntLock(); /\* 屏蔽所有中断 \*/

2

3 /\* 创建硬件中断，用于配置硬件中断并注册硬件中断处理功能 \*/

4 LOS_HwiCreate(KEY1_INT_EXTI_IRQ,

5 /\* 平台的中断向量号，可以在stm32fxxx.h找得到，本例程由bsp_exti.h重新定义了 \*/

6 0, /\* 硬件中断优先级 暂时忽略此参数 \*/

7 0, /\* 硬件中断模式 暂时忽略此参数 \*/

8 KEY1_IRQHandler, /\* 中断服务函数 \*/

9 0); /\* 触发硬件中断时使用的中断处理程序的输入参数 \*/

10

11 /\* 创建硬件中断，用于配置硬件中断并注册硬件中断处理功能 \*/

12 /\* 平台的中断向量号，可以在stm32fxxx.h找得到，本例程由bsp_exti.h重新定义了 \*/

13 LOS_HwiCreate(KEY2_INT_EXTI_IRQ,

14 0, /\* 硬件中断优先级 暂时忽略此参数 \*/

15 0, /\* 硬件中断模式 暂时忽略此参数 \*/

16 KEY2_IRQHandler, /\* 中断服务函数 \*/

17 0); /\* 触发硬件中断时使用的中断处理程序的输入参数 \*/

18

19 LOS_IntRestore(uvIntSave); /\* 恢复所有中断 \*/

20 /\*

21 \* @ 函数名 ： KEY1_IRQHandler

22 \* @ 功能说明： 中断服务函数

23 \* @ 参数 ： 无

24 \* @ 返回值 ： 无

25 \/

26 static void KEY1_IRQHandler(void)

27 {

28 //确保是否产生了EXTI Line中断

29 if (EXTI_GetITStatus(KEY1_INT_EXTI_LINE) != RESET) {

30 Trigger_Num = 1; /\* 标记一下触发的中断,中断中尽可能快进快出 \*/

31 // LED1 取反

32 LED1_TOGGLE;

33 //清除中断标志位

34 EXTI_ClearITPendingBit(KEY1_INT_EXTI_LINE);

35 }

36 }

37 /\*

38 \* @ 函数名 ： KEY1_IRQHandler

39 \* @ 功能说明： 中断服务函数

40 \* @ 参数 ： 无

41 \* @ 返回值 ： 无

42 \/

43 static void KEY2_IRQHandler(void)

44 {

45 //确保是否产生了EXTI Line中断

46 if (EXTI_GetITStatus(KEY2_INT_EXTI_LINE) != RESET) {

47 Trigger_Num = 2; /\* 标记一下触发的中断，中断中尽可能快进快出 \*/

48 // LED2 取反

49 LED2_TOGGLE;

50 //清除中断标志位

51 EXTI_ClearITPendingBit(KEY2_INT_EXTI_LINE);

52 }

53 }

删除硬件中断函数LOS_HwiDelete()
'''''''''''''''''''''''

LiteOS支持删除已注册的硬件中断，当某些中断不再需要使用的时候，可以将其删除，当删除了中断的时候就无法再次使用，系统将不再响应该中断，删除硬件中断函数LOS_HwiDelete()的源码如代码清单 11‑5所示。

代码清单 11‑5删除硬件中断函数LOS_HwiDelete()源码

1 LITE_OS_SEC_TEXT_INIT UINT32 LOS_HwiDelete(HWI_HANDLE_T uwHwiNum)

2 {

3 UINT32 uwIntSave;

4

5 if (uwHwiNum >= OS_HWI_MAX_NUM) { **(1)**

6 return OS_ERRNO_HWI_NUM_INVALID;

7 }

8

9 NVIC_DisableIRQ((IRQn_Type)uwHwiNum); **(2)**

10

11 uwIntSave = LOS_IntLock();

12

13 m_pstHwiForm[uwHwiNum + OS_SYS_VECTOR_CNT] = **(3)**

14 (HWI_PROC_FUNC)osHwiDefaultHandler;

15 LOS_IntRestore(uwIntSave);

16

17 return LOS_OK;

18 }

代码清单 11‑5\ **(1)**\ ：判断中断向量号是否大于OS_HWI_MAX_NUM，若是则返回错误代码。

代码清单 11‑5\ **(2)**\ ：根据中断向量号失能对应中断。

代码清单 11‑5\ **(3)**\ ：解除已经创建的中断向量号与中断服务函数的映射关系。

如果使用LiteOS接管中断，需要使能LOSCFG_PLATFORM_HWI宏定义，并配置系统支持的最大中断数：LOSCFG_PLATFORM_HWI_LIMIT，此外还需要注意以下几点。

1. 创建中断并不等于已经初始化中断了，真正的中断初始化部分还是由用户编写，所以在注册之前应先将中断初始完成。

2. 根据具体硬件平台，配置支持的最大中断数及中断初始化操作的寄存器地址。在 Cortex-m3， Cortex-m4，Cortex-m7中基本无需修改，LiteOS已经处理好，直接使用即可。

3. 中断处理程序耗时不能过长，否则影响CPU对其他中断的及时响应。

4. 关中断后不能执行引起调度的函数。

非接管中断
^^^^^

Cortex-M 系列内核的中断是由硬件管理的，而LiteOS是软件，它可以不接管系统相关中断（接管中断是指：系统中所有的中断都由RTOS的软件管理，硬件产生中断时，由软件决定是否响应，可以挂起中断，延迟响应或者不响应）。而非接管中断方式的使用其实跟裸机是差不多的，需要用户自己配置中断，并且使能中断
，编写中断服务函数，在中断服务函数中使用内核IPC通信机制，一般建议使用信号量或事件做标记，等退出中断后再由相关任务处理。

NVIC支持中断嵌套功能：当一个中断触发并且系统进行响应时，处理器硬件会将当前运行的部分上下文寄存器自动压入中断栈中，这部分的寄存器包括PSR，R0，R1，R2，R3以及R12寄存器。当系统正在服务一个中断时，如果有一个更高优先级的中断触发，那么处理器同样的会打断当前运行的中断服务例程，然后把老的中
断服务例程上下文的PSR，R0，R1，R2，R3和R12寄存器自动保存到中断栈中。这些部分上下文寄存器保存到中断栈的行为完全是硬件行为，这一点是与其他ARM处理器最大的区别（以往都需要依赖于软件保存上下文）。

另外，在ARM Cortex-M系列处理器上，所有中断都采用中断向量表的方式进行处理，即当一个中断触发时，处理器将直接判定是哪个中断源，然后直接跳转到相应的固定位置进行处理。而在ARM7、ARM9中，一般是先跳转进入IRQ入口，然后再由软件进行判断是哪个中断源触发，获得了相对应的中断服务例程入口地址
后，再进行后续的中断处理。ARM7、ARM9的好处在于，所有中断它们都有统一的入口地址，便于OS的统一管理。而ARM Cortex-
M系列处理器则恰恰相反，每个中断服务例程必须排列在一起放在统一的地址上（这个地址必须要设置到NVIC的中断向量偏移寄存器中）。中断向量表一般由一个数组定义（或在起始代码中指定），在STM32上，默认采用起始代码指定，如代码清单11‑6所示。

代码清单11‑6中断向量表（部分）

1 \__Vectors DCD \__initial_sp ; Top of Stack

2 DCD Reset_Handler ; Reset Handler

3 DCD NMI_Handler ; NMI Handler

4 DCD HardFault_Handler ; Hard Fault Handler

5 DCD MemManage_Handler ; MPU Fault Handler

6 DCD BusFault_Handler ; Bus Fault Handler

7 DCD UsageFault_Handler ; Usage Fault Handler

8 DCD 0 ; Reserved

9 DCD 0 ; Reserved

10 DCD 0 ; Reserved

11 DCD 0 ; Reserved

12 DCD SVC_Handler ; SVCall Handler

13 DCD DebugMon_Handler ; Debug Monitor Handler

14 DCD 0 ; Reserved

15 DCD PendSV_Handler ; PendSV Handler

16 DCD SysTick_Handler ; SysTick Handler

17

18 ; External Interrupts

19 DCD WWDG_IRQHandler ; Window Watchdog

20 DCD PVD_IRQHandler ; PVD through EXTI Line detect

21 DCD TAMPER_IRQHandler ; Tamper

22 DCD RTC_IRQHandler ; RTC

23 DCD FLASH_IRQHandler ; Flash

24 DCD RCC_IRQHandler ; RCC

25 DCD EXTI0_IRQHandler ; EXTI Line 0

26 DCD EXTI1_IRQHandler ; EXTI Line 1

27 DCD EXTI2_IRQHandler ; EXTI Line 2

28 DCD EXTI3_IRQHandler ; EXTI Line 3

29 DCD EXTI4_IRQHandler ; EXTI Line 4

30 DCD DMA1_Channel1_IRQHandler ; DMA1 Channel 1

31 DCD DMA1_Channel2_IRQHandler ; DMA1 Channel 2

32 DCD DMA1_Channel3_IRQHandler ; DMA1 Channel 3

33 DCD DMA1_Channel4_IRQHandler ; DMA1 Channel 4

34 DCD DMA1_Channel5_IRQHandler ; DMA1 Channel 5

35 DCD DMA1_Channel6_IRQHandler ; DMA1 Channel 6

36 DCD DMA1_Channel7_IRQHandler ; DMA1 Channel 7

37

37 ………

39

LiteOS在Cortex-M系列处理器上也遵循与裸机中断一致的方法，当用户需要使用自定义的中断服务函数时，只需要定义相同名称的函数覆盖弱化符号即可。

中断实验
~~~~

接管中断方式
^^^^^^

中断管理实验（接管中断方式）是在LiteOS中创建了两个被LiteOS管理的中断，并编写相关的中断服务函数，在触发的时候将信号量传递给任务，任务获取到信号量将相关信息从串口输出，如代码清单 11‑7加粗部分所示。

代码清单 11‑7 LiteOS中断管理实验(接管中断方式)

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief 这是一个[野火]-STM32F103霸道LiteOS中断管理实验！

8 \\*

9 \* @attention

10 \*

11 \* 实验平台:野火 STM32 F103 开发板

12 \* 论坛 :http://www.firebbs.cn

13 \* 淘宝 :https://fire-stm32.taobao.com

14 \*

15 \\*

16 \*/

17

18 /\* LiteOS 头文件 \*/

19 #include "los_sys.h"

20 #include "los_typedef.h"

21 #include "los_task.ph"

22 #include "los_sem.h"

23 /\* 板级外设头文件 \*/

24 #include "stm32f10x.h"

25 #include "bsp_usart.h"

26 #include "bsp_led.h"

27 #include "bsp_key.h"

28 #include "bsp_exti.h"

29

30 /\* 任务ID \/

31 /\*

32 \* 任务ID是一个从0开始的数字，用于索引任务，当任务创建完成之后，它就具有了一个任务ID

33 \* 以后要想操作这个任务都需要通过这个任务ID，

34 \*

35 \*/

36 /\* 定义任务ID变量 \*/

37 UINT32 Test_Task_Handle;

38

39 /\* 定义二值信号量的ID变量 \*/

40 UINT32 BinarySem1_Handle;

41 UINT32 BinarySem2_Handle;

42 /\* 全局变量声明 \/

43 /\*

44 \* 当在写应用程序的时候，可能需要用到一些全局变量。

45 \*/

46

47

48 /\* 函数声明 \*/

49 static void KEY1_IRQHandler(void);

50 static void KEY2_IRQHandler(void);

51

52 static UINT32 Creat_Test_Task(void);

53 static void Test_Task(void);

54

55 static void BSP_Init(void);

56 static void AppTaskCreate(void);

57

58 /*\*

59 \* @brief 主函数

60 \* @param 无

61 \* @retval 无

62 \* @note 第一步：开发板硬件初始化

63 第二步：创建App应用任务

64 第三步：启动LiteOS，开始多任务调度，启动不成功则输出错误信息

65 \*/

66 int main(void)

67 {

68 UINT32 uwRet = LOS_OK;/\* 定义一个创建任务的返回类型，初始化为创建成功的返回值 \*/

69

70 /\* 板级初始化，所有的跟开发板硬件相关的初始化都可以放在这个函数里面 \*/

71 BSP_Init();

72 /\* 发送一个字符串 \*/

73 printf("这是一个[野火]-STM32F103霸道LiteOS中断管理实验！\n");

74

75 /\* LiteOS 核心初始化 \*/

76 uwRet = LOS_KernelInit();

77 if (uwRet != LOS_OK) {

78 printf("LiteOS 核心初始化失败！\n");

79 return LOS_NOK;

80 }

81 /\* 创建App应用任务，所有的应用任务都可以放在这个函数里面 \*/

82 AppTaskCreate();

83

84 /\* 开启LiteOS任务调度 \*/

85 LOS_Start();

86 }

87

88 /\*

89 \* @ 函数名 ： AppTaskCreate

90 \* @ 功能说明： 任务创建，为了方便管理，所有的任务创建函数都可以放在这个函数里面

91 \* @ 参数 ： 无

92 \* @ 返回值 ： 无

93 \/

94 static void AppTaskCreate(void)

95 {

96 UINTPTR uvIntSave;

97 UINT32 uwRet = LOS_OK;

98 /\* 创建一个二值信号量*/

99 uwRet = LOS_BinarySemCreate(0,&BinarySem1_Handle);

100 if (uwRet != LOS_OK) {

101 printf("BinarySem_Handle二值信号量创建失败！\n");

102 }

103 uwRet = LOS_BinarySemCreate(0,&BinarySem2_Handle);

104 if (uwRet != LOS_OK) {

105 printf("BinarySem_Handle二值信号量创建失败！\n");

106 }

107 uwRet = Creat_Test_Task();

108 if (uwRet != LOS_OK) {

109 printf("Test_Task任务创建失败！\n");

110 }

111

**112 uvIntSave = LOS_IntLock();/\* 屏蔽所有中断 \*/**

**113**

**114 /\* 创建硬件中断，用于配置硬件中断并注册硬件中断处理功能 \*/**

**115 LOS_HwiCreate( KEY1_INT_EXTI_IRQ,**

**116 /\* 平台的中断向量号，可以在stm32fxxx.h找得到，本例程由bsp_exti.h重新定义了 \*/**

**117 0, /\* 硬件中断优先级 暂时忽略此参数 \*/**

**118 0, /\* 硬件中断模式 暂时忽略此参数 \*/**

**119 KEY1_IRQHandler, /\* 中断服务函数 \*/**

**120 0); /\* 触发硬件中断时使用的中断处理程序的输入参数 \*/**

**121**

**122 /\* 创建硬件中断，用于配置硬件中断并注册硬件中断处理功能 \*/**

**123 LOS_HwiCreate( KEY2_INT_EXTI_IRQ,**

**124 /\* 平台的中断向量号，可以在stm32fxxx.h找得到，本例程由bsp_exti.h重新定义了 \*/**

**125 0, /\* 硬件中断优先级 暂时忽略此参数 \*/**

**126 0, /\* 硬件中断模式 暂时忽略此参数 \*/**

**127 KEY2_IRQHandler, /\* 中断服务函数 \*/**

**128 0); /\* 触发硬件中断时使用的中断处理程序的输入参数 \*/**

**129**

**130 LOS_IntRestore(uvIntSave); /\* 恢复所有中断 \*/**

131

132 }

133 /\*

134 \* @ 函数名 ： Creat_Test_Task

135 \* @ 功能说明： 创建Test_Task任务

136 \* @ 参数 ： 无

137 \* @ 返回值 ： 无

138 \/

139 static UINT32 Creat_Test_Task()

140 {

141 UINT32 uwRet = LOS_OK; /\* 定义一个创建任务的返回类型，初始化为创建成功的返回值 \*/

142 TSK_INIT_PARAM_S task_init_param;

143

144 task_init_param.usTaskPrio = 5;/\* 优先级，数值越小，优先级越高 \*/

145 task_init_param.pcName = "Test_Task";/\* 任务名，字符串形式，方便调试 \*/

146 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)Test_Task;

147 task_init_param.uwStackSize = 0x1000;/\* 栈大小，单位为字，即4个字节 \*/

148

149 uwRet = LOS_TaskCreate(&Test_Task_Handle, &task_init_param);

150

151 return uwRet;

152 }

153

154 /\*

155 \* @ 函数名 ： Test_Task

156 \* @ 功能说明： 在串口打印触发中断的信息

157 \* @ 参数 ： 无

158 \* @ 返回值 ： 无

159 \/

**160 static void Test_Task(void)**

**161 {**

**162 UINT32 uwRet = LOS_OK;**

**163 while (1) {**

**164 //获取二值信号量,没获取到则不等待**

**165 uwRet = LOS_SemPend( BinarySem1_Handle , 0 );**

**166 if (uwRet == LOS_OK) {**

**167 printf("触发中断的是Key1!\n\n");**

**168 } //获取二值信号量,没获取到则不等待**

**169 uwRet = LOS_SemPend( BinarySem2_Handle , 0 );**

**170 if (uwRet == LOS_OK) {**

**171 printf("触发中断的是Key2!\n\n");**

**172 }**

**173 LOS_TaskDelay(20);**

**174 }**

**175 }**

176 /\*

177 \* @ 函数名 ： BSP_Init

178 \* @ 功能说明： 板级初始化，所有的跟开发板硬件相关的初始化都可以放在这个函数里面

179 \* @ 参数 ： 无

180 \* @ 返回值 ： 无

181 \/

182 static void BSP_Init(void)

183 {

184 /\*

185 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

186 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

187 \* 都统一用这个优先级分组，千万不要再分组，切忌。

188 \*/

189 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

190

191 /\* LED 初始化 \*/

192 LED_GPIO_Config();

193

194 /\* 串口初始化 \*/

195 USART_Config();

196

**197 /\* 按键EXTI初始化 \*/**

**198 EXTI_Key_Config();**

199 }

200 /\*

201 \* @ 函数名 ： KEY1_IRQHandler

202 \* @ 功能说明： 中断服务函数

203 \* @ 参数 ： 无

204 \* @ 返回值 ： 无

205 \/

**206 static void KEY1_IRQHandler(void)**

**207 {**

**208 //确保是否产生了EXTI Line中断**

**209 if (EXTI_GetITStatus(KEY1_INT_EXTI_LINE) != RESET) {**

**210 LOS_SemPost( BinarySem1_Handle ); //释放二值信号量 BinarySem_Handle**

**211 //清除中断标志位**

**212 EXTI_ClearITPendingBit(KEY1_INT_EXTI_LINE);**

**213 }**

**214 }**

215 /\*

216 \* @ 函数名 ： KEY1_IRQHandler

217 \* @ 功能说明： 中断服务函数

218 \* @ 参数 ： 无

219 \* @ 返回值 ： 无

220 \/

**221 static void KEY2_IRQHandler(void)**

**222 {**

**223 //确保是否产生了EXTI Line中断**

**224 if (EXTI_GetITStatus(KEY2_INT_EXTI_LINE) != RESET) {**

**225 LOS_SemPost( BinarySem2_Handle ); //释放二值信号量 BinarySem_Handle**

**226 //清除中断标志位**

**227 EXTI_ClearITPendingBit(KEY2_INT_EXTI_LINE);**

**228 }**

**229 }**

230 /END OF FILE/

非接管中断方式
^^^^^^^

中断管理实验是在LiteOS中创建了两个任务分别获取信号量与消息队列，并且定义了两个按键KEY1与KEY2的触发方式为中断触发，在中断触发的时候通过消息队列将消息传递给任务，任务接收到消息就将信息通过串口调试助手显示出来。而且中断管理实验也实现了一个串口的DMA传输+空闲中断功能，当串口接收完不定长
的数据之后产生一个空闲中断，在中断中将信号量传递给任务，任务在收到信号量的时候将串口的数据读取出来并且在串口调试助手中回显，如代码清单 11‑8加粗部分所示。

代码清单 11‑8 LiteOS中断管理实验(非接管中断方式)

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief 这是一个[野火]-STM32F103霸道LiteOS中断管理实验！

8 \\*

9 \* @attention

10 \*

11 \* 实验平台:野火 STM32 F103 开发板

12 \* 论坛 :http://www.firebbs.cn

13 \* 淘宝 :https://fire-stm32.taobao.com

14 \*

15 \\*

16 \*/

17

18 /\* LiteOS 头文件 \*/

19 #include "los_sys.h"

20 #include "los_typedef.h"

21 #include "los_task.ph"

22 #include "los_sem.h"

23 /\* 板级外设头文件 \*/

24 #include "stm32f10x.h"

25 #include "bsp_usart.h"

26 #include "bsp_led.h"

27 #include "bsp_key.h"

28 #include "bsp_exti.h"

29

30 /\* 任务ID \/

31 /\*

32 \* 任务ID是一个从0开始的数字，用于索引任务，当任务创建完成之后，它就具有了一个任务ID

33 \* 以后要想操作这个任务都需要通过这个任务ID，

34 \*

35 \*/

36 /\* 定义任务ID变量 \*/

37 UINT32 Test_Task_Handle;

38 /\* 定义二值信号量的ID变量 \*/

39 UINT32 BinarySem1_Handle;

40 UINT32 BinarySem2_Handle;

41 /\* 全局变量声明 \/

42 /\*

43 \* 在写应用程序的时候，可能需要用到一些全局变量。

44 \*/

45 UINT16 Trigger_Num = 0; //用于标记的触发中断的变量

46

47 /\* 函数声明 \*/

48 static void KEY1_IRQHandler(void);

49 static void KEY2_IRQHandler(void);

50

51 static UINT32 Creat_Test_Task(void);

52 static void Test_Task(void);

53

54 static void BSP_Init(void);

55 static void AppTaskCreate(void);

56

57 /*\*

58 \* @brief 主函数

59 \* @param 无

60 \* @retval 无

61 \* @note 第一步：开发板硬件初始化

62 第二步：创建App应用任务

63 第三步：启动LiteOS，开始多任务调度，启动不成功则输出错误信息

64 \*/

65 int main(void)

66 {

67 UINT32 uwRet = LOS_OK;/\* 定义一个创建任务的返回类型，初始化为创建成功的返回值 \*/

68

69 /\* 板级初始化，所有的跟开发板硬件相关的初始化都可以放在这个函数里面 \*/

70 BSP_Init();

71 /\* 发送一个字符串 \*/

72 printf("这是一个[野火]-STM32F103霸道LiteOS中断管理实验！\n");

73

74 /\* LiteOS 核心初始化 \*/

75 uwRet = LOS_KernelInit();

76 if (uwRet != LOS_OK) {

77 printf("LiteOS 核心初始化失败！\n");

78 return LOS_NOK;

79 }

80 /\* 创建App应用任务，所有的应用任务都可以放在这个函数里面 \*/

81 AppTaskCreate();

82

83 /\* 开启LiteOS任务调度 \*/

84 LOS_Start();

85 }

86

87 /\*

88 \* @ 函数名 ： AppTaskCreate

89 \* @ 功能说明： 任务创建，为了方便管理，所有的任务创建函数都可以放在这个函数里面

90 \* @ 参数 ： 无

91 \* @ 返回值 ： 无

92 \/

93 static void AppTaskCreate(void)

94 {

95 UINTPTR uvIntSave;

96 UINT32 uwRet = LOS_OK;

97 /\* 创建一个二值信号量*/

98 uwRet = LOS_BinarySemCreate(0,&BinarySem1_Handle);

99 if (uwRet != LOS_OK) {

100 printf("BinarySem_Handle二值信号量创建失败！\n");

101 }

102 uwRet = LOS_BinarySemCreate(0,&BinarySem2_Handle);

103 if (uwRet != LOS_OK) {

104 printf("BinarySem_Handle二值信号量创建失败！\n");

105 }

106 uwRet = Creat_Test_Task();

107 if (uwRet != LOS_OK) {

108 printf("Test_Task任务创建失败！\n");

109 }

110 }

111 /\*

112 \* @ 函数名 ： Creat_Test_Task

113 \* @ 功能说明： 创建Test_Task任务

114 \* @ 参数 ： 无

115 \* @ 返回值 ： 无

116 \/

117 static UINT32 Creat_Test_Task()

118 {

119 UINT32 uwRet = LOS_OK; /\* 定义一个创建任务的返回类型，初始化为创建成功的返回值 \*/

120 TSK_INIT_PARAM_S task_init_param;

121

122 task_init_param.usTaskPrio = 5; /\* 优先级，数值越小，优先级越高 \*/

123 task_init_param.pcName = "Test_Task";/\* 任务名，字符串形式，方便调试 \*/

124 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)Test_Task;

125 task_init_param.uwStackSize = 0x1000;/\* 栈大小，单位为字，即4个字节 \*/

126

127 uwRet = LOS_TaskCreate(&Test_Task_Handle, &task_init_param);

128

129 return uwRet;

130 }

131

132 /\*

133 \* @ 函数名 ： Test_Task

134 \* @ 功能说明： 在串口打印触发中断的信息

135 \* @ 参数 ： 无

136 \* @ 返回值 ： 无

137 \/

**138 static void Test_Task(void)**

**139 {**

**140 UINT32 uwRet = LOS_OK;**

**141 while (1) { //获取二值信号量,没获取到则不等待**

**142 uwRet = LOS_SemPend( BinarySem1_Handle , 0 );**

**143 if (uwRet == LOS_OK) {**

**144 printf("触发中断的是Key1!\n\n");**

**145 } //获取二值信号量,没获取到则不等待**

**146 uwRet = LOS_SemPend( BinarySem2_Handle , 0 );**

**147 if (uwRet == LOS_OK) {**

**148 printf("触发中断的是Key2!\n\n");**

**149 }**

**150 LOS_TaskDelay(20);**

**151 }**

**152 }**

153 /\*

154 \* @ 函数名 ： BSP_Init

155 \* @ 功能说明： 板级初始化，所有的跟开发板硬件相关的初始化都可以放在这个函数里面

156 \* @ 参数 ： 无

157 \* @ 返回值 ： 无

158 \/

159 static void BSP_Init(void)

160 {

161 /\*

162 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

163 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

164 \* 都统一用这个优先级分组，千万不要再分组，切忌。

165 \*/

166 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

167

168 /\* LED 初始化 \*/

169 LED_GPIO_Config();

170

171 /\* 串口初始化 \*/

172 USART_Config();

173

**174 /\* 按键EXTI初始化 \*/**

**175 EXTI_Key_Config();**

176 }

177

178 /END OF FILE/

而中断服务函数则需要用户自己编写，并且通过信号量告知任务，如代码清单 11‑9加粗部分所示。

代码清单 11‑9 中断服务函数（stm32f1xx_it.c部分代码）

1 /\* Includes -----------------------------------------------------------*/

2 #include "stm32f10x_it.h"

3 #include "los_typedef.h"

4 #include "bsp_exti.h"

5 #include "bsp_led.h"

6 #include "los_sem.h"

7

8 /\* 定义二值信号量的ID变量 \*/

9 extern UINT32 BinarySem1_Handle;

10 extern UINT32 BinarySem2_Handle;

11 /\*

12 \* @ 函数名 ： KEY1_IRQHandler

13 \* @ 功能说明： 中断服务函数

14 \* @ 参数 ： 无

15 \* @ 返回值 ： 无

16 \/

**17 void KEY1_IRQHandler(void)**

**18 {**

**19 //确保是否产生了EXTI Line中断**

**20 if (EXTI_GetITStatus(KEY1_INT_EXTI_LINE) != RESET) {**

**21 LOS_SemPost( BinarySem1_Handle ); //释放二值信号量 BinarySem_Handle**

**22 //清除中断标志位**

**23 EXTI_ClearITPendingBit(KEY1_INT_EXTI_LINE);**

**24 }**

**25 }**

26 /\*

27 \* @ 函数名 ： KEY1_IRQHandler

28 \* @ 功能说明： 中断服务函数

29 \* @ 参数 ： 无

30 \* @ 返回值 ： 无

31 \/

**32 void KEY2_IRQHandler(void)**

**33 {**

**34 //确保是否产生了EXTI Line中断**

**35 if (EXTI_GetITStatus(KEY2_INT_EXTI_LINE) != RESET) {**

**36 LOS_SemPost( BinarySem2_Handle ); //释放二值信号量 BinarySem_Handle**

**37 //清除中断标志位**

**38 EXTI_ClearITPendingBit(KEY2_INT_EXTI_LINE);**

**39 }**

**40 }**

41

实验现象
~~~~

程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据读者买的开发板而定，每个型号的开发板都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，按下开发板的KEY1按
键触发中断发送消息1，按下KEY2按键发送消息2；按下KEY1与KEY2试试，在串口调试助手中可以看到运行结果，然后通过串口调试助手发送一段不定长信息，触发中断会在中断服务函数发送信号量通知任务，任务接收到信号量的时候将串口信息打印出来，如图11‑3所示。

|interr004|

图11‑3中断管理的实验现象

.. |interr002| image:: media\interr002.png
   :width: 5.40625in
   :height: 2.49583in
.. |interr003| image:: media\interr003.png
   :width: 4.81181in
   :height: 2.94792in
.. |interr004| image:: media\interr004.png
   :width: 5.49306in
   :height: 4.34028in

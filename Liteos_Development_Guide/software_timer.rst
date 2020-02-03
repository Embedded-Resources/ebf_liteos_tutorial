.. vim: syntax=rst

软件定时器
-----

可能有不少读者了解定时器，但可能没听说过软件定时器。在操作系统中，软件定时器提供了软件层次的接口，软件定时器与底层硬件无关，使得程序拥有更好的可移植性。

软件定时器简介
~~~~~~~

软件定时器的基本概念
^^^^^^^^^^

定时器，是指从指定的时刻开始，经过指定时间，然后触发一个超时事件，用户可以自定义定时器的周期与频率。类似生活中的闹钟，可以设置闹钟提示的时刻，也可以设置提示的次数。

定时器有硬件定时器和软件定时器之分。

1. 硬件定时器是芯片本身提供的定时功能。一般是由外部晶振提供给芯片输入时钟，芯片向软件模块提供一组配置寄存器，接受控制输入，到达设定时间值后芯片中断控制器产生时钟中断。硬件定时器的精度一般很高，可以达到纳秒级别，并且是中断触发方式。

2. 软件定时器，软件定时器是由操作系统提供的一类系统接口（函数），它构建在硬件定时器基础之上，使系统能够提供不受硬件定时器资源限制的定时器服务，它实现的功能与硬件定时器也是类似的。

使用硬件定时器时，每次在定时时间到达之后就会自动触发一个中断，用户在中断中处理信息；而使用软件定时器时，需要在创建软件定时器时指定时间到达后要调用的函数（也称超时函数/回调函数，为了统一，下文均用回调函数描述），在回调函数中处理信息。

注意：软件定时器回调函数的上下文是任务，下文所说的定时器均为软件定时器。

软件定时器在被创建之后，当经过设定的超时时间后会触发回调函数。定时精度与系统时钟的周期有关。一般系统利用SysTick作为软件定时器的时基，软件定时器的回调函数类似硬件的中断服务函数，所以，在回调函数中处理也应快进快出，而且回调函数中不能有任何阻塞任务运行的情况，如LOS_TaskDelay()以及
其他能阻塞任务运行的函数，因为软件定时器回调函数的上下文环境是任务，该任务会处理系统中所有超时的软件定时器，调用阻塞函数会影响其他定时器；两次触发回调函数的时间间隔叫定时器的定时周期。

软件定时器的使用相当于扩展了定时器的数量，允许创建更多的定时业务，LiteOS软件定时器支持以下功能。

1. 裁剪：能通过宏关闭软件定时器功能。

2. 软件定时器创建。

3. 软件定时器启动。

4. 软件定时器停止。

5. 软件定时器删除。

6. 软件定时器剩余Tick数获取。

LiteOS提供的软件定时器支持单次模式和周期模式，单次模式和周期模式的定时时间到之后都会调用软件定时器的回调函数，用户可以在回调函数中加入要执行的工程代码。

单次模式：当用户创建了定时器并启动了定时器后，指定超时时间到达，只执行一次回调函数之后就将该定时器删除，不再重新执行。

周期模式：这个定时器会按照指定的定时时间循环执行回调函数，直到用户将定时器删除，具体如图 9‑1所示。

|softwa002|

图 9‑1软件定时器的单次模式与周期模式

LiteOS通过一个软件定时器任务（osSwTmrTask）管理软定时器，软件定时器任务的优先级是0，即最高优先级，它是在LiteOS内核初始化的时候自动创建的。为了满足用户定时需求，osSwTmrTask任务会在其执行期间检查用户启动的定时器，在超时后调用对应回调函数。

软件定时器的应用场景
^^^^^^^^^^

在很多应用中，可能需要一些定时器任务，硬件定时器受硬件的限制，数量上不足以满足用户的实际需求，无法提供更多的定时器，可以采用软件定时器，由软件定时器代替硬件定时器任务。但需要注意的是软件定时器的精度是无法和硬件定时器相比的，因为在软件定时器的定时过程中是极有可能被其他中断打断，因此软件定时器更适用于
对时间精度要求不高的任务。

软件定时器的精度
^^^^^^^^

在操作系统中，通常软件定时器以系统节拍周期为计时单位。系统节拍是系统时钟的频率，系统节拍配置为LOSCFG_BASE_CORE_TICK_PER_SECOND，默认是1000HZ，因此系统的时钟节拍周期就为1Tick。由于节拍定义了系统中定时器能够分辨的精确度，系统可以根据实际系统CPU的处理能力和
实时性需求设置合适的数值，系统节拍宏定义的值越大，精度越高，但是系统开销也将越大，因为这代表在1s内系统进入中断的次数也就越多。

软件定时器的运作机制
^^^^^^^^^^

软件定时器是可选的系统组件，在模块初始化的时候已经分配了一块连续的内存，系统支持的最大定时器个数由target_config.h中的LOSCFG_BASE_CORE_SWTMR_LIMIT宏配置。

软件定时器使用了系统中一个队列和一个任务资源，系统通过软件定时器命令队列处理软件定时器。

软件定时器以Tick为基本计时单位，当用户创建并启动一个软件定时器时， LiteOS会根据当前系统Tick与用户指定的超时时间计算出该定时器超时的Tick，并将该定时器插入定时器列表。

系统会在SysTick中断处理函数中扫描软件定时器列表，如果有定时器超时则通过“定时器命令队列”向软件定时器任务发送一个命令，任务在接收到命令就会去处理命令对应的程序，调用对应软件定时器的回调函数。

如果软件定时器的定时时间到来，那么在Tick中断处理函数结束后，软件定时器任务osSwTmrTask（优先级为最高）被唤醒，在该任务中调用创建软件定时器时用户指定的回调函数。

定时器状态有以下几种：

1. OS_SWTMR_STATUS_UNUSED（未使用），系统在定时器模块初始化的时候将系统中所有定时器资源初始化成该状态。

2. OS_SWTMR_STATUS_CREATED（创建未启动/停止），在未使用状态下调用LOS_SwtmrCreate接口或者启动后调用LOS_SwtmrStop接口后，定时器将变成该状态。

3. OS_SWTMR_STATUS_TICKING（运行），在定时器创建后调用LOS_SwtmrStart()函数接口，定时器将变成该状态，表示定时器运行时的状态。

使用软件定时器时候要注意以下几点：

1. 在软件定时器的回调函数中处理时间应该尽可能短，不允许使用导致任务挂起或者阻塞的函数，如LOS_TaskDelay()。

2. 软件定时器占用了系统的一个队列和一个任务资源，软件定时器任务的优先级设定为0，且不允许修改 。

3. 创建单次软件定时器，该定时器超时执行完回调函数后，系统会自动删除该软件定时器，并回收资源。

软件定时器的使用讲解
~~~~~~~~~~

软件定时器控制块
^^^^^^^^

LiteOS最大支持LOSCFG_BASE_CORE_SWTMR_LIMIT个软件定时器，该宏在target_config.h文件中配置，每个软件定时器都有对应的软件定时器控制块，每个软件定时器控制块都包含了软件定时器的基本信息，如软件定时器的状态、软件定时器工作模式、软件定时器的计数值，以及软件定
时器回调函数等信息，如代码清单 9‑1所示。

代码清单 9‑1软件定时器控制块

1 /*\*

2 \* @ingroup los_swtmr

3 \* 软件定时器控制块结构体

4 \*/

5 typedef struct tagSwTmrCtrl {

6 struct tagSwTmrCtrl \*pstNext; **(1)**

7 UINT8 ucState; **(2)**

8 UINT8 ucMode; **(3)**

9 #if (LOSCFG_BASE_CORE_SWTMR_ALIGN == YES)

10 UINT8 ucRouses; **(4)**

11 UINT8 ucSensitive; **(5)**

12 #endif

13 UINT16 usTimerID; **(6)**

14 UINT32 uwCount; **(7)**

15 UINT32 uwInterval; **(8)**

16 UINT32 uwArg; **(9)**

17 SWTMR_PROC_FUNC pfnHandler; **(10)**

18 } SWTMR_CTRL_S;

代码清单 9‑1\ **(1)**\ ：指向下一个软件定时器控制块的指针。

代码清单 9‑1\ **(2)**\ ：软件定时器状态有以下三种：OS_SWTMR_STATUS_UNUSED（未使用状态）、OS_SWTMR_STATUS_CREATED（创建未启动/停止状态）、OS_SWTMR_STATUS_TICKING（运行状态）。

代码清单 9‑1\ **(3)**\ ：软件定时器模式：单次模式、周期模式等。

代码清单 9‑1\ **(4)**\ ：如果定义了LOSCFG_BASE_CORE_SWTMR_ALIGN则使能软件定时器唤醒功能。

代码清单 9‑1\ **(5)**\ ：如果定义了LOSCFG_BASE_CORE_SWTMR_ALIGN则使能软件定时器对齐。

代码清单 9‑1\ **(6)**\ ：软件定时器ID。

代码清单 9‑1\ **(7)**\ ：软件计时器的计数值，用来记录软件定时器距离超时的剩余时间。

代码清单 9‑1\ **(8)**\ ：软件定时器的超时时间间隔，即调用回调函数的周期。

代码清单 9‑1\ **(9)**\ ：调用回调函数时传入的参数。

代码清单 9‑1\ **(10)**\ ：处理软件定时器超时的回调函数。

软件定时器错误代码
^^^^^^^^^

在LiteOS中，与软件定时器相关的函数大多数都会有返回值，其返回值是一些错误代码，方便使用者进行调试，本书列出一些常见的错误代码与参考解决方案，如表 9‑1所示。

表 9‑1常见软件定时器错误代码

.. list-table::
   :widths: 25 25 25 25
   :header-rows: 0


   * - 序号 |
     - 义              | 描述
     - | 参考解决
     - 案      |

   * - 1
     - LOS_ERR NO_SWTMR_PTR_NULL
     - 软件定            | 定 时器回调函数为空  | 件定时器回调
     - 软            | 数  |

   * - 2
     - L OS_ERRNO_SWTMR_IN TERVAL_NOT_SUITED
     - 软件              | 定时器间隔时间为0 |
     - 新定义间隔时间  | |

   * - 3
     - LOS_ERRNO_S WTMR_MODE_INVALID
     - 不正确            | 确 的软件定时器模式  | 认软件定时器
     - |

   * - 4
     - LOS_ERRNO_S WTMR_RET_PTR_NULL
     - 软件定时器        | 定义 ID指针入参为NULL  | ID变
     - |

   * - 5
     - LOS_ER RNO_SWTMR_MAXSIZE
     - 软件定时          | 重新 器个数超过最大值  | 件定时器最大
     - 义软        | 数  | ，或者等待一个软  | 件定时器释放资源  |

   * - 6
     - LOS_ERRNO _SWTMR_ID_INVALID
     - 不正确的          | 确保 软件定时器ID入参  |
     - 参合法      | |

   * - 7
     - LOS_ERRNO_ SWTMR_NOT_CREATED
     - 软件定时器未创建  | 创建软件定时
     - |

   * - 8
     - LOS_ERRN O_SWTMR_NO_MEMORY
     - 软件定时器        | 申请 链表创建内存不足  | 一块足够大的
     - |

   * - 9
     - LOS_ERRNO_SWTM R_MAXSIZE_INVALID
     - 不正确的软件      | 重新定义 定时器个数最大值  |
     - 值      | |

   * - 10
     - LOS_ERRNO _SWTMR_HWI_ACTIVE
     - 在                | 中断中使用定时器  | 保不在中断中
     - 修改源代码确      | 用  |

   * - 11
     - L OS_ERRNO_SWTMR_HA NDLER_POOL_NO_MEM
     - membox内存不足    | 扩大
     - 存          |

   * - 12
     - L OS_ERRNO_SWTMR_QU EUE_CREATE_FAILED
     - 软件定            | 检 时器队列创建失败  | 列的内存是否
     - 用以创建队    | 够  |

   * - 13
     - LOS_ERRNO_SWTMR_T ASK_CREATE_FAILED
     - 软件定            | 检 时器任务创建失败  | 查用以创建软
     - |

   * - 14
     - LOS_ERRNO_ SWTMR_NOT_STARTED
     - 未启动软件定时器  | 启动软件定时
     - |

   * - 15
     - LOS_ERRNO_SWT MR_STATUS_INVALID
     - 不正确            | 检 的软件定时器状态  | 认软件定时器
     - 确            | 态  |

   * - 16
     - LOS_ERRNO_SW TMR_TICK_PTR_NULL
     - 用以获取软件      | 创 定时器超时Tick数  | 建一个有 的入参指针为NULL  |
     - |

        |


软件定时器典型开发流程
^^^^^^^^^^^

1. 在target_config.h文件中确认配置项LOSCFG_BASE_CORE_SWTMR和LOSCFG_BASE_IPC_QUEUE为YES打开状态。

2. 在target_config.h文件中配置LOSCFG_BASE_CORE_SWTMR_LIMIT最大支持的软件定时器数。

3. 在target_config.h文件中配置OS_SWTMR_HANDLE_QUEUE_SIZE软件定时器队列最大长度。

4. 创建一个指定定时时间、指定超时处理函数、指定触发模式的软件定时器。

5. 编写软件定时器回调函数。

6. 启动定时器LOS_SwtmrStart。

7. 停止定时器LOS_SwtmrStop。

8. 删除定时器LOS_SwtmrDelete。

软件定时器创建函数LOS_SwtmrCreate()
^^^^^^^^^^^^^^^^^^^^^^^^^^

LiteOS提供软件定时器创建函数LOS_SwtmrCreate()，读者在使用软件定时器前需要先创建软件定时器，同时还需要定义一个软件定时器ID变量，用于保存创建成功后返回的软件定时器ID，其源码如代码清单 9‑2所示，使用实例如代码清单 9‑4加粗部分所示。

代码清单 9‑2软件定时器创建函数LOS_SwtmrCreate()源码

1 /\*

2 Function : LOS_SwtmrCreate

3 Description: 创建一个软件定时器

4 Input : uwInterval ：软件定时器的定时时间（Tick）

5 usMode ：软件定时器的工作模式

6 pfnHandler ：软件定时器的回调函数

7 uwArg ：软件定时器传入参数

8 Output : pusSwTmrID ：软件定时器ID指针

9 Return : 返回LOS_OK表示创建成功,或者其他失败的错误代码

10 \/

11 LITE_OS_SEC_TEXT_INIT UINT32 LOS_SwtmrCreate(UINT32 uwInterval,

12 UINT8 ucMode,

13 SWTMR_PROC_FUNC pfnHandler,

14 UINT16 \*pusSwTmrID,

15 UINT32 uwArg

16 #if (LOSCFG_BASE_CORE_SWTMR_ALIGN == YES) **(1)**

17 ,UINT8 ucRouses,

18 UINT8 ucSensitive

19 #endif

20 )

21 {

22 SWTMR_CTRL_S \*pstSwtmr;

23 UINTPTR uvIntSave;

24

25 if (0 == uwInterval) { **(2)**

26 return LOS_ERRNO_SWTMR_INTERVAL_NOT_SUITED;

27 }

28

29 if ((LOS_SWTMR_MODE_ONCE != ucMode) **(3)**

30 && (LOS_SWTMR_MODE_PERIOD != ucMode)

31 && (LOS_SWTMR_MODE_NO_SELFDELETE != ucMode)) {

32 return LOS_ERRNO_SWTMR_MODE_INVALID;

33 }

34

35 if (NULL == pfnHandler) { **(4)**

36 return LOS_ERRNO_SWTMR_PTR_NULL;

37 }

38

39 if (NULL == pusSwTmrID) { **(5)**

40 return LOS_ERRNO_SWTMR_RET_PTR_NULL;

41 }

42

43 #if (LOSCFG_BASE_CORE_SWTMR_ALIGN == YES)

44 if((OS_SWTMR_ROUSES_IGNORE != ucRouses)&&(OS_SWTMR_ROUSES_ALLOW != ucRouses)) {

45 return OS_ERRNO_SWTMR_ROUSES_INVALID;

46 }

47

48 if ((OS_SWTMR_ALIGN_INSENSITIVE != ucSensitive)&&

49 (OS_SWTMR_ALIGN_SENSITIVE != ucSensitive)) {

50 return OS_ERRNO_SWTMR_ALIGN_INVALID;

51 }

52 #endif

53

54 uvIntSave = LOS_IntLock();

55 if (NULL == m_pstSwtmrFreeList) { **(6)**

56 LOS_IntRestore(uvIntSave);

57 return LOS_ERRNO_SWTMR_MAXSIZE;

58 }

59

60 pstSwtmr = m_pstSwtmrFreeList;

61 m_pstSwtmrFreeList = pstSwtmr->pstNext;

62 LOS_IntRestore(uvIntSave);

63 pstSwtmr->pfnHandler = pfnHandler; **(7)**

64 pstSwtmr->ucMode = ucMode; **(8)**

65 pstSwtmr->uwInterval = uwInterval; **(9)**

66 pstSwtmr->pstNext = (SWTMR_CTRL_S \*)NULL; **(10)**

67 pstSwtmr->uwCount = 0; **(11)**

68 pstSwtmr->uwArg = uwArg; **(12)**

69 #if (LOSCFG_BASE_CORE_SWTMR_ALIGN == YES)

70 pstSwtmr->ucRouses = ucRouses;

71 pstSwtmr->ucSensitive = ucSensitive;

72 #endif

73 pstSwtmr->ucState = OS_SWTMR_STATUS_CREATED; **(13)**

74 \*pusSwTmrID = pstSwtmr->usTimerID; **(14)**

75

76 return LOS_OK;

77 }

代码清单 9‑2\ **(1)**\ ：如果配置了LOSCFG_BASE_CORE_SWTMR_ALIGN，则需要传入ucRouses与ucSensitive参数，这是关于软件定时器对齐的，暂时无需理会。

代码清单 9‑2\ **(2)**\ ：如果软件定时器间隔时间为0，返回错误代码。

代码清单 9‑2\ **(3)**\ ：如果软件定时器的工作模式不正确，返回错误代码。 LiteOS的软件定时器支持的工作模式有以下几种，目前支持的仅有前3种，如代码清单 9‑3所示。

代码清单 9‑3LiteOS软件定时器工作模式

1 enum enSwTmrType {

2 LOS_SWTMR_MODE_ONCE, /**< 单次模式 \*/

3 LOS_SWTMR_MODE_PERIOD, /**< 周期模式 \*/

4 LOS_SWTMR_MODE_NO_SELFDELETE, /**< 单次模式，但不能删除自己 \*/

5 LOS_SWTMR_MODE_OPP, /**<在一次性定时器完成定时后，启用定期

6 软件定时器。 暂时不支持此模式。*/

7 };

代码清单 9‑2\ **(4)**\ ：如果用户没有实现软件定时器的回调函数，也返回错误代码，用户需要自己编写软件定时器回调函数。

代码清单 9‑2\ **(5)**\ ：如果软件定时器ID变量的地址为NULL，则返回错误代码。

代码清单 9‑2\ **(6)**\ ：当系统已经使用的软件定时器个数超过支持的最大值时，返回错误代码，读者可以在target_config.h文件中修改LOSCFG_BASE_CORE_SWTMR_LIMIT宏定义以增加系统支持的软件定时器最大个数。

代码清单 9‑2\ **(7)**\ ：从软件定时器未使用列表中取下一个软件定时器，然后根据用户指定参数对软件定时器进行初始化，首先初始化软件定时器的回调函数。

代码清单 9‑2\ **(8)**\ ：初始化软件定时器的工作模式。

代码清单 9‑2\ **(9)**\ ：初始化软件定时器的处理周期。

代码清单 9‑2\ **(10)**\ ：初始化pstNext指针为NULL，在启动软件定时器的时候会按照唤醒时间升序插入软件定时器列表中。

代码清单 9‑2\ **(11)**\ ：初始化软件定时器的剩余唤醒时间为0，在启动软件定时器的时候会重新计算。

代码清单 9‑2\ **(12)**\ ：初始软件定时器回调函数的传入参数。

代码清单 9‑2\ **(13)**\ ：初始化软件定时器的状态为OS_SWTMR_STATUS_CREATED，表示软件定时器是处于创建状态，尚未启动。

代码清单 9‑2\ **(14)**\ ：将软件定时器ID通过pusSwTmrID指针返回给用户。

代码清单 9‑4软件定时器创建函数LOS_SwtmrCreate()实例

1 UINT32 uwRet = LOS_OK;/\* 定义一个创建任务的返回类型，初始化为创建成功的返回值 \*/

2

**3 /\* 创建一个软件定时器定时器*/**

**4 uwRet = LOS_SwtmrCreate(5000, /\* 软件定时器的定时时间（Tick）*/**

**5 LOS_SWTMR_MODE_ONCE, /\* 软件定时器模式 一次模式 \*/**

**6 (SWTMR_PROC_FUNC)Timer1_Callback, //软件定时器的回调函数**

**7 &Timer1_Handle, /\* 软件定时器的id \*/**

**8 0); /*软件定时器的回调函数传入参数 \*/**

**9**

10 if (uwRet != LOS_OK)

11 {

12 printf("软件定时器Timer1创建失败！\n");

13 }

注意：如果使能了LOSCFG_BASE_CORE_SWTMR_ALIGN宏定义则还需传入两个参数：ucRouses与ucSensitive。

软件定时器的回调函数是由用户实现的，类似于中断服务函数，在回调函数中的处理时间尽可能短，虽然软件定时器回调函数的上下文环境是任务，但不允许调用任何阻塞任务运行的函数，回调函数的应用实例如代码清单 9‑5加粗部分所示。

代码清单 9‑5软件定时器回调函数

1 /\*

2 \* @ 函数名 ： Timer1_Callback

3 \* @ 功能说明： 软件定时器回调函数

4 \* @ 参数 ： 传入1个参数，但未使用

5 \* @ 返回值 ： 无

6 \/

**7 static void Timer1_Callback(UINT32 arg)**

**8 {**

**9 UINT32 tick_num;**

**10**

**11 TmrCb_Count++; /\* 每回调一次加一 \*/**

**12 LED1_TOGGLE;**

**13 tick_num1 = (UINT32)LOS_TickCountGet(); /\* 获取滴答定时器的计数值 \*/**

**14**

**15 printf("Timer_CallBack_Count=%d\n", TmrCb_Count);**

**16 printf("tick_num=%d\n", tick_num);**

**17 }**

软件定时器删除函数LOS_SwtmrDelete()
^^^^^^^^^^^^^^^^^^^^^^^^^^

LiteOS允许用户主动删除软件定时器，被删除的软件定时器不会继续执行，回调函数也无法再次被调用，关于该软件定时器的所有资源都会被系统回收。软件定时器删除函数LOS_SwtmrDelete()的源码如代码清单 9‑6所示。

代码清单 9‑6软件定时器删除函数LOS_SwtmrDelete()源码

1 /\*

2 Function : LOS_SwtmrDelete

3 Description: 删除一个软件定时器

4 Input : usSwTmrID ------- 软件定时器ID

5 Output : None

6 Return : 返回LOS_OK表示删除成功,或者其他失败的错误代码

7 \/

8 LITE_OS_SEC_TEXT UINT32 LOS_SwtmrDelete(UINT16 usSwTmrID)

9 {

10 SWTMR_CTRL_S \*pstSwtmr;

11 UINTPTR uvIntSave;

12 UINT32 uwRet = LOS_OK;

13 UINT16 usSwTmrCBID;

14

15 CHECK_SWTMRID(usSwTmrID, uvIntSave, usSwTmrCBID, pstSwtmr); **(1)**

16 switch (pstSwtmr->ucState) {

17 case OS_SWTMR_STATUS_UNUSED: **(2)**

18 uwRet = LOS_ERRNO_SWTMR_NOT_CREATED;

19 break;

20 case OS_SWTMR_STATUS_TICKING: **(3)**

21 osSwtmrStop(pstSwtmr);

22 case OS_SWTMR_STATUS_CREATED: **(4)**

23 osSwtmrDelete(pstSwtmr);

24 break;

25 default:

26 uwRet = LOS_ERRNO_SWTMR_STATUS_INVALID;

27 break;

28 }

29

30 LOS_IntRestore(uvIntSave);

31 return uwRet;

32 }

代码清单 9‑6\ **(1)**\ ：检查要删除的软件定时器的ID是否有效，CHECK_SWTMRID其实上一个宏定义，在los_swtmr.c文件中定义，在这个宏定义中实现了检查软件定时器ID是否有效，如果有效则根据软件定时器ID进行获取软件定时器控制块pstSwtmr。

代码清单 9‑6\ **(2)**\ ：获取软件定时器的状态，并根据软件定时器的状态进行删除操作，如果要删除的软件定时器是没有被创建或者已经被删除的，则直接返回错误代码LOS_ERRNO_SWTMR_NOT_CREATED。

代码清单 9‑6\ **(3)**\ ：如果软件定时器还在运行中，则先停止软件定时器而不是直接删除，在软件定时器被停止之后，它没有break，所以是不会退出switch语句，然后再进行删除操作。

代码清单 9‑6\ **(4)**\
：如果软件定时器已经停止了，则表示可以进行删除操作，调用osSwtmrDelete()函数进行删除操作：将软件定时器归还到系统软件定时器未使用列表中，并且将软件定时器的状态变为OS_SWTMR_STATUS_UNUSED，以便在下次创建软件定时器的时候能从未使用列表获取到软件定时器，如代码清单
9‑7所示。

代码清单 9‑7 osSwtmrDelete()删除软件定时器源码

1 LITE_OS_SEC_TEXT STATIC_INLINE VOID osSwtmrDelete(SWTMR_CTRL_S \*pstSwtmr)

2 {

3 /*\* 插入软件定时器未使用列表中 \**/

4 pstSwtmr->pstNext = m_pstSwtmrFreeList;

5 m_pstSwtmrFreeList = pstSwtmr;

6 pstSwtmr->ucState = OS_SWTMR_STATUS_UNUSED;

7

8 #if (LOSCFG_BASE_CORE_SWTMR_ALIGN == YES)

9 m_uwSwTmrAlignID[pstSwtmr->usTimerID % LOSCFG_BASE_CORE_SWTMR_LIMIT] = 0;

10 #endif

11 }

进行软件定时器删除操作要传入正确的软件定时器ID，并且应先将软件定时器停止工作，再进行软件定时器删除，其使用实例如代码清单 9‑8加粗部分所示。

代码清单 9‑8软件定时器删除函数LOS_SwtmrDelete()实例

1 UINT32 uwRet = LOS_OK;

**2 uwRet = LOS_SwtmrDelete(Timer_Handle);//删除软件定时器**

3 if (LOS_OK != uwRet)

4 {

5 printf("删除软件定时器失败\n");

6 } else

7 {

8 printf("删除成功\n");

9 }

软件定时器启动函数LOS_SwtmrStart()
^^^^^^^^^^^^^^^^^^^^^^^^^

在创建成功软件定时器的时候，软件定时器的状态从OS_SWTMR_STATUS_UNUSED（未使用状态）变成OS_SWTMR_STATUS_CREATED（创建未启动/停止状态），创建完成的软件定时器是未运行的，用户在需要的时候可以启动它，LirteOS提供了软件定时器启动函数LOS_SwtmrSt
art()，如代码清单 9‑9所示，使用实例如代码清单 9‑11加粗部分所示。

代码清单 9‑9软件定时器启动函数LOS_SwtmrStart()

1 /\*

2 Function : LOS_SwtmrStart

3 Description: 启动一个软件定时器

4 Input : usSwTmrID ------- 软件定时器ID

5 Output : None

6 Return : 返回LOS_OK表示启动成功,或者其他失败的错误代码

7 \/

8 LITE_OS_SEC_TEXT UINT32 LOS_SwtmrStart(UINT16 usSwTmrID)

9 {

10 SWTMR_CTRL_S \*pstSwtmr;

11 UINTPTR uvIntSave;

12 #if (LOSCFG_BASE_CORE_SWTMR_ALIGN == YES)

13 UINT32 uwTimes;

14 #endif

15 UINT32 uwRet = LOS_OK;

16 UINT16 usSwTmrCBID;

17

18 CHECK_SWTMRID(usSwTmrID, uvIntSave, usSwTmrCBID, pstSwtmr);

19 #if (LOSCFG_BASE_CORE_SWTMR_ALIGN == YES) **(1)**

20 if ( OS_SWTMR_ALIGN_INSENSITIVE == pstSwtmr->ucSensitive &&

21 LOS_SWTMR_MODE_PERIOD == pstSwtmr->ucMode ) {

22 SET_ALIGN_SWTMR_CAN_ALIGNED(m_uwSwTmrAlignID[pstSwtmr->

23 usTimerID % LOSCFG_BASE_CORE_SWTMR_LIMIT]);

24 if (pstSwtmr->uwInterval % LOS_COMMON_DIVISOR == 0) {

25 SET_ALIGN_SWTMR_CAN_MULTIPLE(m_uwSwTmrAlignID[pstSwtmr->

26 usTimerID % LOSCFG_BASE_CORE_SWTMR_LIMIT]);

27 uwTimes = pstSwtmr->uwInterval / (LOS_COMMON_DIVISOR);

28 SET_ALIGN_SWTMR_DIVISOR_TIMERS(m_uwSwTmrAlignID[pstSwtmr->

29 usTimerID % LOSCFG_BASE_CORE_SWTMR_LIMIT], uwTimes);

30 }

31 }

32 #endif

33

34 switch (pstSwtmr->ucState) {

35 case OS_SWTMR_STATUS_UNUSED: **(2)**

36 uwRet = LOS_ERRNO_SWTMR_NOT_CREATED;

37 break;

38 case OS_SWTMR_STATUS_TICKING: **(3)**

39 osSwtmrStop(pstSwtmr);

40 case OS_SWTMR_STATUS_CREATED: **(4)**

41 osSwTmrStart(pstSwtmr);

42 break;

43 default:

44 uwRet = LOS_ERRNO_SWTMR_STATUS_INVALID;

45 break;

46 }

47

48 LOS_IntRestore(uvIntSave);

49 return uwRet;

50 }

代码清单 9‑9\ **(1)**\ ：当配置了LOSCFG_BASE_CORE_SWTMR_ALIGN才会对软件定时器进行对齐操作，此处暂时无需理会。

代码清单 9‑9\ **(2)**\ ：在CHECK_SWTMRID这个宏定义中会根据软件定时器ID获取软件定时器的状态，现在判断一下其状态，如果软件定时器没有创建或者已经删除了，是无法启动的，返回错误代码LOS_ERRNO_SWTMR_NOT_CREATED。

代码清单 9‑9 **(3)**\ ：如果软件定时器已经启动了，再次调用LOS_SwtmrStart()函数将会停止已经启动的定时器，然后重新启动软件定时器，因为停止软件定时器之后，并没有退出switch语句。

代码清单 9‑9 **(4)**\ ：调用osSwTmrStart()函数启动软件定时器，该函数源码如代码清单 9‑10所示。

代码清单 9‑10 osSwTmrStart()源码

1 /\*

2 Function : osSwTmrStart

3 Description: 启动一个软件定时器

4 Input : pstSwtmr ---- 需要启动软件定时器

5 Output : None

6 Return : None

7 \/

8 LITE_OS_SEC_TEXT VOID osSwTmrStart(SWTMR_CTRL_S \*pstSwtmr)

9 {

10 SWTMR_CTRL_S \*pstPrev = (SWTMR_CTRL_S \*)NULL;

11 SWTMR_CTRL_S \*pstCur = (SWTMR_CTRL_S \*)NULL;

12

13 /\*

14

15 \* 中间省略配置了LOSCFG_BASE_CORE_SWTMR_ALIGN才有用的代码

16 \* 本例程中未使用LOSCFG_BASE_CORE_SWTMR_ALIGN

17 \* .....

18 \* .....

19

20 \/

21

22 pstSwtmr->uwCount = pstSwtmr->uwInterval;

23

24 pstCur = m_pstSwtmrSortList; **(1)**

25 while (pstCur != NULL) {

26 if (pstCur->uwCount > pstSwtmr->uwCount) { **(2)**

27 break;

28 }

29

30 pstSwtmr->uwCount -= pstCur->uwCount; **(3)**

31 pstPrev = pstCur;

32 pstCur = pstCur->pstNext; **(4)**

33 }

34

35 pstSwtmr->pstNext = pstCur; **(5)**

36

37 if (pstCur != NULL) {

38 pstCur->uwCount -= pstSwtmr->uwCount; **(6)**

39 }

40

41 if (pstPrev == NULL) {

42 m_pstSwtmrSortList = pstSwtmr; **(7)**

43 } else {

44 pstPrev->pstNext = pstSwtmr; **(8)**

45 }

46

47 pstSwtmr->ucState = OS_SWTMR_STATUS_TICKING; **(9)**

48

49 return;

50 }

在启动的过程中，会将软件定时器按唤醒时间升序插入软件定时器列表中，距离唤醒时间越短的软件定时器排在列表头部，距离唤醒时间越长的软件定时器排在尾部。例如，软件定时器列表中一开始只有一个周期为200个Tick的软件定时器A，那么A定时器在200个Tick后就会被唤醒，调用对应的回调函数；此时插入一个周期
为100个Tick的软件定时器B，那么100个Tick之后，软件定时器B就会被唤醒，而原来在200个Tick后唤醒的软件定时器A，将会在软件定时器B调用之后的100个Tick唤醒；同理，插入一个周期为50个Tick的软件定时器C也是一样的，如图 9‑2与图 9‑3所示。

|softwa003|

图 9‑2软件定时器插入队列时的排序

|softwa004|

图 9‑3软件定时器插入队列时的排序

上文简单分析了插入软件定时器列表的过程，那么结合源码分析LiteOS将软件定时器插入软件定时器列表的实现过程：

代码清单 9‑10\ **(1)**\ ：m_pstSwtmrSortList是LiteOS管理软件定时器的列表，所有被创建并且启动的软件定时器都会被插入这个软件定时器列表中，首先获取软件定时器列表的第一个软件软件定时器，保存在局部变量pstCur中。

代码清单 9‑10\ **(2)**\ ：当pstCur不为空的时候，表明软件定时器列表中存在软件定时器，那就进行新的软件定时器插入操作，系统将列表中的第一个软件定时器（pstCur）唤醒时间与新插入的软件定时器唤醒时间比较一下。如果pstCur的唤醒时间是大于新插入的软件定时器的唤醒时间，那就直接
退出循环，说明新插入的软件定时器应该处于软件定时器列表头部，因为它距离唤醒的时间是最小的，如图 9‑2\ **(2)**\ 所示。

代码清单 9‑10\ **(3)**\ ：如果插入的软件定时器距离唤醒时间不是最小的，则继续寻找，直到应该合适的位置。这时候新插入的软件定时器唤醒的时间应该要减去前一个唤醒的时间，如图
9‑3所示插入的软件定时器C，本来插入的周期是130个Tick，减去软件定时器A唤醒的时间50个Tick，这表明在软件定时器A唤醒之后的80个Tick再去唤醒软件定时器C，而软件定时器A距离唤醒的时间是50个Tick，等到唤醒软件定时器C也是经过的时间是130个Tick（50+80），与设定的一致。

代码清单 9‑10\ **(4)**\ ：继续寻找要插入的位置，直到找到合适的位置，才退出循环。

代码清单 9‑10\ **(5)**\ ：找到合适的插入位置，那么需要进行插入操作，新插入的软件定时器的执向下一个软件定时器就是pstCur，如图 9‑2\ **(3)**\ 和图 9‑3\ **(2)**\ 所示。

代码清单 9‑10\ **(6)**\ ：如果pstCur不为NULL，表示插入的软件定时器后面还是有定时器的，那么需要改变其唤醒的时间，减去插入的软件定时器时间，如图 9‑2所示中软件定时器A、B和图 9‑3所示中软件定时器B。

代码清单 9‑10\ **(7)**\ ：如果新插入的软件定时器前面没有定时器了，表示该软件定时器插入到软件定时器列表头部，所以m_pstSwtmrSortList要指向新插入的软件定时器，如图 9‑2所示中的软件定时器C。

代码清单 9‑10\ **(8)**\ ：而新插入的软件定时器前面还存在软件定时器，那么就让该软件定时器的pstNext指针指向新插入的软件定时器，如图 9‑3\ **(3)**\ 所示。

代码清单 9‑10\ **(9)**\ ：设置软件定时器状态为工作状态。

代码清单 9‑11软件定时器启动函数LOS_SwtmrStart()实例

1 UINT32 uwRet = LOS_OK;

**2 /\* 启动一个软件定时器定时器*/**

**3 uwRet = LOS_SwtmrStart(Timer2_Handle);**

4 if (LOS_OK != uwRet)

5 {

6 printf("start Timer2 failed\n");

7 } else

8 {

9 printf("start Timer2 sucess\n");

10 }

软件定时器停止函数LOS_SwtmrStop()
^^^^^^^^^^^^^^^^^^^^^^^^

与软件定时器启动函数相反的是软件定时器停止函数，软件定时器停止函数LOS_SwtmrStop()是用于停止正在运行的软件定时器，在不需要使用的时候可以停止软件定时器，或者是需要删除某个软件定时器之前应先把软件定时器停止，所以，软件定时器的停止也是很常用的函数，其源码如代码清单 9‑12所示。

代码清单 9‑12软件定时器停止函数LOS_SwtmrStop()源码

1 /\*

2 Function : LOS_SwtmrStop

3 Description: 停止一个软件定时器

4 Input : usSwTmrID ------- 软件定时器ID

5 Output : None

6 Return : 返回LOS_OK表示停止成功,或者其他失败的错误代码

7 \/

8 LITE_OS_SEC_TEXT UINT32 LOS_SwtmrStop(UINT16 usSwTmrID)

9 {

10 SWTMR_CTRL_S \*pstSwtmr;

11 UINTPTR uvIntSave;

12 UINT16 usSwTmrCBID;

13 UINT32 uwRet = LOS_OK;

14

15 CHECK_SWTMRID(usSwTmrID, uvIntSave, usSwTmrCBID, pstSwtmr); **(1)**

16 switch (pstSwtmr->ucState) {

17 case OS_SWTMR_STATUS_UNUSED: **(2)**

18 uwRet = LOS_ERRNO_SWTMR_NOT_CREATED;

19 break;

20 case OS_SWTMR_STATUS_CREATED: **(3)**

21 uwRet = LOS_ERRNO_SWTMR_NOT_STARTED;

22 break;

23 case OS_SWTMR_STATUS_TICKING: **(4)**

24 osSwtmrStop(pstSwtmr);

25 break;

26 default:

27 uwRet = LOS_ERRNO_SWTMR_STATUS_INVALID;

28 break;

29 }

30

31 LOS_IntRestore(uvIntSave);

32 return uwRet;

33 }

代码清单 9‑12\ **(1)**\ ：通过宏定义CHECK_SWTMRID检查软件定时器ID是否有效，并且根据软件定时器ID获取对应的软件定时器控制块。

代码清单 9‑12\ **(2)**\ ：获取当前定时器的状态，如果软件定时器没有创建或者已经被删除了，返回错误代码LOS_ERRNO_SWTMR_NOT_CREATED。

代码清单 9‑12\ **(3)**\ ：如果软件定时器没有启动，则返回错误代码。

代码清单 9‑12\ **(4)**\ ：如果软件定时器已经启动了，调用软件定时器停止函数LOS_SwtmrStop()将会停止已经启动的定时器。而真正停止软件定时器的代码是osSwtmrStop()，如代码清单 9‑13所示。

代码清单 9‑13软件定时器停止函数osSwtmrStop源码

1 /\*

2 Function : osSwtmrStop

3 Description: 停止一个软件定时器

4 Input : pstSwtmr

5 Output : None

6 Return : None

7 \/

8 LITE_OS_SEC_TEXT VOID osSwtmrStop(SWTMR_CTRL_S \*pstSwtmr)

9 {

10 SWTMR_CTRL_S \*pstPrev = (SWTMR_CTRL_S \*)NULL;

11 SWTMR_CTRL_S \*pstCur = (SWTMR_CTRL_S \*)NULL;

12

13 if (!m_pstSwtmrSortList)

14 return;

15

16 pstCur = m_pstSwtmrSortList; **(1)**

17

18 while (pstCur != pstSwtmr) {

19 pstPrev = pstCur;

20 pstCur = pstCur->pstNext; **(2)**

21 }

22

23 if (pstCur->pstNext != NULL) {

24 pstCur->pstNext->uwCount += pstCur->uwCount; **(3)**

25 }

26

27 if (pstPrev == NULL) {

28 m_pstSwtmrSortList = pstCur->pstNext; **(4)**

29 } else {

30 pstPrev->pstNext = pstCur->pstNext; **(5)**

31 }

32

33 pstCur->pstNext = (SWTMR_CTRL_S \*)NULL;

34 pstCur->ucState = OS_SWTMR_STATUS_CREATED; **(6)**

35

36 #if (LOSCFG_BASE_CORE_SWTMR_ALIGN == YES)

37 SET_ALIGN_SWTMR_ALREADY_NOT_ALIGNED(m_uwSwTmrAlignID[

38 pstSwtmr->usTimerID % LOSCFG_BASE_CORE_SWTMR_LIMIT]);

39 #endif

40 }

代码清单 9‑13\ **(1)**\ ：获取软件定时器列表的第一个软件定时器，并且保存在pstCur中，为遍历定时器列表做准备。

代码清单 9‑13\ **(2)**\ ：如果pstCur不是要停止的软件定时器，那就需要遍历软件定时器列表，直到找到要停止的软件定时器。

代码清单 9‑13\ **(3)**\ ：如果要停止的软件定时器后面还有定时器的话，那么要修改该定时器唤醒的时间，即加上要停止的软件定时器的时间。

代码清单 9‑13\ **(4)**\ ：如果停止的软件定时器是列表中第一个的话，那么将m_pstSwtmrSortList指向列表中第二个定时器（当前软件定时器的下一个）。

代码清单 9‑13\ **(5)**\ ：如果停止的不是列表中第一个软件定时器的话，就要将软件定时器前后的两个定时器连接起来。

代码清单 9‑13\ **(6)**\ ：设置软件定时器的状态是停止状态。

软件定时器实验
~~~~~~~

软件定时器实验是在LiteOS中创建了两个软件定时器，其中一个软件定时器是单次模式，5000Tick调用一次回调函数，另一个软件定时器是周期模式，1000Tick调用一次回调函数，在回调函数中输出相关信息，实验源码如代码清单 9‑14加粗部分所示。

代码清单 9‑14软件定时器实验源码

1 /\*

2 \* @file main.c

3 \* @author fire

4 \* @version V1.0

5 \* @date 2018-xx-xx

6 \* @brief STM32全系列开发板-LiteOS！

7 \\*

8 \* @attention

9 \*

10 \* 实验平台:野火 F103-霸道 STM32 开发板

11 \* 论坛 :http://www.firebbs.cn

12 \* 淘宝 :http://firestm32.taobao.com

13 \*

14 \\*

15 \*/

16 /\* LiteOS 头文件 \*/

17 #include "los_sys.h"

18 #include "los_task.ph"

19 #include "los_swtmr.h"

20 /\* 板级外设头文件 \*/

21 #include "bsp_usart.h"

22 #include "bsp_led.h"

23 #include "bsp_key.h"

24

25 /\* 任务ID \/

26 /\*

27 \* 任务ID是一个从0开始的数字，用于索引任务，当任务创建完成之后，它就具有了一个任务ID

28 \* 以后要想操作这个任务都需要通过这个任务ID，

29 \*

30 \*/

31

**32 /\* 定义定时器ID变量*/**

**33 UINT16 Timer1_Handle;**

**34 UINT16 Timer2_Handle;**

35

36 /\* 内核对象ID \/

37 /\*

38 \* 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核

39 \* 对象，必须先创建，创建成功之后会返回一个相应的ID。实际上就是一个整数，后续

40 \* 就可以通过这个ID操作这些内核对象。

41 \*

42 \*

43 内核对象就是一种全局的数据结构，通过这些数据结构可以实现任务间的通信，

44 \* 任务间的事件同步等各种功能。至于这些功能的实现是通过调用这些内核对象的函数

45 \* 来完成的

46 \*

47 \*/

48

49 /\* 全局变量声明 \/

50 /\*

51 \* 在写应用程序的时候，可能需要用到一些全局变量。

52 \*/

53 static UINT32 TmrCb_Count1 = 0;

54 static UINT32 TmrCb_Count2 = 0;

55

56

57 /\* 函数声明 \*/

58 static UINT32 AppTaskCreate(void);

59 static void Timer1_Callback(UINT32 arg);

60 static void Timer2_Callback(UINT32 arg);

61

62 static void LED_Task(void);

63 static void Key_Task(void);

64 static void BSP_Init(void);

65

66

67 /\*

68 \* @brief 主函数

69 \* @param 无

70 \* @retval 无

71 \* @note 第一步：开发板硬件初始化

72 第二步：创建App应用任务

73 第三步：启动LiteOS，开始多任务调度，启动失败则输出错误信息

74 \/

75 int main(void)

76 {

77 //定义一个返回类型变量，初始化为LOS_OK

78 UINT32 uwRet = LOS_OK;

79

80 /\* 板载相关初始化 \*/

81 BSP_Init();

82

83 printf("这是一个[野火]-STM32全系列开发板-LiteOS软件定时器实验！\n\n");

84 printf("Timer1_Callback只执行一次就被销毁\n");

85 printf("Timer2_Callback则循环执行\n");

86

87 /\* LiteOS 内核初始化 \*/

88 uwRet = LOS_KernelInit();

89

90 if (uwRet != LOS_OK) {

91 printf("LiteOS 核心初始化失败！失败代码0x%X\n",uwRet);

92 return LOS_NOK;

93 }

94

95 /\* 创建App应用任务，所有的应用任务都可以放在这个函数里面 \*/

96 uwRet = AppTaskCreate();

97 if (uwRet != LOS_OK) {

98 printf("AppTaskCreate创建任务失败！失败代码0x%X\n",uwRet);

99 return LOS_NOK;

100 }

101

102 /\* 开启LiteOS任务调度 \*/

103 LOS_Start();

104

105 //正常情况下不会执行到这里

106 while (1);

107 }

108

109

110 /\*

111 \* @ 函数名 ： AppTaskCreate

112 \* @ 功能说明： 任务创建，为了方便管理，所有的任务创建函数都可以放在这个函数里面

113 \* @ 参数 ： 无

114 \* @ 返回值 ： 无

115 \/

116 static UINT32 AppTaskCreate(void)

117 {

118 /\* 定义一个返回类型变量，初始化为LOS_OK \*/

119 UINT32 uwRet = LOS_OK;

120

**121 /\* 创建一个软件定时器定时器*/**

**122 uwRet = LOS_SwtmrCreate(5000, /\* 软件定时器的定时时间*/**

**123 LOS_SWTMR_MODE_ONCE, /\* 软件定时器模式 一次模式 \*/**

**124 (SWTMR_PROC_FUNC)Timer1_Callback,/*软件定时器的回调函数 \*/**

**125 &Timer1_Handle, /\* 软件定时器的id \*/**

**126 0);**

**127 if (uwRet != LOS_OK) {**

**128 printf("软件定时器Timer1创建失败！\n");**

**129 }**

**130 uwRet = LOS_SwtmrCreate(1000, /\* 软件定时器的定时时间（Tick）*/**

**131 LOS_SWTMR_MODE_PERIOD,/\* 软件定时器模式 周期模式 \*/**

**132 (SWTMR_PROC_FUNC)Timer2_Callback,/\* 软件定时器的回调函数 \*/**

**133 &Timer2_Handle, /\* 软件定时器的id \*/**

**134 0);**

**135 if (uwRet != LOS_OK) {**

**136 printf("软件定时器Timer2创建失败！\n");**

**137 return uwRet;**

**138 }**

139

**140 /\* 启动一个软件定时器定时器*/**

**141 uwRet = LOS_SwtmrStart(Timer1_Handle);**

**142 if (LOS_OK != uwRet) {**

**143 printf("start Timer1 failed\n");**

**144 return uwRet;**

**145 } else {**

**146 printf("start Timer1 sucess\n");**

**147 }**

**148 /\* 启动一个软件定时器定时器*/**

**149 uwRet = LOS_SwtmrStart(Timer2_Handle);**

**150 if (LOS_OK != uwRet) {**

**151 printf("start Timer2 failed\n");**

**152 return uwRet;**

**153 } else {**

**154 printf("start Timer2 sucess\n");**

**155 }**

156

157 return LOS_OK;

158 }

159

160 /\*

161 \* @ 函数名 ： Timer1_Callback

162 \* @ 功能说明： 软件定时器回调函数1

163 \* @ 参数 ： 传入1个参数，但未使用

164 \* @ 返回值 ： 无

165 \/

**166 static void Timer1_Callback(UINT32 arg)**

**167 {**

**168 UINT32 tick_num1;**

**169**

**170 TmrCb_Count1++; /\* 每回调一次加一 \*/**

**171 LED1_TOGGLE;**

**172 tick_num1 = (UINT32)LOS_TickCountGet(); /\* 获取滴答定时器的计数值 \*/**

**173**

**174 printf("Timer_CallBack_Count1=%d\n", TmrCb_Count1);**

**175 printf("tick_num1=%d\n", tick_num1);**

**176 }**

177 /\*

178 \* @ 函数名 ： Timer2_Callback

179 \* @ 功能说明： 软件定时器回调函数2

180 \* @ 参数 ： 传入1个参数，但未使用

181 \* @ 返回值 ： 无

182 \/

**183 static void Timer2_Callback(UINT32 arg)**

**184 {**

**185 UINT32 tick_num2;**

**186**

**187 TmrCb_Count2++; /\* 每回调一次加一 \*/**

**188 LED2_TOGGLE;**

**189 tick_num2 = (UINT32)LOS_TickCountGet(); /\* 获取滴答定时器的计数值 \*/**

**190**

**191 printf("Timer_CallBack_Count2=%d\n", TmrCb_Count2);**

**192**

**193 printf("tick_num2=%d\n", tick_num2);**

**194**

**195 }**

196

197 /\*

198 \* @ 函数名 ： BSP_Init

199 \* @ 功能说明： 板级外设初始化，所有开发板上的初始化均可放在这个函数里面

200 \* @ 参数 ：

201 \* @ 返回值 ： 无

202 \/

203 static void BSP_Init(void)

204 {

205 /\*

206 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

207 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

208 \* 都统一用这个优先级分组，千万不要再分组，切忌。

209 \*/

210 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

211

212 /\* LED 初始化 \*/

213 LED_GPIO_Config();

214

215 /\* 串口初始化 \*/

216 USART_Config();

217

218 /\* 按键初始化 \*/

219 Key_GPIO_Config();

220 }

221

222

223 /END OF FILE/

实验现象
~~~~

程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据读者买的开发板而定，每个型号的开发板都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，在串口调试助手中可以看
到运行结果：每1000个Tick时候软件定时器就会触发一次回调函数，当5000个Tick到来的时候，触发软件定时器单次模式的回调函数，如图 9‑4所示。

|softwa005|

图 9‑4软件定时器实验现象

.. |softwa002| image:: media\softwa002.png
   :width: 5.76806in
   :height: 2.35764in
.. |softwa003| image:: media\softwa003.png
   :width: 5.20302in
   :height: 4.22388in
.. |softwa004| image:: media\softwa004.png
   :width: 5.22083in
   :height: 3.18611in
.. |softwa005| image:: media\softwa005.png
   :width: 5.65486in
   :height: 4.46806in

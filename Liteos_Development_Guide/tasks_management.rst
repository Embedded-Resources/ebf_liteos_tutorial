.. vim: syntax=rst

任务管理
----

任务管理是LiteOS的核心组成部分，本章主要介绍任务及调度器相关的概念，分析任务相关函数的实现过程。

基本概念
~~~~

任务的基本概念
^^^^^^^

从系统的角度看，任务是竞争系统资源的最小运行单元。LiteOS是一个支持多任务的操作系统。在LiteOS中，任务可以使用CPU、内存空间等系统资源，并独立于其他任务运行。理论上任何数量的任务都可以共享同一个优先级，处于就绪态的多个最高优先级任务将会以时间片切换的方式共享CPU。

简而言之： LiteOS的任务可认为是一系列独立任务的集合，每个任务在独立的环境中运行。在任何时刻，有且只有一个任务处于运行态，由LiteOS调度器决定哪个任务可以运行。调度器的主要职责是找到处于就绪态的最高优先级任务，并且在任务切入切出时保存任务上下文内容（寄存器值、任务栈数据），为了实现这点，
LiteOS的每个任务都需要有独立的任务栈，当任务切出时，它的上下文环境会被保存在任务栈中，当任务恢复运行时，就能从任务栈中正确的恢复上次的运行环境。系统的任务越多，整需要SRAM就越大，一个系统能够运行多少个任务，取决于系统的SRAM大小。

任务享有独立的栈空间，系统决定任务的状态，它表示任务是否可以运行，同时还能使用内核的IPC通信资源，实现中断与任务、任务与任务之间的通信。

LiteOS支持抢占式调度机制，高优先级的任务可打断低优先级任务，低优先级任务必须在高优先级任务阻塞或结束后才能得到调度，同时LiteOS也支持时间片轮转调度机制。

任务通常会运行在一个死循环中不会退出，如果一个任务不再需要运行时，可以调用LiteOS中的任务删除函数将其删除。

LiteOS的任务默认有32个优先级(0-31)，最高优先级为0，最低优先级为31。

任务调度器的基本概念
^^^^^^^^^^

LiteOS中提供的任务调度器是基于优先级的全抢占式调度：在系统中除了中断处理函数、调度器上锁和处于临界段中的情况是不可抢占之外，系统的其他部分都是可以抢占的。系统默认可以支持32个优先级(0～31)，优先级数值越大的任务优先级越低，31为最低优先级，分配给空闲任务使用，一般不建议用户来使用这个优先
级。在一些资源比较紧张的系统中，可以根据实际情况选择系统支持的任务优先级个数，创建合适的任务，以节约内存资源。在系统中，当有比当前任务优先级更高的任务就绪时，当前任务将立刻被切出，高优先级任务抢占处理器运行。

LiteOS内核中也允许创建相同优先级的任务，相同优先级的任务采用时间片轮转方式进行调度（也就是通常说的分时调度器），时间片轮转调度仅在当前系统有多个最高优先级就绪任务存在的情况下才有效。为了保证系统的实时性，系统尽最大可能地保证高优先级的任务得以运行，任务调度的原则是一旦任务状态发生了改变，并且当
前运行的任务优先级小于就绪列表中最高优先级任务时，立刻进行任务切换（除非当前系统处于中断处理程序中或禁止任务切换的状态）。

任务状态的概念
^^^^^^^

LiteOS系统中的任务都有多种运行状态。系统初始化完成后，创建的任务就可以在系统中竞争资源，由内核进行调度。

任务状态通常分为以下四种。

1. 就绪态（Ready）：该任务处于就绪列表中，就绪的任务已经具备执行的能力，只等待调度器进行调度，新创建的任务会被初始化为该状态。

2. 运行态（Running）：该状态表明任务正在执行，此时它占用处理器，LiteOS调度器选择运行的永远是处于最高优先级的就绪态任务，当任务被运行的一刻，它的任务状态就变成了运行态。

3. 阻塞态（Blocked）：如果任务当前正在等待某个时序或外部中断，那么该任务处于阻塞状态，它不处于就绪列表中，无法得到调度器的调度。阻塞态包含任务被挂起、任务被延时、任务正在等待信号量、读写队列或者等待读写事件等。

4. 退出态（Dead）：该任务运行结束，等待系统回收资源。

任务状态迁移
^^^^^^

LiteOS系统中的每一个任务都有多种运行状态，它们之间的转换关系是怎么样的呢？从运行态任务变成阻塞态，或者从阻塞态变成就绪态，这些任务状态是怎么迁移的呢，下面就一起了解任务状态迁移吧，如图 4‑1所示。

|tasksm002|

图 4‑1任务状态示意图

创建任务→就绪态：任务创建完成后进入就绪态，表明任务已准备就绪，随时可以运行，只等待调度器进行调度。

就绪态→运行态：任务创建后进入就绪态，发生任务切换时，就绪列表中最高优先级的任务被执行，从而进入运行态，但此刻该任务依旧在就绪列表中。

运行态→就绪态：有更高优先级任务创建或者恢复后，会发生任务调度，此刻就绪列表中最高优先级任务变为运行态，而原先运行的任务由运行态变为就绪态，仍处于就绪列表中，等待最高优先级的任务运行完毕后继续运行（CPU使用权被更高优先级的任务抢占了）。

运行态→阻塞态：正在运行的任务发生阻塞（挂起、延时、读信号量等待）时，该任务会从就绪列表中删除。任务状态由运行态变成阻塞态，然后发生任务切换，系统运行就绪列表中最高优先级任务。

阻塞态→就绪态（阻塞态→运行态）：阻塞的任务被恢复后（任务恢复、延时时间超时、读信号量超时或读到信号量等），此时被恢复的任务会被加入就绪列表，从而由阻塞态变成就绪态；如果此时被恢复任务的优先级高于正在运行任务的优先级，则会发生任务切换，将该任务将再次转换任务状态，由就绪态变成运行态。

就绪态→阻塞态：任务也有可能在就绪态时被阻塞（挂起），此时任务状态会由就绪态变为阻塞态，该任务从就绪列表中删除，不会参与系统调度，直到该任务被恢复就绪态。

运行态、阻塞态→退出态：调用系统中删除任务的函数，无论是处于何种状态的任务都将变为退出态。

常用的任务函数讲解
~~~~~~~~~

任务创建函数LOS_TaskCreate()
^^^^^^^^^^^^^^^^^^^^^^

在前面的章节中，本书已经讲解了任务创建函数的使用，而未分析LOS_TaskCreate()的源码，那么LiteOS中任务创建函数LOS_TaskCreate()是如何实现的呢？如代码清单 4‑1所示。

代码清单 4‑1任务创建函数LOS_TaskCreate()源码

1 LITE_OS_SEC_TEXT_INIT UINT32 LOS_TaskCreate(UINT32 \*puwTaskID,

2 TSK_INIT_PARAM_S \*pstInitParam){

3 UINT32 uwRet = LOS_OK;

4 UINTPTR uvIntSave;

5 LOS_TASK_CB \*pstTaskCB; **(1)**

6

7 uwRet = LOS_TaskCreateOnly(puwTaskID, pstInitParam); **(2)**

8 if (LOS_OK != uwRet) {

9 return uwRet;

10 }

11 pstTaskCB = OS_TCB_FROM_TID(*puwTaskID); **(3)**

12

13 uvIntSave = LOS_IntLock();

14 pstTaskCB->usTaskStatus &= (~OS_TASK_STATUS_SUSPEND);

15 pstTaskCB->usTaskStatus \|= OS_TASK_STATUS_READY; **(4)**

16

17 #if (LOSCFG_BASE_CORE_CPUP == YES)

18 g_pstCpup[pstTaskCB->uwTaskID].uwID = pstTaskCB->uwTaskID;

19 g_pstCpup[pstTaskCB->uwTaskID].usStatus = pstTaskCB->usTaskStatus;

20 #endif

21

22 osPriqueueEnqueue(&pstTaskCB->stPendList, pstTaskCB->usPriority); **(5)**

23 g_stLosTask.pstNewTask = LOS_DL_LIST_ENTRY(osPriqueueTop(),

24 LOS_TASK_CB, stPendList);

25 if ((g_bTaskScheduled) && (g_usLosTaskLock == 0)) {

26 if (g_stLosTask.pstRunTask != g_stLosTask.pstNewTask) { **(6)**

27 if (LOS_CHECK_SCHEDULE) {

28 (VOID)LOS_IntRestore(uvIntSave);

29 osSchedule(); **(7)**

30 return LOS_OK;

31 }

32 }

33 }

34

35 (VOID)LOS_IntRestore(uvIntSave);

36 return LOS_OK; **(8)**

37 }

代码清单 4‑1\ **(1)**\ ：定义一个新创建任务的任务控制块结构体指针，用于保存新创建任务的任务信息。

代码清单 4‑1\ **(2)**\ ：调用 LOS_TaskCreateOnly()函数进行任务的创建并且阻塞任务，该函数仅创建任务，而不配置任务状态信息，参数puwTaskID是任务的ID的指针，指向用户定义任务ID变量的地址，在创建任务成功后将通过该指针返回一个任务ID给用户，任务配置与pst
InitParam一致，在创建新任务时，会对之前已删除任务的任务控制块和任务栈进行回收。

代码清单 4‑1\ **(3)**\ ：通过任务ID获取对应任务控制块的信息。

代码清单 4‑1\ **(4)**\ ：将新创建的任务从阻塞态中解除，然后将任务状态设置为就绪态，这步操作之后任务状态由新创建的阻塞态变为就绪态（Ready），表明任务可以参与系统调度。

代码清单 4‑1\ **(5)**\ ：首先获取新创建任务的优先级，并且将任务按照优先级顺序插入任务就绪列表。

代码清单 4‑1\ **(6)**\ ：如果开启了任务调度，并且调度器没有被上锁，则进行第二次判断：如果新建的任务优先级比当前的任务优先级更高，则进行一次任务调度，否则将返回任务创建成功\ **(8)**\ 。

代码清单 4‑1\ **(7)**\ ：如果满足了\ **(6)** 中的条件，则进行任务的调度，任务的调度是用汇编代码实现的，如代码清单 4‑2 所示，然后返回任务创建成功。

代码清单 4‑2 LiteOS任务调度的实现

1 OS_NVIC_INT_CTRL EQU 0xE000ED04

2 OS_NVIC_PENDSVSET EQU 0x10000000

3

4 osTaskSchedule

5 LDR R0, =OS_NVIC_INT_CTRL

6 LDR R1, =OS_NVIC_PENDSVSET

7 STR R1, [R0]

8 BX LR

在Cortex-M系列处理器中，LiteOS的调度是利用PendSV进行任务调度的，LiteOS向0xE000ED04这个地址写入0x10000000，即将SCB寄存器的第28位置1，触发PendSV中断，真正的任务切换是在PendSV中断中进行的，如图 4‑2所示。

|tasksm003|

图 4‑2任务调度将PendSV置1

任务删除函数LOS_TaskDelete()
^^^^^^^^^^^^^^^^^^^^^^

在LiteOS中支持显式删除任务，当任务不需要的时候，可以删除它，例如，在“小心翼翼，十分谨慎”法启动流程中，就是对启动任务进行了删除操作，因为系统只需要运行一次该任务，删除任务后，LiteOS会回收任务的相关资源，任务删除的实现过程如代码清单 4‑3所示。

代码清单 4‑3任务删除函数 LOS_TaskDelete()源码

1 LITE_OS_SEC_TEXT_INIT UINT32 LOS_TaskDelete(UINT32 uwTaskID)

2 {

3 UINTPTR uvIntSave;

4 LOS_TASK_CB \*pstTaskCB;

5 UINT16 usTempStatus;

6 UINT32 uwErrRet = OS_ERROR;

7

8 CHECK_TASKID(uwTaskID);

9 uvIntSave = LOS_IntLock();

10

11 pstTaskCB = OS_TCB_FROM_TID(uwTaskID);

12

13 usTempStatus = pstTaskCB->usTaskStatus;

14

15 if (OS_TASK_STATUS_UNUSED & usTempStatus) { **(1)**

16 uwErrRet = LOS_ERRNO_TSK_NOT_CREATED;

17 OS_GOTO_ERREND();

18 }

19

20 if ((OS_TASK_STATUS_RUNNING & usTempStatus)

21 && (g_usLosTaskLock != 0)) { **(2)**

22 PRINT_INFO("In case of task lock,task deletion is not recommended\n");

23 g_usLosTaskLock = 0;

24 }

25

26 if (OS_TASK_STATUS_READY & usTempStatus) { **(3)**

27 osPriqueueDequeue(&pstTaskCB->stPendList);

28 pstTaskCB->usTaskStatus &= (~OS_TASK_STATUS_READY);

29 } else if ((OS_TASK_STATUS_PEND & usTempStatus)

30 \|\| (OS_TASK_STATUS_PEND_QUEUE & usTempStatus)) {

31 LOS_ListDelete(&pstTaskCB->stPendList); **(4)**

32 }

33 if ((OS_TASK_STATUS_DELAY \| OS_TASK_STATUS_TIMEOUT) & usTempStatus) {

34 osTimerListDelete(pstTaskCB); **(5)**

35 }

36

37 pstTaskCB->usTaskStatus &= (~(OS_TASK_STATUS_SUSPEND));

38 pstTaskCB->usTaskStatus \|= OS_TASK_STATUS_UNUSED;

39 pstTaskCB->uwEvent.uwEventID = 0xFFFFFFFF;

40 pstTaskCB->uwEventMask = 0;

41

42 g_stLosTask.pstNewTask = LOS_DL_LIST_ENTRY(osPriqueueTop(),

43 LOS_TASK_CB, stPendList); **(6)**

44

45 if (OS_TASK_STATUS_RUNNING & pstTaskCB->usTaskStatus) { **(7)**

46 LOS_ListTailInsert(&g_stTskRecyleList, &pstTaskCB->stPendList);

47 g_stLosTask.pstRunTask = &g_pstTaskCBArray[g_uwTskMaxNum];

48 g_stLosTask.pstRunTask->uwTaskID = uwTaskID;

49 g_stLosTask.pstRunTask->usTaskStatus = pstTaskCB->usTaskStatus;

50 g_stLosTask.pstRunTask->uwTopOfStack = pstTaskCB->uwTopOfStack;

51 g_stLosTask.pstRunTask->pcTaskName = pstTaskCB->pcTaskName;

52 pstTaskCB->usTaskStatus = OS_TASK_STATUS_UNUSED;

53 (VOID)LOS_IntRestore(uvIntSave);

54 osSchedule();

55 return LOS_OK;

56 } else {

57 pstTaskCB->usTaskStatus = OS_TASK_STATUS_UNUSED; **(8)**

58 LOS_ListAdd(&g_stLosFreeTask, &pstTaskCB->stPendList); **(9)**

59 (VOID)LOS_MemFree(m_aucSysMem0, (VOID \*)pstTaskCB->uwTopOfStack);\ **(10)**

60 pstTaskCB->uwTopOfStack = (UINT32)NULL; **(11)**

61 }

62

63 (VOID)LOS_IntRestore(uvIntSave);

64 return LOS_OK; **(12)**

65

66 LOS_ERREND:

67 (VOID)LOS_IntRestore(uvIntSave);

68 return uwErrRet; **(13)**

69 }

代码清单 4‑3\ **(1)**\ ：如果要删除的任务的任务状态是OS_TASK_STATUS_UNUSED，表示任务尚未创建，系统无法删除，将返回错误代码LOS_ERRNO_TSK_NOT_CREATED。

代码清单 4‑3\ **(2)**\ ：如果要删除的任务正在运行且调度器已经被上锁，系统会将任务解锁，g_usLosTaskLock 被设置为0，然后接着进行删除操作。

代码清单 4‑3\ **(3)**\ ：如果要删除的任务在就绪态，那么LiteOS会将要删除的任务从就绪列表中移除，并且取消任务的就绪状态。

代码清单 4‑3\ **(4)**\ ：如果要删除的任务在阻塞态或者任务在队列中被阻塞，那么LiteOS会将要删除的任务从阻塞列表中删除。

代码清单 4‑3\ **(5)**\ ：如果要删除的任务正在处于延时状态或者任务正在等待信号量/事件等阻塞超时状态，那么LiteOS将从延时列表中删除任务。

代码清单 4‑3\ **(6)**\ ：系统重新在就绪列表中寻找处于就绪态的最高优先级任务，保证系统能正常运行，因为如果删除的任务是下一个即将要切换的任务，那么删除之后系统将无法正常进行任务切换。

代码清单 4‑3\ **(7)**\ ：如果删除的任务是当前正在运行的任务，因为删除任务以后要调度新的任务运行，而调度的过程需要当前任务的参与，所以还不能直接将当前任务彻底删除掉，只是将任务添加到系统的回收列表中（g_stTskRecyleList），在创建任务的时候将回收列表中的任务进行回收，而当
前任务需要继续执行，直到系统调度完成，就完成了当前任务的使命。

代码清单 4‑3\ **(8)**\ ：如果被删除的任务不是当前任务，那么直接将任务状态变为未使用状态。

代码清单 4‑3\ **(9)**\ ：将任务控制块插入系统可用任务链表中，为了以后能再创建任务，系统支持的任务个数是有限的，当删除了一个任务之后，就要归还，否则当系统可用任务链表中没有可用的任务控制块，那么就不能创建任务了，因为任务控制块的内存控制在系统初始化的时候就已经分配了。

代码清单 4‑3\ **(10)**\ ：将任务控制块的内存进行释放，回收利用。

代码清单 4‑3\ **(11)**\ ：将任务的栈顶指针指向NULL。

代码清单 4‑3\ **(12)-(13)**\ ：如果删除成功则返回LOS_OK，否则将返回错误代码。

任务延时函数LOS_TaskDelay()
^^^^^^^^^^^^^^^^^^^^^

延时函数是在使用操作系统的时候是经常用到的函数，延时函数的作用是将调用延时函数的任务进入阻塞态而放弃CPU 的使用权，这样子系统中其他任务优先级较低的任务就能完成获得CPU的使用权。否则的话，高优先级任务一直占用CPU，导致系统无法进行任务切换，比它优先级低的任务将永远得不到运行，延时的基本单位为T
ick，配置LOSCFG_BASE_CORE_TICK_PER_SECOND宏定义即可改变系统节拍，如果LOSCFG_BASE_CORE_TICK_PER_SECOND配置为1000，那么一个Tick为1ms，延时函数的实现方式如代码清单 4‑4所示。

代码清单 4‑4 任务延时函数LOS_TaskDelay()源码

1 LITE_OS_SEC_TEXT UINT32 LOS_TaskDelay(UINT32 uwTick)

2 {

3 UINTPTR uvIntSave;

4

5 if (OS_INT_ACTIVE) { **(1)**

6 return LOS_ERRNO_TSK_DELAY_IN_INT;

7 }

8

9 if (g_usLosTaskLock != 0) { **(2)**

10 return LOS_ERRNO_TSK_DELAY_IN_LOCK;

11 }

12

13 if (uwTick == 0) { **(3)**

14 return LOS_TaskYield();

15 } else {

16 uvIntSave = LOS_IntLock();

17 osPriqueueDequeue(&(g_stLosTask.pstRunTask->stPendList)); **(4)**

18 g_stLosTask.pstRunTask->usTaskStatus &= (~OS_TASK_STATUS_READY);

19 osTaskAdd2TimerList((LOS_TASK_CB \*)g_stLosTask.pstRunTask,uwTick);

20 g_stLosTask.pstRunTask->usTaskStatus \|= OS_TASK_STATUS_DELAY;

21 (VOID)LOS_IntRestore(uvIntSave);

22 LOS_Schedule(); **(5)**

23 }

24

25 return LOS_OK;

26 }

代码清单 4‑4\ **(1)**\ ：如果在中断中进行延时，这将是非法的，LiteOS会返回错误代码，因为LiteOS不允许在中断中调用延时操作。

代码清单 4‑4\ **(2)**\ ：如果在调度器被锁定时进行延时，这也是非法的，因为延时操作需要依赖调度器的调度， 因此LiteOS也会返回错误代码。

代码清单 4‑4\ **(3)**\ ：如果要进行0个Tick的延时，那么当前任务将主动放弃CPU的使用权，进行一次强制切换任务。

代码清单 4‑4\ **(4)-(5)**\ ：如果任务可以进行延时，
LiteOS将调用延时函数的任务从就绪列表中删除，同时将该任务的任务状态从就绪态中解除；然后将该任务添加到延时链表中，最后将任务的状态变为延时状态（阻塞态），当延时的时间到达，任务将从阻塞态直接变为就绪态，最后，LiteOS进行一次任务的切换，再返回LOS_OK表示延时成功。

注意，在每个任务的循环中必须要有阻塞的出现，否则，比该任务优先级低的任务是永远无法获得CPU的使用权的。

任务挂起函数LOS_TaskSuspend()
^^^^^^^^^^^^^^^^^^^^^^^

LiteOS支持挂起指定任务，被挂起的任务不会得到CPU使用权，不管该任务具有什么优先级。

调用LOS_TaskSuspend()函数挂起任务的次数是不会累计的：即使多次调用LOS_TaskSuspend()函数将一个任务挂起，也只需调用一次任务恢复函数LOS_TaskResume()就能使挂起的任务解除挂起状态。任务挂起是经常使用的一个函数，如果读者想要某个任务长时间不需要执行的时候，就
可以使用LOS_TaskSuspend()函数将该任务挂起，任务挂起函数的源码实现如代码清单 4‑5所示。

代码清单 4‑5任务挂起函数LOS_TaskSuspend()源码

1 LITE_OS_SEC_TEXT_INIT UINT32 LOS_TaskSuspend(UINT32 uwTaskID)

2 {

3 UINTPTR uvIntSave;

4 LOS_TASK_CB \*pstTaskCB;

5 UINT16 usTempStatus;

6 UINT32 uwErrRet = OS_ERROR;

7

8 CHECK_TASKID(uwTaskID);

9 pstTaskCB = OS_TCB_FROM_TID(uwTaskID); **(1)**

10 uvIntSave = LOS_IntLock();

11 usTempStatus = pstTaskCB->usTaskStatus;

12 if (OS_TASK_STATUS_UNUSED & usTempStatus) { **(2)**

13 uwErrRet = LOS_ERRNO_TSK_NOT_CREATED;

14 OS_GOTO_ERREND();

15 }

16

17 if (OS_TASK_STATUS_SUSPEND & usTempStatus) { **(3)**

18 uwErrRet = LOS_ERRNO_TSK_ALREADY_SUSPENDED;

19 OS_GOTO_ERREND();

20 }

21

22 if((OS_TASK_STATUS_RUNNING & usTempStatus)&&(g_usLosTaskLock != 0)) {

23 uwErrRet = LOS_ERRNO_TSK_SUSPEND_LOCKED; **(4)**

24 OS_GOTO_ERREND();

25 }

26

27 if (OS_TASK_STATUS_READY & usTempStatus) { **(5)**

28 osPriqueueDequeue(&pstTaskCB->stPendList); **(6)**

29 pstTaskCB->usTaskStatus &= (~OS_TASK_STATUS_READY); **(7)**

30 }

31

32 pstTaskCB->usTaskStatus \|= OS_TASK_STATUS_SUSPEND; **(8)**

33 if (uwTaskID == g_stLosTask.pstRunTask->uwTaskID) {

34 (VOID)LOS_IntRestore(uvIntSave);

35 LOS_Schedule(); **(9)**

36 return LOS_OK;

37 }

38

39 (VOID)LOS_IntRestore(uvIntSave);

40 return LOS_OK;

41

42 LOS_ERREND:

43 (VOID)LOS_IntRestore(uvIntSave);

44 return uwErrRet;

45 }

代码清单 4‑5\ **(1)**\ ：根据任务ID获取对应的任务控制块。

代码清单 4‑5\ **(2)**\ ：判断要挂起任务的状态，如果是未使用状态，就返回错误代码。

代码清单 4‑5\ **(3)**\ ：判断要挂起任务的状态，如果该任务已经被挂起了，会返回错误代码，用户可以在恢复任务后再挂起。

代码清单 4‑5\ **(4)**\ ：如果任务运行中并且调度器已经被上锁了，那么也无法进行挂起任务，返回错误代码。

代码清单 4‑5\ **(5)**\ ：如果任务处于就绪态，则可以进行挂起任务。

代码清单 4‑5\ **(6)**\ ：将任务从就绪列表中删除。

代码清单 4‑5\ **(7)**\ ：将任务从就绪态中解除。

代码清单 4‑5\ **(8)**\ ：将任务的状态变为挂起态。

代码清单 4‑5\ **(9)**\ ：进行一次任务调度。

任务恢复函数LOS_TaskResume()
^^^^^^^^^^^^^^^^^^^^^^

任务恢复就是让挂起的任务重新进入就绪状态，恢复的任务会保留挂起前的状态信息，在恢复的时候继续运行。如果被恢复任务在所有就绪态任务中，处于系统中的最高优先级，那么系统将进行一次任务切换。任务恢复函数LOS_TaskResume()的源码实现如代码清单 4‑6所示。

代码清单 4‑6任务恢复函数LOS_TaskResume()源码

1 LITE_OS_SEC_TEXT_INIT UINT32 LOS_TaskResume(UINT32 uwTaskID)

2 {

3 UINTPTR uvIntSave;

4 LOS_TASK_CB \*pstTaskCB;

5 UINT16 usTempStatus;

6 UINT32 uwErrRet = OS_ERROR;

7

8 if (uwTaskID > LOSCFG_BASE_CORE_TSK_LIMIT) { **(1)**

9 return LOS_ERRNO_TSK_ID_INVALID;

10 }

11

12 pstTaskCB = OS_TCB_FROM_TID(uwTaskID); **(2)**

13 uvIntSave = LOS_IntLock();

14 usTempStatus = pstTaskCB->usTaskStatus;

15

16 if (OS_TASK_STATUS_UNUSED & usTempStatus) { **(3)**

17 uwErrRet = LOS_ERRNO_TSK_NOT_CREATED;

18 OS_GOTO_ERREND();

19 } else if (!(OS_TASK_STATUS_SUSPEND & usTempStatus)) { **(4)**

20 uwErrRet = LOS_ERRNO_TSK_NOT_SUSPENDED;

21 OS_GOTO_ERREND();

22 }

23

24 pstTaskCB->usTaskStatus &= (~OS_TASK_STATUS_SUSPEND); **(5)**

25 if (!(OS_CHECK_TASK_BLOCK & pstTaskCB->usTaskStatus) ) {

26 pstTaskCB->usTaskStatus \|= OS_TASK_STATUS_READY; **(6)**

27 osPriqueueEnqueue(&pstTaskCB->stPendList, pstTaskCB->usPriority);

28 if (g_bTaskScheduled) { **(7)**

29 (VOID)LOS_IntRestore(uvIntSave);

30 LOS_Schedule(); **(8)**

31 return LOS_OK;

32 }

33 g_stLosTask.pstNewTask = LOS_DL_LIST_ENTRY(osPriqueueTop(),

34 LOS_TASK_CB, stPendList);

35 }

36 (VOID)LOS_IntRestore(uvIntSave);

37 return LOS_OK;

38

39 LOS_ERREND:

40 (VOID)LOS_IntRestore(uvIntSave);

41 return uwErrRet;

42 }

代码清单 4‑6\ **(1)**\ ：判断任务ID是否有效，如果无效则返回错误代码。

代码清单 4‑6\ **(2)**\ ：根据任务ID获取任务控制块。

代码清单 4‑6\ **(3)**\ ：判断要恢复任务的状态，如果是未使用状态，返回错误代码。

代码清单 4‑6\ **(4)**\ ：判断要恢复任务的状态，如果是未挂起状态，那就无需恢复了，也会返回错误代码。

代码清单 4‑6\ **(5)**\ ：经过前面的代码的判断，可以确认任务已经是挂起的，那么可以恢复任务，将任务的状态从阻塞态解除。

代码清单 4‑6\ **(6)**\ ：将任务状态变成就绪态。

代码清单 4‑6\ **(7)**\ ：将任务按照本身的优先级数值添加到就绪列表中。

代码清单 4‑6\ **(8)**\ ：如果调度器已经运行了，则发起一次任务调度，在任务调度中会寻找处于就绪态的最高优先级任务，如果被恢复的任务刚好是就绪态任务中的最高优先级，那么系统会立即运行该任务。

常用Task错误代码说明
~~~~~~~~~~~~

在LiteOS中，与任务相关的函数大多数都会有返回值，其返回值是一些错误代码，方便用户进行调试，本书将列出一些常见的错误代码与参考解决方案，如表 4‑1所示。

表 4‑1常用Task函数返回的错误代码说明

.. list-table::
   :widths: 25 25 25 25
   :header-rows: 0


   * - 序号 |
     - 义              | 描述
     - | 参考解决
     - 案      |

   * - 1
     - LOS_ER RNO_TSK_NO_MEMORY
     - 内存空间不足      | 分配更大
     - 内存    |

   * - 2
     - LOS_E RRNO_TSK_PTR_NULL
     - 任务参数为空      | 检查任务
     - 数      |

   * - 3
     - LOS_ERRNO_TS K_STKSZ_NOT_ALIGN
     - 任务栈未对齐      | 对齐任务
     - |

   * - 4
     - LOS_ERRN O_TSK_PRIOR_ERROR
     - 不                | 正确的任务优先级  |
     - 检查任务优先级    | |

   * - 5
     - LOS_ERR NO_TSK_ENTRY_NULL
     - 任务入口函数为空  | 定义任务入口
     - 数  |

   * - 6
     - LOS_ERR NO_TSK_NAME_EMPTY
     - 任务名为空        | 设置任
     - 名        |

   * - 7
     - LOS_ERRNO_TS K_STKSZ_TOO_SMALL
     - 任务栈太小        | 扩大任
     - 栈        |

   * - 8
     - LOS_ERR NO_TSK_ID_INVALID
     - 无效的任务ID      | 检查任
     - ID        |

   * - 9
     - LOS_ERRNO_TSK_ ALREADY_SUSPENDED
     - 任务已经被挂起    | 等待这个任
     - |
       再去  |
       任务  |

   * - 10
     - LOS_ERRNO_ TSK_NOT_SUSPENDED
     - 任务未被挂起      | 挂起这个
     - 务      |

   * - 11
     - LOS_ERRN O_TSK_NOT_CREATED
     - 任务未被创建      | 创建这个
     - 务      |

   * - 12
     - LOS_ERRNO_ TSK_DELETE_LOCKED
     - 删除任务时，      | 等待解锁 任务处于被锁状态  | 后再进行删除
     - 务之    | 作  |

   * - 13
     - LOS_ERRN O_TSK_MSG_NONZERO
     - 任务信息非零      | 暂
     - |

   * - 14
     - LOS_ERRNO _TSK_DELAY_IN_INT
     - 中断期            | 等 间，进行任务延时  | 后再进行延时
     - 退出中断      | 作  |

   * - 15
     - LOS_ERRNO_ TSK_DELAY_IN_LOCK
     - 任务被锁的        | 等待解 状态下，进行延时  | 后再进行延时
     - 任务之    | 作  |

   * - 16
     - LOS_ERRNO_TSK_Y IELD_INVALID_TASK
     - 将被排入行        | 检查这 程的任务是无效的  |
     - 任务      | |

   * - 17
     - L OS_ERRNO_TSK_YIEL D_NOT_ENOUGH_TASK
     - 没有或            | 增 者仅有一个可用任  | 务能进行行程安排  |
     - 任务数        | | |

   * - 18
     - LOS_ERRNO_TS K_TCB_UNAVAILABLE
     - 没有空闲          | 增 的任务控制块可用  | 加任务控制块
     - |

   * - 19
     - LOS_ERRNO_T SK_HOOK_NOT_MATCH
     - 任务              | 的钩子函数不匹配  | 不使用该错误
     - |

   * - 20
     - LOS_ERRNO _TSK_HOOK_IS_FULL
     - 任务的钩子        | 暂 函数数量超过界限  | 不使用该错误
     - |

   * - 21
     - LOS_ERRNO _TSK_OPERATE_IDLE
     - 这是个IDLE任务    | 检查任
     - ID，不要  | 试图操作IDLE任务  |

   * - 22
     - LOS_ERRNO_T SK_SUSPEND_LOCKED
     - 将被挂起的        | 等待任 任务处于被锁状态  | 后再尝试挂起
     - 解锁      | 务  |

   * - 23
     - LOS_ERRNO_TSK_ FREE_STACK_FAILED
     - 任务栈free失败    | 该
     - |

   * - 24
     - LOS_ERRNO_TSK_ STKAREA_TOO_SMALL
     - 任务栈区域太小    | 该
     - |
       |

   * - 25
     - LOS_ERRNO_ TSK_ACTIVE_FAILED
     - 任务触发失败      | 创建一个
     - DLE任    | 务后执行任务转换  |

   * - 26
     - LOS_ERRNO_TS K_CONFIG_TOO_MANY
     - 过多的任务配置项  | 该
     - |
        |

   * - 27
     - LOS_ERRNO_TS K_STKSZ_TOO_LARGE
     - 任                | 务栈大小设置过大  |
     - 减小任务栈大小    | |

   * - 28
     - LOS_E RRNO_TSK_SUSPEND_ SWTMR_NOT_ALLOWED
     - 不允许挂          | 检查 起软件定时器任务  | 不要试图挂
     - 务ID,       | | 起软件定时器任务  |


常用任务函数的使用方法
~~~~~~~~~~~

.. _任务创建函数los_taskcreate-1:

任务创建函数LOS_TaskCreate()
^^^^^^^^^^^^^^^^^^^^^^

LOS_TaskCreate()函数原型如代码清单 4‑7所示。创建任务函数是创建每个独立任务的时候是必须使用的，在使用函数的时候，需要提前定义任务ID变量，并且要自定义实现任务创建的pstInitParam，如代码清单
4‑8加粗部分所示。如果任务创建成功，则返回LOS_OK，否则返回对应的错误代码。

代码清单 4‑7LOS_TaskCreate()函数原型

1 UINT32 LOS_TaskCreate(UINT32 \*puwTaskID, TSK_INIT_PARAM_S \*pstInitParam);

代码清单 4‑8自定义实现任务的相关配置

1 UINT32 Test1_Task_Handle; /\* 定义任务ID变量 \*/

**2 TSK_INIT_PARAM_S task_init_param; /\* 自定义任务配置的相关参数 \*/**

**3**

**4 task_init_param.usTaskPrio = 5; /\* 优先级，数值越小，优先级越高 \*/**

**5 task_init_param.pcName = "Test1_Task"; /\* 任务名，字符串形式，方便调试 \*/**

**6 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)Test1_Task; /\* 任务函数名 \*/**

**7 task_init_param.uwStackSize = 0x1000; /\* 栈大小，单位为字，即4个字节 \*/**

8

**9 uwRet = LOS_TaskCreate(&Test1_Task_Handle, &task_init_param);/\* 创建任务 \*/**

自定义任务配置的TSK_INIT_PARAM_S结构体在los_task.h中，其内部的配置参数具体作用如代码清单
4‑9所示，读者可以根据自己的任务需要来配置，重要的任务优先级可以设置高一点，任务栈可以设置大一点，防止溢出导致系统崩溃，若指定的任务栈大小为0，则系统使用配置项LOSCFG_BASE_CORE_TSK_DEFAULT_STACK_SIZE指定默认的任务栈大小，任务栈的大小按8字节大小对齐。

代码清单 4‑9 TSK_INIT_PARAM_S结构体

1 typedef struct tagTskInitParam {

2 TSK_ENTRY_FUNC pfnTaskEntry; /**< 任务的入口函数 \*/

3 UINT16 usTaskPrio; /**< 任务优先级 \*/

4 UINT32 uwArg; /**< 任务参数（未使用） \*/

5 UINT32 uwStackSize; /**< 任务栈大小 \*/

6 CHAR \*pcName; /**< 任务名字 \*/

7 UINT32 uwResved; /**< LiteOS保留未使用 \*/

8 } TSK_INIT_PARAM_S;

.. _任务删除函数los_taskdelete-1:

任务删除函数LOS_TaskDelete()
^^^^^^^^^^^^^^^^^^^^^^

任务删除函数是根据任务ID直接删除任务，任务控制块与任务栈将被系统回收，所有保存的信息都会被清空。uwTaskID是LOS_TaskDelete()传入的任务ID，表示的是要删除哪个任务，如代码清单 4‑10所示。

代码清单 4‑10任务删除函数LOS_TaskDelete()原型

1 /\*

2 功能：LOS_TaskDelete

3 描述：删除任务

4 输入：uwTaskID ---任务ID

5 输出：无

6 返回：LOS_OK成功或失败时出现错误代码

7 \/

8 LITE_OS_SEC_TEXT_INIT UINT32 LOS_TaskDelete(UINT32 uwTaskID)

任务删除函数的实例：如代码清单 4‑11加粗部分所示，如果任务删除成功，则返回LOS_OK，否则返回其他错误代码。

代码清单 4‑11 任务删除函数的用法

1 UINT32 uwRet = LOS_OK;/\* 定义一个任务的返回类型，初始化为LOS_OK \*/

2

**3 uwRet = LOS_TaskDelete(Test_Task_Handle)**

4 if (uwRet != LOS_OK)

5 {

6 printf("任务删除失败\n");

7 }

.. _任务延时函数los_taskdelay-1:

任务延时函数LOS_TaskDelay()
^^^^^^^^^^^^^^^^^^^^^

任务延时函数只有一个传入的参数uwTick，它的延时单位是Tick，支持传入0个Tick。读者根据实际情况对任务进行延时即可，其函数原型如代码清单 4‑12所示。

代码清单 4‑12延时函数任务原型

1 extern UINT32 LOS_TaskDelay(UINT32 uwTick);

任务延时函数有几点需要注意的地方，第一点：延时函数不允许在中断中使用；第二点：延时函数不允许在任务调度被锁定的时候使用；第三点：如果传入0并且未锁定任务调度，则执行具有当前任务相同优先级的任务队列中的下一个任务，如果没有当前任务优先级的就绪任务可用，则不会发生任务调度，并继续执行当前任务；第四点：不
允许在系统初始化之前使用该函数；第五点：延时函数也是有返回值的，如果使用时候发生错误，可以根据返回的错误代码来进行调整；第六点：这种延时并不精确。任务延时函数的使用方法如代码清单 4‑13加粗部分所示。

代码清单 4‑13延时函数的使用方法

1 static void Test1_Task(void)

2 {

3 /\* 每个任务都是无限循环 \*/

4 while (1) {

5 LED2_TOGGLE; //LED2翻转

**6 LOS_TaskDelay(1000); //1000个Tick 延时**

7 }

8 }

任务挂起与恢复函数
^^^^^^^^^

任务的挂起与恢复函数在很多时候都是很有用的，比如想长时间暂停运行某个任务，但是又需要在其恢复的时候继续工作，那么是不可能删除任务的，因为删除了任务的话，任务的所有的信息都是不可能恢复的。但是可以使用挂起任务函数，仅仅是将任务进入阻塞态，其内部的资源都会保留在任务栈中，同时也不会参与任务的调度，当调用
恢复函数的时候，整个任务立即从阻塞态进入就绪态，参与任务的调度，如果该任务的优先级是当前就绪态优先级最高的任务，那么系统立即会进行一次任务切换，而恢复的任务将按照挂起前的任务状态继续运行，从而达到需要的效果，注意，是继续运行，也就是说，挂起任务之前的任务状态信息，都会被系统保留下来，在恢复的瞬间，继
续运行，挂起任务与恢复任务的函数原型如代码清单 4‑14所示。

代码清单 4‑14 挂起与恢复任务函数的原型

1 /\*

2 \* 暂停任务。

3  \* 此API用于挂起指定的任务，该任务将从就绪列表中删除。

4  \* 无法暂停正在运行和锁定的任务。

5  \* 无法暂停idle task和swtmr任务。

6 \*/

7 extern UINT32 LOS_TaskSuspend(UINT32 uwTaskID);

8

9 /\*

10 \* 恢复任务。

11  \* 此API用于恢复暂停的任务。

12  \* 如果任务被延迟或阻止，请恢复任务，而不将其添加到准备任务的队列中。

13  \* 如果在系统初始化后任务的优先级高于当前任务并且任务计划未锁定，则计划运行。

14 \*/

15 extern UINT32 LOS_TaskResume(UINT32 uwTaskID);

这两个任务函数的使用方法是根据传入的任务ID来挂起/恢复对应的任务，任务ID是每个任务的唯一识标，本书提供的例程将通过按键来挂起与恢复LED任务，如代码清单 4‑15加粗部分所示。

代码清单 4‑15 任务挂起与恢复的使用实例

1 static void Key_Task(void)

2 {

3 UINT32 uwRet = LOS_OK;/\* 定义一个任务的返回类型，初始化为成功的返回值 \*/

4 /\* 任务都是一个无限循环，不能返回 \*/

5 while (1) {/\* KEY1 被按下 \*/

6 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {

7 printf("挂起LED1任务！\n");

**8 uwRet = LOS_TaskSuspend(LED_Task_Handle);/\* 挂起LED任务 \*/**

9 if (LOS_OK == uwRet) {

10 printf("挂起LED1任务成功！\n");

11 }/\* KEY2 被按下 \*/

12 } else if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {

13 printf("恢复LED1任务!\n");

**14 uwRet = LOS_TaskResume(LED_Task_Handle); /\* 恢复LED任务 \*/**

15 if (LOS_OK == uwRet) {

16 printf("恢复LED1任务成功！\n");

17 }

18 }

19 LOS_TaskDelay(20); /\* 20Ticks扫描一次 \*/

20 }

21 }

任务的设计要点
~~~~~~~

作为一个嵌入式开发人员，要对自己设计的嵌入式系统要了如指掌，如任务的优先级信息、任务与中断的处理、任务的运行时间、逻辑、状态等，才能设计出好的系统，因此在设计的时候需要根据需求制定框架，并且应该考虑以下几点因素：任务运行的上下文环境（中断与任务）、空闲任务以及任务的执行时间合理设计。

1. 中断服务函数

中断服务函数是一种需要特别注意的上下文环境，它运行在非任务的执行环境下（一般为芯片的一种特殊运行模式），在这个上下文环境中不能使用挂起当前任务的操作，不能有任何阻塞的操作，在中断中不允许调用带有阻塞机制的API函数。另外需要注意的是，中断服务程序最好保持精简短小，快进快出，一般在中断服务函数中只做标
记事件的发生，然后通知任务，让对应的处理任务去执行相关处理，因为中断的优先级高于系统中任何任务，在中断处理时间过长，可能会导致整个系统任务无法正常运行。所以在设计的时候必须考虑中断的频率、中断的处理时间等重要因素，以便配合对应中断处理任务的工作。

2. 普通任务

任务看似没有什么限制程序执行的因素，似乎所有的操作都可以执行。但是做为一个优先级明确的实时系统，如果一个任务中的程序出现了死循环操作（此处的死循环是指没有阻塞机制的任务循环体），那么比该任务优先级低的任务都将无法执行，当然也包括了空闲任务，因为没有阻塞的任务不会主动让出CPU，而低优先级的任务是不允
许抢占高优先级任务的CPU的，而高优先级的任务可以抢占低优先级的CPU，如此一来低优先级将无法运行，这种情况在实时操作系统中是必须注意的一点，所以在任务中不允许出现死循环。如果一个任务只有就绪态而无阻塞态，势必会影响到其他低优先级任务的运行，所以在进行任务设计时，就应该保证任务在不活跃的时候，任务可
以进入阻塞态以让出CPU使用权，这就需要设计者明确知道什么情况下让任务进入阻塞态，保证低优先级任务可以正常运行。在实际设计中，一般会将紧急的处理事件的任务优先级设置得高一些。

3. 空闲任务

空闲任务是LiteOS系统中没有其他工作进行时自动进入的系统任务。开发者可以通过宏定义LOSCFG_KERNEL_TICKLESS与LOSCFG_KERNEL_RUNSTOP选择自己需要的特殊功能，如低功耗模式，睡眠模式等。不过需要注意的是，空闲任务是不允许阻塞也不允许被挂起的，空闲任务是唯一一个不
允许出现阻塞情况的任务，因为LiteOS需要保证系统永远都有一个可运行的任务。

4. 任务的执行时间

..

   任务的执行时间一般是指两个方面，一是任务从开始到结束的时间，二是任务的周期。

在系统设计的时候这两个时间都需要用户去考虑清楚，例如，对于事件A对应的服务任务Ta，系统要求的实时响应指标是10ms，而Ta的最大运行时间是1ms，那么10ms就是任务Ta的周期了，1ms则是任务的运行时间，简单来说任务Ta在10ms内完成对事件A的响应即可。此时，系统中还存在着以50ms为周期的另
一任务Tb，它每次运行的最大时间长度是100us。在这种情况下，即使把任务Tb的优先级抬到比Ta更高的位置，对系统的实时性指标也没什么影响，因为即使在Ta的运行过程中，Tb抢占了Ta的资源，等到Tb执行完毕，消耗的时间也只不过是100us，还是在事件A规定的响应时间内(10ms)，Ta能够安全完成对
事件A的响应。但是假如系统中还存在任务Tc，其运行时间为20ms，假如将Tc的优先级设置比Ta更高，那么在Ta运行的时候，突然间被Tc打断，等到Tc执行完毕，那Ta已经错过对事件A（10ms）的响应了，这是不允许的。所以在设计的时候，必须考虑任务的时间，一般来说处理时间更短的任务优先级应设置更高一些
。

任务管理实验
~~~~~~

任务管理实验是使用任务常用的函数进行一次实验，本书将在野火STM32开发板上进行该试验，实验将创建两个任务，一个是LED任务，另一个是按键任务，LED任务的功能是显示任务运行的状态，而按键任务则是通过检测按键的按下情况来将LED任务的挂起/恢复，实验的源码如代码清单 4‑16加粗部分所示。

代码清单 4‑16 任务管理实验源码

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

19 /\* 板级外设头文件 \*/

20 #include "bsp_usart.h"

21 #include "bsp_led.h"

22 #include "bsp_key.h"

23

24 /\* 任务ID \/

25 /\*

26 \* 任务ID是一个从0开始的数字，用于索引任务，当任务创建完成之后，它就具有了一个任务ID

27 \* 以后要想操作这个任务都需要通过这个任务ID

28 \*

29 \*/

30

31 /\* 定义任务ID变量 \*/

**32 UINT32 LED_Task_Handle;**

**33 UINT32 Key_Task_Handle;**

34

35 /\* 函数声明 \*/

36 static UINT32 AppTaskCreate(void);

37 static UINT32 Creat_LED_Task(void);

38 static UINT32 Creat_Key_Task(void);

39

40 static void LED_Task(void);

41 static void Key_Task(void);

42 static void BSP_Init(void);

43

44

45 /\*

46 \* @brief 主函数

47 \* @param 无

48 \* @retval 无

49 \* @note 第一步：开发板硬件初始化

50 第二步：创建App应用任务

51 第三步：启动LiteOS，开始多任务调度，启动失败则输出错误信息

52 \/

53 int main(void)

54 {

55 UINT32 uwRet = LOS_OK; //定义一个任务创建的返回值，默认为创建成功

56

57 /\* 板载相关初始化 \*/

58 BSP_Init();

59

60 printf("这是一个[野火]-STM32全系列开发板-LiteOS任务管理实验！\n\n");

61 printf("按下KEY1挂起任务，按下KEY2恢复任务\n");

62

63 /\* LiteOS 内核初始化 \*/

64 uwRet = LOS_KernelInit();

65

66 if (uwRet != LOS_OK) {

67 printf("LiteOS 核心初始化失败！失败代码0x%X\n",uwRet);

68 return LOS_NOK;

69 }

70

71 uwRet = AppTaskCreate();

72 if (uwRet != LOS_OK) {

73 printf("AppTaskCreate创建任务失败！失败代码0x%X\n",uwRet);

74 return LOS_NOK;

75 }

76

77 /\* 开启LiteOS任务调度 \*/

78 LOS_Start();

79

80 //正常情况下不会执行到这里

81 while (1);

82 }

83

84

85 /\*

86 \* @ 函数名 ： AppTaskCreate

87 \* @ 功能说明： 任务创建，为了方便管理，所有的任务创建函数都可以放在这个函数里面

88 \* @ 参数 ： 无

89 \* @ 返回值 ： 无

90 \/

91 static UINT32 AppTaskCreate(void)

92 {

93 /\* 定义一个返回类型变量，初始化为LOS_OK \*/

94 UINT32 uwRet = LOS_OK;

95

96 uwRet = Creat_LED_Task();

97 if (uwRet != LOS_OK) {

98 printf("LED_Task任务创建失败！失败代码0x%X\n",uwRet);

99 return uwRet;

100 }

101

102 uwRet = Creat_Key_Task();

103 if (uwRet != LOS_OK) {

104 printf("Key_Task任务创建失败！失败代码0x%X\n",uwRet);

105 return uwRet;

106 }

107 return LOS_OK;

108 }

109

110

111 /\*

112 \* @ 函数名 ： Creat_LED_Task

113 \* @ 功能说明： 创建LED_Task任务

114 \* @ 参数 ：

115 \* @ 返回值 ： 无

116 \/

117 static UINT32 Creat_LED_Task()

118 {

119 //定义一个创建任务的返回类型，初始化为创建成功的返回值

120 UINT32 uwRet = LOS_OK;

121

122 //定义一个用于创建任务的参数结构体

123 TSK_INIT_PARAM_S task_init_param;

124

125 task_init_param.usTaskPrio = 5; /\* 任务优先级，数值越小，优先级越高 \*/

126 task_init_param.pcName = "LED_Task";/\* 任务名 \*/

127 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)LED_Task;

128 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

129

130 uwRet=LOS_TaskCreate(&LED_Task_Handle,&task_init_param);/*创建任务 \*/

131 return uwRet;

132 }

133 /\*

134 \* @ 函数名 ： Creat_Key_Task

135 \* @ 功能说明： 创建Key_Task任务

136 \* @ 参数 ：

137 \* @ 返回值 ： 无

138 \/

139 static UINT32 Creat_Key_Task()

140 {

141 // 定义一个创建任务的返回类型，初始化为创建成功的返回值

142 UINT32 uwRet = LOS_OK;

143 TSK_INIT_PARAM_S task_init_param;

144

145 task_init_param.usTaskPrio = 4; /\* 任务优先级，数值越小，优先级越高 \*/

146 task_init_param.pcName = "Key_Task"; /\* 任务名*/

147 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)Key_Task;

148 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

149

150 uwRet = LOS_TaskCreate(&Key_Task_Handle,&task_init_param);/*创建任务 \*/

151

152 return uwRet;

153 }

154

155 /\*

156 \* @ 函数名 ： LED_Task

157 \* @ 功能说明： LED_Task任务实现

158 \* @ 参数 ： NULL

159 \* @ 返回值 ： NULL

160 \/

**161 static void LED_Task(void)**

**162 {**

**163 /\* 任务都是一个无限循环，不能返回 \*/**

**164 while (1) {**

**165 LED2_TOGGLE; //LED2翻转**

**166 printf("LED任务正在运行！\n");**

**167 LOS_TaskDelay(1000);**

**168 }**

**169 }**

170 /\*

171 \* @ 函数名 ： Key_Task

172 \* @ 功能说明： Key_Task任务实现

173 \* @ 参数 ： NULL

174 \* @ 返回值 ： NULL

175 \/

**176 static void Key_Task(void)**

**177 {**

**178 UINT32 uwRet = LOS_OK;**

**179**

**180 /\* 任务都是一个无限循环，不能返回 \*/**

**181 while (1) {**

**182 /\* K1 被按下 \*/**

**183 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {**

**184 printf("挂起LED任务！\n");**

**185 uwRet = LOS_TaskSuspend(LED_Task_Handle);/\* 挂起LED1任务 \*/**

**186 if (LOS_OK == uwRet) {**

**187 printf("挂起LED任务成功！\n");**

**188 }**

**189 }**

**190 /\* K2 被按下 \*/**

**191 else if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {**

**192 printf("恢复LED任务！\n");**

**193 uwRet = LOS_TaskResume(LED_Task_Handle); /\* 恢复LED1任务 \*/**

**194 if (LOS_OK == uwRet) {**

**195 printf("恢复LED任务成功！\n");**

**196 }**

**197**

**198 }**

**199 LOS_TaskDelay(20); /\* 20ms扫描一次 \*/**

**200 }**

**201 }**

202

203

204 /\*

205 \* @ 函数名 ： BSP_Init

206 \* @ 功能说明： 板级外设初始化，所有开发板上的初始化均可放在这个函数里面

207 \* @ 参数 ：

208 \* @ 返回值 ： 无

209 \/

210 static void BSP_Init(void)

211 {

212 /\*

213 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

214 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

215 \* 都统一用这个优先级分组，千万不要再分组，切忌。

216 \*/

217 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

218

219 /\* LED 初始化 \*/

220 LED_GPIO_Config();

221

222 /\* 串口初始化 \*/

223 USART_Config();

224

225 /\* 按键初始化 \*/

226 Key_GPIO_Config();

227 }

228

229 /END OF FILE/

实验现象
~~~~

将程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据读者买的开发板而定，每个型号的开发板都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，在开发板可以看到，L
ED在闪烁，按下KEY1后可以看到开发板上的灯也不闪烁了，同时在串口调试助手也输出了相应的信息，说明任务已经被挂起，按下KEY2后可以看到开发板上的灯也恢复闪烁了，同时在串口调试助手也输出了相应的信息，说明任务已经被恢复，如图 4‑3所示。

|tasksm004|

图 4‑3任务管理实验现象

.. |tasksm002| image:: media\tasksm002.png
   :width: 4.57014in
   :height: 2.63681in
.. |tasksm003| image:: media\tasksm003.png
   :width: 5.17153in
   :height: 1.69375in
.. |tasksm004| image:: media\tasksm004.png
   :width: 5.13681in
   :height: 4.05833in

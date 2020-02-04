.. vim: syntax=rst

互斥锁
=======

在学习第6章 信号量的时候，本书就提过“互斥”一词，但是在操作系统中使用信号量做临界资源的互斥保护并不是最明智的选择，LiteOS提供另一种机制——互斥锁，专门用于临界资源互斥保护。

互斥锁的基本概念
~~~~~~~~

互斥锁又称互斥信号量，是一种特殊的二值信号量，它和信号量不同的是，它具有互斥锁所有权、递归访问以及优先级继承等特性，常用于实现对临界资源的独占式处理。任意时刻互斥锁的状态只有两种，开锁或闭锁。当互斥锁被任务持有时，该互斥锁处于闭锁状态，任务获得互斥锁的所有权。当该任务释放互斥锁时，该互斥锁处于开锁状
态，任务失去该互斥锁的所有权。当一个任务持有互斥锁时，其他任务将不能再对该互斥锁进行开锁或持有。持有该互斥锁的任务能够再次获得这个锁而不被挂起，这就是互斥量的递归访问，这个特性与一般的信号量有很大的不同，在信号量中，由于已经不存在可用的信号量，任务递归获取信号量时会发生主动挂起任务最终形成死锁。

如果想要实现同步功能（任务与任务间或者任务与中断间同步），二值信号量或许是更好的选择，虽然互斥锁也可以用于任务与任务间的同步，但互斥锁更多的是用于保护资源的互斥。

互斥锁可以充当保护资源的令牌，当一个任务希望访问某个资源时，它必须先获取令牌；当任务使用资源后，必须归还令牌，以便其他任务可以访问该资源。

在二值信号量中也可以用于保护临界资源，当任务获取信号量后才能开始使用该资源，当不需要使用时就释放信号量，如此一来其他任务也能获取到信号量从而可用使用资源。但信号量会导致的另一个潜在问题：可能发生任务优先级翻转（会在下文详细讲解）。而LiteOS提供的互斥锁可以通过优先级继承算法，降低优先级翻转产生的
危害，所以，在临界资源的保护中一般使用互斥锁。

互斥锁的优先级继承机制
~~~~~~~~~~~

在 LiteOS中为了降低优先级翻转产生的危害使用了优先级继承算法，优先级继承算法是指：暂时提高占有某种临界资源的低优先级任务的优先级，使之与在所有等待该资源的任务中，优先级最高的任务优先级相等，而当这个低优先级任务执行完毕释放该资源时，优先级恢复初始设定值（此处可以看作是低优先级任务继承了高优先级
任务的优先级）。因此，继承优先级的任务避免了系统资源被任何中间优先级的任务抢占。

互斥锁与二值信号量最大的区别是：互斥锁具有优先级继承机制，而信号量没有。也就是说，某个临界资源受到一个互斥锁保护。任务访问该资源时需要获得互斥锁，如果这个资源正在被一个低优先级任务使用，那么此时的互斥锁是闭锁状态，其他任务不能获得该互斥锁，如果此时一个高优先级任务想要访问该资源，那么高优先级任务会因
为获取不到互斥锁而进入阻塞态，系统会将当前持有该互斥锁任务的优先级临时提升到与高优先级任务相同，这就是优先级继承机制，它确保高优先级任务进入阻塞状态的时间尽可能短，以及将已经出现的“优先级翻转”危害降低到最小。

任务的优先级在任务创建的时候就已经指定，高优先级的任务可以打断低优先级的任务，抢占CPU的使用权。但是在很多场合中，某些资源只有一个，当低优先级任务正在占用该资源的时候，互斥锁处于闭锁状态，即便是高优先级任务也只能等待低优先级任务释放互斥锁，然后高优先级任务才能获得互斥锁去访问该资源。高优先级任务无
法运行而低优先级任务可以运行的现象称为“优先级翻转”。

为什么说优先级翻转在操作系统中是危害很大？因为在一开始设计系统的时候，就已经指定任务的优先级了，越重要的任务优先级越高。但是发生优先级翻转，会导致系统的高优先级任务阻塞时间过长，得不到有效的处理，有可能对整个系统产生严重的危害，同时也违法了操作系统可抢占调度的原则。

举个例子，当前系统中存在三个任务，分别为H任务（High）、M任务（Middle）、L任务（Low），三个任务的优先级顺序为H任务>M任务>L任务。正常运行的时候H任务可以打断M任务与L任务，M任务可以打断L任务。假设系统中存在一个临界资源，此时该资源被L任务正在使用中，某一时刻，H任务需要使用该资
源，但此时L任务还未释放资源，H任务则因为获取不到该资源使用权而进入阻塞态，L任务继续使用该资源，此时已经出现了“优先级翻转”现象，高优先级任务在等待低优先级的任务执行，如果在L任务执行的时候刚好M任务被唤醒了，由于M任务优先级比L任务优先级高，那么会打断L任务，抢占L任务的CPU使用权，直到M任务
执行完，再把CPU使用权归还给L任务，L任务继续执行，等到执行完毕之后释放该资源，此时H任务才能从阻塞态解除，使用该资源。这个过程，本来是最高优先级的H任务，等待了更低优先级的L任务与M任务，其阻塞的时间是M任务运行时间+L任务运行时间，假如系统中有多个如M任务这样的中间优先级任务抢占最低优先级任务
CPU使用权，那么系统最高优先级的任务将持续阻塞，这是绝对不允许出现的。因此在没有优先级继承的情况下，保护临界资源将有可能产生优先级翻转，其危害极大，优先级翻转过程示意图如图 7‑1所示。

|mutex002|

图 7‑1优先级翻转示意图

图 7‑1\ **(1)**\ ：L任务正在使用某临界资源， H任务被唤醒，执行H任务。但L任务并未执行完毕，此时临界资源还未释放。

图 7‑1\ **(2)**\ ：这个时刻H任务也要对该临界资源进行访问，但 L任务还未释放资源，由于保护机制，H任务进入阻塞态，L任务得以继续运行，此时已经发生了优先级翻转现象。

图 7‑1\ **(3)**\ ：某个时刻M任务被唤醒，由于M任务的优先级高于L任务， M任务抢占了CPU的使用权，M任务开始运行，此时L任务尚未执行完，临界资源还未被释放。

图 7‑1\ **(4)**\ ：M任务运行结束，归还CPU使用权，L任务继续运行。

图 7‑1\ **(5)**\ ：L任务运行结束，释放临界资源，H任务得以对资源进行访问，H任务开始运行。

如果H任务的等待时间过长，对整个系统来说可能是致命的，所以尽可能降低高优先级任务的等待时间以此降低优先级翻转的危害，而互斥锁就是用于临界资源的保护，并且其特有的优先级继承机制可以降低优先级翻转的产生的危害。

假如系统使用互斥锁保护临界资源，那么就具有优先级继承特性，任务需要在获取互斥锁后才能访问临界资源。在H任务获取互斥锁时，由于获取不到互斥锁进入阻塞态，那么系统就会把当前正在使用资源的L任务的优先级临时提升到与H任务优先级相同，当M任务被唤醒时，因为它的优先级比H任务低，所以无法打断L任务，因为此时L
任务的优先级被临时提升到H，所以当L任务使用完该资源候释放互斥锁，H任务将获得互斥锁而恢复运行。因此H任务的阻塞时间仅仅是L任务的执行时间，此时的优先级的危害降到了最低，这就是优先级继承的优势，其示意图如图 7‑2所示。

|mutex003|

图 7‑2优先级继承示意图

图 7‑2\ **(1)**\ ： L任务正在使用某临界资源， H任务被唤醒，执行H任务。但L任务尚未运行完毕，此时互斥锁还未释放。

图 7‑2\ **(2)**\ ：某一时刻H任务也要获取互斥锁访问该资源，由于互斥锁对临界资源的保护机制，H任务无法获得互斥锁而进入阻塞态。此时发生优先级继承，系统将L任务的优先级暂时提升到与H任务优先级相同，L任务继续执行。

图 7‑2\ **(3)**\ ：在某一时刻M任务被唤醒，由于此时M任务的优先级暂时低于L任务，所以M任务仅在就绪态，而无法获得CPU使用权。

图 7‑2\ **(4)**\ ：L任务运行完毕释放互斥锁，H任务获得互斥锁后恢复运行，此时L任务的优先级会恢复初始指定的优先级。

图 7‑2\ **(5)**\ ：当H任务运行完毕，M任务得到CPU使用权，开始执行。

图 7‑2\ **(6)**\ ：系统正常运行，按照初始指定的优先级运行。

使用互斥锁的时候一定需要注意以下几点。

1. 在获得互斥锁后，请尽快释放互斥锁。

2. 在任务持有互斥锁的这段时间，不得更改任务的优先级。

3. LiteOS的优先级继承机制不能解决优先级翻转，只能将这种情况的影响降低到最小，硬实时系统在一开始设计时就要避免优先级翻转发生。

4. 互斥锁不能在中断服务函数中使用。

互斥锁的应用场景
~~~~~~~~

互斥锁的使用比较单一，因为它是信号量的一种，并且它是以锁的形式存在。在初始化的时候，互斥锁处于开锁的状态，而当被任务持有的时候则立刻转为闭锁的状态。互斥锁更适合于以下场景。

1. 可能会引起优先级翻转的情况。

2. 任务可能会多次获取互斥锁的情况下。这样可以避免同一任务多次递归持有而造成死锁的问题。

多任务环境下往往存在多个任务竞争同一临界资源的应用场景，互斥锁可被用于对临界资源的保护从而实现独占式访问。另外，互斥锁可以降低信号量存在的优先级翻转问题带来的影响。

比如有两个任务需要对串口进行发送数据，其硬件资源只有一个，那么两个任务肯定不能同时发送数据，不然将导致数据错误，那么，就可以用互斥锁对串口资源进行保护，当一个任务正在使用串口的时候，另一个任务则无法使用串口，等到任务使用串口完毕之后，另外一个任务才能获得串口的使用权。

互斥锁的运作机制
~~~~~~~~

多任务环境下会存在多个任务访问同一临界资源的场景，该资源会被任务独占处理。其他任务在资源被占用的情况下不允许对该临界资源进行访问，这个时候就需要用到LiteOS的互斥锁来进行资源保护，那么互斥锁是怎样来避免这种冲突？

使用互斥锁处理不同任务对临界资源的同步访问时，任务想要获得互斥锁才能访问资源，如果一旦有任务成功获得了互斥锁，则互斥锁立即变为闭锁状态，此时其他任务会因为获取不到互斥锁而不能访问该资源，任务会根据用户指定的阻塞时间进行等待，直到互斥锁被持有任务释放后，其他任务才能获取互斥锁从而得以访问该临界资源，此
时互斥锁再次上锁，如此一来就可以确保同一时刻只有一个任务正在访问这个临界资源，保证了临界资源操作的安全性，其过程如图 7‑3所示。

|mutex004|

图 7‑3互斥锁运作机制

图 7‑3\ **(1)**\ ：因为互斥锁具有优先级继承机制，一般选择使用互斥锁对资源进行保护，如果资源被占用的时候，无论是何种优先级的任务想要使用该资源都会被阻塞。

图 7‑3\ **(2)**\ ：假如正在使用该资源的任务1比阻塞中的任务2的优先级低，那么任务1将被系统临时提升到与高优先级任务2相等的优先级（任务1的优先级从L 变成H）。

图 7‑3\ **(3)**\ ：当任务1使用完资源之后，释放互斥锁，此时任务1的优先级从H恢复L。

图 7‑3\ **(4)-(5)**\ ：任务2此时可以获得互斥锁，然后访问资源，当任务2访问了资源的时候，该互斥锁的状态又为闭锁状态，其他任务无法获取互斥锁。

互斥锁的使用讲解
~~~~~~~~

互斥锁控制块
^^^^^^

互斥锁控制块与信号量控制类似，系统中每一个互斥锁都有对应的互斥锁控制块，它记录了互斥锁的所有信息，比如互斥锁的状态，持有次数、ID、所属任务等，如代码清单 7‑1所示。

代码清单 7‑1互斥锁控制块

1 typedef struct {

2 UINT8 ucMuxStat; **(1)**

3 UINT16 usMuxCount; **(2)**

4 UINT32 ucMuxID; **(3)**

5 LOS_DL_LIST stMuxList; **(4)**

6 LOS_TASK_CB \*pstOwner; **(5)**

7 UINT16 usPriority; **(6)**

8 } MUX_CB_S;

代码清单 7‑1\ **(1)**\ ：ucMuxStat是互斥锁状态，其状态有两个：OS_MUX_UNUSED或OS_MUX_USED，表示互斥锁是否被使用。

代码清单 7‑1\ **(2)**\ ：usMuxCount是互斥锁持有次数，在每次获取互斥锁的时候，该成员变量会增加，用于记录持有的次数，当usMuxCount为0的时候表示互斥锁处于开锁状态，任务可以随时获取，当它是一个正值的时候，表示互斥锁已经被获取了，只有持有互斥锁的任务才能释放它。

代码清单 7‑1\ **(3)**\ ：ucMuxID是互斥锁ID。

代码清单 7‑1\ **(4)**\ ：stMuxList是互斥锁阻塞列表。用于记录阻塞在此互斥锁的任务。

代码清单 7‑1\ **(5)**\ ：*pstOwner是一个任务控制块指针，指向当前持有该互斥锁任务，如此一来系统就能够知道该互斥锁的所有权归哪个任务，互斥锁的释放只能是持有互斥锁的任务进行释放，其他任务都没有权利操作已经处于闭锁状态的互斥锁。

代码清单 7‑1\ **(6)**\ ：usPriority是记录持有互斥锁任务的初始优先级，用于处理优先级继承。

互斥锁错误代码
^^^^^^^

在LiteOS中，与互斥锁相关的函数大多数都会有返回值，其返回值是一些错误代码，方便使用者进行调试，下面列出一些常见的错误代码与参考解决方案，具体如表 7‑1所示。

表 7‑1常见互斥锁错误代码说明

.. list-table::
   :widths: 25 25 25 25
   :header-rows: 0


   * - 序号 |
     - 义              | 描述
     - | 参考解决
     - 案      |

   * - 1
     - LOS_ER RNO_MUX_NO_MEMORY
     - 内存请求失败      | 减少互斥
     - |
         |

   * - 2
     - LOS_ ERRNO_MUX_INVALID
     - 互斥锁不可用      | 传入
     - |
        |

   * - 3
     - LOS_E RRNO_MUX_PTR_NULL
     - 传入空指针        | 传入合
     - 指针      |

   * - 4
     - LOS_E RRNO_MUX_ALL_BUSY
     - 没有互斥锁可用    | 增加互斥
     - |
       限  |

   * - 5
     - LOS_ERRN O_MUX_UNAVAILABLE
     - 锁失败，因为      | 等待其他 锁被其他任务使用  | 或者设置等待
     - 务解锁  | 间  |

   * - 6
     - LOS_ERRN O_MUX_PEND_INTERR
     - 在                | 中断中使用互斥锁  | 中禁止调用此
     - 在中断            | 口  |

   * - 7
     - LOS_ERRNO _MUX_PEND_IN_LOCK
     - 任务调度没        | 设置 有使能，任务等待  | PEND为非 另一个任务释放锁  | 或者使能任务
     - |

   * - 8
     - LOS_ ERRNO_MUX_TIMEOUT
     - 互斥锁PEND超时    | 增加等
     - 时间或者  | 设置一直等待模式  |

   * - 9
     - LOS _ERRNO_MUX_PENDED
     - 删除正在使用的锁  | 等待解锁再删
     - 锁  |


互斥锁创建函数LOS_MuxCreate()
^^^^^^^^^^^^^^^^^^^^^^

LiteOS提供互斥锁创建函数接口——LOS_MuxCreate()，该函数用于创建一个互斥锁，在创建互斥锁后，系统会返回互斥锁ID，以后对互斥锁的操作也是通过互斥锁ID进行操作的，因此需要用户定义一个互斥锁ID变量，并将变量的地址传入互斥锁创建函数中，LOS_MuxCreate()函数源码如
代码清单 7‑2所示，其使用实例如代码清单 7‑3加粗部分所示。

代码清单 7‑2 互斥锁创建函数LOS_MuxCreate()源码

1 /\*

2 Function : LOS_MuxCreate

3 Description : 创建一个互斥锁,

4 Input : None

5 Output : puwMuxHandle --- 互斥锁ID（句柄）

6 Return : 返回LOS_OK表示创建成功,或者其他失败的错误代码

7 \/

8 LITE_OS_SEC_TEXT_INIT UINT32 LOS_MuxCreate (UINT32 \*puwMuxHandle)

9 {

10 UINT32 uwIntSave;

11 MUX_CB_S \*pstMuxCreated;

12 LOS_DL_LIST \*pstUnusedMux;

13 UINT32 uwErrNo;

14 UINT32 uwErrLine;

15

16 if (NULL == puwMuxHandle) { **(1)**

17 return LOS_ERRNO_MUX_PTR_NULL;

18 }

19

20 uwIntSave = LOS_IntLock();

21 if (LOS_ListEmpty(&g_stUnusedMuxList)) { **(2)**

22 LOS_IntRestore(uwIntSave);

23 OS_GOTO_ERR_HANDLER(LOS_ERRNO_MUX_ALL_BUSY);

24 }

25

26 pstUnusedMux = LOS_DL_LIST_FIRST(&(g_stUnusedMuxList));

27 LOS_ListDelete(pstUnusedMux);

28 pstMuxCreated = (GET_MUX_LIST(pstUnusedMux)); **(3)**

29 pstMuxCreated->usMuxCount = 0; **(4)**

30 pstMuxCreated->ucMuxStat = OS_MUX_USED; **(5)**

31 pstMuxCreated->usPriority = 0; **(6)**

32 pstMuxCreated->pstOwner = (LOS_TASK_CB \*)NULL; **(7)**

33 LOS_ListInit(&pstMuxCreated->stMuxList); **(8)**

34 \*puwMuxHandle = (UINT32)pstMuxCreated->ucMuxID; **(9)**

35 LOS_IntRestore(uwIntSave);

36 return LOS_OK;

37 ErrHandler:

38 OS_RETURN_ERROR_P2(uwErrLine, uwErrNo);

39 }

代码清单 7‑2\ **(1)**\ ：判断互斥锁ID变量地址是否有效，如果为NULL则返回错误代码。

代码清单 7‑2\ **(2)**\
：从系统的互斥锁未使用列表取下一个互斥锁控制块，如果系统中没有可用的互斥锁控制块，则返回错误代码，因为系统可用的互斥锁个数达到系统支持的上限，读者可以在target_config.h文件中修改LOSCFG_BASE_IPC_MUX_LIMIT宏定义以增加系统支持的互斥锁数量。

代码清单 7‑2\ **(3)**\ ：如果系统中互斥锁尚未达到上限，就从互斥锁未使用列表中获取一个互斥锁控制块。

代码清单 7‑2\ **(4)**\ ：初始化互斥锁中的持有次数为0，表示互斥锁处于开锁状态，因为新创建的互斥锁是没有被任何任务持有的。

代码清单 7‑2\ **(5)**\ ：初始化互斥锁的状态信息为已使用的状态。

代码清单 7‑2\ **(6)**\ ：初始化占用互斥锁的任务的优先级，为最高优先级，此时互斥锁没有被任何任务持有，当有任务持有互斥锁时，这个值会设置为持有任务的优先级数值。

代码清单 7‑2\ **(7)**\ ：将指向任务控制块的指针初始化为NULL表示没有任务持有互斥锁。

代码清单 7‑2\ **(8)**\ ：初始化互斥锁的阻塞列表。

代码清单 7‑2\ **(9)**\ ：返回已经创建成功的互斥锁ID。

代码清单 7‑3互斥锁创建函数LOS_MuxCreate()实例

**1 /\* 定义互斥锁的ID变量 \*/**

**2 UINT32 Mutex_Handle;**

3 UINT32 uwRet = LOS_OK;/\* 定义一个创建任务的返回类型，初始化为创建成功的返回值 \*/

4

**5 /\* 创建一个互斥锁*/**

**6 uwRet = LOS_MuxCreate(&Mutex_Handle);**

7 if (uwRet != LOS_OK)

8 {

9 printf("Mutex_Handle互斥锁创建失败！\n");

10 }

互斥锁删除函数LOS_MuxDelete()
^^^^^^^^^^^^^^^^^^^^^^

读者可以根据互斥锁ID将互斥锁删除，删除后的互斥锁将不能被使用，它所有信息都会被系统回收，如果系统中有任务持有互斥锁或者有任务阻塞在互斥锁上时，互斥锁是不能被删除的。uwMuxHandle是互斥锁ID，表示的是要删除哪个互斥锁，其函数源码如代码清单 7‑4所示。

代码清单 7‑4互斥锁删除函数LOS_MuxDelete()源码

1 /\*

2 Function : LOS_MuxDelete

3 Description : 删除一个互斥锁

4 Input : uwMuxHandle------互斥锁ID

5 Output : None

6 Return : 返回LOS_OK表示删除成功,或者其他失败的错误代码

7 \/

8 LITE_OS_SEC_TEXT_INIT UINT32 LOS_MuxDelete(UINT32 uwMuxHandle)

9 {

10 UINT32 uwIntSave;

11 MUX_CB_S \*pstMuxDeleted;

12 UINT32 uwErrNo;

13 UINT32 uwErrLine;

14

15 if (uwMuxHandle >= (UINT32)LOSCFG_BASE_IPC_MUX_LIMIT) { **(1)**

16 OS_GOTO_ERR_HANDLER(LOS_ERRNO_MUX_INVALID);

17 }

18

19 pstMuxDeleted = GET_MUX(uwMuxHandle); **(2)**

20 uwIntSave = LOS_IntLock();

21 if (OS_MUX_UNUSED == pstMuxDeleted->ucMuxStat) { **(3)**

22 LOS_IntRestore(uwIntSave);

23 OS_GOTO_ERR_HANDLER(LOS_ERRNO_MUX_INVALID);

24 }

25

26 if (!LOS_ListEmpty(&pstMuxDeleted->stMuxList) \|\| pstMuxDeleted->usMuxCount) {

27 LOS_IntRestore(uwIntSave);

28 OS_GOTO_ERR_HANDLER(LOS_ERRNO_MUX_PENDED); **(4)**

29 }

30

31 LOS_ListAdd(&g_stUnusedMuxList, &pstMuxDeleted->stMuxList); **(5)**

32 pstMuxDeleted->ucMuxStat = OS_MUX_UNUSED; **(6)**

33

34 LOS_IntRestore(uwIntSave);

35

36 return LOS_OK;

37 ErrHandler:

38 OS_RETURN_ERROR_P2(uwErrLine, uwErrNo);

39 }

代码清单 7‑4\ **(1)**\ ：判断互斥锁ID是否有效，如果无效则返回错误代码LOS_ERRNO_MUX_INVALID。

代码清单 7‑4\ **(2)**\ ：根据互斥锁ID获取要删除的互斥锁控制块指针。

代码清单 7‑4\ **(3)**\ ：如果该互斥锁是未使用的，则返回错误代码。

代码清单 7‑4\ **(4)**\ ：如果系统中有任务持有互斥锁或者有任务阻塞在互斥锁上时，系统不会删除该互斥锁，返回错误代码LOS_ERRNO_MUX_PENDED，读者需要确保没有任务持有互斥锁或者没有任务阻塞在互斥锁上时再进行删除操作。

代码清单 7‑4\ **(5)**\ ：把互斥锁添加到互斥锁未使用列表中。

代码清单 7‑4\ **(6)**\ ：将互斥锁的状态改变为未使用，表示互斥锁已经删除。

互斥锁删除函数的使用方法，如代码清单 7‑5加粗部分所示。

代码清单 7‑5互斥锁删除函数LOS_MuxDelete()实例

1 UINT32 uwRet = LOS_OK;/\* 定义一个返回类型，初始化为删除成功的返回值 \*/

**2 uwRet = LOS_MuxDelete(Mutex_Handle); /\* 删除互斥锁 \*/**

3 if (LOS_OK == uwRet)

4 {

5 printf("互斥锁删除成功！\n");

6 }

互斥锁释放函数LOS_MuxPost()
^^^^^^^^^^^^^^^^^^^^

任务想要访问某个临界资源时，需要先获取互斥锁，然后才能访问该资源，在任务使用完该资源后必须要及时释放互斥锁，其他任务才能获取互斥锁从而访问该资源。在前面章节的讲解中，读者应该都知道当互斥锁处于开锁状态的时候，任务才能获取互斥锁，那么，是什么函数使互斥锁处于开锁状态呢？LiteOS提供了互斥锁释放函数
LOS_MuxPost()，持有互斥锁的任务可以调用该函数将互斥锁释放，释放后的互斥锁处于开锁状态，系统中其他任务可以获取互斥锁。但互斥锁允许在任务中释放而不能在中断中释放，原因有以下两点。

1. 中断上下文没有一个任务的概念。

2. 互斥锁只能被持有者释放，持有者是任务。

互斥锁有所属关系，只有持有者才能释放锁，而这个持有者是任务，因为中断上下文没有任务概念，所以中断上下文不能持有，也不能释放互斥锁。

使用该函数接口时，只有已持有互斥锁所有权的任务才能释放它，当持有互斥锁的任务调用LOS_MuxPost()函数时会将互斥锁变为开锁状态，如果有其他任务在等待获取该互斥锁时，等待的任务将被唤醒，然后持有该互斥锁。如果任务的优先级被临时提升，那么当互斥锁被释放后，任务的优先级将恢复为任务初始设定的优先级
，LOS_MuxPost()源码如代码清单 7‑6所示。

代码清单 7‑6互斥锁释放函数LOS_MuxPost()源码

1 /\*

2 Function : LOS_MuxPost

3 Description : 释放一个互斥锁

4 Input : uwMuxHandle ------ 互斥锁ID

5 Output : None

6 Return : 返回LOS_OK表示释放成功,或者其他失败的错误代码

7 \/

8 LITE_OS_SEC_TEXT UINT32 LOS_MuxPost(UINT32 uwMuxHandle)

9 {

10 UINT32 uwIntSave;

11 MUX_CB_S \*pstMuxPosted = GET_MUX(uwMuxHandle);

12 LOS_TASK_CB \*pstResumedTask;

13 LOS_TASK_CB \*pstRunTsk;

14

15 uwIntSave = LOS_IntLock();

16

17 if ((uwMuxHandle >= (UINT32)LOSCFG_BASE_IPC_MUX_LIMIT) \|\|

18 (OS_MUX_UNUSED == pstMuxPosted->ucMuxStat)) { **(1)**

19 LOS_IntRestore(uwIntSave);

20 OS_RETURN_ERROR(LOS_ERRNO_MUX_INVALID);

21 }

22

23 pstRunTsk = (LOS_TASK_CB \*)g_stLosTask.pstRunTask;

24 if ((pstMuxPosted->usMuxCount == 0)||(pstMuxPosted->pstOwner != pstRunTsk)) {

25 LOS_IntRestore(uwIntSave);

26 OS_RETURN_ERROR(LOS_ERRNO_MUX_INVALID); **(2)**

27 }

28

29 if (--(pstMuxPosted->usMuxCount) != 0) { **(3)**

30 LOS_IntRestore(uwIntSave);

31 return LOS_OK;

32 }

33

34 if ((pstMuxPosted->pstOwner->usPriority)!=pstMuxPosted->usPriority){

35 osTaskPriModify(pstMuxPosted->pstOwner, pstMuxPosted->usPriority);

36 } **(4)**

37

38 if (!LOS_ListEmpty(&pstMuxPosted->stMuxList)) {

39 pstResumedTask = OS_TCB_FROM_PENDLIST(

40 LOS_DL_LIST_FIRST(&(pstMuxPosted->stMuxList))); **(5)** **(5)**

41 pstMuxPosted->usMuxCount = 1; **(6)**

42 pstMuxPosted->pstOwner = pstResumedTask; **(7)**

43 pstMuxPosted->usPriority = pstResumedTask->usPriority;\ **(8)**

44 pstResumedTask->pTaskMux = NULL; **(9)**

45

46 osTaskWake(pstResumedTask, OS_TASK_STATUS_PEND); **(10)**

47

48 (VOID)LOS_IntRestore(uwIntSave);

49 LOS_Schedule(); **(11)**

50 } else {

51 (VOID)LOS_IntRestore(uwIntSave);

52 }

53

54 return LOS_OK;

55 }

代码清单 7‑6\ **(1)**\ ：如果互斥锁ID是无效的，或者要释放的信号量状态是未使用的，则返回错误代码。

代码清单 7‑6\ **(2)**\ ：如果互斥锁没有被任务持有，那就无需释放互斥锁；如果持有互斥锁的任务不是当前任务，则不允许进行互斥锁释放操作，因为互斥锁的所有权仅归持有互斥锁的任务所有，其他任务不具备释放/获取互斥锁的权利。

代码清单 7‑6\ **(3)**\ ：满足释放互斥锁的条件，释放一次互斥锁后usMuxCount持有次数不为0，这就表明当前任务还持有互斥锁，此时互斥锁还处于闭锁状态，返回LOS_OK表示释放成功。

代码清单 7‑6\ **(4)**\ ：如果当前任务已经完全释放了持有的互斥锁，由于可能发生过优先级继承从而修改了任务的优先级，那么系统就需要恢复任务初始的优先级，如果当前任务的优先级与初始设定的优先级不一样，则调用osTaskPriModify()函数使任务的优先级恢复为初始设定的优先级。

代码清单 7‑6\ **(5)**\ ：如果有任务阻塞在该互斥锁上，获取阻塞任务的任务控制块。

代码清单 7‑6\ **(6)**\ ：设置互斥锁的持有次数为1，新任务持有互斥锁。

代码清单 7‑6\ **(7)**\ ：互斥锁的任务控制块指针指向新任务控制块。

代码清单 7‑6\ **(8)**\ ：记录持有互斥锁任务的优先级。

代码清单 7‑6\ **(9)**\ ：将新任务控制块中pTaskMux指针指向NULL。

代码清单 7‑6\ **(10)**\ ：将新任务从阻塞列表中移除，并且添加到就绪列表中。

代码清单 7‑6\ **(11)**\ ：进行一次任务调度。

被释放前的互斥锁是处于上锁状态，被释放后互斥锁是开锁状态，除了将互斥锁控制块中usMuxCount变量减一外，还要判断一下持有互斥锁的任务是否发生优先级继承，如果有的话，要将任务的优先级恢复到初始值；并且判断一下是否有任务阻塞在该互斥锁上，如果有则将任务恢复就绪态并持有互斥锁。互斥锁释放函数的使用实
例如代码清单 7‑7加粗部分所示。

代码清单 7‑7互斥锁释放函数LOS_MuxPost()实例

1 /\* 定义互斥锁的ID变量 \*/

2 UINT32 Mutex_Handle;

3

4 UINT32 uwRet = LOS_OK;/\* 定义一个返回类型，初始化为成功的返回值 \*/

**5 /\* 释放一个互斥锁*/**

**6 uwRet = LOS_MuxPost(Mutex_Handle);**

7 if (LOS_OK == uwRet)

8 {

9 printf("互斥锁释放成功！\n");

10 }

互斥锁获取函数LOS_MuxPend()
^^^^^^^^^^^^^^^^^^^^

当互斥锁处于开锁状态时，任务才能够获取互斥锁，当任务持有了某个互斥锁的时候，其他任务就无法获取这个互斥锁，需要等到持有互斥锁的任务进行释放后，其他任务才能获取成功，任务通过互斥锁获取函数来获取互斥锁的所有权。任务对互斥锁的所有权是独占的，任意时刻互斥锁只能被一个任务持有，如果互斥锁处于开锁状态，那么
获取该互斥锁的任务将成功获得该互斥锁，并拥有互斥锁的使用权；如果互斥锁处于闭锁状态，获取该互斥锁的任务将无法获得互斥锁，任务将被挂起，在任务被挂起之前，会进行优先级继承，如果当前任务优先级比持有互斥锁的任务优先级高，那么将会临时提升持有互斥锁任务的优先级。互斥锁的获取函数是LOS_MuxPend()
，其源码如代码清单 7‑8所示。

代码清单 7‑8互斥锁获取函数LOS_MuxPend()源码

1 /\*

2 Function : LOS_MuxPend

3 Description : 对指定的互斥锁ID获取互斥锁,

4 Input : uwMuxHandle ------ 互斥锁ID,

5 uwTimeOut ------- 等待时间

6 Output : None

7 Return : 返回LOS_OK表示获取成功,或者其他失败的错误代码

8 \/

9 LITE_OS_SEC_TEXT UINT32 LOS_MuxPend(UINT32 uwMuxHandle, UINT32 uwTimeout)

10 {

11 UINT32 uwIntSave;

12 MUX_CB_S \*pstMuxPended;

13 UINT32 uwRetErr;

14 LOS_TASK_CB \*pstRunTsk;

15

16 if (uwMuxHandle >= (UINT32)LOSCFG_BASE_IPC_MUX_LIMIT) {

17 OS_RETURN_ERROR(LOS_ERRNO_MUX_INVALID); **(1)**

18 }

19

20 pstMuxPended = GET_MUX(uwMuxHandle);

21 uwIntSave = LOS_IntLock();

22 if (OS_MUX_UNUSED == pstMuxPended->ucMuxStat) { **(2)**

23 LOS_IntRestore(uwIntSave);

24 OS_RETURN_ERROR(LOS_ERRNO_MUX_INVALID);

25 }

26

27 if (OS_INT_ACTIVE) { **(3)**

28 LOS_IntRestore(uwIntSave);

29 return LOS_ERRNO_MUX_PEND_INTERR;

30 }

31

32 pstRunTsk = (LOS_TASK_CB \*)g_stLosTask.pstRunTask; **(4)**

33 if (pstMuxPended->usMuxCount == 0) { **(5)**

34 pstMuxPended->usMuxCount++;

35 pstMuxPended->pstOwner = pstRunTsk;

36 pstMuxPended->usPriority = pstRunTsk->usPriority;

37 LOS_IntRestore(uwIntSave);

38 return LOS_OK;

39 }

40

41 if (pstMuxPended->pstOwner == pstRunTsk) { **(6)**

42 pstMuxPended->usMuxCount++;

43 LOS_IntRestore(uwIntSave);

44 return LOS_OK;

45 }

46

47 if (!uwTimeout) { **(7)**

48 LOS_IntRestore(uwIntSave);

49 return LOS_ERRNO_MUX_UNAVAILABLE;

50 }

51

52 if (g_usLosTaskLock) { **(8)**

53 uwRetErr = LOS_ERRNO_MUX_PEND_IN_LOCK;

54 PRINT_ERR("!!!LOS_ERRNO_MUX_PEND_IN_LOCK!!!\n");

55 #if (LOSCFG_PLATFORM_EXC == YES)

56 osBackTrace();

57 #endif

58 goto errre_uniMuxPend;

59 }

60

61 pstRunTsk->pTaskMux = (VOID \*)pstMuxPended; **(9)**

62

63 if (pstMuxPended->pstOwner->usPriority > pstRunTsk->usPriority) {

64 osTaskPriModify(pstMuxPended->pstOwner, pstRunTsk->usPriority);

65 } **(10)**

66

67 osTaskWait(&pstMuxPended->stMuxList, OS_TASK_STATUS_PEND, uwTimeout);

68

69 (VOID)LOS_IntRestore(uwIntSave);

70 LOS_Schedule(); **(11)**

71

72 if (pstRunTsk->usTaskStatus & OS_TASK_STATUS_TIMEOUT) { **(12)**

73 uwIntSave = LOS_IntLock();

74 pstRunTsk->usTaskStatus &= (~OS_TASK_STATUS_TIMEOUT);

75 (VOID)LOS_IntRestore(uwIntSave);

76 uwRetErr = LOS_ERRNO_MUX_TIMEOUT;

77 goto error_uniMuxPend;

78 }

79

80 return LOS_OK;

81

82 errre_uniMuxPend:

83 (VOID)LOS_IntRestore(uwIntSave);

84 error_uniMuxPend:

85 OS_RETURN_ERROR(uwRetErr);

86 }

代码清单 7‑8\ **(1)**\ ：如果互斥锁ID是无效的，返回错误代码。

代码清单 7‑8\ **(2)**\ ：根据互斥锁ID获取互斥锁控制块，如果该互斥锁是未使用的，返回错误代码LOS_ERRNO_MUX_INVALID。

代码清单 7‑8\ **(3)**\ ：如果在中断中调用此函数，则是非法的，返回错误代码LOS_ERRNO_MUX_PEND_INTERR，因为互斥锁是不允许在中断中使用，只能在任务中获取互斥锁。

代码清单 7‑8\ **(4)**\ ：获取当前运行的任务控制块。

代码清单 7‑8\ **(5)**\ ：如果此互斥锁处于开锁状态，则可以获取互斥锁，并且将互斥锁的锁定次数加1，互斥锁控制块的成员变量pstOwner指向当前任务控制块，记录该互斥锁归哪个任务所有；记录持有互斥锁的任务的优先级，用于优先级继承机制，获取成功返回LOS_OK。

代码清单 7‑8\ **(6)**\ ：如果当前任务是持有互斥锁的任务，系统允许再次获取互斥锁，则只需记录次互斥锁被持有的次数即可，返回LOS_OK。

代码清单 7‑8\ **(7)**\ ：如果互斥锁处于闭锁状态，那么当前任务将无法获取互斥锁，如果用户指定的阻塞时间为0，则直接返回错误代码LOS_ERRNO_MUX_UNAVAILABLE。

代码清单 7‑8\ **(8)**\ ：如果调度器已上锁则返回LOS_ERRNO_MUX_PEND_IN_LOCK 。

代码清单 7‑8\ **(9)**\ ：标记一下当前任务是由于获取不到哪个互斥锁而进入阻塞态。

代码清单 7‑8\ **(10)**\ ：如果持有该互斥锁的任务优先级比当前任务的优先级低，系统会把持有互斥锁任务的优先级暂时提升到与当前任务优先级一致，除此之外系统还会将当前任务添加到互斥锁的阻塞列表中。

代码清单 7‑8\ **(11)**\ ：进行一次任务调度。

代码清单 7‑8\ **(12)**\
：程序能运行到这，说明持有互斥锁的任务释放了互斥锁，或者是阻塞时间已超时，那么系统要判断一下解除阻塞的原因，如果是由于阻塞时间超时，则返回错误代码LOS_ERRNO_MUX_TIMEOUT；而如果是持有互斥锁任务释放了互斥锁，那么在释放互斥锁的时候，阻塞的任务已经恢复运行，并且持有互斥锁了。

至此，获取互斥锁的操作就完成了，如果任务获取互斥锁成功，那么在使用完毕需要立即释放，否则造成其他任务无法获取互斥锁而导致系统无法正常运作，因为互斥锁的优先级继承机制是只能将优先级危害降低，而不能完全消除。同时还需注意的是，互斥锁是不允许在中断中操作的，互斥锁获取函数的使用实例如代码清单
7‑9加粗部分所示。

代码清单 7‑9互斥锁获取函数LOS_MuxPend()实例

1 /\* 定义互斥锁的ID变量 \*/

2 UINT32 Mutex_Handle;

3

4 UINT32 uwRet = LOS_OK;/\* 定义一个返回类型，初始化为成功的返回值 \*/

**5 //获取互斥锁，没获取到则一直等待**

**6 uwRet = LOS_MuxPend(Mutex_Handle,LOS_WAIT_FOREVER);**

7 if (LOS_OK == uwRet)

8 {

9 printf("互斥获取成功！\n");

10 }

使用互斥锁的注意事项
^^^^^^^^^^

1. 两个任务不能获取同一个互斥锁。如果某任务尝试获取已被持有的互斥锁，则该任务会被阻塞，直到持有该互斥锁的任务释放互斥锁。

2. 互斥锁不能在中断服务函数中使用。

3. LiteOS作为实时操作系统需要保证任务调度的实时性，尽量避免任务的长时间阻塞，因此在获得互斥锁之后，应该尽快释放互斥锁。

4. 任务持有互斥锁的过程中，不允许再调用LOS_TaskPriSet()等函数接口更改持有互斥锁任务的优先级。

5. 互斥锁和信号量的区别在于：互斥锁可以被已经持有互斥锁的任务重复获取，而不会形成死锁。这个递归调用功能是通过互斥锁控制块usMuxCount成员变量实现的，这个变量用于记录任务持有互斥锁的次数，在每次获取互斥锁后该变量加1，在释放互斥锁后该变量减1。只有当usMuxCount的值为0时，互斥锁才处于开
   锁状态，其他任务才能获取该互斥锁。

互斥锁实验
~~~~~

模拟优先级翻转实验
^^^^^^^^^

模拟优先级翻转实验是在LiteOS中创建了三个任务与一个二值信号量，任务分别是高优先级任务，中优先级任务，低优先级任务，用于模拟产生优先级翻转。低优先级任务在获取信号量的时候，被中优先级打断，中优先级的任务执行时间较长，因为低优先级还未释放信号量，那么高优先级任务就无法获得信号量而进入阻塞态，此时就
发生了优先级翻转，任务在运行中通过串口打印出相关信息，实验源码如代码清单 7‑10加粗部分所示。

代码清单 7‑10模拟优先级翻转实验

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

19 #include "los_sem.h"

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

**32 /\* 定义任务ID变量 \*/**

**33 UINT32 HighPriority_Task_Handle;**

**34 UINT32 MidPriority_Task_Handle;**

**35 UINT32 LowPriority_Task_Handle;**

36

37 /\* 内核对象ID \/

38 /\*

39 \* 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核

40 \* 对象，必须先创建，创建成功之后会返回一个相应的ID。实际上就是一个整数，后续

41 \* 就可以通过这个ID操作这些内核对象。

42 \*

43 \*

44 内核对象就是一种全局的数据结构，通过这些数据结构可以实现任务间的通信，

45 \* 任务间的事件同步等各种功能。至于这些功能的实现是通过调用这些内核对象的函数

46 \* 来完成的

47 \*

48 \*/

**49 /\* 定义二值信号量的ID变量 \*/**

**50 UINT32 BinarySem_Handle;**

51

52 /\* 全局变量声明 \/

53 /\*

54 \* 在写应用程序的时候，可能需要用到一些全局变量。

55 \*/

56

57

58 /\* 函数声明 \*/

59 static UINT32 AppTaskCreate(void);

60 static UINT32 Creat_HighPriority_Task(void);

61 static UINT32 Creat_MidPriority_Task(void);

62 static UINT32 Creat_LowPriority_Task(void);

63

64 static void HighPriority_Task(void);

65 static void MidPriority_Task(void);

66 static void LowPriority_Task(void);

67 static void BSP_Init(void);

68

69

70 /\*

71 \* @brief 主函数

72 \* @param 无

73 \* @retval 无

74 \* @note 第一步：开发板硬件初始化

75 第二步：创建App应用任务

76 第三步：启动LiteOS，开始多任务调度，启动失败则输出错误信息

77 \/

78 int main(void)

79 {

80 UINT32 uwRet = LOS_OK; //定义一个任务创建的返回值，默认为创建成功

81

82 /\* 板载相关初始化 \*/

83 BSP_Init();

84

85 printf("这是一个[野火]-STM32全系列开发板-LiteOS优先级翻转实验！\n\n");

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

**121 /\* 创建一个二值信号量*/**

**122 uwRet = LOS_BinarySemCreate(1,&BinarySem_Handle);**

**123 if (uwRet != LOS_OK) {**

**124 printf("BinarySem创建失败！失败代码0x%X\n",uwRet);**

**125 }**

126

127 uwRet = Creat_HighPriority_Task();

128 if (uwRet != LOS_OK) {

129 printf("HighPriority_Task任务创建失败！失败代码0x%X\n",uwRet);

130 return uwRet;

131 }

132

133 uwRet = Creat_MidPriority_Task();

134 if (uwRet != LOS_OK) {

135 printf("MidPriority_Task任务创建失败！失败代码0x%X\n",uwRet);

136 return uwRet;

137 }

138

139 uwRet = Creat_LowPriority_Task();

140 if (uwRet != LOS_OK) {

141 printf("LowPriority_Task任务创建失败！失败代码0x%X\n",uwRet);

142 return uwRet;

143 }

144

145 return LOS_OK;

146 }

147

148

149 /\*

150 \* @ 函数名 ： Creat_HighPriority_Task

151 \* @ 功能说明： 创建HighPriority_Task任务

152 \* @ 参数 ：

153 \* @ 返回值 ： 无

154 \/

155 static UINT32 Creat_HighPriority_Task()

156 {

157 //定义一个返回类型变量，初始化为LOS_OK

158 UINT32 uwRet = LOS_OK;

159

160 //定义一个用于创建任务的参数结构体

161 TSK_INIT_PARAM_S task_init_param;

162

163 task_init_param.usTaskPrio = 3; /\* 任务优先级，数值越小，优先级越高 \*/

164 task_init_param.pcName = "HighPriority_Task";/\* 任务名 \*/

165 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)HighPriority_Task;

166 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

167

168 uwRet = LOS_TaskCreate(&HighPriority_Task_Handle,&task_init_param);

169 return uwRet;

170 }

171 /\*

172 \* @ 函数名 ： Creat_MidPriority_Task

173 \* @ 功能说明： 创建MidPriority_Task任务

174 \* @ 参数 ：

175 \* @ 返回值 ： 无

176 \/

177 static UINT32 Creat_MidPriority_Task()

178 {

179 //定义一个返回类型变量，初始化为LOS_OK

180 UINT32 uwRet = LOS_OK;

181 TSK_INIT_PARAM_S task_init_param;

182

183 task_init_param.usTaskPrio = 4; /\* 任务优先级，数值越小，优先级越高 \*/

184 task_init_param.pcName = "MidPriority_Task"; /\* 任务名*/

185 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)MidPriority_Task;

186 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

187

188 uwRet = LOS_TaskCreate(&MidPriority_Task_Handle, &task_init_param);

189

190 return uwRet;

191 }

192

193 /\*

194 \* @ 函数名 ： Creat_MidPriority_Task

195 \* @ 功能说明： 创建MidPriority_Task任务

196 \* @ 参数 ：

197 \* @ 返回值 ： 无

198 \/

199 static UINT32 Creat_LowPriority_Task()

200 {

201 //定义一个返回类型变量，初始化为LOS_OK

202 UINT32 uwRet = LOS_OK;

203 TSK_INIT_PARAM_S task_init_param;

204

205 task_init_param.usTaskPrio = 5; /\* 任务优先级，数值越小，优先级越高 \*/

206 task_init_param.pcName = "LowPriority_Task"; /\* 任务名*/

207 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)LowPriority_Task;

208 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

209

210 uwRet = LOS_TaskCreate(&LowPriority_Task_Handle, &task_init_param);

211

212 return uwRet;

213 }

214

215 /\*

216 \* @ 函数名 ： HighPriority_Task

217 \* @ 功能说明： HighPriority_Task任务实现

218 \* @ 参数 ： NULL

219 \* @ 返回值 ： NULL

220 \/

**221 static void HighPriority_Task(void)**

**222 {**

**223 //定义一个返回类型变量，初始化为LOS_OK**

**224 UINT32 uwRet = LOS_OK;**

**225**

**226 /\* 任务都是一个无限循环，不能返回 \*/**

**227 while (1) {**

**228 //获取二值信号量 BinarySem_Handle,没获取到则一直等待**

**229 uwRet = LOS_SemPend( BinarySem_Handle , LOS_WAIT_FOREVER );**

**230 if (uwRet == LOS_OK)**

**231 printf("HighPriority_Task Running\n");**

**232**

**233 LED1_TOGGLE;**

**234 LOS_SemPost( BinarySem_Handle ); //释放二值信号量 BinarySem_Handle**

**235 LOS_TaskDelay ( 1000 ); /\* 延时100Ticks \*/**

**236 }**

**237 }**

238 /\*

239 \* @ 函数名 ： MidPriority_Task

240 \* @ 功能说明： MidPriority_Task任务实现

241 \* @ 参数 ： NULL

242 \* @ 返回值 ： NULL

243 \/

**244 static void MidPriority_Task(void)**

**245 {**

**246 /\* 任务都是一个无限循环，不能返回 \*/**

**247 while (1) {**

**248 printf("MidPriority_Task Running\n");**

**249 LOS_TaskDelay ( 1000 ); /\* 延时100Ticks \*/**

**250 }**

**251 }**

252

253 /\*

254 \* @ 函数名 ： LowPriority_Task

255 \* @ 功能说明： LowPriority_Task任务实现

256 \* @ 参数 ： NULL

257 \* @ 返回值 ： NULL

258 \/

**259 static void LowPriority_Task(void)**

**260 {**

**261 //定义一个返回类型变量，初始化为LOS_OK**

**262 UINT32 uwRet = LOS_OK;**

**263**

**264 static uint32_t i;**

**265**

**266 /\* 任务都是一个无限循环，不能返回 \*/**

**267 while (1) {**

**268 //获取二值信号量 BinarySem_Handle，没获取到则一直等待**

**269 uwRet = LOS_SemPend( BinarySem_Handle , LOS_WAIT_FOREVER );**

**270 if (uwRet == LOS_OK)**

**271 printf("LowPriority_Task Running\n");**

**272**

**273 LED2_TOGGLE;**

**274**

**275 for (i=0; i<2000000; i++) { //模拟低优先级任务占用信号量**

**276 //放弃剩余时间片，进行一次任务切换**

**277 LOS_TaskYield();**

**278 }**

**279 printf("LowPriority_Task 释放信号量!\r\n");**

**280 LOS_SemPost( BinarySem_Handle ); //释放二值信号量 BinarySem_Handle**

**281**

**282 LOS_TaskDelay ( 1000 ); /\* 延时100Ticks \*/**

**283 }**

**284 }**

285

286 /\*

287 \* @ 函数名 ： BSP_Init

288 \* @ 功能说明： 板级外设初始化，所有开发板上的初始化均可放在这个函数里面

289 \* @ 参数 ：

290 \* @ 返回值 ： 无

291 \/

292 static void BSP_Init(void)

293 {

294 /\*

295 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

296 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

297 \* 都统一用这个优先级分组，千万不要再分组，切忌。

298 \*/

299 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

300

301 /\* LED 初始化 \*/

302 LED_GPIO_Config();

303

304 /\* 串口初始化 \*/

305 USART_Config();

306

307 /\* 按键初始化 \*/

308 Key_GPIO_Config();

309 }

310

311

312

313 /END OF FILE/

.. _互斥锁实验-1:

互斥锁实验
^^^^^

互斥锁实验是基于优先级翻转实验进行修改的，将二值信号替换为互斥锁，目的是为了测试互斥锁的优先级继承机制是否有效，实验源码如代码清单 7‑11加粗部分所示。

代码清单 7‑11互斥锁实验

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

19 #include "los_mux.h"

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

**32 /\* 定义任务ID变量 \*/**

**33 UINT32 HighPriority_Task_Handle;**

**34 UINT32 MidPriority_Task_Handle;**

**35 UINT32 LowPriority_Task_Handle;**

36

37 /\* 内核对象ID \/

38 /\*

39 \* 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核

40 \* 对象，必须先创建，创建成功之后会返回一个相应的ID。实际上就是一个整数，后续

41 \* 就可以通过这个ID操作这些内核对象。

42 \*

43 \*

44 内核对象就是一种全局的数据结构，通过这些数据结构可以实现任务间的通信，

45 \* 任务间的事件同步等各种功能。至于这些功能的实现是通过调用这些内核对象的函数

46 \* 来完成的

47 \*

48 \*/

**49 /\* 定义互斥锁的ID变量 \*/**

**50 UINT32 Mutex_Handle;**

51

52 /\* 全局变量声明 \/

53 /\*

54 \* 在写应用程序的时候，可能需要用到一些全局变量。

55 \*/

56

57

58 /\* 函数声明 \*/

59 static UINT32 AppTaskCreate(void);

60 static UINT32 Creat_HighPriority_Task(void);

61 static UINT32 Creat_MidPriority_Task(void);

62 static UINT32 Creat_LowPriority_Task(void);

63

64 static void HighPriority_Task(void);

65 static void MidPriority_Task(void);

66 static void LowPriority_Task(void);

67 static void BSP_Init(void);

68

69

70 /\*

71 \* @brief 主函数

72 \* @param 无

73 \* @retval 无

74 \* @note 第一步：开发板硬件初始化

75 第二步：创建App应用任务

76 第三步：启动LiteOS，开始多任务调度，启动失败则输出错误信息

77 \/

78 int main(void)

79 {

80 UINT32 uwRet = LOS_OK; //定义一个任务创建的返回值，默认为创建成功

81

82 /\* 板载相关初始化 \*/

83 BSP_Init();

84

85 printf("这是一个[野火]-STM32全系列开发板-LiteOS互斥锁实验！\n\n");

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

**121 /\* 创建一个互斥锁*/**

**122 uwRet = LOS_MuxCreate(&Mutex_Handle);**

**123 if (uwRet != LOS_OK) {**

**124 printf("Mutex创建失败！失败代码0x%X\n",uwRet);**

**125 }**

126

127 uwRet = Creat_HighPriority_Task();

128 if (uwRet != LOS_OK) {

129 printf("HighPriority_Task任务创建失败！失败代码0x%X\n",uwRet);

130 return uwRet;

131 }

132

133 uwRet = Creat_MidPriority_Task();

134 if (uwRet != LOS_OK) {

135 printf("MidPriority_Task任务创建失败！失败代码0x%X\n",uwRet);

136 return uwRet;

137 }

138

139 uwRet = Creat_LowPriority_Task();

140 if (uwRet != LOS_OK) {

141 printf("LowPriority_Task任务创建失败！失败代码0x%X\n",uwRet);

142 return uwRet;

143 }

144

145 return LOS_OK;

146 }

147

148

149 /\*

150 \* @ 函数名 ： Creat_HighPriority_Task

151 \* @ 功能说明： 创建HighPriority_Task任务

152 \* @ 参数 ：

153 \* @ 返回值 ： 无

154 \/

155 static UINT32 Creat_HighPriority_Task()

156 {

157 //定义一个返回类型变量，初始化为LOS_OK

158 UINT32 uwRet = LOS_OK;

159

160 //定义一个用于创建任务的参数结构体

161 TSK_INIT_PARAM_S task_init_param;

162

163 task_init_param.usTaskPrio = 3; /\* 任务优先级，数值越小，优先级越高 \*/

164 task_init_param.pcName = "HighPriority_Task";/\* 任务名 \*/

165 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)HighPriority_Task;

166 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

167

168 uwRet = LOS_TaskCreate(&HighPriority_Task_Handle, &task_init_param);

169 return uwRet;

170 }

171 /\*

172 \* @ 函数名 ： Creat_MidPriority_Task

173 \* @ 功能说明： 创建MidPriority_Task任务

174 \* @ 参数 ：

175 \* @ 返回值 ： 无

176 \/

177 static UINT32 Creat_MidPriority_Task()

178 {

179 //定义一个返回类型变量，初始化为LOS_OK

180 UINT32 uwRet = LOS_OK;

181 TSK_INIT_PARAM_S task_init_param;

182

183 task_init_param.usTaskPrio = 4; /\* 任务优先级，数值越小，优先级越高 \*/

184 task_init_param.pcName = "MidPriority_Task"; /\* 任务名*/

185 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)MidPriority_Task;

186 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

187

188 uwRet = LOS_TaskCreate(&MidPriority_Task_Handle, &task_init_param);

189

190 return uwRet;

191 }

192

193 /\*

194 \* @ 函数名 ： Creat_MidPriority_Task

195 \* @ 功能说明： 创建MidPriority_Task任务

196 \* @ 参数 ：

197 \* @ 返回值 ： 无

198 \/

199 static UINT32 Creat_LowPriority_Task()

200 {

201 //定义一个返回类型变量，初始化为LOS_OK

202 UINT32 uwRet = LOS_OK;

203 TSK_INIT_PARAM_S task_init_param;

204

205 task_init_param.usTaskPrio = 5; /\* 任务优先级，数值越小，优先级越高 \*/

206 task_init_param.pcName = "LowPriority_Task"; /\* 任务名*/

207 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)LowPriority_Task;

208 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

209

210 uwRet = LOS_TaskCreate(&LowPriority_Task_Handle, &task_init_param);

211

212 return uwRet;

213 }

214

215 /\*

216 \* @ 函数名 ： HighPriority_Task

217 \* @ 功能说明： HighPriority_Task任务实现

218 \* @ 参数 ： NULL

219 \* @ 返回值 ： NULL

220 \/

**221 static void HighPriority_Task(void)**

**222 {**

**223 //定义一个返回类型变量，初始化为LOS_OK**

**224 UINT32 uwRet = LOS_OK;**

**225**

**226 /\* 任务都是一个无限循环，不能返回 \*/**

**227 while (1) {**

**228 //获取互斥锁,没获取到则一直等待**

**229 uwRet = LOS_MuxPend( Mutex_Handle , LOS_WAIT_FOREVER );**

**230 if (uwRet == LOS_OK)**

**231 printf("HighPriority_Task Running\n");**

**232**

**233 LED1_TOGGLE;**

**234 LOS_MuxPost( Mutex_Handle ); //释放互斥锁**

**235 LOS_TaskDelay ( 1000 ); /\* 延时100Ticks \*/**

**236 }**

**237 }**

238 /\*

239 \* @ 函数名 ： MidPriority_Task

240 \* @ 功能说明： MidPriority_Task任务实现

241 \* @ 参数 ： NULL

242 \* @ 返回值 ： NULL

243 \/

**244 static void MidPriority_Task(void)**

**245 {**

**246 /\* 任务都是一个无限循环，不能返回 \*/**

**247 while (1) {**

**248 printf("MidPriority_Task Running\n");**

**249 LOS_TaskDelay ( 1000 ); /\* 延时100Ticks \*/**

**250 }**

**251 }**

252

253 /\*

254 \* @ 函数名 ： LowPriority_Task

255 \* @ 功能说明： LowPriority_Task任务实现

256 \* @ 参数 ： NULL

257 \* @ 返回值 ： NULL

258 \/

**259 static void LowPriority_Task(void)**

**260 {**

**261 //定义一个返回类型变量，初始化为LOS_OK**

**262 UINT32 uwRet = LOS_OK;**

**263**

**264 static uint32_t i;**

**265**

**266 /\* 任务都是一个无限循环，不能返回 \*/**

**267 while (1) {**

**268 //获取互斥锁，没获取到则一直等待**

**269 uwRet = LOS_MuxPend( Mutex_Handle , LOS_WAIT_FOREVER );**

**270 if (uwRet == LOS_OK)**

**271 printf("LowPriority_Task Running\n");**

**272**

**273 LED2_TOGGLE;**

**274**

**275 for (i=0; i<2000000; i++) { //模拟低优先级任务占用信号量**

**276 //放弃剩余时间片，进行一次任务切换**

**277 LOS_TaskYield();**

**278 }**

**279 printf("LowPriority_Task 释放互斥锁!\r\n");**

**280 LOS_MuxPost( Mutex_Handle ); //释放互斥锁**

**281**

**282 LOS_TaskDelay ( 1000 ); /\* 延时100Ticks \*/**

**283 }**

**284 }**

285

286 /\*

287 \* @ 函数名 ： BSP_Init

288 \* @ 功能说明： 板级外设初始化，所有开发板上的初始化均可放在这个函数里面

289 \* @ 参数 ：

290 \* @ 返回值 ： 无

291 \/

292 static void BSP_Init(void)

293 {

294 /\*

295 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

296 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

297 \* 都统一用这个优先级分组，千万不要再分组，切忌。

298 \*/

299 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

300

301 /\* LED 初始化 \*/

302 LED_GPIO_Config();

303

304 /\* 串口初始化 \*/

305 USART_Config();

306

307 /\* 按键初始化 \*/

308 Key_GPIO_Config();

309 }

310

311

312 /END OF FILE/

实验现象
~~~~

模拟优先级翻转实验现象
^^^^^^^^^^^

将程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据读者买的开发板而定，每个型号的开发板都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，它里面输出了信息表明
任务正在运行中，并且很明确可以看到：高优先级任务在等待低优先级任务运行完毕才能获得信号量继续运行，而期间中优先级的任务一直能得到运行，如图 7‑4所示。

|mutex005|

图 7‑4优先级翻转实验现象

互斥锁实验现象
^^^^^^^

将程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据读者买的开发板而定，每个型号的开发板都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，它里面输出了信息表明
任务正在运行中，并且很明确可以看到：在低优先级任务运行的时候，中优先级任务无法抢占低优先级的任务，这是因为互斥锁的优先级继承机制，从而最大程度降低了优先级翻转产生的危害，如图 7‑5所示。

|mutex006|

图 7‑5互斥锁同步实验现象

.. |mutex002| image:: media\mutex002.png
   :width: 5.38542in
   :height: 3.04583in
.. |mutex003| image:: media\mutex003.png
   :width: 5.26181in
   :height: 2.88611in
.. |mutex004| image:: media\mutex004.png
   :width: 5.76806in
   :height: 3.14167in
.. |mutex005| image:: media\mutex005.png
   :width: 5.18056in
   :height: 4.09306in
.. |mutex006| image:: media\mutex006.png
   :width: 5.475in
   :height: 4.32639in

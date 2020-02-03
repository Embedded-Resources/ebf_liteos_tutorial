.. vim: syntax=rst

消息队列
----

不知读者是否在裸机编程中这样使用过数组？它用于存储数据，在需要的时候再从数组中读取。本章的消息队列是LiteOS也能实现数据存储的功能，并且更加完善，它在操作系统中常被用于传输数据。

消息队列的基本概念
~~~~~~~~~

队列又称消息队列，是一种常用于系统中通信的数据结构，队列可以在任务与任务间、中断和任务间传送信息，实现了接收来自任务或中断的不固定长度的消息，并根据LiteOS提供不同函数接口选择传递消息是否存放在自己空间。任务能够从队列中读取消息，当队列中的消息为空时，读取消息的任务将被阻塞，用户可以指定任务阻塞
的时间uwTimeOut，在这段时间中，如果队列的消息一直为空，该任务将保持阻塞状态以等待消息到了。当队列中有新消息时，阻塞的任务会被唤醒；当任务等待的时间超过了指定的阻塞时间，即使队列中依然没有消息，任务也会自动从阻塞态转为就绪态。消息队列是一种异步的通信方式。

通过消息队列服务，任务或中断服务例程可以将一条或多条消息放入消息队列中。同样，一个或多个任务可以从消息队列中获得消息。当有多个消息写入到消息队列时，通常是先进入的消息先传递给任务，也就是说，任务先得到的是最先进入队列的消息，即先进先出原则（First In First Out
，FIFO），但是也支持后进先出原则（Last In First Out，LIFO）。

用户在处理业务时，消息队列提供了异步处理机制，允许将一个消息放入队列，但并不立即处理它，这样子队列就起到缓存消息的作用。

LiteOS中使用队列数据结构实现任务异步通信工作，具有如下特性。

1. 消息以先进先出方式排队，支持异步读写工作方式。

2. 读队列和写队列都支持超时机制。

3. 写入消息类型由通信双方约定，可以允许不同长度（不超过消息节点最大值）的任意类型消息。

4. 消息支持后进先出方式排队（LIFO）。

5. 一个任务能够从任意一个消息队列读取和写入消息。

6. 多个任务能够从同一个消息队列读取和写入消息。

消息队列的运作机制
~~~~~~~~~

创建队列时，根据用户传入队列长度和消息节点大小来开辟相应的内存空间以供该队列使用，初始化消息队列的相关信息，创建成功后返回队列ID。

在队列控制块中维护一个消息头节点位置usQueueHead和一个消息尾节点位置usQueueTail变量，用于记录当前队列中消息存储情况。usQueueHead表示队列中被占用消息节点的起始位置，usQueueTail表示占用消息节点的结束位置（或者可以理解为队列中空闲消息的起始位置），在消息队列刚
创建时usQueueHead和usQueueTail均指向队列起始位置。

写队列时，根据usQueueTail找到消息节点末尾的空闲节点作为消息写入区域。如果usQueueTail已经指向队列尾则采用回卷方式（可以将LiteOS的消息队列看做是一个环形队列，这样子操作很方便，也能避免溢出，达到缓冲的效果）。根据usReadWriteableCnt
[OS_QUEUE_WRITE]判断队列是否可以写入，不能对已满的队列进行写队列操作。

读队列时，根据usQueueHead找到最先写入队列中的消息节点进行读取。如果usQueueHead已经指向队列尾也采用回卷方式。根据usReadWriteableCnt [OS_QUEUE_READ]判断队列是否有消息读取，对没有消息的队列进行读队列操作会引起任务挂起（假设用户指定阻塞时间的话）。

删除队列时，根据传入的队列ID寻找到对应的队列，把队列状态置为未使用，释放原队列所占的空间，对应的队列控制头置为初始状态。

LiteOS的消息队列采用两个双向链表来维护，一个链表指向消息队列的头部，一个链表指向消息队列的尾部（stReadWriteList[QUEUE_HEAD_TAIL]），通过访问这两个链表就能直接访问对应的消息空间（消息空间中的每个节点称之为消息节点），并且通过消息队列控制块中的读写类型（usRea
dWriteableCnt[QUEUE_READ_WRITE]）来操作消息队列，消息队列的运作过程如图 5‑1所示。

|messag002|

图 5‑1消息队列运作过程

消息队列的读写运作如图 5‑2所示。

|messag003|

图 5‑2队列读写消息操作示意图

消息队列的传输机制
~~~~~~~~~

既然队列是任务间通信的数据结构，那么它必然是可以存储消息数据的，消息是存储在消息节点中，而消息节点的大小在创建队列的时候由用户指定。LiteOS提供的队列是一种先进先出线性表，只允许在一端插入，在另一端进行读取（出队），支持异步读写工作方式，就像来买车票的人一样，先到的人先买到票，后到的人后买到票，
不允许插队。当然除此之外，LiteOS也提供一种后进先出的队列操作方式，这种方式能支持传输紧急的消息，在某些场合也是会比较常用，就像插队一样，后来买票的人能先买到票。

一般来说，数据的传递是有复制与引用传递两种方式，所谓复制就是将某个数据直接复制到另一个存储数据的地方，就像在电脑中将某个文件复制到另一个文件中，这两个文件都是一模一样的，修改源文件并不会影响已经复制的文件，但是文件占用的内存是同样的。而引用传递则是传递数据的指针，该指针指向源数据存储的地址，就好比是
从电脑中的源文件创建了一个文件的快捷方式，通过快捷方式也能打开源文件，并且快捷方式的占用内存是非常小的，但是有个缺点，假如修改了源文件的内容，那么，通过快捷方式打开的文件，其内容也会相应被修改，这样子就造成数据的可变性，在某些场合下是不安全的。

LiteOS提供了两种消息的传输方式，一种是复制方式，另一种是引用方式，通过上文的类比，读者可以选择自己需要的消息传输方式。作者这里有个小小的建议，读者可以根据消息的大小与重要性来选择消息的传递方式，假如消息是很重要的话，选择复制的方式会更加安全，假如消息的数据量很小的话，也是可以选择复制的方式。假
如消息只是一些不重要的内容或者消息数据量很大，可以选择引用方式。

消息队列的阻塞机制
~~~~~~~~~

在系统中创建了一个队列，每个任务都可以去对它进行读写操作，但是为了保护每个任务对它进行读写操作的过程，则必须要有阻塞机制，在某个任务对它读写操作的时候，必须保证该任务能正常完成读写操作，而不受后来的任务干扰。除此之外，当队列已满的时候，其他任务就不能将消息写入而导致消息的覆盖，同理，当队列为空的时候
，读取消息的任务也无法读取消息，这种机制可以称之为阻塞机制。

出队阻塞
^^^^

假设有一个任务A对某个队列进行读操作的时候（也就是出队），发现它没有消息，那么此时任务A有三个选择：第一个选择，任务A不进行等待，既然队列没有消息，那任务A也不必阻塞等待消息的到来，这样子任务A不会进入阻塞态；第二个选择，任务A阻塞等待，其等待时间由用户定义，比如可以是1000个Tick，在超时时间
到来之前假如队列有消息了，那任务A恢复就绪态，读取队列的消息，如果任务A刚好是最高优先级的就绪态，那么系统将进行一次任务调度；假如已经超出等待的时间，队列还没有消息可以读取，那任务A将恢复为就绪态继续运行；第三个选择，任务A进入阻塞态一直等待消息的到来，直到完成读取队列的消息。

入队阻塞
^^^^

同理，对某个队列的写操作也是一样的（写操作就是将消息写入队列中，也就是入队），当任务A向某个队列中写入一个消息，发现这个队列已经满了， LiteOS出于对队列中消息的保护，使这个队列无法被写入消息，如此一来任务的写操作就会被阻塞。在消息入队的时候，当且仅当队列允许入队时，任务才能成功写入消息；队列中
无可用消息节点时，说明消息队列已满，此时，系统会根据用户指定的阻塞超时时间将任务阻塞，在指定的超时时间内如果还不能完成入队操作，写入消息的任务会收到一个错误代码LOS_ERRNO_QUEUE_ISFULL，然后解除阻塞状态；当然，只有在任务中写入消息才允许进行阻塞状态，而在中断中写入消息不允许带有阻
塞机制，用户必须将阻塞时间设置为0，否则就直接返回错误代码LOS_ERRNO_QUEUE_READ_IN_INTERRUPT，因为写入消息的上下文环境是在中断中，不允许出现阻塞的情况。

假如有多个任务阻塞在一个消息队列中，那么这些阻塞的任务将按照任务优先级进行排序，优先级高的任务将优先获得队列的访问权。

消息队列应用场景
~~~~~~~~

消息队列可以应用于传递不定长消息的场合，包括任务与任务间的消息传递，中断和任务间传递信息。

常用Queue错误代码说明
~~~~~~~~~~~~~

在LiteOS中，与队列相关的函数大多数都会有返回值，其返回值是一些错误代码，方便使用者进行调试，下面列出一些常见的错误代码与参考解决方案如表 5‑1所示。

表 5‑1常用队列错误代码说明

.. list-table::
   :widths: 25 25 25 25
   :header-rows: 0


   * - 序号 |
     - 义              | 描述
     - | 参考解决
     - 案      |

   * - 1
     - LOS_ERRNO_ QUEUE_MAXNUM_ZERO
     - 队列资源          | 配置 的最大数目配置为0 | 大于0的队列
     - |


       |

   * - 2
     - LOS_ERRN O_QUEUE_NO_MEMORY
     - 队列              | 块内存无法初始化  | 列块分配更大
     - 队              | 内  | 存分区，或减少队  | 列资源的最大数量  |

   * - 3
     - LOS_ERRNO_QUEUE _CREATE_NO_MEMORY
     - 队列创建          | 为队 的内存未能被请求  | 内存，或减少
     - 分配更多的  | 创  | 建的队列中的队列  | 长度和节点的数目  |

   * - 4
     - LOS_ERRNO_Q UEUE_SIZE_TOO_BIG
     - 队列创建时        | 更改创 消息长度超过上限  | 队列中最大消
     - |

        |

   * - 5
     - LOS_ERR NO_TSK_ENTRY_NULL
     - 已超过创建的      | 增加队 队列的数量的上限  | 列的配置资源
     - |

   * - 6
     - LOS_ERRN O_QUEUE_NOT_FOUND
     - 无效的队列        | 确
     - |

   * - 7
     - LOS_ERRNO_Q UEUE_PEND_IN_LOCK
     - 当                | 任务被锁定时，禁  | 用队列前解锁 止在队列中被阻塞  |
     - 使                | 务  | |

   * - 8
     - LOS_ER RNO_QUEUE_TIMEOUT
     - 等待处            | 检 理队列的时间超时  | 超时时间是否
     - 设置的        | 适  |

   * - 9
     - LOS_ERRN O_QUEUE_IN_TSKUSE
     - 阻塞任务          | 使任 的队列不能被删除  | 能够获得资源
     - |

   * - 10
     - LOS_ERRNO_QUEUE_W RITE_IN_INTERRUPT
     - 在中断处理        | 将写队 程序中不能写队列  | 列设为非阻塞
     - |

   * - 11
     - LOS_ERRNO _QUEUE_NOT_CREATE
     - 队列未创建        | 检查队
     - 中        | 传递的ID是否有效  |

   * - 12
     - LOS_ERRNO_ QUEUE_IN_TSKWRITE
     - 队列读写不同步    | 同步队列的
     - 写    |

   * - 13
     - LOS_ERRNO_QUE UE_CREAT_PTR_NULL
     - 队列创建过程中传  | 确保传递 递的参数为空指针  | 的参数不为空
     - |

   * - 14
     - LOS_ERRNO_ QUEUE_PARA_ISZERO
     - 队列创建过程      | 传入正确 中传递的队列长度  | 度和消息节点 或消息节点大小为0 |
     - 队列长  | 小  | |

   * - 15
     - LOS_ERRNO_Q UEUE_READ_INVALID
     - 读取的            | 检 队列的handle无效  | 的ha
     - 队列中传递    | dle是否有效  |

   * - 16
     - LOS_ERRNO_QU EUE_READ_PTR_NULL
     - 队列读取过程      | 检查指针 中传递的指针为空  | 中传递的是否
     - |

   * - 17
     - LOS_ERRNO_QUEU E_READSIZE_ISZERO
     - 队列读取过程中传  | 通过一个 递的缓冲区大小为0 | 正确的缓冲区
     - |

   * - 18
     - LOS_ERRNO_QU EUE_WRITE_INVALID
     - 队                | 列写入过程中传递  | 的handl 的队列handle无效  |
     - 检查队列中传递    | 是否有效  | |

   * - 19
     - LOS_ERRNO_QUE UE_WRITE_PTR_NULL
     - 队列写入过程      | 检查指针 中传递的指针为空  | 中传递的是否
     - |

   * - 20
     - LOS_ERRNO_QUEUE _WRITESIZE_ISZERO
     - 队列写入过程中传  | 通过一个 递的缓冲区大小为0 | 正确的缓冲区
     - |

   * - 21
     - LOS_ERRNO_QUEUE _WRITE_NOT_CREATE
     - 写入              | 消息的队列未创建  |
     - 入有效队列ID    | |

   * - 22
     - LOS_ERRNO_QUEUE_W RITE_SIZE_TOO_BIG
     - 队列写入过程      | 减少缓冲 中传递的缓冲区大  | ，或增大队列 小比队列大小要大  |
     - 大小    | 点  | |

   * - 23
     - LOS_E RRNO_QUEUE_ISFULL
     - 在                | 队列写入过程中没  | 队列写入之前 有可用的空闲节点  | 以使用空闲的
     - 确保在            | 可  | 点  |

   * - 24
     - LOS_ERR NO_QUEUE_PTR_NULL
     - 正在获取队列信息  | 检查指针 时传递的指针为空  | 中传递的是否
     - |

   * - 25
     - LOS_ERRNO_QUEUE_ READ_IN_INTERRUPT
     - 在中断处理        | 将读队 程序中不能读队列  | 列设为非阻塞
     - |

   * - 26
     - L OS_ERRNO_QUEUE_MA IL_HANDLE_INVALID
     - 正在释放队        | 检查队 列的内存时传递的  | 的handl 队列的handle无效  |
     - 中传递    | 是否有效  | |

   * - 27
     - LOS_ERRNO_QUEUE _MAIL_PTR_INVALID
     - 传入的消          | 检查 息内存池指针为空  |
     - 针是否为空  | |

   * - 28
     - LOS_ERRNO_QUEU E_MAIL_FREE_ERROR
     - m embox内存释放失败 | 空mem
     - 传入非            | x内存指针  |

   * - 29
     - LOS_ERRNO_QUEU E_READ_NOT_CREATE
     - 待                | 读取的队列未创建  |
     - 传入有效队列ID    | |

   * - 30
     - LOS_ER RNO_QUEUE_ISEMPTY
     - 队列已空          | 确保
     - 读          | 取队列时包含消息  |

   * - 31
     - L OS_ERRNO_QUEUE_RE AD_SIZE_TOO_SMALL
     - 读缓冲区          | 增 大小小于队列大小  | 加缓冲区大小
     - |


常用消息队列的函数讲解
~~~~~~~~~~~

使用消息队列的典型流程如下。

1. 创建消息队列LOS_QueueCreate()。

2. 创建成功后，可以得到消息队列的ID值。

3. 写队列操作函数LOS_QueueWrite()。

4. 读队列操作函数LOS_QueueRead()。

5. 删除队列LOS_QueueDelete()。

消息队列创建函数LOS_QueueCreate()
^^^^^^^^^^^^^^^^^^^^^^^^^

消息队列创建函数LOS_QueueCreate()用于创建一个队列，读者可以根据自己的需要去创建队列，可以指定队列的长度以及消息节点的大小等信息，LiteOS创建队列的函数原型如代码清单 5‑1所示。

创建消息队列时系统会先给消息队列分配一块内存空间，这块内存的大小等于(单个消息节点大小+4个字节)与消息队列长度的乘积，接着再初始化消息队列，此时消息队列为空。LiteOS的消息队列控制块由多个元素组成，当系统初始化时，系统会为控制块分配对应的内存空间，用于保存消息队列的基本信息如消息的存储位置，头
指针usQueueHead、尾指针usQueueTail、消息大小usQueueSize以及队列长度usQueueLen等。在消息队列创建成功的时候，这些内存就被占用了，只有删除了消息队列的时候，这段内存才会被释放掉，创建成功的队列已经确定队列的长度与消息节点的大小，且无法再次更改，每个消息节点可以
存放不大于消息大小usQueueSize的任意类型的消息，消息节点个数的总和就是队列的长度，用户可以在消息队列创建时指定。

代码清单 5‑1队列创建函数LOS_QueueCreate()函数原型

1 extern UINT32 LOS_QueueCreate(CHAR \*pcQueueName, **(1)**

2 UINT16 usLen, **(2)**

3 UINT32 \*puwQueueID, **(3)**

4 UINT32 uwFlags, **(4)**

5 UINT16 usMaxMsgSize); **(5)**

代码清单 5‑1\ **(1)**\ ：pcQueueName是消息队列名称，LiteOS保留，暂时未使用。

代码清单 5‑1\ **(2)**\ ：usLen是队列长度，值范围是1~0xFFFF。

代码清单 5‑1\ **(3)**\ ：puwQueueID是消息队列ID变量指针，该变量用于保存创建队列成功时返回的消息队列ID，由用户定义，对消息队列的读写操作都是通过消息队列ID来操作的。

代码清单 5‑1\ **(4)**\ ：uwFlags是队列模式，保留参数，暂不使用。

代码清单 5‑1\ **(5)**\ ：usMaxMsgSize是消息节点大小（单位为字节），其取值范围为1~(0xFFFF-4)。

队列控制块与任务控制类似，每一个队列都由对应的队列控制块维护，队列控制块中包含了队列的所有信息，比如队列的一些状态信息，使用情况等，如代码清单 5‑2所示。

代码清单 5‑2队列控制块

1 typedef struct tagQueueCB {

2 UINT8 \*pucQueue; /**< 队列指针 \*/

3 UINT16 usQueueState; /**< 队列状态 \*/

4 UINT16 usQueueLen; /**< 队列中消息个数 \*/

5 UINT16 usQueueSize; /**< 消息节点大小 \*/

6 UINT16 usQueueID; /**< 队列ID \*/

7 UINT16 usQueueHead; /**< 消息头节点位置（数组下标）*/

8 UINT16 usQueueTail; /**< 消息尾节点位置（数组下标）*/

9 UINT16 usReadWriteableCnt[2]; /**< 可读或可写资源的计数，0：可读，1：可写\* /

10 LOS_DL_LIST stReadWriteList[2]; /**< 指向要读取或写入的链表的指针，0：读列表，1：写列表/

11 LOS_DL_LIST stMemList; / \*\* <指向内存链表的指针\* /

12 } QUEUE_CB_S;

创建队列必须是调用LOS_QueueCreate()函数进行创建，在创建成功后返回一个队列ID。在创建队列时会返回创建的情况的，如果返回LOS_OK，则表明队列创建成功，若是其他错误代码，读者可以根据表 5‑1定位错误并解决，创建消息队列的应用实例如代码清单 5‑3加粗部分所示，其源码如代码清单
5‑4所示。

代码清单 5‑3队列创建函数LOS_QueueCreate()实例

1 UINT32 uwRet = LOS_OK;/\* 定义一个创建队列的返回类型，初始化为创建成功的返回值 \*/

2

**3 /\* 创建一个测试队列*/**

**4 uwRet = LOS_QueueCreate("Test_Queue", /\* 队列的名称，保留，未使用*/**

**5 128, /\* 队列的长度 \*/**

**6 &Test_Queue_Handle, /\* 队列的ID(句柄) \*/**

**7 0, /\* 队列模式，官方暂时未使用 \*/**

**8 16); /\* 最大消息大小（字节）*/**

9 if (uwRet != LOS_OK)

10 {

11 printf("Test_Queue队列创建失败！\n");

12 }

代码清单 5‑4队列创建函数LOS_QueueCreate()源码

1 /\*

2 Function : LOS_QueueCreate

3 Description : 创建一个队列

4 Input : pcQueueName --- 队列名称，官方保留未用

5 usLen --- 队列长度

6 uwFlags --- 队列模式，FIFO或PRIO，官方保留未用

7 usMaxMsgSize --- 最大消息大小（字节）

8 Output : puwQueueID --- 队列ID

9 Return : LOS_OK表示成功或失败时其他的错误代码

10 \/

11 LITE_OS_SEC_TEXT_INIT UINT32 LOS_QueueCreate(CHAR \*pcQueueName,

12 UINT16 usLen,

13 UINT32 \*puwQueueID,

14 UINT32 uwFlags,

15 UINT16 usMaxMsgSize )

16 {

17 QUEUE_CB_S \*pstQueueCB;

18 UINTPTR uvIntSave;

19 LOS_DL_LIST \*pstUnusedQueue;

20 UINT8 \*pucQueue;

21 UINT16 usMsgSize = usMaxMsgSize + sizeof(UINT32);

22

23 (VOID)pcQueueName; **(1)**

24 (VOID)uwFlags;

25

26 if (NULL == puwQueueID) { **(2)**

27 return LOS_ERRNO_QUEUE_CREAT_PTR_NULL;

28 }

29

30 if (usMaxMsgSize > OS_NULL_SHORT -4) {

31 return LOS_ERRNO_QUEUE_SIZE_TOO_BIG;

32 }

33

34 if ((0 == usLen) \|\| (0 == usMaxMsgSize)) { **(3)**

35 return LOS_ERRNO_QUEUE_PARA_ISZERO;

36 }

37

38

39

40 pucQueue = (UINT8 \*)LOS_MemAlloc(m_aucSysMem0, usLen \* usMsgSize);\ **(4)**

41 if (NULL == pucQueue) {

42 return LOS_ERRNO_QUEUE_CREATE_NO_MEMORY;

43 }

44

45 uvIntSave = LOS_IntLock();

46 if (LOS_ListEmpty(&g_stFreeQueueList)) { **(5)**

47 LOS_IntRestore(uvIntSave);

48 (VOID)LOS_MemFree(m_aucSysMem0, pucQueue);

49 return LOS_ERRNO_QUEUE_CB_UNAVAILABLE;

50 }

51

52 pstUnusedQueue = LOS_DL_LIST_FIRST(&(g_stFreeQueueList)); **(6)**

53 LOS_ListDelete(pstUnusedQueue);

54 pstQueueCB = (GET_QUEUE_LIST(pstUnusedQueue));

55 pstQueueCB->usQueueLen = usLen; **(7)**

56 pstQueueCB->usQueueSize = usMsgSize; **(8)**

57 pstQueueCB->pucQueue = pucQueue; **(9)**

58 pstQueueCB->usQueueState = OS_QUEUE_INUSED;

59 pstQueueCB->usReadWriteableCnt[OS_QUEUE_READ] = 0; **(10)**

60 pstQueueCB->usReadWriteableCnt[OS_QUEUE_WRITE] = usLen; **(11)**

61 pstQueueCB->usQueueHead = 0; **(12)**

62 pstQueueCB->usQueueTail = 0;

63 LOS_ListInit(&pstQueueCB->stReadWriteList[OS_QUEUE_READ]); **(13)**

64 LOS_ListInit(&pstQueueCB->stReadWriteList[OS_QUEUE_WRITE]);

65 LOS_ListInit(&pstQueueCB->stMemList);

66 LOS_IntRestore(uvIntSave);

67

68 \*puwQueueID = pstQueueCB->usQueueID; **(14)**

69

70 return LOS_OK;

71 }

代码清单 5‑4\ **(1)**\ ：由于LiteOS对队列的名称、队列模式等进行了保留，未使用，所以，传进来的队列名称与队列模式参数会强制被转换成空类型。

代码清单 5‑4\ **(2)**\ ：如果传递进来的队列ID指针puwQueueID为NULL，则返回错误代码。

代码清单 5‑4\ **(3)**\ ：如果传递进来的usMaxMsgSize过大或者是为0，则返回错误代码。

代码清单 5‑4\ **(4)**\ ：使用LOS_MemAlloc为队列分配内存，分配的大小根据传递进来的usLen（队列长度）与usMaxMsgSize（消息节点大小（字节））进行动态分配。

代码清单 5‑4\ **(5)**\ ：判断一下系统当前是否还可以创建消息队列，因为在系统配置中已经定义了最大可创建的消息队列个数，并且在系统核心初始化的时候将可以创建的消息队列进行初始化，采用空闲消息队控制块列表进行管理，此时如果g_stFreeQueueList为空，那么表示系统当前的消息队列已
经达到支持的最大，无法进行创建，所以刚刚申请的内存就需要调用LOS_MemFree()函数进行释放，然后返回一个错误代码LOS_ERRNO_QUEUE_CB_UNAVAILABLE。用户可以在traget_config.h文件修改宏定义LOSCFG_BASE_IPC_QUEUE_LIMIT，以增加系
统支持的消息队列个数。

代码清单 5‑4\ **(6)**\ ：从系统管理的空闲消息队列控制块列表中取下一个消息队列控制块，表示消息队列已经被创建。

代码清单 5‑4\ **(7)**\ ：创建一个队列的具体过程，根据传进来的参数进行配置队列的长度usLen。

代码清单 5‑4\ **(8)**\ ：配置消息队列的每个消息节点的大小usMsgSize。

代码清单 5‑4\ **(9)**\ ：配置消息队列存放消息的起始地址pucQueue，即消息空间的内存地址，并且将消息队列的状态要设置为OS_QUEUE_INUSED表示队列已使用。

代码清单 5‑4\ **(10)**\ ：初始化消息队列可读的消息个数为0。

代码清单 5‑4\ **(11)**\ ：初始化消息队列可写的消息个数是usLen。

代码清单 5‑4\ **(12)**\ ：创建消息队列时，usQueueHead和usQueueTail都是0，也就是指向初始位置，随着消息队列的读写，这两个指针位置会改变。

代码清单 5‑4\ **(13)**\ ：初始化读写操作的消息空间的链表。

代码清单 5‑4\ **(14)**\ ：将队列ID通过puwQueueID指针返回给用户，后续用户可以使用这个队列ID即可对队列操作，创建完成之后返回LOS_OK。

消息队列删除函数LOS_QueueDelete()
^^^^^^^^^^^^^^^^^^^^^^^^^

队列删除函数是根据队列ID直接删除的，删除之后这个队列的所有信息都会被系统回收清空，而且不能再次使用这个队列了，但是需要注意的是，队列在使用或者阻塞中是不能被删除的，如果某个队列没有被创建，那也是无法被删除的，uwQueueID是LOS_QueueDelete()函数传入的参数，是队列ID，表示的是
要删除哪个队列，其函数原型如代码清单 5‑5所示。

代码清单 5‑5 LOS_TaskDelete()函数原型

1 /*\*

2 \* 此API用于删除队列。

3 \* 此API不能用于删除未创建的队列。

4 \* 如果同步队列被阻塞，或正在读取或写入某些队列，则同步队列将无法删除。

5 \*/

6 extern UINT32 LOS_QueueDelete(UINT32 uwQueueID);

队列删除函数的实例：如代码清单 5‑6加粗部分所示，如果队列删除成功，则返回LOS_OK，否则返回其他错误代码。

代码清单 5‑6 LOS_TaskDelete()函数使用实例

1 UINT32 uwRet = LOS_OK;/\* 定义一个删除队列的返回类型，初始化为删除成功的返回值 \*/

2

**3 uwRet = LOS_QueueDelete(Test_Queue_Handle); /\* 删除队列 \*/**

**4 if (uwRet != LOS_OK) /\* 删除队列失败，返回其他错误代码 \*/**

**5 {**

**6 printf("删除队列失败！\n");**

**7 } else /\* 删除队列成功，返回LOS_OK \*/**

**8 {**

**9 printf("删除队列成功！\n");**

**10 }**

LOS_TaskDelete()函数的实现如代码清单 5‑7所示。

代码清单 5‑7 LOS_TaskDelete()函数源码

1 /\*

2 Function : LOS_QueueDelete

3 Description : 删除一个队列

4 Input : puwQueueID --- 队列ID

5 Output : None

6 Return : LOS_OK表示成功或失败时返回其他错误代码

7 \/

8 LITE_OS_SEC_TEXT_INIT UINT32 LOS_QueueDelete(UINT32 uwQueueID)

9 {

10 QUEUE_CB_S \*pstQueueCB;

11 UINT8 \*pucQueue = NULL;

12 UINTPTR uvIntSave;

13 UINT32 uwRet;

14

15 if (uwQueueID >= LOSCFG_BASE_IPC_QUEUE_LIMIT) { **(1)**

16 return LOS_ERRNO_QUEUE_NOT_FOUND;

17 }

18

19 uvIntSave = LOS_IntLock();

20 pstQueueCB = (QUEUE_CB_S \*)GET_QUEUE_HANDLE(uwQueueID); **(2)**

21 if (OS_QUEUE_UNUSED == pstQueueCB->usQueueState) {

22 uwRet = LOS_ERRNO_QUEUE_NOT_CREATE;

23 goto QUEUE_END;

24 }

25

26 if (!LOS_ListEmpty(&pstQueueCB->stReadWriteList[OS_QUEUE_READ])) {**(3)**

27 uwRet = LOS_ERRNO_QUEUE_IN_TSKUSE;

28 goto QUEUE_END;

29 }

30

31 if (!LOS_ListEmpty(&pstQueueCB->stReadWriteList[OS_QUEUE_WRITE])) {**(4)**

32 uwRet = LOS_ERRNO_QUEUE_IN_TSKUSE;

33 goto QUEUE_END;

34 }

35

36 if (!LOS_ListEmpty(&pstQueueCB->stMemList)) { **(5)**

37 uwRet = LOS_ERRNO_QUEUE_IN_TSKUSE;

38 goto QUEUE_END;

39 }

40

41 if ((pstQueueCB->usReadWriteableCnt[OS_QUEUE_WRITE] + pstQueueCB->

42 usReadWriteableCnt[OS_QUEUE_READ]) != pstQueueCB->usQueueLen) {

43 uwRet = LOS_ERRNO_QUEUE_IN_TSKWRITE; **(6)**

44 goto QUEUE_END;

45 }

46

47 pucQueue = pstQueueCB->pucQueue;

48 pstQueueCB->pucQueue = (UINT8 \*)NULL;

49 pstQueueCB->usQueueState = OS_QUEUE_UNUSED; **(7)**

50 LOS_ListAdd(&g_stFreeQueueList, &pstQueueCB->stReadWriteList[OS_QUEUE_WRITE]);

51 LOS_IntRestore(uvIntSave);

52

53 uwRet = LOS_MemFree(m_aucSysMem0, (VOID \*)pucQueue); **(8)**

54 return uwRet;

55

56 QUEUE_END:

57 LOS_IntRestore(uvIntSave);

58 return uwRet;

59 }

代码清单 5‑7\ **(1)**\ ：判断队列ID是否有效，如果是无效的队列，则返回错误代码。

代码清单 5‑7\ **(2)**\ ：根据队列ID获取对应的队列控制块，并且获取队列当前状态，如果队列是未使用状态，则返回错误代码。

代码清单 5‑7\ **(3)**\ ：如果当前系统中有任务在等待队列中的消息，那么这个队列是无法被删除的，返回错误代码。

代码清单 5‑7\ **(4)**\ ：如果当前系统有任务等待写入消息到队列中，那么这个队列也是无法被删除的，返回错误代码。

代码清单 5‑7\ **(5)**\ ：如果当前队列非空，系统为了保证任务获得资源，此时的队列也是无法被删除的，返回错误代码。

代码清单 5‑7\ **(6)**\ ：如果队列的读写是不同步的，那么返回错误代码。

代码清单 5‑7\ **(7)**\ ：将要删除的队列变为未使用状态，并且添加到消息队列控制块空闲列表中，归还给系统，以便系统创建可以新的消息队列。

代码清单 5‑7\ **(8)**\ ：将队列的内存进行释放。

消息队列写消息函数
^^^^^^^^^

不带复制方式写入LOS_QueueWrite()
''''''''''''''''''''''''

任务或者中断服务程序都可以给消息队列写入消息，当写入消息时，如果队列未满，LiteOS会将消息复制到消息队列队尾，否则，会根据用户指定的阻塞超时时间进行阻塞，在这段时间中，如果队列还是满的，该任务将保持阻塞状态以等待队列有空闲的消息节点。如果系统中有任务从其等待的队列中读取了消息（队列未满），该任务
将自动由阻塞态转为就绪态。当任务等待的时间超过了指定的阻塞时间，即使队列中还是满的，任务也会自动从阻塞态变成就绪态，此时写入消息的任务或者中断程序会收到一个错误代码LOS_ERRNO_QUEUE_ISFULL。

同时LiteOS支持后进先出（LIFO）方式写入消息，即支持写入紧急消息，写入紧急消息的过程与普通写入消息几乎一样，唯一的不同是，当写入紧急消息时，写入的位置是消息队列队头而非队尾，这样读取任务就能够优先读取到紧急消息，从而及时进行消息处理。

LiteOS消息队列的传递方式有两种，一种是不带复制传递消息，另一种是带复制传递消息，不带复制传递消息的函数原型如代码清单 5‑8所示，其实验实例如代码清单 5‑9加粗部分所示。

代码清单 5‑8 LOS_QueueWrite()函数原型

1 extern UINT32 LOS_QueueWrite(UINT32 uwQueueID, **(1)**

2 VOID \*pBufferAddr, **(2)**

3 UINT32 uwBufferSize, **(3)**

4 UINT32 uwTimeOut); **(4)**

代码清单 5‑8\ **(1)**\ ：uwQueueID是队列ID，由LOS_QueueCreate()函数返回的，其值范围为1~LOSCFG_BASE_IPC_QUEUE_LIMIT。

代码清单 5‑8\ **(2)**\ ：pBufferAddr：消息的起始地址。

代码清单 5‑8\ **(3)**\ ：uwBufferSize是写入消息的大小。

代码清单 5‑8\ **(4)**\ ：uwTimeOut是等待时间，其值范围为0~LOS_WAIT_FOREVER，单位为Tick，当uwTimeOut为0的时候是不等待，为LOS_WAIT_FOREVER时候是一直等待，在中断中使用该函数uwTimeOut的值必须为0。

代码清单 5‑9 LOS_QueueWrite()函数实例

1 /\*

2 \* @ 函数名 ： Send_Task

3 \* @ 功能说明： 通过按键进行对队列的写操作

4 \* @ 参数 ：

5 \* @ 返回值 ： 无

6 \/

7 UINT32 send_data1 = 1; /\* 写入队列的第一个消息 \*/

8 UINT32 send_data2 = 2; /\* 写入队列的第二个消息 \*/

9 static void Send_Task(void)

10 {

11 UINT32 uwRet = LOS_OK; /\* 定义一个返回类型，初始化为成功的返回值 \*/

12 /\* 任务都是一个无限循环，不能返回 \*/

13 while (1) { /\* K1 被按下 \*/

14 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {

**15 /\* 将消息写入到队列中，等待时间为 0 \*/**

**16 uwRet = LOS_QueueWrite(Test_Queue_Handle, /\* 写入的队列ID \*/**

**17 &send_data1, /\* 写入的消息 \*/**

**18 sizeof(send_data1),/\* 消息的大小 \*/**

**19 0); /\* 等待时间为 0 \*/**

20 if (LOS_OK != uwRet) {

21 printf("消息不能写入到消息队列！错误代码0x%x \\n",uwRet);

22 }/\* K2 被按下 \*/

23 } else if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {

**24 /\* 将消息写入到队列中，等待时间为 0 \*/**

**25 uwRet = LOS_QueueWrite(Test_Queue_Handle, /\* 写入的队列ID \*/**

**26 &send_data2, /\* 写入的消息 \*/**

**27 sizeof(send_data2), /\* 消息的长度 \*/**

**28 0); /\* 等待时间为 0 \*/**

29 if (LOS_OK != uwRet) {

30 printf("消息不能写入到消息队列！错误代码0x%x \\n",uwRet);

31 }

32

33 }

34 /\* 20Ticks扫描一次 \*/

35 LOS_TaskDelay(20);

36 }

37 }

写入队列按照LiteOS的API进行操作即可，但是有几个点需要注意。

1. 在使用写入队列的操作前应先创建要写入的队列。

2. 在中断上下文环境中，必须使用非阻塞模式写入，也就是等待时间为0个Tick。

3. 在初始化LiteOS之前无法调用此API。

4. 将写入由uwBufferSize指定大小的消息，该值不能大于消息节点的大小。

5. 写入队列节点中的是消息的地址。

LOS_QueueWrite()函数的源码具体实现如代码清单 5‑10所示。

代码清单 5‑10 LOS_QueueWrite()函数源码

1 LITE_OS_SEC_TEXT UINT32 LOS_QueueWrite(UINT32 uwQueueID,

2 VOID \*pBufferAddr,

3 UINT32 uwBufferSize,

4 UINT32 uwTimeOut)

5 {

6 if (pBufferAddr == NULL) {

7 return LOS_ERRNO_QUEUE_WRITE_PTR_NULL;

8 }

9 uwBufferSize = sizeof(UINT32*);

10 return LOS_QueueWriteCopy(uwQueueID,

11 &pBufferAddr,

12 uwBufferSize,

13 uwTimeOut);

14 }

其实代码很简单，LiteOS实际上是对LOS_QueueWriteCopy()函数进行封装，该函数会在下文进行讲解。只不过在该函数中复制的是消息的地址，而非内容。

带复制写入LOS_QueueWriteCopy()
'''''''''''''''''''''''''

LOS_QueueWriteCopy()是带复制写入的函数接口，函数原型如代码清单 5‑11所示，其使用实例如代码清单 5‑12加粗部分所示。

代码清单 5‑11 LOS_QueueWriteCopy()函数原型

1 extern UINT32 LOS_QueueWriteCopy(UINT32 uwQueueID, **(1)**

2 VOID \*pBufferAddr, **(2)**

3 UINT32 uwBufferSize, **(3)**

4 UINT32 uwTimeOut); **(4)**

代码清单 5‑11\ **(1)**\ ：uwQueueID是由LOS_QueueCreate创建的队列ID，其值范围为1~LOSCFG_BASE_IPC_QUEUE_LIMIT。

代码清单 5‑11\ **(2)**\ ：pBufferAddr是存储要写入的消息的起始地址，起始地址不能为空。

代码清单 5‑11\ **(3)**\ ：uwBufferSize是指定写入消息的大小，其值不能大于消息节点大小。

代码清单 5‑11\ **(4)**\ ：uwTimeOut是等待时间，其值范围为0~LOS_WAIT_FOREVER，单位为Tick，当uwTimeOut为0的时候是不等待，为LOS_WAIT_FOREVER时候是一直等待。

代码清单 5‑12 LOS_QueueWriteCopy()函数实例

1 /\*

2 \* @ 函数名 ： Send_Task

3 \* @ 功能说明： 通过按键进行对队列的写操作

4 \* @ 参数 ：

5 \* @ 返回值 ： 无

6 \/

7 UINT32 send_data1 = 1; /\* 写入队列的第一个消息 \*/

8 UINT32 send_data2 = 2; /\* 写入队列的第二个消息 \*/

9 static void Send_Task(void)

10 {

11 UINT32 uwRet = LOS_OK; /\* 定义一个返回类型，初始化为成功的返回值 \*/

12 /\* 任务都是一个无限循环，不能返回 \*/

13 while (1) { /\* KEY1 被按下 \*/

14 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {

**15 /\* 将消息写入到队列中，等待时间为 0 \*/**

**16 uwRet = LOS_QueueWriteCopy (Test_Queue_Handle,/*写入的队列ID \*/**

**17 &send_data1, /\* 写入的消息 \*/**

**18 sizeof(send_data1),/\* 消息的长度 \*/**

**19 0); /\* 等待时间为 0 \*/**

20 if (LOS_OK != uwRet) {

21 printf("消息不能写入到消息队列！错误代码0x%x\n",uwRet);

22 }/\* KEY2 被按下 \*/

23 } else if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {

**24 /\* 将消息写入到队列中，等待时间为 0 \*/**

**25 uwRet = LOS_QueueWriteCopy (Test_Queue_Handle,/*写入的队列ID \*/**

**26 &send_data2, /\* 写入的消息 \*/**

**27 sizeof(send_data2),/\* 消息的长度 \*/**

**28 0); /\* 等待时间为 0 \*/**

29 if (LOS_OK != uwRet) {

30 printf("消息不能写入到消息队列！错误代码0x%x\n",uwRet);

31 }

32

33 }

34 /\* 20Ticks扫描一次 \*/

35 LOS_TaskDelay(20);

36 }

37 }

带复制写入操作有几点需要注意的地方。

1. 使用写入队列的操作前应先创建要写入的队列。

2. 在中断上下文环境中，必须使用非阻塞模式写入，也就是等待时间为0个Tick。

3. 在初始化LiteOS之前无法调用此API。

4. 将写入由uwBufferSize指定大小的消息，不能大于消息节点的大小。

5. 写入队列节点中的是存储在BufferAddr中的消息。

LOS_QueueWriteCopy()函数源码如代码清单 5‑13所示。

代码清单 5‑13 LOS_QueueWriteCopy()函数源码

1 LITE_OS_SEC_TEXT UINT32 LOS_QueueWriteCopy( UINT32 uwQueueID,

2 VOID \* pBufferAddr,

3 UINT32 uwBufferSize,

4 UINT32 uwTimeOut )

5 {

6 UINT32 uwRet;

7 UINT32 uwOperateType;

8

9 uwRet = osQueueWriteParameterCheck(uwQueueID,

10 pBufferAddr,

11 &uwBufferSize,

12 uwTimeOut); **(1)**

13 if (uwRet != LOS_OK) {

14 return uwRet;

15 }

16

17 uwOperateType = OS_QUEUE_OPERATE_TYPE(OS_QUEUE_WRITE, OS_QUEUE_TAIL); **(2)**

18 return osQueueOperate(uwQueueID,

19 uwOperateType,

20 pBufferAddr,

21 &uwBufferSize,

22 uwTimeOut); **(3)**

23 }

代码清单 5‑13\ **(1)**\ ：对传递进来的参数进行检查，如果参数非法就返回错误代码，并且消息不会写入到队列中。

代码清单 5‑13\ **(2)**\ ：保存处理的类型，LiteOS采用一种通用的处理消息队列的方法进行处理消息，对于复制写入消息，其操作方式是写入OS_QUEUE_WRITE，位置是队列尾部OS_QUEUE_TAIL。

代码清单 5‑13\ **(3)**\ ：osQueueOperate()函数源码实现如代码清单 5‑14所示。

通用的消息队列处理函数
^^^^^^^^^^^

osQueueOperate()函数是LiteOS的一个通用处理函数，根据处理类型uwOperateType进行处理。

代码清单 5‑14 osQueueOperate()源码

1 LITE_OS_SEC_TEXT UINT32 osQueueOperate(UINT32 uwQueueID,

2 UINT32 uwOperateType,

3 VOID \*pBufferAddr,

4 UINT32 \*puwBufferSize,

5 UINT32 uwTimeOut)

6 {

7 QUEUE_CB_S \*pstQueueCB;

8 LOS_TASK_CB \*pstRunTsk;

9 UINTPTR uvIntSave;

10 LOS_TASK_CB \*pstResumedTask;

11 UINT32 uwRet = LOS_OK;

12 UINT32 uwReadWrite = OS_QUEUE_READ_WRITE_GET(uwOperateType); **(1)**

13

14 uvIntSave = LOS_IntLock(); **(2)**

15

16 pstQueueCB = (QUEUE_CB_S \*)GET_QUEUE_HANDLE(uwQueueID); **(3)**

17 if (OS_QUEUE_UNUSED == pstQueueCB->usQueueState) {

18 uwRet = LOS_ERRNO_QUEUE_NOT_CREATE;

19 goto QUEUE_END;

20

21 }

22

23 if (OS_QUEUE_IS_READ(uwOperateType) &&

24 (*puwBufferSize < pstQueueCB->usQueueSize - sizeof(UINT32))){ **(4)**

25 uwRet = LOS_ERRNO_QUEUE_READ_SIZE_TOO_SMALL;

26 goto QUEUE_END;

27 } else if (OS_QUEUE_IS_WRITE(uwOperateType) &&

28 (*puwBufferSize > pstQueueCB->usQueueSize - sizeof(UINT32))) {**(5)**

29 uwRet = LOS_ERRNO_QUEUE_WRITE_SIZE_TOO_BIG;

30 goto QUEUE_END;

31 }

32

33 if (0 == pstQueueCB->usReadWriteableCnt[uwReadWrite]) { **(6)**

34 if (LOS_NO_WAIT == uwTimeOut) {

35 uwRet = OS_QUEUE_IS_READ(uwOperateType) ?

36 LOS_ERRNO_QUEUE_ISEMPTY : LOS_ERRNO_QUEUE_ISFULL; **(7)**

37 goto QUEUE_END;

38 }

39

40 if (g_usLosTaskLock) {

41 uwRet = LOS_ERRNO_QUEUE_PEND_IN_LOCK; **(8)**

42 goto QUEUE_END;

43 }

44

45 pstRunTsk = (LOS_TASK_CB \*)g_stLosTask.pstRunTask; **(9)**

46 osTaskWait(&pstQueueCB->stReadWriteList[uwReadWrite],

47 OS_TASK_STATUS_PEND_QUEUE, uwTimeOut); **(10)**

48 LOS_IntRestore(uvIntSave);

49 LOS_Schedule(); **(11)**

50

51 uvIntSave = LOS_IntLock();

52

53 if (pstRunTsk->usTaskStatus & OS_TASK_STATUS_TIMEOUT) { **(12)**

54 pstRunTsk->usTaskStatus &= (~OS_TASK_STATUS_TIMEOUT);

55 uwRet = LOS_ERRNO_QUEUE_TIMEOUT;

56 goto QUEUE_END;

57 }

58 } else {

59 pstQueueCB->usReadWriteableCnt[uwReadWrite]--; **(13)**

60 }

61

62 osQueueBufferOperate(pstQueueCB,

63 uwOperateType,

64 pBufferAddr,

65 puwBufferSize); **(14)**

66

67 if (!LOS_ListEmpty(&pstQueueCB->stReadWriteList[!uwReadWrite])) {**(15)**

68 pstResumedTask = OS_TCB_FROM_PENDLIST(LOS_DL_LIST_FIRST(&

69 pstQueueCB->stReadWriteList[!uwReadWrite]));

70

71 osTaskWake(pstResumedTask, OS_TASK_STATUS_PEND_QUEUE); **(16)**

72

73 LOS_IntRestore(uvIntSave);

74

75 LOS_Schedule(); **(17)**

76 return LOS_OK;

77 } else {

78 pstQueueCB->usReadWriteableCnt[!uwReadWrite]++; **(18)**

79 }

80

81 QUEUE_END:

82 LOS_IntRestore(uvIntSave);

83 return uwRet;

84 }

代码清单 5‑14\ **(1)**\ ：通过OS_QUEUE_READ_WRITE_GET()得到即将处理的操作类型，如果是读，该值为0，如果是写，该值为1。

代码清单 5‑14\ **(2)**\ ：屏蔽中断，因为在后续的操作中，系统不希望被打扰，否则有可能影响对阻塞在消息队列中任务的操作。

代码清单 5‑14\ **(3)**\ ：通过消息队列ID获取对应的消息队列控制块，并且判断消息队列是否已使用，如果是未使用的，则返回一个错误代码并退出操作。

代码清单 5‑14\ **(4)**\ ：如果要操作队列的方式是读取，那么还需要判断一下存放消息的地址空间大小是否足以放得下消息队列的消息，如果放不下就会返回一个错误代码并且退出操作。

代码清单 5‑14\ **(5)**\ ：如果要操作队列的方式是写入，那么还需要判断一下要写入消息队列中的消息大小，消息节点大小是否能存储即将要写入的消息，如果无法存储就会返回一个错误代码并且退出操作。

代码清单 5‑14\ **(6)**\ ：对于读取消息操作，如果当前消息队列中的可读的消息个数是0，那表明当队列是空的，则不能读取消息；对于写入消息操作，如果当前消息队列中可以写入的消息个数也是0，表明此时队列已满，不允许写入消息。反之则跳转到代码清单 5‑14\ **(13)** 处执行。

代码清单 5‑14\ **(7)**\ ：在不可读写消息的情况下，如果用户不设置阻塞超时的话，那么如果是读消息队列操作，则返回一个错误代码LOS_ERRNO_QUEUE_ISEMPTY；如果是写消息队列操作，则返回一个错误代码LOS_ERRNO_QUEUE_ISFULL。

代码清单 5‑14\ **(8)**\ ：如果任务被上锁，那不允许操作消息队列，返回一个错误代码LOS_ERRNO_QUEUE_PEND_IN_LOCK。

代码清单 5‑14\ **(9)**\ ：获取当前任务的任务控制块。

代码清单 5‑14\ **(10)**\ ：根据用户指定的阻塞超时时间uwTimeOut进行等待，把当前任务添加到对应操作队列的阻塞列表中，如果是写消息操作，将任务添加到写操作阻塞列表，当队列有空闲的消息节点时，任务就会恢复就绪态执行写入操作，或者当阻塞时间超时任务也会恢复就绪态；如果是读消息操作，
将任务添加到读操作阻塞列表中，等到其他任务/中断写入消息，当队列有可读消息时，任务恢复就绪态执行读消息操作，或者当阻塞时间超时任务也会恢复就绪态。

代码清单 5‑14\ **(11)**\ ：进行切换任务。

代码清单 5‑14\ **(12)**\
：程序能运行到这一步，说明任务已经解除阻塞了，有可能是阻塞时间超时，也可能是有其他任务操作了消息队列，导致阻塞在消息队列的任务解除阻塞。系统需要进一步判断任务解除阻塞的原因，如果是阻塞时间超时，直接返回一个错误代码LOS_ERRNO_QUEUE_TIMEOUT并且退出操作。

代码清单 5‑14\ **(13)**\ ：如果任务不是因为超时恢复就绪态的，那就说明消息队列可以进行读写操作，可读写的消息个数减一。

代码清单 5‑14\ **(14)**\ ：调用osQueueBufferOperate()函数进行对应的操作，源码实现如代码清单 5‑15所示。

代码清单 5‑14\ **(15)**\ ：如果与操作相反的阻塞列表中有任务在阻塞，那么在操作完成后需要恢复任务。LiteOS直接采用stReadWriteList[!uwReadWrite]表示操作相反的阻塞列表。例如：当前是进行读消息操作，在读取消息之后，那么队列就有空闲的消息节点了，此时队列将
允许写入消息，因此系统就会判断一下写操作阻塞列表是否有任务在等待写入，如果有那就将任务恢复就绪态；对于写消息操作也是如此。

代码清单 5‑14\ **(16)**\ ：调用osTaskWake()函数唤醒任务。

代码清单 5‑14\ **(17)**\ ：进行一次任务调度。

代码清单 5‑14\ **(18)**\ ：如果没有任务阻塞在与当前操作相反的阻塞列表中，那么与当前操作相反的可用消息个数加一。比如：当前是读消息操作，那么读取完消息之后，可写消息的操作个数就要加一；如果当前是写消息操作，那么可读消息的个数就要加一。

代码清单 5‑15 osQueueBufferOperate()源码

1 LITE_OS_SEC_TEXT static VOID osQueueBufferOperate(QUEUE_CB_S \*pstQueueCB,

2 UINT32 uwOperateType,

3 VOID \*pBufferAddr,

4 UINT32 \*puwBufferSize)

5 {

6 UINT8 \*pucQueueNode;

7 UINT32 uwMsgDataSize = 0;

8 UINT16 usQueuePosion = 0;

9

10 /\* 获取消息队列操作类型 \*/

11 switch (OS_QUEUE_OPERATE_GET(uwOperateType)) {

12 case OS_QUEUE_READ_HEAD:

13 usQueuePosion = pstQueueCB->usQueueHead;

14 (pstQueueCB->usQueueHead + 1 == pstQueueCB->usQueueLen) ?

15 (pstQueueCB->usQueueHead = 0) : (pstQueueCB->usQueueHead++);\ **(1)**

16 break;

17

18 case OS_QUEUE_WRITE_HEAD:

19 (0 == pstQueueCB->usQueueHead) ?

20 (pstQueueCB->usQueueHead = pstQueueCB->usQueueLen - 1)

21 : (--pstQueueCB->usQueueHead);

22 usQueuePosion = pstQueueCB->usQueueHead; **(2)**

23 break;

24

25 case OS_QUEUE_WRITE_TAIL :

26 usQueuePosion = pstQueueCB->usQueueTail;

27 (pstQueueCB->usQueueTail + 1 == pstQueueCB->usQueueLen) ?

28 (pstQueueCB->usQueueTail = 0) : (pstQueueCB->usQueueTail++);\ **(3)**

29 break;

30

31 default:

32 PRINT_ERR("invalid queue operate type!\n");

33 return;

34 }

35

36 pucQueueNode = &(pstQueueCB->pucQueue[(usQueuePosion \*

37 (pstQueueCB->usQueueSize))]);

38

39 if (OS_QUEUE_IS_READ(uwOperateType)) {

40 memcpy((VOID \*)&uwMsgDataSize,

41 (VOID \*)(pucQueueNode + pstQueueCB->usQueueSize - sizeof(UINT32)),

42 sizeof(UINT32));

43 memcpy((VOID \*)pBufferAddr,

44 (VOID \*)pucQueueNode, uwMsgDataSize);

45 \*puwBufferSize = uwMsgDataSize;

46 } else {

47 memcpy((VOID \*)pucQueueNode,

48 (VOID \*)pBufferAddr, \*puwBufferSize);

49 memcpy((VOID \*)(pucQueueNode +

50 pstQueueCB->usQueueSize - sizeof(UINT32)),

51 puwBufferSize, sizeof(UINT32));

52 }

53 }

代码清单 5‑15\ **(1)(3)**\ ：LiteOS的消息队列支持回卷方式操作，即当可读或者可写指针达到消息队列的末尾时，将重置指针从0开始，可以把队列看作是一个环形队列。

代码清单 5‑15\ **(2)**\ ：LiteOS的消息队列支持LIFO，处理紧急消息，从消息队列头部写入。

消息队列读消息函数
^^^^^^^^^

不带复制方式读取LOS_QueueRead()
'''''''''''''''''''''''

消息队列的传输方式分为两种，一种是不带复制的，另一种是带复制的，不带复制读取消息函数原型如代码清单 5‑16所示。该函数用于读取指定队列中的消息，并将获取的消息存储到pBufferAddr指定的地址，用户需要指定读取消息的存储地址与大小，其实验实例如代码清单 5‑17加粗部分所示。

代码清单 5‑16 LOS_QueueRead()函数原型

1 extern UINT32 LOS_QueueRead(UINT32 uwQueueID, **(1)**

2 VOID \*pBufferAddr, **(2)**

3 UINT32 uwBufferSize, **(3)**

4 UINT32 uwTimeOut); **(4)**

代码清单 5‑16\ **(1)**\ ：uwQueueID是由LOS_QueueCreate创建的队列ID，其值范围为1~LOSCFG_BASE_IPC_QUEUE_LIMIT。

代码清单 5‑16\ **(2)**\ ：pBufferAddr是存储获取消息的起始地址。

代码清单 5‑16\ **(3)**\ ：uwBufferSize是读取消息缓冲区的大小，该值不能小于消息节点大小。

代码清单 5‑16\ **(4)**\ ：uwTimeOut是等待时间，其值范围为0~LOS_WAIT_FOREVER，单位为Tick，当uwTimeOut为0的时候是不等待，为LOS_WAIT_FOREVER时候是一直等待。

代码清单 5‑17 LOS_QueueRead()实例

1 /\*

2 \* @ 函数名 ： Receive_Task

3 \* @ 功能说明： 读取队列的消息

4 \* @ 参数 ：

5 \* @ 返回值 ： 无

6 \/

7 static void Receive_Task(void)

8 {

9 UINT32 uwRet = LOS_OK;

10 UINT32 r_queue; /\* r_queue地址作为队列读取来的存放地址的变量 \*/

11 UINT32 buffsize = 10;

12 while (1) {

**13 /\* 队列读取，等待时间为一直等待 \*/**

**14 uwRet = LOS_QueueRead(Test_Queue_Handle,/\* 读取队列的ID(句柄) \*/**

**15 &r_queue, /\* 读取的消息保存位置 \*/**

**16 buffsize,/\* 读取消息的长度 \*/**

**17 LOS_WAIT_FOREVER); /\* 等待时间：一直等 \*/**

18 if (LOS_OK == uwRet) {

19 printf("本次读取到的消息是%d\n", \*(UINT32 \*)r_queue );

20 } else {

21 printf("消息读取出错\n");

22 }

23 LOS_TaskDelay(20);

24 }

25 }

读取消息的时候需要注意以下几点。

1. 使用LOS_QueueRead()这个函数之前应先创建需要读取消息的队列，并根据队列ID进行读取消息。

2. 队列读取采用的是先进先出（FIFO）模式，首先读取首先存储在队列中的消息。

3. 必须要用户定义一个存储地址的变量，假设为r_queue，并且把存储消息的地址传递给 LOS_QueueRead()函数，否则，将发生地址非法的错误。

4. 在中断上下文环境中，必须使用非阻塞模式写入，也就是等待时间为0个Tick。

5. 在初始化LiteOS之前无法调用此API。

6. r_queue变量中存放的是队列节点的地址。

7. LOS_QueueReadCopy()和LOS_QueueWriteCopy()是一组接口，LOS_QueueRead()和LOS_QueueWrite()是一组接口，两组接口需要配套使用。

LOS_QueueRead()函数的源码的实现如代码清单 5‑18所示，实际上LOS_QueueRead()是LiteOS对LOS_QueueReadCopy()函数的封装，只不过读取的消息是地址而非内容。

代码清单 5‑18LOS_QueueRead()函数源码

1 LITE_OS_SEC_TEXT UINT32 LOS_QueueRead(UINT32 uwQueueID,

2 VOID \*pBufferAddr,

3 UINT32 uwBufferSize,

4 UINT32 uwTimeOut)

5 {

6 return LOS_QueueReadCopy(uwQueueID,

7 pBufferAddr,

8 &uwBufferSize,

9 uwTimeOut);

10 }

带复制读取LOS_QueueReadCopy()
''''''''''''''''''''''''

LOS_QueueReadCopy()是带复制读取读取消息函数，其函数原型如代码清单 5‑19所示，实验实例如代码清单 5‑20加粗部分所示。

代码清单 5‑19 LOS_QueueReadCopy()函数原型

1 extern UINT32 LOS_QueueReadCopy(UINT32 uwQueueID,

2 VOID \*pBufferAddr,

3 UINT32 \*puwBufferSize,

4 UINT32 uwTimeOut);

代码清单 5‑19\ **(1)**\ ：uwQueueID是由LOS_QueueCreate创建的队列ID，其值范围为1~LOSCFG_BASE_IPC_QUEUE_LIMIT。

代码清单 5‑19\ **(2)**\ ：pBufferAddr是存储获取消息的起始地址。

代码清单 5‑19\ **(3)**\ ：uwBufferSize是读取消息缓冲区的大小，该值不能小于消息节点大小。

代码清单 5‑19\ **(4)**\ ：uwTimeOut是等待时间，其值范围为0~LOS_WAIT_FOREVER，单位为Tick，当uwTimeOut为0的时候表示不等待，为LOS_WAIT_FOREVER的时候表示一直等待。

代码清单 5‑20 LOS_QueueReadCopy()函数实例

1 /\*

2 \* @ 函数名 ： Receive_Task

3 \* @ 功能说明： Receive_Task任务实现

4 \* @ 参数 ： NULL

5 \* @ 返回值 ： NULL

6 \/

7 static void Receive_Task(void)

8 {

9 /\* 定义一个返回类型变量，初始化为LOS_OK \*/

10 UINT32 uwRet = LOS_OK;

11 UINT32 r_queue;

12 UINT32 buffsize = 10;

13 /\* 任务都是一个无限循环，不能返回 \*/

14 while (1)

15 {

**16 buffsize = 10; //更新传递进来的buffsize大小**

**17 /\* 队列读取，等待时间为一直等待 \*/**

**18 uwRet = LOS_QueueReadCopy(Test_Queue_Handle,**

**19 &r_queue, /\* 读取消息保存位置 \*/**

**20 &buffsize, /\* 读取消息的大小 \*/**

**21 LOS_WAIT_FOREVER); /\* 等待时间：一直等 \*/**

22

23 if (LOS_OK == uwRet)

24 {

25 printf("本次读取到的消息是%d\n",r_queue);

26 }

27 else

28 {

29 printf("消息读取出错,错误代码0x%X\n",uwRet);

30 }

31 }

32 }

LOS_QueueReadCopy()函数需要注意以下几点。

1. 使用LOS_QueueReadCopy()这个函数之前应先创建需要读取消息的队列，并根据队列ID进行读取消息。

2. 队列读取采用的是先进先出（FIFO）模式，首先读取首先存储在队列中的消息。

3. 必须要用户自己定义一个存储空间，如r_queue，并且把存储消息的起始地址传递给 LOS_QueueReadCopy()函数，否则，将发生地址非法的错误。

4. 不要在非阻塞模式下读取或写入队列，例如中断，如果非要在中断中读取消息（一般中断是不读取消息的，但是也有例外，比如在某个定时器中断中读取信息判断一下），请将队列设为非阻塞模式，也就是等待时间为0个Tick。

5. 在初始化LiteOS之前无法调用此API。

6. r_queue中存放的是队列节点中的消息而非地址，因此该空间必须是足够大的。

7. 用户必须在读取消息时指定读取消息的大小，其值不能小于消息节点大小。如buffsize，该变量既作为输入又作为输出，作为输入是指定读取缓冲区的大小；作为输出，buffsize是用于保存读取到消息的大小，把读取到的消息大小写在buffsize变量中，在调用LOS_QueueWriteCopy()函数前应
   该注意更新buffsize的值。

LOS_QueueReadCopy()源码的实现过程如代码清单 5‑21所示，实际上也是通过调用消息队列通用处理函数osQueueOperate()进行处理，处理的方式是读操作OS_QUEUE_READ，位置是队列头部OS_QUEUE_HEAD。

代码清单 5‑21 LOS_QueueReadCopy()源码

1 LITE_OS_SEC_TEXT UINT32 LOS_QueueReadCopy(UINT32 uwQueueID,

2 VOID \* pBufferAddr,

3 UINT32 \* puwBufferSize,

4 UINT32 uwTimeOut)

5 {

6 UINT32 uwRet;

7 UINT32 uwOperateType;

8

9 uwRet = osQueueReadParameterCheck(uwQueueID,

10 pBufferAddr,

11 puwBufferSize,

12 uwTimeOut);

13 if (uwRet != LOS_OK) {

14 return uwRet;

15 }

16

17 uwOperateType = OS_QUEUE_OPERATE_TYPE(OS_QUEUE_READ, OS_QUEUE_HEAD);

18 return osQueueOperate(uwQueueID,

19 uwOperateType,

20 pBufferAddr,

21 puwBufferSize,

22 uwTimeOut);

23 }

消息队列实验
~~~~~~

消息队列实验是在LiteOS中创建了两个任务，一个是写消息任务，另一个是读消息任务，两个任务独立运行，写消息任务是通过检测按键的按下情况来写入消息；而读消息任务则一直等待消息的到来，当读取消息成功就通过串口把消息打印在串口调试助手中，实验源码如代码清单 5‑22加粗部分所示。

代码清单 5‑22 消息队列实验源码

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

19 #include "los_queue.h"

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

32 /\* 定义任务ID变量 \*/

33 UINT32 Receive_Task_Handle;

34 UINT32 Send_Task_Handle;

35

36 /\* 内核对象ID \/

37 /\*

38 \* 信号量，消息队列，事件标志组，软件定时器这些都属于内核的对象，要想使用这些内核

39 \* 对象，必须先创建，创建成功之后会返回一个相应的ID。实际上就是一个整数，后续

40 \* 就可以通过这个ID操作这些内核对象。

41 \*

42 \*

43 \* 内核对象就是一种全局的数据结构，通过这些数据结构可以实现任务间的通信，

44 \* 任务间的事件同步等各种功能。至于这些功能的实现是通过调用这些内核对象的函数

45 \* 来完成的

46 \*

47 \*/

**48 /\* 定义消息队列的ID变量 \*/**

**49 UINT32 Test_Queue_Handle;**

**50 /\* 定义消息队列长度 \*/**

**51 #define TEST_QUEUE_LEN 16**

**52 #define TEST_QUEUE_SIZE 16**

53

54 /\* 全局变量声明 \/

55 /\*

56 \* 在写应用程序的时候，可能需要用到一些全局变量。

57 \*/

58 UINT32 send_data1 = 1;

59 UINT32 send_data2 = 2;

60 /\* 函数声明 \*/

61 static UINT32 AppTaskCreate(void);

62 static UINT32 Creat_Receive_Task(void);

63 static UINT32 Creat_Send_Task(void);

64

65 static void Receive_Task(void);

66 static void Send_Task(void);

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

80 //定义一个返回类型变量，初始化为LOS_OK

81 UINT32 uwRet = LOS_OK;

82

83 /\* 板载相关初始化 \*/

84 BSP_Init();

85

86 printf("这是一个[野火]-STM32全系列开发板-LiteOS消息队列实验！\n\n");

87 printf("按下KEY1或者KEY2写入队列消息\n");

88 printf("Receive_Task任务读取到消息在串口回显\n\n");

89

90 /\* LiteOS 内核初始化 \*/

91 uwRet = LOS_KernelInit();

92

93 if (uwRet != LOS_OK) {

94 printf("LiteOS 核心初始化失败！失败代码0x%X\n",uwRet);

95 return LOS_NOK;

96 }

97

98 uwRet = AppTaskCreate();

99 if (uwRet != LOS_OK) {

100 printf("AppTaskCreate创建任务失败！失败代码0x%X\n",uwRet);

101 return LOS_NOK;

102 }

103

104 /\* 开启LiteOS任务调度 \*/

105 LOS_Start();

106

107 //正常情况下不会执行到这里

108 while (1);

109 }

110

111

112 /\*

113 \* @ 函数名 ： AppTaskCreate

114 \* @ 功能说明： 任务创建，为了方便管理，所有的任务创建函数都可以放在这个函数里面

115 \* @ 参数 ： 无

116 \* @ 返回值 ： 无

117 \/

118 static UINT32 AppTaskCreate(void)

119 {

120 /\* 定义一个返回类型变量，初始化为LOS_OK \*/

121 UINT32 uwRet = LOS_OK;

122

**123 /\* 创建一个测试队列*/**

**124 uwRet = LOS_QueueCreate("Test_Queue", /\* 队列的名称 \*/**

**125 TEST_QUEUE_LEN, /\* 队列的长度 \*/**

**126 &Test_Queue_Handle, /\* 队列的ID(句柄) \*/**

**127 0, /\* 队列模式，官方暂时未使用 \*/**

**128 TEST_QUEUE_SIZE); /\* 节点大小，单位为字节 \*/**

**129 if (uwRet != LOS_OK) {**

**130 printf("Test_Queue队列创建失败！失败代码0x%X\n",uwRet);**

**131 return uwRet;**

**132 }**

133

134 uwRet = Creat_Receive_Task();

135 if (uwRet != LOS_OK) {

136 printf("Receive_Task任务创建失败！失败代码0x%X\n",uwRet);

137 return uwRet;

138 }

139

140 uwRet = Creat_Send_Task();

141 if (uwRet != LOS_OK) {

142 printf("Send_Task任务创建失败！失败代码0x%X\n",uwRet);

143 return uwRet;

144 }

145 return LOS_OK;

146 }

147

148 /\*

149 \* @ 函数名 ： Creat_Receive_Task

150 \* @ 功能说明： 创建Receive_Task任务

151 \* @ 参数 ：

152 \* @ 返回值 ： 无

153 \/

154 static UINT32 Creat_Receive_Task()

155 {

156 //定义一个返回类型变量，初始化为LOS_OK

157 UINT32 uwRet = LOS_OK;

158

159 //定义一个用于创建任务的参数结构体

160 TSK_INIT_PARAM_S task_init_param;

161

162 task_init_param.usTaskPrio = 5; /\* 任务优先级，数值越小，优先级越高 \*/

163 task_init_param.pcName = "Receive_Task";/\* 任务名 \*/

164 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)Receive_Task;

165 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

166

167 uwRet = LOS_TaskCreate(&Receive_Task_Handle, &task_init_param);

168 return uwRet;

169 }

170

171

172 /\*

173 \* @ 函数名 ： Creat_Send_Task

174 \* @ 功能说明： 创建Send_Task任务

175 \* @ 参数 ：

176 \* @ 返回值 ： 无

177 \/

178 static UINT32 Creat_Send_Task()

179 {

180 // 定义一个返回类型变量，初始化为LOS_OK

181 UINT32 uwRet = LOS_OK;

182 TSK_INIT_PARAM_S task_init_param;

183

184 task_init_param.usTaskPrio = 4; /\* 任务优先级，数值越小，优先级越高 \*/

185 task_init_param.pcName = "Send_Task"; /\* 任务名*/

186 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)Send_Task;

187 task_init_param.uwStackSize = 1024; /\* 栈大小 \*/

188

189 uwRet = LOS_TaskCreate(&Send_Task_Handle, &task_init_param);

190

191 return uwRet;

192 }

193

194

195 /\*

196 \* @ 函数名 ： Receive_Task

197 \* @ 功能说明： Receive_Task任务实现

198 \* @ 参数 ： NULL

199 \* @ 返回值 ： NULL

200 \/

**201 static void Receive_Task(void)**

**202 {**

**203 /\* 定义一个返回类型变量，初始化为LOS_OK \*/**

**204 UINT32 uwRet = LOS_OK;**

**205 UINT32 r_queue;**

**206 UINT32 buffsize = 10;**

**207 /\* 任务都是一个无限循环，不能返回 \*/**

**208 while (1) {**

**209 /\* 队列读取，等待时间为一直等待 \*/**

**210 uwRet = LOS_QueueRead(Test_Queue_Handle, /\* 读取队列的ID(句柄) \*/**

**211 &r_queue, /\* 读取的消息保存位置 \*/**

**212 buffsize, /\* 读取的消息的长度 \*/**

**213 LOS_WAIT_FOREVER); /\* 等待时间：一直等 \*/**

**214 if (LOS_OK == uwRet) {**

**215 printf("本次读取到的消息是%d\n",*(UINT32 \*)r_queue);**

**216 } else {**

**217 printf("消息读取出错,错误代码0x%X\n",uwRet);**

**218 }**

**219 }**

**220 }**

221

222

223 /\*

224 \* @ 函数名 ： Send_Task

225 \* @ 功能说明： Send_Task任务实现

226 \* @ 参数 ： NULL

227 \* @ 返回值 ： NULL

228 \/

**229 static void Send_Task(void)**

**230 {**

**231 /\* 定义一个返回类型变量，初始化为LOS_OK \*/**

**232 UINT32 uwRet = LOS_OK;**

**233**

**234**

**235 /\* 任务都是一个无限循环，不能返回 \*/**

**236**

**237 while (1)**

**238 {**

**239**

**240 /\* K1 被按下 \*/**

**241 if ( Key_Scan(KEY1_GPIO_PORT,KEY1_GPIO_PIN) == KEY_ON ) {**

**242 /\* 将消息写入到队列中，等待时间为 0 \*/**

**243 uwRet = LOS_QueueWrite(Test_Queue_Handle, /\* 写入队列的ID(句柄) \*/**

**244 &send_data1, /\* 写入的消息 \*/**

**245 sizeof(send_data1), /\* 消息的长度 \*/**

**246 0); /\* 等待时间为 0 \*/**

**247 if (LOS_OK != uwRet) {**

**248 printf("消息不能写入到消息队列！错误代码0x%X\n",uwRet);**

**249 }**

**250 }**

**251**

**252 /\* K2 被按下 \*/**

**253 if ( Key_Scan(KEY2_GPIO_PORT,KEY2_GPIO_PIN) == KEY_ON ) {**

**254 /\* 将消息写入到队列中，等待时间为 0 \*/**

**255 uwRet = LOS_QueueWrite( Test_Queue_Handle,**

**256 &send_data2, /\* 写入的消息 \*/**

**257 sizeof(send_data2), /\* 消息的长度 \*/**

**258 0); /\* 等待时间为 0 \*/**

**259 if (LOS_OK != uwRet) {**

**260 printf("消息不能写入到消息队列！错误代码0x%X\n",uwRet);**

**261 }**

**262 }**

**263 /\* 20ms扫描一次 \*/**

**264 LOS_TaskDelay(20);**

**265 }**

**266 }**

267

268 /\*

269 \* @ 函数名 ： BSP_Init

270 \* @ 功能说明： 板级外设初始化，所有开发板上的初始化均可放在这个函数里面

271 \* @ 参数 ：

272 \* @ 返回值 ： 无

273 \/

274 static void BSP_Init(void)

275 {

276 /\*

277 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

278 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

279 \* 都统一用这个优先级分组，千万不要再分组，切忌。

280 \*/

281 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

282

283 /\* LED 初始化 \*/

284 LED_GPIO_Config();

285

286 /\* 串口初始化 \*/

287 USART_Config();

288

289 /\* 按键初始化 \*/

290 Key_GPIO_Config();

291 }

292

293 /\* END OF FILE \/

实验现象
~~~~

将程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据读者买的开发板而定，每个型号的开发板都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板，就可以在调试助手中看到串口的打印信息，按下开发板的KEY
1按键写入消息1，按下KEY2按键写入消息2；按下KEY1后在串口调试助手中可以看到读取到消息1，按下KEY2后在串口调试助手中可以看到读取到消息2，如图 5‑3所示。

|messag004|

图 5‑3消息队列实验现象

.. |messag002| image:: media\messag002.png
   :width: 5.76806in
   :height: 2.18681in
.. |messag003| image:: media\messag003.png
   :width: 4.65278in
   :height: 3.03472in
.. |messag004| image:: media\messag004.png
   :width: 5.58669in
   :height: 4.44268in

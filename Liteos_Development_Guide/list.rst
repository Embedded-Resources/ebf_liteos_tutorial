.. vim: syntax=rst

链表
=========

在LiteOS中存在着大量的基础数据结构链表（或者称之为列表List）的操作，要想读懂LiteOS的源码，就必须弄懂链表的基本操作，在列表中的成员称之为节点（Node），在后续的讲解中，本书所说的链表就是列表。

C语言链表简介
~~~~~~~

链表作为C语言中一种基础的数据结构，在平时写程序的时候用的并不多，但在操作系统里面使用的非常多。链表就好比一个圆形的晾衣架，如图 12‑1所示，晾衣架上面有很多钩子，钩子首尾相连。链表也是如此，链表由节点组成，节点与节点之间首尾相连。

|list002|

图 12‑1 圆形晾衣架

晾衣架的钩子不能代表很多东西，但是钩子本身却可以挂很多东西。同样，链表也类似，链表的节点本身不能存储太多东西，但是节点跟晾衣架的钩子一样，可以挂载很多数据。

链表分为单向链表和双向链表，本书讲解的链表为双向链表。

双向链表也叫双链表，是链表的一种，是在操作系统中常用的数据结构，它的每个数据节点中都有两个指针，分别指向前驱节点和后继节点。因此，从双向链表中的任意一个节点开始，都可以很方便地访问它的前驱节点和后继节点，这种数据结构形式使得双向链表在查找时更加方便，特别是大量数据的遍历，能方便地完成各种插入、删除等
操作。

在C语言中，链表与数组很类似，数组的特性是便于索引，而链表的特性是便于插入与删除，两者的示意图如图 12‑2所示，本书以双向链表为例。

|list003|

图 12‑2 链表与数组的对比

链表是通过节点把离散的数据链接成一个表，通过对节点的插入和删除操作从而实现对数据的存取。而数组是通过开辟一段连续的内存来存储数据，这是数组和链表最大的区别。数组的每个成员对应链表的节点，成员和节点的数据类型可以是标准的C类型或者是用户自定义的结构体，数组有起始地址和结束地址，而链表是一个圈。

链表的使用讲解
~~~~~~~

LiteOS提供了很多操作链表的函数，如链表的初始化、添加节点、删除节点等。

LiteOS的链表节点结构体中只有两个指针，一个是指向前驱节点的指针，另一个是指向后继节点的指针，如代码清单 12‑1所示。

代码清单 12‑1链表节点结构体

1 typedef struct LOS_DL_LIST {

2 struct LOS_DL_LIST \*pstPrev;

3 struct LOS_DL_LIST \*pstNext;

4 } LOS_DL_LIST;

链表初始化函数LOS_ListInit()
^^^^^^^^^^^^^^^^^^^^^

在使用链表的时候必须要先初始化，将链表的指针指向自己，为后续添加节点做准备 ，链表初始化函数LOS_ListInit()的源码如代码清单 12‑2所示，链表初始化示意图如图 12‑3所示。

代码清单 12‑2链表初始化函数LOS_ListInit()源码

1 LITE_OS_SEC_ALW_INLINE STATIC_INLINE VOID LOS_ListInit(LOS_DL_LIST \*pstList)

2 {

3 pstList->pstNext = pstList;

4 pstList->pstPrev = pstList;

5 }

|list004|

图 12‑3链表初始化示意图

在初始化完成后可以检查一下链表初始化是否成功，判断链表是否为空，链表的初始化实例如代码清单 12‑3所示。

代码清单 12‑3链表初始化函数LOS_ListInit()实例

1 LOS_DL_LIST \*head; /\* 定义一个双向链表的头节点 \*/

2 head = (LOS_DL_LIST \*)LOS_MemAlloc(m_aucSysMem0, sizeof(LOS_DL_LIST));

3 /\* 动态申请头节点的内存 \*/

4 LOS_ListInit(head); /\* 初始化双向链表 \*/

5 if (!LOS_ListEmpty(head)) /\* 判断是否初始化成功 \*/

6 {

7 printf("双向链表初始化失败!\n\n");

8 } else

9 {

10 printf("双向链表初始化成功!\n\n");

11 }

向链表添加节点函数LOS_ListAdd()
^^^^^^^^^^^^^^^^^^^^^^

LiteOS运行向链表中插入节点，插入过程是需要选择插入链表的位置，再执行插入操作，如代码清单 12‑4所示（源码标注序号对应图片序号），使用实例如代码清单 12‑5所示。

代码清单 12‑4向链表添加节点函数LOS_ListAdd()源码

1 LITE_OS_SEC_ALW_INLINE STATIC_INLINE VOID LOS_ListAdd(LOS_DL_LIST \*pstList,

2 LOS_DL_LIST \*pstNode)

3 {

4 pstNode->pstNext = pstList->pstNext; **(1)**

5 pstNode->pstPrev = pstList; **(2)**

6 pstList->pstNext->pstPrev = pstNode; **(3)**

7 pstList->pstNext = pstNode; **(4)**

8 }

插入节点的思想很简单，其过程如图 12‑4所示（pstList 可以看作是Node1）。

|list005|

图 12‑4插入节点的过程示意图

代码清单 12‑5向链表添加节点函数LOS_ListAdd()实例

1 printf("添加节点......\n");/\* 插入节点*/

2

3 LOS_DL_LIST \*node1 = /*动态申请第一个节点的内存 \*/

4 (LOS_DL_LIST \*)LOS_MemAlloc(m_aucSysMem0, sizeof(LOS_DL_LIST));

5 LOS_DL_LIST \*node2 = /*动态申请第二个节点的内存 \*/

6 (LOS_DL_LIST \*)LOS_MemAlloc(m_aucSysMem0, sizeof(LOS_DL_LIST));

7

8 printf("添加第一个节点与第二个节点.....\n");

9 LOS_ListAdd(head,node1); /\* 添加第一个节点，连接在头节点上 \*/

10 LOS_ListAdd(node1,node2); /\* 添加第二个节点，连接在第一个节点上 \*/

11 if ((node1->pstPrev == head) && (node2->pstPrev == node1))

12 {/\* 判断是否插入成功 \*/

13 printf("添加节点成功!\n\n");

14 } else

15 {

16 printf("添加节点失败!\n\n");

17 }

从链表删除节点函数LOS_ListDelete()
^^^^^^^^^^^^^^^^^^^^^^^^^

LiteOS支持删除链表中的节点，用户可以使用LOS_ListDelete()函数将节点删除，只需将要删除节点传递到函数中即可，该函数把该节点的前驱节点与后继节点链接在一起，，然后将该节点的指针指向NULL就表示节点已删除，如代码清单 12‑6所示，其过程示意图如图
12‑5所示（源码标注序号对应图片序号），LOS_ListDelete()函数使用实例如代码清单 12‑7所示。

代码清单 12‑6从链表删除节点函数LOS_ListDelete()源码

1 LITE_OS_SEC_ALW_INLINE STATIC_INLINE VOID LOS_ListDelete(LOS_DL_LIST \*pstNode)

2 {

3 pstNode->pstNext->pstPrev = pstNode->pstPrev; **(1)**

4 pstNode->pstPrev->pstNext = pstNode->pstNext; **(2)**

5 pstNode->pstNext = (LOS_DL_LIST \*)NULL; **(3)**

6 pstNode->pstPrev = (LOS_DL_LIST \*)NULL; **(4)**

7 }

|list006|

图 12‑5节点删除过程示意图

代码清单 12‑7从链表删除节点函数LOS_ListDelete()实例

1 printf("删除节点......\n");

2 LOS_ListDelete(node1); /\* 删除第一个节点 \*/

3 LOS_MemFree(m_aucSysMem0, node1); /\* 释放第一个节点的内存， \*/

4 if (head->pstNext == node2) /\* 判断是否删除成功 \*/

5 {

6 printf("删除节点成功\n\n");

7 } else

8 {

9 printf("删除节点失败\n\n");

10

11 }

双向链表实验
~~~~~~

双向链表实验实现如下功能：

1. 调用LOS_ListInit初始双向链表。

2. 调用LOS_ListAdd向链表中增加节点。

3. 调用LOS_ListTailInsert向链表尾部插入节点。

4. 调用LOS_ListDelete删除指定节点。

5. 调用LOS_ListEmpty判断链表是否为空。

6. 测试操作是否成功。

实验源码如代码清单 12‑8加粗部分所示。

代码清单 12‑8双向链表实验

1 /*\*

2 \\*

3 \* @file main.c

4 \* @author fire

5 \* @version V1.0

6 \* @date 2018-xx-xx

7 \* @brief 这是一个[野火]-STM32F103霸道LiteOS的双向链表实验！

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

22 #include "los_memory.h"

23 /\* 板级外设头文件 \*/

24 #include "stm32f10x.h"

25 #include "bsp_usart.h"

26 #include "bsp_led.h"

27 #include "bsp_key.h"

28

29 /\* 任务ID \/

30 /\*

31 \* 任务ID是一个从0开始的数字，用于索引任务，当任务创建完成之后，它就具有了一个任务ID

32 \* 以后读者要想操作这个任务都需要通过这个任务ID，

33 \*

34 \*/

35 /\* 定义定时器ID变量 \*/

36 UINT32 Test_Task_Handle;

37

38

39 /\* 函数声明 \*/

40 extern LITE_OS_SEC_BSS UINT8\* m_aucSysMem0;

41

42 static void AppTaskCreate(void);

43 static UINT32 Creat_Test_Task(void);

44 static void Test_Task(void);

45 static void BSP_Init(void);

46

47 /*\*

48 \* @brief 主函数

49 \* @param 无

50 \* @retval 无

51 \* @note 第一步：开发板硬件初始化

52 第二步：创建App应用任务

53 第三步：启动LiteOS，开始多任务调度，启动不成功则输出错误信息

54 \*/

55 int main(void)

56 {

57 UINT32 uwRet = LOS_OK;

58 /\* 板级初始化，所有的跟开发板硬件相关的初始化都可以放在这个函数里面 \*/

59 BSP_Init();

60 /\* 发送一个字符串 \*/

61 printf("这是一个[野火]-STM32全系列开发板- LiteOS的双向链表实验！\n");

62 /\* LiteOS 核心初始化 \*/

63 uwRet = LOS_KernelInit();

64 if (uwRet != LOS_OK) {

65 printf("LiteOS 核心初始化失败！\n");

66 return LOS_NOK;

67 }

68 /\* 创建App应用任务，所有的应用任务都可以放在这个函数里面 \*/

69 AppTaskCreate();

70

71 /\* 开启LiteOS任务调度 \*/

72 LOS_Start();

73 }

74 static void AppTaskCreate(void)

75 {

76 UINT32 uwRet = LOS_OK;/\* 定义一个创建任务的返回类型，初始化为创建成功的返回值 \*/

77 /\* 创建Test_Task任务 \*/

78 uwRet = Creat_Test_Task();

79 if (uwRet != LOS_OK) {

80 printf("Test_Task任务创建失败！\n");

81 }

82

83 }

84

85

86 /\* 创建Test_Task任务*/

87 static UINT32 Creat_Test_Task(void)

88 {

89 UINT32 uwRet = LOS_OK; /\* 定义一个创建任务的返回类型，初始化为创建成功的返回值 \*/

90 TSK_INIT_PARAM_S task_init_param;

91

92 task_init_param.usTaskPrio = 4;/\* 优先级，数值越小，优先级越高 \*/

93 task_init_param.pcName = "Test_Task";/\* 任务名，字符串形式，方便调试 \*/

94 task_init_param.pfnTaskEntry = (TSK_ENTRY_FUNC)Test_Task;

95 task_init_param.uwStackSize = 0x1000;/\* 栈大小，单位为字，即4个字节 \*/

96

97 uwRet = LOS_TaskCreate(&Test_Task_Handle, &task_init_param);

98 return uwRet;

99 }

100

101

102

103 /\*

104 \* @ 函数名 ： Clear_Task

105 \* @ 功能说明： 写入已经初始化成功的内存池地址数据

106 \* @ 参数 ： void

107 \* @ 返回值 ： 无

108 \/

109 static void Test_Task(void)

110 {

**111 UINT32 uwRet = LOS_OK; /\* 定义一个初始化的返回类型，初始化为成功的返回值 \*/**

**112 printf("\n双向链表初始化中......\n");**

**113**

**114 LOS_DL_LIST \*head; /\* 定义一个双向链表的头节点 \*/**

**115 head = (LOS_DL_LIST \*)LOS_MemAlloc(m_aucSysMem0, sizeof(LOS_DL_LIST));**

**116 /\* 动态申请头节点的内存 \*/**

**117 LOS_ListInit(head); /\* 初始化双向链表 \*/**

**118 if (!LOS_ListEmpty(head)) { /\* 判断是否初始化成功 \*/**

**119 printf("双向链表初始化失败!\n\n");**

**120 } else {**

**121 printf("双向链表初始化成功!\n\n");**

**122 }**

**123**

**124 printf("添加节点和尾节点添加......\n");/\* 插入节点：顺序插入与从末尾插入 \*/**

**125**

**126 LOS_DL_LIST \*node1 = /*动态申请第一个节点的内存 \*/**

**127 (LOS_DL_LIST \*)LOS_MemAlloc(m_aucSysMem0, sizeof(LOS_DL_LIST));**

**128 LOS_DL_LIST \*node2 = /*动态申请第二个节点的内存 \*/**

**129 (LOS_DL_LIST \*)LOS_MemAlloc(m_aucSysMem0, sizeof(LOS_DL_LIST));**

**130 LOS_DL_LIST \*tail = /*动态申请尾节点的内存 \*/**

**131 (LOS_DL_LIST \*)LOS_MemAlloc(m_aucSysMem0, sizeof(LOS_DL_LIST));**

**132**

**133 printf("添加第一个节点与第二个节点.....\n");**

**134 LOS_ListAdd(head,node1); /\* 添加第一个节点，连接在头节点上 \*/**

**135 LOS_ListAdd(node1,node2); /\* 添加第二个节点，连接在一个节点上 \*/**

**136 if ((node1->pstPrev == head) && (node2->pstPrev == node1)) {**

**137 printf("添加节点成功!\n\n"); /\* 判断是否插入成功 \*/**

**138 } else {**

**139 printf("添加节点失败!\n\n");**

**140 }**

**141 printf("将尾节点插入双向链表的末尾.....\n");**

**142 LOS_ListTailInsert(head, tail); /\* 将尾节点插入双向链表的末尾 \*/**

**143 if (tail->pstPrev == node2) {/\* 判断是否插入成功 \*/**

**144 printf("链表尾节点添加成功!\n\n");**

**145 } else {**

**146 printf("链表尾节点添加失败!\n\n");**

**147 }**

**148**

**149 printf("删除节点......\n"); /\* 删除已有节点 \*/**

**150 LOS_ListDelete(node1); /\* 删除第一个节点 \*/**

**151 LOS_MemFree(m_aucSysMem0, node1); /\* 释放第一个节点的内存， \*/**

**152 if (head->pstNext == node2) {/\* 判断是否删除成功 \*/**

**153 printf("删除节点成功\n\n");**

**154 } else {**

**155 printf("删除节点失败\n\n");**

**156**

**157 }**

158

159 while (1) {

160 LED2_TOGGLE; //LED2翻转

161 printf("任务运行中!\n");

162 LOS_TaskDelay (2000);

163 }

164 }

165

166

167 static void BSP_Init(void)

168 {

169 /\*

170 \* STM32中断优先级分组为4，即4bit都用来表示抢占优先级，范围为：0~15

171 \* 优先级分组只需要分组一次即可，以后如果有其他的任务需要用到中断，

172 \* 都统一用这个优先级分组，千万不要再分组，切忌。

173 \*/

174 NVIC_PriorityGroupConfig( NVIC_PriorityGroup_4 );

175

176 /\* LED 初始化 \*/

177 LED_GPIO_Config();

178

179 /\* 串口初始化 \*/

180 USART_Config();

181

182 /\* 按键初始化 \*/

183 Key_GPIO_Config();

184 }

185

186 /END OF FILE/

双向链表实验现象
~~~~~~~~

程序编译好，用USB线连接电脑和开发板的USB接口（对应丝印为USB转串口），用DAP仿真器把配套程序下载到野火STM32开发板（具体型号根据读者买的开发板而定，每个型号的开发板都配套有对应的程序），在电脑上打开串口调试助手，然后复位开发板就可以在调试助手中看到串口的打印信息，在串口调试助手中可以看
到运行结果，它里面输出了信息表明双向链表的操作已经全部完成，如图 12‑6所示。

|list007|

图 12‑6双向链表实验现象

.. |list002| image:: media\list002.jpeg
   :width: 1.94167in
   :height: 1.94167in
.. |list003| image:: media\list003.png
   :width: 5.76806in
   :height: 1.44792in
.. |list004| image:: media\list004.png
   :width: 3.27778in
   :height: 1.71528in
.. |list005| image:: media\list005.png
   :width: 5.86806in
   :height: 2.07986in
.. |list006| image:: media\list006.png
   :width: 5.97778in
   :height: 1.55556in
.. |list007| image:: media\list007.png
   :width: 5.7125in
   :height: 4.51389in

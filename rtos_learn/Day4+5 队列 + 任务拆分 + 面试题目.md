

### Day4 + Day5 队列 + 任务拆分 + 面试题目

#### 一 . 队列理论

数据的操作采用先进先出的方法(FIFO)：写数据时放到尾部，读数据时从头部读。也可以强制写队列头部：覆盖头部数据。可以理解为是一个环形缓冲区。

某个任务读队列时，如果队列没有数据，则该任务可以进入阻塞状态：还可以指定阻塞的时间。如果队列有数据了，则该阻塞的任务会变为就绪态。如果一直都没有数据，则时间到之后它也会进入就绪态。

当多个任务读取空队列时，这些任务都会进入阻塞状态：有多个任务在等待同一个队列的数据。当队列中有数据时，哪个任务会进入就绪态？

1. 优先级最高的任务

2. 如果大家的优先级相同，那等待时间最久的任务会进入就绪态

队列里面存的可以是值也可以是地址

![image-20260319115650743](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260319115650743.png)

#### 二 . 队列函数

```c
// 创建队列
QueueHandle_t xQueueCreate( UBaseType_t uxQueueLength, UBaseType_t uxItemSize );
// pcHeadto 和 pcWriteTo 都是指向头部
```

| 参数          | 说明                                                         |
| :------------ | :----------------------------------------------------------- |
| uxQueueLength | 队列长度，最多能存放多少个数据(item)                         |
| uxItemSize    | **每个**数据(item)的大小：以字节为单位                       |
| 返回值        | 非0：成功，返回句柄，以后使用句柄来操作队列<br>NULL：失败，因为内存不足 |


```c
/* 等同于xQueueSendToBack
 * 往队列尾部写入数据，如果没有空间，阻塞时间为xTicksToWait
 * 满了之后就跳回去队列的头部
 */
BaseType_t xQueueSend(
                      QueueHandle_t xQueue,
                      const void *pvItemToQueue, // 原来的队列里面有ItemSize
                      TickType_t xTicksToWait
                      );
```
| 参数          | 说明                                                         |
| :------------ | :----------------------------------------------------------- |
| xQueue        | 队列句柄，要写哪个队列                                       |
| pvItemToQueue | 数据指针，这个数据的值会被复制进队列<br>复制多大的数据？在创建队列时已经指定了数据大小 |
| xTicksToWait  | 如果队列满则无法写入新数据，可以让任务进入阻塞状态，<br>xTicksToWait表示阻塞的最大时间(Tick Count)。<br>如果被设为0，无法写入数据时函数会立刻返回；<br>如果被设为portMAX_DELAY，则会一直阻塞直到有空间可写 |
| 返回值        | pdPASS：数据成功写入了队列<br>errQUEUE_FULL：写入失败，因为队列满了。 |
```c
/*
 * 读队列数据，如果没有空间，阻塞时间为xTicksToWait
 * pcHeadto 是指向buffer的首地址，是不会变的
 * pcReadFrom 是上一次读的位置，如果是一开始的话就指向N-1
 * 不可能有人同时读一个队列，因为这个代码里面有禁止中断的操作（独占）
 */
BaseType_t xQueueReceive(QueueHandle_t xQueue, // 读哪一个队列
                         void * const pvBuffer, // 存储的buffer
                         TickType_t xTicksToWait);
```
| 参数          | 说明                                                         |
| :------------ | :----------------------------------------------------------- |
| xQueue         | 队列句柄，要读哪个队列                                       |
| pvBuffer       | buffer指针，队列的数据会被复制到这个buffer<br>复制多大的数据？在创建队列时已经指定了数据大小 |
| xTicksToWait   | 如果队列空则无法读出数据，可以让任务进入阻塞状态，<br>xTicksToWait表示阻塞的最大时间(Tick Count)。<br>如果被设为0，无法读出数据时函数会立刻返回；<br>如果被设为portMAX_DELAY，则会一直阻塞直到有数据可读 |
| 返回值         | pdPASS：从队列读出数据入<br>errQUEUE_EMPTY：读取失败，因为队列空了。 |


```c
/*
 * 往队列头部写入数据，如果没有空间，阻塞时间为xTicksToWait
 * 假设原来的排列是 0,1,...,N-1 只有第一二还有最后一位有数据
 * 那么应用这个函数之后不会覆盖之前的数据，而是往N-1的位置写数据，接着N-2，pcReadFrom-=ItemSize
 */
BaseType_t xQueueSendToFront(
                            QueueHandle_t xQueue,
                            const void *pvItemToQueue,
                            TickType_t xTicksToWait
                           );
```

#### 三 . 队列使用

1. ##### 同步 保护**共享资源**不被同时访问（联想卖包子）  **信号量** / **事件组 / **消息队列

![image-20260318011511531](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260318011511531.png)

```c
QueueHandle_t QueueCaclHandle;

/*-----------------------------------------------------------*/

void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 10000000; i++)
		{
			sum++;
		}
		
		xQueueSend(QueueCaclHandle, &sum, portMAX_DELAY); //这里要阻塞发送

		sum = 1;
	}
}

void Task2Function(void * param)
{
	while (1)
	{
		int val; //设置一个值来存储读出来的数据
		
		Task2runflag = 0;
		xQueueReceive(QueueCaclHandle, &val, portMAX_DELAY);
		printf("sum = %d\r\n", val);
		Task2runflag = 1;
		
	}
}

// 主函数
QueueCaclHandle =  xQueueCreate(2 , sizeof(int));
if(QueueCaclHandle == NULL)
{
    printf("can not create queue!");
}
```

2. ##### 互斥 协调**任务执行顺序**/ 传递事件（联想上厕所，全局变量）  互斥锁（采取开中断的方法）

![image-20260318011559702](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260318011559702.png)

```c
int InitUARTLock(void)
{
	int val;
	// 创建队列
	QueueCaclHandle =  xQueueCreate(1 , sizeof(int));
	if(QueueCaclHandle == NULL)
	{
		printf("can not create queue!");
		return -1;
	}	
	xQueueSend(QueueCaclHandle, &val, portMAX_DELAY);
	return 0;
}

void GetUARTLock(void)
{
	int val;
	xQueueReceive(QueueCaclHandle, &val, portMAX_DELAY);
}

// 释放锁
void PullUARTLock(void)
{
	int val;
	xQueueSend(QueueCaclHandle, &val, portMAX_DELAY);
}

void TaskGenericFunction(void * param)
{
	while (1)
	{
		GetUARTLock(); // 往队列里面读数据
		printf("%s\r\n",(char *)param);
		PullUARTLock(); //往队列里面写数据 这个时候可以切换任务 printf 执行速度极快
		vTaskDelay(2); // 如果不加这个延时就会一直运行后面创建的任务
	}
}
```

#### 四 . 队列创建细节

```c
// 在底层里面
#if ( configSUPPORT_DYNAMIC_ALLOCATION == 1 )
    #define xQueueCreate( uxQueueLength, uxItemSize )    
	  xQueueGenericCreate( ( uxQueueLength ), ( uxItemSize ), ( queueQUEUE_TYPE_BASE ) )
          
// 在xQueueGenericCreate函数里面
// 可以看到在每一个队列里面的开头都有一个结构体 Queue_t
pxNewQueue = ( Queue_t * ) pvPortMalloc( sizeof( Queue_t ) + xQueueSizeInBytes );
```
Queue_t 结构体定义：
![image-20260317001303953](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260317001303953.png)

![image-20260317001438683](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260317001438683.png)

```c
// 其中 pcHead和pcTail是队列起始和结束的位置，是不变的。
// 在prvInitialiseNewQueue的任务之后是xQueueGenericReset函数
// pxQueue->pcWriteTo = pxQueue->pcHead;
// pxQueue->u.xQueue.pcReadFrom = pxQueue->pcHead + ( ( pxQueue->uxLength - 1U ) * pxQueue->uxItemSize ); 

/* 写数据的作用：存入数据，唤醒等待数据的任务，若被唤醒的任务优先级更高就调度
```

#### 五  . 队列发送细节

```c
/* 写数据的作用：存入数据，唤醒等待数据的任务，若被唤醒的任务优先级更高就调度 */

// 跟创建任务一样，xQueueGenericSend被重定义为xQueueSend
#define xQueueSend( xQueue, pvItemToQueue, xTicksToWait ) \
    xQueueGenericSend( ( xQueue ), ( pvItemToQueue ), ( xTicksToWait ), queueSEND_TO_BACK )

// 在这里面涉及到了唤醒任务
if( listLIST_IS_EMPTY( &( pxQueue->xTasksWaitingToReceive ) ) == pdFALSE )//有任务在等待data
{
      if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToReceive ) ) != pdFALSE )
          // 唤醒某个任务
          prvResetNextTaskUnblockTime(); // 发起调度
          
// 其中在xTaskRemoveFromEventList函数里面
// 这里面也涉及到了链表的操作，把要唤醒的任务（这里面是有优先级的不同）从链表里面拿出来，把他从delaylist里面放到readylist里面，本来它是在readylist里面，但是因为读不到数据，所以要去delaylist
pxUnblockedTCB = listGET_OWNER_OF_HEAD_ENTRY( pxEventList ); // 在这个链表里面第一个取出Unblocked任务
configASSERT( pxUnblockedTCB );
listREMOVE_ITEM( &( pxUnblockedTCB->xEventListItem ) ); // 将这个任务从等待唤醒的链表里面去除掉
if( uxSchedulerSuspended == ( UBaseType_t ) pdFALSE )
{
    listREMOVE_ITEM( &( pxUnblockedTCB->xStateListItem ) );
    prvAddTaskToReadyList( pxUnblockedTCB );
 
// 那在if( xTaskRemoveFromEventList( &( pxQueue->xTasksWaitingToReceive ) ) != pdFALSE )这里是怎么判断pdFALSE的呢
// 在xTaskRemoveFromEventList函数里对当前任务和被唤醒的任务优先级进行比较，然后发起调度
if( pxUnblockedTCB->uxPriority > pxCurrentTCB->uxPriority )
{
    xReturn = pdTRUE;
    xYieldPending = pdTRUE;
}
else
{
    xReturn = pdFALSE;
}
```

#### 六  . 队列集

流程：
![image-20260317110321064](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260317110321064.png)

代码思路：

```c
// FreeRTOSConfig.h 定义Queue_set开关
#define configUSE_QUEUE_SETS        1

void Task1Function(void * param)
{
	int i = 1;
	
	while (1)
	{
		i++;
		
		xQueueSend(QueueHandle1, &i, portMAX_DELAY);

		vTaskDelay(10);
	}
}

void Task2Function(void * param)
{
	int i = -1;
	
	while (1)
	{	
		i--;
		
		xQueueSend(QueueHandle2, &i, portMAX_DELAY);

		vTaskDelay(20);
	}
}

void Task3Function(void * param)
{
	while (1)
	{
		int i;
		QueueSetMemberHandle_t Handle;
		
		// 1. 读Queue_set :看一下哪个队列有数据
		Handle = xQueueSelectFromSet(QueueSetHandle, portMAX_DELAY);

		// 2. 读数据
		xQueueReceive(Handle, &i, 0);

		// 3. 打印
		printf("data is %d\r\n", i);
	}
}

// 主函数
// 1. 创建2个队列
QueueHandle1 =  xQueueCreate(2 , sizeof(int));
QueueHandle2 =  xQueueCreate(2 , sizeof(int));

// 2. 创建1个队列集合
QueueSetHandle = xQueueCreateSet(4);

// 3. 把两个队列添加进队列集合
xQueueAddToSet(QueueHandle1, QueueSetHandle);
xQueueAddToSet(QueueHandle2, QueueSetHandle);

// 4. 创建3个任务
xTaskCreate(Task1Function, "Task1", 100, NULL, 1, &xHandleTask1);
xTaskCreate(Task2Function, "Task2", 100, NULL, 1, NULL);
// Task3用来检测哪个队列有数据
xTaskCreate(Task3Function, "Task3", 100, NULL, 1, NULL);
```

#### 六  . 任务拆分

1. 如果面对**Task3是优先级2**，**Task1和Task2是优先级0**的任务，他们的调度策略和内部原理是怎么样的呢？

顺序如下：Task3会在运行之后延时5个tick，给其他任务执行

![image-20260318002450868](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260318002450868.png)

完整执行任务的顺序：

![image-20260318002825895](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260318002825895.png)

**Idle Task**第一次进去的时候先执行**释放任务**的函数，然后会**主动放弃**，释放CPU资源，第二次进去的时候再执行**钩子函数**。

```c
prvCheckTasksWaitingTermination(); // 看看有没有需要释放内存的任务

#if ( configUSE_PREEMPTION == 0 )
    {
        taskYIELD();
    }

#if ( ( configUSE_PREEMPTION == 1 ) && ( configIDLE_SHOULD_YIELD == 1 ) )
    {
        if( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ tskIDLE_PRIORITY ] ) ) > ( UBaseType_t ) 1 )
        {
            taskYIELD(); // 释放CPU资源
        }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
    }
#endif /* ( ( configUSE_PREEMPTION == 1 ) && ( configIDLE_SHOULD_YIELD == 1 ) ) */

#if ( configUSE_IDLE_HOOK == 1 )
    {
        extern void vApplicationIdleHook( void ); // 钩子函数

        vApplicationIdleHook();
    }
```



2. 如果三个**优先级相同**（但是不为0）的任务在main函数里面依次创建，为什么先执行的是**Task3**？

```c
xTaskCreate(Task1Function, "Task1", 100, NULL, 1, &xHandleTask1);
xTaskCreate(Task2Function, "Task2", 100, NULL, 1, NULL);
xTaskCreate(Task3Function, "Task3", 100, NULL, 1, NULL);

// 启动调度器
vTaskStartScheduler();
```

首先在`xTaskCreate`中找到`prvAddNewTaskToReadyList( pxNewTCB );`这个函数

这是放入新创建任务的TCB结构体
```c
// 如果是空，则创建TCB结构体
if( pxCurrentTCB == NULL )
{
    pxCurrentTCB = pxNewTCB;
// 这里面写到如果是同等优先级，指针会指向最新创建的TCB结构体
if( pxCurrentTCB->uxPriority <= pxNewTCB->uxPriority )
{
    pxCurrentTCB = pxNewTCB;
}
```

然后会生成一个任务就绪链表

在调度器`vTaskStartScheduler`里面

![image-20260318004910173](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260318004910173.png)

在`xPortStartScheduler`他会触发一个中断`prvStartFirstTask`，在这里面运行第一个任务

```c
__asm void prvStartFirstTask( void )
{
/* *INDENT-OFF* */
    PRESERVE8

    /* Use the NVIC offset register to locate the stack. */
    ldr r0, =0xE000ED08
    ldr r0, [ r0 ]
    ldr r0, [ r0 ]

    /* Set the msp back to the start of the stack. */
    msr msp, r0
    /* Globally enable interrupts. */
    cpsie i
    cpsie f
    dsb
    isb
    /* Call SVC to start the first task. */
    svc 0
    nop
    nop
/* *INDENT-ON* */
}
```



3. 如果三个**优先级为0**的任务在main函数里面依次创建，是最后创建的任务，也就是空闲任务先运行

```c
/* 因为在任务器调度的时候，空闲任务在里面被创建 */
/* 在执行完空闲任务之后运行的是Task1，Task2，Task3 */
xIdleTaskHandle = xTaskCreateStatic( prvIdleTask,
                                     configIDLE_TASK_NAME,
                                     ulIdleTaskStackSize,
                                     ( void * ) NULL,       
                                     portPRIVILEGE_BIT,     
                                     pxIdleTaskStackBuffer,
                                     pxIdleTaskTCBBuffer );
```

拆分任务：

① 任务功能相对单一

② 优先级    

a. 一直要运行的任务：优先级最高 = 0    

b. 高优先级的任务：事件驱动（平时是阻塞）




面试问题：

![image-20260317133711892](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260317133711892.png)

1. 记住**Readylist**就可以弄清楚这个调度策略，程序是从上往下去查找这个**Readylist**，高优先级的任务永远会优先执行

2. 主要是队列，传数据一般是队列，还有任务通知

3. Mutex（互斥量）：通过优先级继承解决优先级翻转问题

   优先级翻转：低优先级的任务并没有释放信号量，高优先级的任务无法获得信号量，导致其他比低优先级高的任务一直运行 

   笔记：

   ![image-20260324125400724](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260324125400724.png)

![image-20260324125412220](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260324125412220.png)

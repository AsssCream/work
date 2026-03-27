### Day8 任务通知

#### 一、理论知识

使用队列、信号量、事件组时，我们都要事先创建对应的结构体，双方通过中间的结构体通信：

该通信对象是一个公共对象，多对多的关系

**![image-20260323095538694](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260323095538694.png)**

使用任务通知时，任务结构体**TCB**中就包含了内部对象，可以直接接收别人发过来的"通知"：

多个任务都可以往里面填充obj，多对一的关系

**![image-20260323095612248](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260323095612248.png)**





发送数据的任务和接收数据的任务都有可能被挂在这两个链表上面；

如果接收任务正在进行，发送数据的任务等待发送的时候会处于阻塞状态挂在链表上；

如果发送任务正在进行，接收数据的任务等待发送的时候会处于阻塞状态挂在链表上。

![image-20260323100001094](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260323100001094.png)

TCB结构体：

其他任务往里面放入值`ulNotifiedValue`的时候并不会处于阻塞状态，要么成功要么失败，都不等待；

目标任务可以等待，`ucNotifyState`无数据的时候等待，有数据的时候即可返回

![image-20260323100443987](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260323100443987.png)

`ucNotifyState`有三种取值

- **taskNOT_WAITING_NOTIFICATION**：任务没有在等待通知
- **taskWAITING_NOTIFICATION**：任务在等待通知
- **taskNOTIFICATION_RECEIVED**：任务接收到了通知，也被称为 **pending**（有数据了，待处理）



函数原型：

```c
// 第一套函数
BaseType_t xTaskNotifyGive( TaskHandle_t xTaskToNotify ); //
uint32_t ulTaskNotifyTake( BaseType_t xClearCountOnExit, TickType_t xTicksToWait );
```

第一套函数：

`xTaskNotifyGive` 函数参数说明：

（这个函数只有目标任务，但是没有val对应的值，要改变val需要目标任务val++）

| 参数            | 说明                                                 |
| --------------- | ---------------------------------------------------- |
| `xTaskToNotify` | 任务句柄（创建任务时得到），指定要向哪个任务发送通知 |
| **返回值**      | 必定返回 `pdPASS`                                    |



`ulTaskNotifyTake` 函数参数说明：

| 参数                | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| `xClearCountOnExit` | 函数返回前是否清零：<br />`pdTRUE`：把通知值清零<br />`pdFALSE`：如果通知值大于 0，则把通知值减一 |
| `xTicksToWait`      | 任务进入阻塞态的超时时间，它在等待通知值大于 0。<br />0：不等待，即刻返回；<br />`portMAX_DELAY`：一直等待，直到通知值大于 0；<br />其他值：Tick Count，可以用 `pdMS_TO_TICKS()` 把 ms 转换为 Tick Count |
| **返回值**          | 函数返回之前，在清零或减一**之前**的通知值。<br />如果`xTicksToWait`非 0，则返回值有 2 种情况：<br />1. 大于 0：在超时前，通知值被增加了<br />2. 等于 0：一直没有其他任务增加通知值，最后超时返回 0 |



第二套函数：

```c
// 第二套函数
BaseType_t xTaskNotify( TaskHandle_t xTaskToNotify, uint32_t ulValue, eNotifyAction eAction );
BaseType_t xTaskNotifyWait( uint32_t ulBitsToClearOnEntry,
                            uint32_t ulBitsToClearOnExit,
                            uint32_t *pulNotificationValue,
                            TickType_t xTicksToWait );
```

`xTaskNotify` 函数参数说明：

| 参数            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| `xTaskToNotify` | 任务句柄（创建任务时得到），指定要向哪个任务发送通知         |
| `ulValue`       | 用该值去更改接收任务的val，具体使用方式由 `eAction` 参数决定 |
| `eAction`       | 见下表（指定通知的操作类型）                                 |
| **返回值**      | `pdPASS`：成功，大部分调用都会成功<br />`pdFAIL`：仅在 `eAction` 为 `eSetValueWithoutOverwrite`，且通知状态为 "pending"（有新数据未读）时失败 |

| eNotifyAction 取值          | 说明                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `eNoAction`                 | 仅仅是更新通知状态为 "pending"，未使用`ulValue`。<br />这个选项相当于轻量级的、更高效的二值信号量。 |
| `eSetBits`                  | 通知值 = 原来的通知值 \| `ulValue`，按位或。<br />相当于轻量级的、更高效的事件组。 |
| `eIncrement`                | 通知值 = 原来的通知值 + 1，未使用`ulValue`。<br />相当于轻量级的、更高效的二值信号量、计数型信号量。<br />**等价于`xTaskNotifyGive()`函数。** |
| `eSetValueWithoutOverwrite` | 不覆盖。<br />如果通知状态为 "pending"（表示有数据未读），则此次调用`xTaskNotify`不做任何事，返回`pdFAIL`。<br />如果通知状态不是 "pending"（表示没有新数据），则：通知值 = `ulValue`。 |
| `eSetValueWithOverwrite`    | 覆盖。<br />无论如何，不管通知状态是否为 "pending"，通知值 = `ulValue`。 |

`xTaskNotifyWait` 函数参数说明：

| 参数                   | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| `ulBitsToClearOnEntry` | 在`xTaskNotifyWait`入口处，要清除通知值的哪些位？<br />仅在通知状态不是 "pending" 时才会清除。<br />本意：等待事件前先清零 “旧数据” 的某些位。<br />清除公式：`通知值 = 通知值 & ~(ulBitsToClearOnEntry)`<br />例：传入`0x01` → 清除 bit0；传入`0xffffffff`（即`ULONG_MAX`）→ 清除所有位（置 0） |
| `ulBitsToClearOnExit`  | 在`xTaskNotifyWait`出口处，若因收到数据（非超时）退出时：<br />`通知值 = 通知值 & ~(ulBitsToClearOnExit)`<br />在**清除前**，通知值会先赋值给`*pulNotificationValue`。<br />例：传入`0x03` → 清除 bit0、bit1；传入`0xffffffff` → 清除所有位（置 0） |
| `pulNotificationValue` | 用于取出通知值。<br />函数退出时，在`ulBitsToClearOnExit`清除**之前**，将当前通知值赋值给`*pulNotificationValue`。<br />若不需要取出通知值，可设为`NULL` |
| `xTicksToWait`         | 任务阻塞的超时时间（等待通知状态变为 "pending"）。<br /> `0`：不等待，即刻返回<br />`portMAX_DELAY`：无限等待，直到通知状态变为 "pending"<br /> 其他值：指定 Tick 数，可通过`pdMS_TO_TICKS()`将毫秒转换为 Tick Count |
| **返回值**             | 1. `pdPASS`：成功获得通知（可能调用前已是 "pending"，或阻塞期间变为 "pending"）<br />2. `pdFAIL`：未收到通知（超时或其他原因） |



#### 二、 实现轻量级的信号量

与之前的函数对比：

|          | 信号量                                                       | 使用任务通知实现信号量                                       |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **创建** | `SemaphoreHandle_t xSemaphoreCreateCounting( UBaseType_t uxMaxCount, UBaseType_t uxInitialCount ); ` | **无**（任务通知复用 TCB 资源，无需额外创建对象）            |
| **Give** | `xSemaphoreGive( SemaphoreHandle_t xSemaphore ); `           | `BaseType_t xTaskNotifyGive( TaskHandle_t xTaskToNotify ); ` |
| **Take** | `xSemaphoreTake( SemaphoreHandle_t xSemaphore, TickType_t xBlockTime ); ` | `uint32_t ulTaskNotifyTake( BaseType_t xClearCountOnExit, TickType_t xTicksToWait ); ` |

```c
// 之前的写法
void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 10; i++)
		{
			xSemaphoreGive(xSemCacl);
		}
		
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
    volatile int i = 0;
	while (1)
	{
		xSemaphoreTake(xSemCacl, portMAX_DELAY);
		printf("take i = %d", i++);
	}
}

// 任务通知
void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 10; i++)
		{
			xTaskNotifyGive(xHandleTask2);
		}
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	volatile int val = 0;
	volatile int i = 0;
	while (1)
	{
		val = ulTaskNotifyTake(pdFALSE, portMAX_DELAY);
		printf("Notify val = %d，take i = %d",val,i++); // 先打印当前的 i 值，语句执行完成后 i 才会自增。
	}
}
```



#### 三、轻量级的队列

与之前的函数对比：

|          | 队列                                                         | 使用任务通知实现队列                                         |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **创建** | `QueueHandle_t xQueueCreate( UBaseType_t uxQueueLength, UBaseType_t uxItemSize ); ` | **无**（任务通知复用 TCB 资源，无需额外创建队列对象）        |
| **发送** | ` BaseType_t xQueueSend( QueueHandle_t xQueue, const void * pvItemToQueue, TickType_t xTicksToWait ); ` | `BaseType_t xTaskNotify( TaskHandle_t xTaskToNotify, uint32_t ulValue, eNotifyAction eAction ); ` |
| **接收** | ` BaseType_t xQueueReceive( QueueHandle_t xQueue, void * const pvBuffer, TickType_t xTicksToWait ); ` | `BaseType_t xTaskNotifyWait( uint32_t ulBitsToClearOnEntry, uint32_t ulBitsToClearOnExit, uint32_t *pulNotificationValue, TickType_t xTicksToWait ); ` |

```c
// 之前的写法
// 在初始化函数里面写的内容：
QueueCaclHandle =  xQueueCreate(10 , sizeof(int));
// 任务内容
void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 10000; i++)
		{
			sum++;
		}
		
		for(i = 0; i < 10; i++)
		{			
			xQueueSend(QueueCaclHandle, &sum, portMAX_DELAY);
			sum++;
		}
        
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	volatile int val;
	volatile int i = 0;
	while (1)
	{	
		xQueueReceive(QueueCaclHandle, &val, portMAX_DELAY);
		printf("sum = %d,i = %d\r\n", val, i++);		
	}
}

// 任务通知
void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 10000; i++)
		{
			sum++;
		}
		
		for(i = 0; i < 10; i++)
		{			
            // 这里选择不覆盖就是eSetValueWithoutOverwrite，覆盖eSetValueWithOverwrite
            // 前者打印 10000，后者打印 10009
			xTaskNotify(xHandleTask2, sum, eSetValueWithoutOverwrite);
			sum++;
		}
//		sum = 1;
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	volatile int val = 0;
	volatile int i = 0;
	while (1)
	{	
		xTaskNotifyWait(0, 0, &val, portMAX_DELAY);
		printf("sum = %d,i = %d\r\n", val, i++);
		
	}
}
```



#### 四、轻量级的事件组

与之前的函数对比：

|              | 事件组                                                       | 使用任务通知实现事件组                                       |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **创建**     | `EventGroupHandle_t xEventGroupCreate( void ); `             | **无**（任务通知复用 TCB 资源，无需额外创建事件组对象）      |
| **设置事件** | `EventBits_t xEventGroupSetBits( EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToSet ); ` | `BaseType_t xTaskNotify( TaskHandle_t xTaskToNotify, uint32_t ulValue, eNotifyAction eAction ); `（需指定 `eAction = eSetBits`，实现按位或设置事件位） |
| **等待事件** | `EventBits_t xEventGroupWaitBits( EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToWaitFor, const BaseType_t xClearOnExit, const BaseType_t xWaitForAllBits, TickType_t xTicksToWait ); ` | `BaseType_t xTaskNotifyWait( uint32_t ulBitsToClearOnEntry, uint32_t ulBitsToClearOnExit, uint32_t *pulNotificationValue, TickType_t xTicksToWait ); `<br />（通过 `ulBitsToClearOnEntry/Exit` 控制位清除，实现事件等待逻辑） |

在任务通知里面，和事件组不同的是，事件组是可以用位与的操作去唤醒目标任务，其中一位的正确并不会正确激活，而要完全匹配才行；而任务通知则是一直会唤醒目标任务，无法指定触发条件。

例子如下：

```c
// 之前的写法
void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 10000; i++)
		{
			sum++;
		}
		xQueueSend(QueueCaclHandle, &sum, 0);
		xEventGroupSetBits( xEventCacl, (1<<0));
		printf("task1 is ready\r\n");
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	while (1)
	{	
		volatile int i;
		
		for(i = 0; i < 1000000; i++)
		{
			dec++;
		}
		xQueueSend(QueueCaclHandle, &dec, 0);
		xEventGroupSetBits(xEventCacl, (1<<1));
		printf("task2 is ready\r\n");
		vTaskDelete(NULL);
	}
}

void Task3Function(void * param)
{
	int val1,val2;
	while (1)
	{	
		xEventGroupWaitBits(xEventCacl, (1<<0) | (1<<1), pdTRUE, pdTRUE, portMAX_DELAY);
		vTaskDelay(20);
		xQueueReceive(QueueCaclHandle, &val1, 0);
		xQueueReceive(QueueCaclHandle, &val2, 0);

		printf("val1 = %d , val2 = %d\r\n", val1, val2);
	}
}

// 任务通知
void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 10000; i++)
		{
			sum++;
		}
		xQueueSend(QueueCaclHandle, &sum, 0);
		xTaskNotify(xHandleTask3, (1<<0), eSetBits);
		printf("task1 is ready\r\n");
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	while (1)
	{	
		volatile int i;
		
		for(i = 0; i < 1000000; i++)
		{
			dec++;
		}
		xQueueSend(QueueCaclHandle, &dec, 0);
		xTaskNotify(xHandleTask3, (1<<1), eSetBits);
		printf("task2 is ready\r\n");
		vTaskDelete(NULL);
	}
}

void Task3Function(void * param)
{
	int val1,val2;
	int val;
	
	while (1)
	{	
		xTaskNotifyWait(0, 0, &val, portMAX_DELAY);
		
		if(val == 0x3)
		{
			vTaskDelay(20);
			xQueueReceive(QueueCaclHandle, &val1, 0);
			xQueueReceive(QueueCaclHandle, &val2, 0);	
			printf("val1 = %d , val2 = %d\r\n", val1, val2);
			vTaskDelete(NULL);			
		}
		else 
		{
			vTaskDelay(20);
			printf("I am waiting for val, now val is 0x%x\r\n", val);
		}
	}
}
```


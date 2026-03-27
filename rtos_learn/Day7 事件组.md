### Day7 事件组

#### 一、事件组概念

事件组可以简单地认为就是一个整数：

- 该整数的每一位表示一个事件
- 每一位事件的含义由程序员决定，比如：Bit0 表示用来串口是否就绪，Bit1 表示按键是否被按下
- 这些位，值为 1 表示事件发生了，值为 0 表示事件没发生
- 一个或多个任务、ISR 都可以去写这些位；一个或多个任务、ISR 都可以去读这些位
- 可以等待某一位、某些位中的任意一个，也可以等待多位

![image-20260319115627980](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260319115627980.png)

事件组用一个整数来表示，其中的高 8 位留给内核使用，只能用其他的位来表示事件。那么这个整数是多少位的？

- 如果 `configUSE_16_BIT_TICKS` 是 1，那么这个整数就是**16 位**的，低 8 位用来表示事件
- 如果 `configUSE_16_BIT_TICKS` 是 0，那么这个整数就是**32 位**的，低 24 位用来表示事件
- `configUSE_16_BIT_TICKS` 是用来表示 Tick Count 的，怎么会影响事件组？这只是基于效率来考虑
  - 如果 `configUSE_16_BIT_TICKS` 是 1，就表示该处理器使用 16 位更高效，所以事件组也使用 16 位
  - 如果 `configUSE_16_BIT_TICKS` 是 0，就表示该处理器使用 32 位更高效，所以事件组也使用 32 位

事件组和队列、信号量等不太一样，主要集中在 2 个地方：

- **唤醒谁？**
  - 队列、信号量：事件发生时，只会唤醒**一个任务**
  - 事件组：事件发生时，会唤醒**所有符合条件**的任务，简单地说它有 **广播**"的作用
- **是否清除事件？**
  - 队列、信号量：是消耗型的资源，队列的数据被读走就没了；信号量被获取后就减少了
  - 事件组：被唤醒的任务有两个选择，可以让事件**保留不动**，也可以**清除事件**

------

以上图为例，事件组的常规操作如下：

- 先**创建**事件组
- 任务 C、D 等待事件：
  - 等待什么事件？可以等待某一位、某些位中的任意一个，也可以等待多位。简单地说就是 "或"、"与" 的关系。
  - 得到事件时，要不要清除？可选择清除、不清除。
- 任务 A、B **产生**事件：设置事件组里的某一位、某些位

#### 二、事件组函数

1. ##### **创建**

   使用事件组之前，要先创建，得到一个句柄；使用事件组时，要使用句柄来表明使用哪个事件组。

```c
/* 创建一个事件组，返回它的句柄。
 * 此函数内部会分配事件组结构体
 * 返回值：返回句柄，非NULL表示成功
 */
EventGroupHandle_t xEventGroupCreate( void );

/* 创建一个事件组，返回它的句柄。
 * 此函数无需动态分配内存，所以需要先有一个StaticEventGroup_t结构体，并传入它的指针
 * 返回值：返回句柄，非NULL表示成功
 */
EventGroupHandle_t xEventGroupCreateStatic( StaticEventGroup_t *pxEventGroupBuffer );
```

2. ##### **设置**

​	可以设置事件组的某个位、某些位，使用的函数有2个：

```c 
/* 设置事件组中的位
 * xEventGroup: 哪个事件组
 * uxBitsToSet: 设置哪些位?
 *               如果uxBitsToSet的bitX, bitY为1, 那么事件组中的bitX, bitY被设置为1
 *               可以用来设置多个位，比如 0x15 就表示设置bit4, bit2, bit0
 * 返回值: 返回原来的事件值 (没什么意义, 因为很可能已经被其他任务修改了)
 */
EventBits_t xEventGroupSetBits( EventGroupHandle_t xEventGroup,
                                const EventBits_t uxBitsToSet );

/* 设置事件组中的位（中断安全版本）
 * xEventGroup: 哪个事件组
 * uxBitsToSet: 设置哪些位?
 *               如果uxBitsToSet的bitX, bitY为1, 那么事件组中的bitX, bitY被设置为1
 *               可以用来设置多个位，比如 0x15 就表示设置bit4, bit2, bit0
 * pxHigherPriorityTaskWoken: 有没有导致更高优先级的任务进入就绪态? pdTRUE-有, pdFALSE-没有
 * 返回值: pdPASS-成功, pdFALSE-失败
 */
BaseType_t xEventGroupSetBitsFromISR( EventGroupHandle_t xEventGroup,
                                      const EventBits_t uxBitsToSet,
                                      BaseType_t * pxHigherPriorityTaskWoken );
```

**注意**：

其中ISR 中的函数，比如队列函数 `xQueueSendToBackFromISR`、信号量函数 `xSemaphoreGiveFromISR`，它们会唤醒某个任务，最多只会唤醒 1 个任务。

但是设置事件组时，有可能导致多个任务被唤醒，这会带来很大的不确定性。所以 `xEventGroupSetBitsFromISR` 函数不是直接去设置事件组，而是给一个 FreeRTOS 后台任务（**daemon task**）发送队列数据，由这个任务来设置事件组。

如果后台任务的优先级比当前被中断的任务优先级高，`xEventGroupSetBitsFromISR` 会设置 `*pxHigherPriorityTaskWoken` 为 `pdTRUE`。

如果 **daemon task** 成功地把队列数据发送给了后台任务，那么 `xEventGroupSetBitsFromISR` 的返回值就是 `pdPASS`。

3. ##### **等待**

```c
/* 使用 xEventGroupWaitBits来等待事件，可以等待某一位、某些位中的任意一个，也可以等待多位；等到期望的事件后，还可以清除某些位。 */

EventBits_t xEventGroupWaitBits( EventGroupHandle_t xEventGroup,
                                const EventBits_t uxBitsToWaitFor,
                                const BaseType_t xClearOnExit,
                                const BaseType_t xWaitForAllBits,
                                TickType_t xTicksToWait );
```

一个任务在等待事件发生时，它处于阻塞状态；当期望的时间发生时，这个状态就叫叫`unblock condition`，非阻塞条件，或称为`非阻塞条件成立`；当`非阻塞条件成立`后，该任务就可以变为就绪态。

| 参数              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| `xEventGroup`     | 等待哪个事件组？                                             |
| `uxBitsToWaitFor` | 等待哪些位？哪些位要被测试？                                 |
| `xWaitForAllBits` | 怎么测试？是 "AND" 还是 "OR"？`pdTRUE`: 等待的位，**全部为 1**；`pdFALSE`: 等待的位，**某一个为 1**即可 |
| `xClearOnExit`    | 函数退出前是否要清除事件？`pdTRUE`: 清除`uxBitsToWaitFor`指定的位`pdFALSE`: 不清除 |
| `xTicksToWait`    | 如果期待的事件未发生，阻塞多久。可以设置为`0`: 判断后即刻返回；可设置为`portMAX_DELAY`: 一定等到成功才返回；可以设置为期望的 Tick Count，一般用`pdMS_TO_TICKS()`把 ms 转换为 Tick Count |
| **返回值**        | 返回的是事件值，如果期待的事件发生了，返回的是 "非阻塞条件成立" 时的事件值；如果是超时退出，返回的是超时时刻的事件值。 |

**注意**：

可以使用 `xEventGroupWaitBits()` 等待期望的事件，发生之后再使用 `xEventGroupClearBits()` 来清除。但是这两个函数之间，有可能被其他任务或中断**抢占**，它们可能会**修改**事件组。

可以使用设置 `xClearOnExit` 为 `pdTRUE`，使得对事件组的测试、清零都在 `xEventGroupWaitBits()` 函数内部完成，这是一个原子操作。

#### 三、实操
e.g. 创建 3 个任务: 任务 1：累加 100000000 次，然后设置事件 bit 0；任务 2：累减 50000000 次，然后设置事件 bit 1 ；任务 3：等待  事件 0 和事件 1（AND 逻辑）

在这之前要在`FreeRTOS files`里面添加底层文件，这个底层文件在`source`文件夹里面

```c
void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 100000; i++)
		{
			sum++;
		}
		xQueueSend(QueueCaclHandle, &sum, 0);
		xEventGroupSetBits( xEventCacl, (1<<0));
	}
}

void Task2Function(void * param)
{
	while (1)
	{	
		volatile int i;
		
		for(i = 0; i < 100000; i++)
		{
			dec--;
		}
		xQueueSend(QueueCaclHandle, &dec, 0);
		xEventGroupSetBits(xEventCacl, (1<<1));
	}
}

void Task3Function(void * param)
{
	int val1,val2;
	while (1)
	{	
		xEventGroupWaitBits(xEventCacl, (1<<0) | (1<<1), pdTRUE, pdTRUE, portMAX_DELAY);
		xQueueReceive(QueueCaclHandle, &val1, 0);
		xQueueReceive(QueueCaclHandle, &val2, 0);

		printf("val1 = %d , val2 = %d\r\n", val1, val2);
	}
}

// 主函数
QueueCaclHandle = xQueueCreate(2, sizeof(int));

xEventCacl = xEventGroupCreate();	
```

如果编译不成功的话要去掉这一行，显示定义要在函数前面。

![image-20260319132456529](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260319132456529.png)

#### 四、事件组的同步

使用 `xEventGroupSync()` 函数可以同步多个任务：

- 可以设置某位、某些位，表示自己做了什么事
- 可以等待某位、某些位，表示要等等其他任务
- 期望的事件发生后，`xEventGroupSync()` 才会成功返回
- `xEventGroupSync()` 成功返回后，会自动清除事件

```c
EventBits_t xEventGroupSync( EventGroupHandle_t xEventGroup,
                             const EventBits_t uxBitsToSet,
                             const EventBits_t uxBitsToWaitFor,
                             TickType_t xTicksToWait );
```

| 参数              | 说明                                                         |
| ----------------- | ------------------------------------------------------------ |
| `xEventGroup`     | 操作哪个事件组？                                             |
| `uxBitsToSet`     | 要设置哪些事件？我完成了哪些事件？比如 `0x05`（二进制 `0101`）会导致事件组的 `bit0`、`bit2` 被设置为 1 |
| `uxBitsToWaitFor` | 等待哪个 / 哪些位？比如 `0x15`（二进制 `10101`），表示要等待 `bit0`、`bit2`、`bit4` 都为 1 |
| `xTicksToWait`    | 如果期待的事件未发生，阻塞多久：- `0`：判断后即刻返回- `portMAX_DELAY`：一直等到成功才返回- 其他值：期望的 Tick Count，一般用 `pdMS_TO_TICKS()` 把 ms 转换为 Tick Count |
| **返回值**        | 返回事件值：- 事件发生时：返回 “非阻塞条件成立” 时的事件值- 超时时：返回超时时刻的事件值 |

e.g. 有一个事情需要多个任务协同，比如：

- 任务 A：炒菜
- 任务 B：买酒
- 任务 C：摆台
- A、B、C 做好自己的事后，还要等别人做完；大家一起做完，才可开饭

```c
static void vCookingTask( void *pvParameters )
{
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 100UL );		
	int i = 0;
	
	/* 无限循环 */
	for( ;; )
	{
		/* 做自己的事 */
		printf("%s is cooking %d time....\r\n", (char *)pvParameters, i);
		
		/* 表示我做好了, 还要等别人都做好 */
		xEventGroupSync(xEventGroup, COOKING, ALL, portMAX_DELAY);
	
		/* 别人也做好了, 开饭 */
		printf("%s is eating %d time....\r\n", (char *)pvParameters, i++);
		vTaskDelay(xTicksToWait);
	}
}

static void vBuyingTask( void *pvParameters )
{
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 100UL );		
	int i = 0;
	
	/* 无限循环 */
	for( ;; )
	{
		/* 做自己的事 */
		printf("%s is buying %d time....\r\n", (char *)pvParameters, i);
		
		/* 表示我做好了, 还要等别人都做好 */
		xEventGroupSync(xEventGroup, BUYING, ALL, portMAX_DELAY);
	
		/* 别人也做好了, 开饭 */
		printf("%s is eating %d time....\r\n", (char *)pvParameters, i++);
		vTaskDelay(xTicksToWait);
	}
}

static void vTableTask( void *pvParameters )
{
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 100UL );		
	int i = 0;
	
	/* 无限循环 */
	for( ;; )
	{
		/* 做自己的事 */
		printf("%s is do the table %d time....\r\n", (char *)pvParameters, i);
		
		/* 表示我做好了, 还要等别人都做好 */
		xEventGroupSync(xEventGroup, TABLE, ALL, portMAX_DELAY);
	
		/* 别人也做好了, 开饭 */
		printf("%s is eating %d time....\r\n", (char *)pvParameters, i++);
		vTaskDelay(xTicksToWait);
	}
}
```


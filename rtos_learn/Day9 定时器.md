### Day9 定时器

------

#### 一、守护任务

FreeRTOS 中有一个 Tick 中断，软件定时器基于 Tick 来运行。但是 RTOS**不允许**在内核、在**中断**中执行不确定的代码；如果定时器函数很耗时，会影响整个系统。

因此要改成在某个任务里执行，这个任务是**RTOS 守护任务**。当 FreeRTOS 的配置项 `configUSE_TIMERS` 被设置为 1 时，在启动调度器（`vTaskStartScheduler`）时，会自动创建 `RTOS Damemon Task`。通过 "**定时器命令队列**"(timer command queue) 和守护任务交互。

![image-20260326105329236](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260326105329236.png)

状态转换图：

![image-20260326105504341](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260326105504341.png)

#### 二、定时器的使用

准备工作：

```c
/* 1. 工程中 */
添加 timer.c

/* 2. 配置文件FreeRTOSConfig.h中 */
#define configUSE_TIMERS                1    /* 使能定时器 */
#define configTIMER_TASK_PRIORITY       configMAX_PRIORITIES - 1   
                                        /* 守护任务的优先级，尽可能高一些 */
    						       /* 这里的最高优先级只能是-1，并不等于configMAX_PRIORITIES */
#define configTIMER_QUEUE_LENGTH        5    /* 命令队列长度 */
#define configTIMER_TASK_STACK_DEPTH    32   /* 守护任务的栈大小 */

/* 3. 源码中 */
#include "timers.h"
```

创建任务函数：

```c
/* 使用动态分配内存的方法创建定时器
* pcTimerName:定时器名字，用处不大，尽在调试时用到
* xTimerPeriodInTicks: 周期，以Tick为单位
* uxAutoReload: 类型，pdTRUE表示自动加载，pdFALSE表示一次性
* pvTimerID: 回调函数可以使用此参数，比如分辨是哪个定时器
* pxCallbackFunction: 回调函数
* 返回值: 成功则返回TimerHandle_t，否则返回NULL
*/
TimerHandle_t xTimerCreate( const char * const pcTimerName,
                            const TickType_t xTimerPeriodInTicks,
                            const UBaseType_t uxAutoReload,
                            void * const pvTimerID,
                            TimerCallbackFunction_t pxCallbackFunction );
```

启动任务函数：

```c
/* 启动定时器
* xTimer: 哪个定时器
* xTicksToWait: 超时时间
* 返回值: pdFAIL表示"启动命令"在xTicksToWait个Tick内无法写入队列
*        pdPASS表示成功
*/
BaseType_t xTimerStart( TimerHandle_t xTimer, TickType_t xTicksToWait );
```

例程：（观察标志位的翻转）

```c
// 主函数
xHandleTimer = xTimerCreate("myTimer", 100, pdTRUE, NULL, myTimerCallbackFunction);
xTaskCreate(Task1Function, "Task1", 100, NULL, 1, &xHandleTask1);

// Task1
void Task1Function(void * param)
{
	xTimerStart(xHandleTimer, 0); //这里是要放在while循环外面	
	while (1)
	{
		printf("Task1 is running\r\n");
	}
}

// 定时器回调函数
void myTimerCallbackFunction( TimerHandle_t xTimer )
{
	int i = 0;
	Timerflag = !Timerflag;
	printf("Timer Task%d is running\r\n", &i);
}
```

#### 三、利用定时器消除抖动

示意图：

![image-20260326120553290](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260326120553290.png)

每次中断都复位一下定时器，给定时器一点延时，然后触发，记住不能自动重装载，为**单次触发**。



#### 四、中断管理

以写队列为例：

|                    | xQueueSendToBack                     | xQueueSendToBackFromISR                                      |
| ------------------ | ------------------------------------ | ------------------------------------------------------------ |
| **参数不同**       | `xTicksToWait`: 队列满的话阻塞多久   | 没有`xTicksToWait`                                           |
| **唤醒等待的任务** | 写队列后，会唤醒等待数据的任务       | 写队列后，会**唤醒**等待数据的任务                           |
| **调度**           | 如果被唤醒的任务优先级更高，即刻调度 | 如果被唤醒的任务优先级更高，**不会调度**，**只是记录**下来表示：需要调度 |
| **阻塞**           | 如果队列满，可以阻塞                 | 如果队列满，不能阻塞                                         |

函数对比：

| 类型               | 在任务中          | 在 ISR 中                |
| ------------------ | ----------------- | ------------------------ |
| 队列 (queue)       | xQueueSendToBack  | xQueueSendToBackFromISR  |
|                    | xQueueSendToFront | xQueueSendToFrontFromISR |
|                    | xQueueReceive     | xQueueReceiveFromISR     |
|                    | xQueueOverwrite   | xQueueOverwriteFromISR   |
|                    | xQueuePeek        | xQueuePeekFromISR        |
| 信号量 (semaphore) | xSemaphoreGive    | xSemaphoreGiveFromISR    |
|                    | xSemaphoreTake    | xSemaphoreTakeFromISR    |

| 类型                         | 在任务中           | 在 ISR 中                 |
| ---------------------------- | ------------------ | ------------------------- |
| 事件组 (event group)         | xEventGroupSetBits | xEventGroupSetBitsFromISR |
|                              | xEventGroupGetBits | xEventGroupGetBitsFromISR |
| 任务通知 (task notification) | xTaskNotifyGive    | vTaskNotifyGiveFromISR    |
|                              | xTaskNotify        | xTaskNotifyFromISR        |
| 软件定时器 (software timer)  | xTimerStart        | xTimerStartFromISR        |
|                              | xTimerStop         | xTimerStopFromISR         |
|                              | xTimerReset        | xTimerResetFromISR        |
|                              | xTimerChangePeriod | xTimerChangePeriodFromISR |

e.g. 现在在外部中断里面

```c
/*
 * 往队列尾部写入数据，此函数可以在中断函数中使用，不可阻塞
 * 原始函数
 */
BaseType_t xQueueSendToBackFromISR(
        QueueHandle_t xQueue, // 目标队列句柄
        const void *pvItemToQueue, // 待写入数据的指针
        BaseType_t *pxHigherPriorityTaskWoken 
);

/* 用法示例 */
BaseType_t xHigherPriorityTaskWoken = pdFALSE; // 输出参数，用于在中断退出时触发任务调度
xQueueSendToBackFromISR(xQueue, pvItemToQueue, &xHigherPriorityTaskWoken);

/* 最后再决定是否进行任务切换 
 * 假如xQueueSendToBackFromISR在一个for循环里面，portYIELD_FROM_ISR也不要放在里面，放在最后
 * xHigherPriorityTaskWoken为pdTRUE时才切换
 */
portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```

分析任务调度的内核：

```c
// 找 portYIELD_FROM_ISR 原型，为什么需要 xHigherPriorityTaskWoken
#define portYIELD_FROM_ISR( x )                     portEND_SWITCHING_ISR( x )
// 到这里面就能看到 portYIELD 任务调度
#define portEND_SWITCHING_ISR( xSwitchRequired )    do { if( xSwitchRequired != pdFALSE ) portYIELD(); } while( 0 )
// 向 NVIC 寄存器写值 → 触发 PendSV 中断
// PendSV 是 FreeRTOS 专门用来做任务上下文切换的中断 只要发生任务切换，一定是 PendSV 做的 ！！！
#define portYIELD()                                 
{                                                   
    /* 触发 PendSV，请求上下文切换 */
    portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT; 
    
    /* Barriers are normally not required but do ensure the code is completely \
     * within the specified behaviour for the architecture. */ 
    __dsb( portSY_FULL_READ_WRITE );                           
    __isb( portSY_FULL_READ_WRITE );                           
}
// pendsv内核的函数
__asm void xPortPendSVHandler( void )
{
    extern uxCriticalNesting;
    extern pxCurrentTCB;
    extern vTaskSwitchContext;

/* *INDENT-OFF* */
    PRESERVE8

    mrs r0, psp
    isb

    ldr r3, = pxCurrentTCB /* Get the location of the current TCB. */
    ldr r2, [ r3 ]

    stmdb r0 !, { r4 - r11 } /* Save the remaining registers. */
    str r0, [ r2 ] /* Save the new top of stack into the first member of the TCB. */

    stmdb sp !, { r3, r14 }
    mov r0, #configMAX_SYSCALL_INTERRUPT_PRIORITY
    msr basepri, r0
    dsb
    isb
    bl vTaskSwitchContext
    mov r0, #0
    msr basepri, r0
    ldmia sp !, { r3, r14 }

    ldr r1, [ r3 ]
    ldr r0, [ r1 ] /* The first item in pxCurrentTCB is the task top of stack. */
    ldmia r0 !, { r4 - r11 } /* Pop the registers and the critical nesting count. */
    msr psp, r0
    isb
    bx r14
    nop
/* *INDENT-ON* */
}
```

**Q1：滴答定时器有中断吗？**

有！SysTick_Handler，每 1ms 进一次，是系统心跳。

**Q2：系统怎么知道10ms到了？**

SysTick中断计数，数够10次就知道到了。

**Q3：延时到了用哪个中断切换？**

SysTick只负责**唤醒**，真正切换靠**PendSV**。



**问题：**

1. 正在运行：优先级 1 的任务 A

2. **来了一个外部中断**（比如串口发送中断）
3. 中断里调用 `xQueueSendFromISR`
4. **唤醒了另一个 优先级 1 的任务 B**

5. 最后调用 `portYIELD_FROM_ISR()`

答案：同优先级任务，遵循 “时间片轮转” + “中断唤醒立即切换”

![image-20260326230816879](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260326230816879.png)


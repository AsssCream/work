### Day6 信号量

#### 一、信号量理论

信号量框图：

![image-20260318102534256](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260318102534256.png)

队列与信号量的区别：

| 队列 | 信号量 |
|:---|:---|
| 可以容纳多个数据，<br>创建队列时有2部分内存：队列结构体、存储数据的空间 | 只有计数值，无法容纳其他数据。<br>创建信号量时，只需要分配信号量结构体 |
| 生产者：没有空间存入数据时可以阻塞 | 生产者：用于不阻塞，计数值已经达到最大时返回失败 |
| 消费者：没有数据时可以阻塞 | 消费者：没有资源时可以阻塞 |

#### 二、 信号量函数

使用信号量时，先创建、然后去添加资源、获得资源。使用句柄来表示一个信号量。

1.  ##### 创建

使用信号量之前，要先创建，得到一个句柄；使用信号量时，要使用句柄来表明使用哪个信号量。
| | 二进制信号量 | 计数型信号量 |
|:---|:---|:---|
| 动态创建 | xSemaphoreCreateBinary<br>计数值初始值为0 | xSemaphoreCreateCounting |
| | vSemaphoreCreateBinary(过时了)<br>计数值初始值为1 | |
| 静态创建 | xSemaphoreCreateBinaryStatic | xSemaphoreCreateCountingStatic |

```c
/* 创建一个二进制信号量，返回它的句柄。
 * 此函数内部会分配信号量结构体
 * 返回值: 返回句柄，非NULL表示成功
*/
SemaphoreHandle_t xSemaphoreCreateBinary( void );
/* 创建一个二进制信号量，返回它的句柄。
 * 此函数无需动态分配内存，所以需要先有一个StaticSemaphore_t结构体，并传入它的指针
 * 返回值: 返回句柄，非NULL表示成功
*/
SemaphoreHandle_t xSemaphoreCreateBinaryStatic( StaticSemaphore_t *pxSemaphoreBuffer );
```

```c
/* 创建一个计数型信号量，返回它的句柄。
 * 此函数内部会分配信号量结构体
 * uxMaxCount: 最大计数值
 * uxInitialCount: 初始计数值
 * 返回值: 返回句柄，非NULL表示成功
*/
SemaphoreHandle_t xSemaphoreCreateCounting(UBaseType_t uxMaxCount, UBaseType_t uxInitialCount);
```

2. ##### give / take

```c
BaseType_t xSemaphoreGive( SemaphoreHandle_t xSemaphore );
BaseType_t xSemaphoreTake(SemaphoreHandle_t xSemaphore, TickType_t xTicksToWait);
```



e.g. 通过计数型信号量去实现任务间的同步

```c
static SemaphoreHandle_t xSemCacl;

void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 10000000; i++)
		{
			sum++;
		}
		xSemaphoreGive(xSemCacl);
		
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	while (1)
	{
		Task2runflag = 0;
		xSemaphoreTake(xSemCacl, portMAX_DELAY);
		Task2runflag = 1;
		printf("sum = %d\r\n", sum);
	}
}

// 主函数
xSemCacl = xSemaphoreCreateCounting(10 , 0);
```

e.g. 通过二进制型信号量去实现任务间的互斥

```c
void TaskGenericFunction(void * param)
{
	while (1)
	{
		xSemaphoreTake(xSemUART, portMAX_DELAY); // 获得锁
		printf("%s\r\n",(char *)param);
		xSemaphoreGive(xSemUART);  // 释放锁
		vTaskDelay(2);
	}
}

// 主函数
xSemUART = xSemaphoreCreateBinary(); // 初始值为 0
xSemaphoreGive(xSemUART); // 先让这个二进制数为1

xTaskCreate(TaskGenericFunction, "Task3", 100, "task3 is running", 1, NULL);
xTaskCreate(TaskGenericFunction, "Task4", 100, "task4 is running", 1, NULL);
```

#### 三、 互斥量的缺陷与互斥锁

互斥量是特殊的信号量也被称为互斥锁，可以解决谁上锁谁解锁的问题，还有优先级继承的方法去解决优先级翻转。

使用过程如下:

1. 互斥量初始值为1

2. 任务A想访问临界资源，先获得并占有互斥量，然后开始访问

3. 任务B也想访问临界资源，也要先获得互斥量：被别人占有了，于是阻塞

4. 任务A使用完毕，释放互斥量；任务B被唤醒、得到并占有互斥量，然后开始访问临界资源

5. 任务B使用完毕，释放互斥量

函数造成死锁例子：

![image-20260319001930820](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260319001930820.png)

1. ##### 创建

使用互斥量时，先创建、然后去获得、释放它。使用句柄来表示一个互斥量。

```c
/* 创建一个互斥量，返回它的句柄。
 * 此函数内部会分配互斥量结构体
 * 返回值：返回句柄，非NULL表示成功
 */
SemaphoreHandle_t xSemaphoreCreateMutex( void );

/* 创建一个互斥量，返回它的句柄。
 * 此函数无需动态分配内存，所以需要先有一个StaticSemaphore_t结构体，并传入它的指针
 * 返回值：返回句柄，非NULL表示成功
 */
SemaphoreHandle_t xSemaphoreCreateMutexStatic( StaticSemaphore_t *pxMutexBuffer );

// 要想使用互斥量，需要在配置文件FreeRTOSConfig.h中定义:
##define ConfigUSE_MUTEXES 1
```

2. ##### 其他函数比如删除、give/take，跟一般是信号量是一样的。

e.g. 常规使用，通过二进制型信号量去实现任务间的互斥（跟上面一样）

```c
// 主函数
xSemUART = xSemaphoreCreateBinary(); // 初始值为 0
xSemaphoreGive(xSemUART); // 先让这个二进制数为1

// 转换成
xSemUART = xSemaphoreCreateMutex();
```



**注意**：FreeRTOS并没有实现"**谁上锁就得由谁开锁**"的功能。

**A**: 任务 1 的优先级高，先运行，立刻上锁

**B**: 任务 1 阻塞

**C**: 任务 2 开始执行，尝试获得互斥量 (上锁)，超时时间设为 0。根据返回值打印出：上锁失败

**D**: 任务 2 监守自盗，开锁，成功！

**E**: 任务 2 成功获得互斥量

**F**: 任务 2 阻塞



**递归锁**实现了：谁上锁就由谁解锁。

```c
/* 这样任务就会乱套，因为有人胡乱开锁和上锁 */
void TaskGenericFunction(void * param)
{
	while (1)
	{
		xSemaphoreTake(xSemUART, portMAX_DELAY);
		printf("%s\r\n",(char *)param);
		xSemaphoreGive(xSemUART);
		vTaskDelay(2);
	}
}

void Task5Function(void * param)
{
	vTaskDelay(10);
	while (1)
	{
		while(1)
		{
			if(xSemaphoreTake(xSemUART, 0) != pdTRUE)
			{
				
				xSemaphoreGive(xSemUART);
			}
			else
				break;
		}
		printf("task5 is running\r\n");
		xSemaphoreGive(xSemUART);
		vTaskDelay(2);
	}
}
```

解决方法：加入**递归锁**

```c
void TaskGenericFunction(void * param)
{
	while (1)
	{
		xSemaphoreTakeRecursive(xSemUART, portMAX_DELAY);
		printf("%s\r\n",(char *)param);
		xSemaphoreGiveRecursive(xSemUART);
		vTaskDelay(2);
	}
}

void Task5Function(void * param)
{
	vTaskDelay(10);
	while (1)
	{
		while(1)
		{
			if(xSemaphoreTakeRecursive(xSemUART, 0) != pdTRUE)
			{
				
				xSemaphoreGiveRecursive(xSemUART);
			}
			else
				break;
		}
		printf("task5 is running\r\n");
		xSemaphoreGiveRecursive(xSemUART);
		vTaskDelay(2);
	}
}

//主函数 记录谁开锁谁上锁
xSemUART = xSemaphoreCreateRecursiveMutex();
```





**优先级反转**现象：

假设任务 A、B 都想使用串口，A 优先级比较低：

- 任务 A 获得了串口的互斥量
- 任务 B 也想使用串口，它将会阻塞、等待 A 释放互斥量
- 高优先级的任务，被低优先级的任务延迟，这被称为优先级反转**。如果涉及3个任务，可以让"优先级反转"的后果更加恶劣。

而互斥量可以通过**优先级继承**，可以很大程度解决**优先级反转**的问题，这也是FreeRTOS中互斥量和二级制信号量的差别。

PTask/MPTask/HPTask 三个任务的运行过程：

- A: HPTask 优先级最高，它最先运行。在这里故意打印，这样才可以观察到 flagHPTaskRun 的脉冲。
- HP Delay: HPTask 阻塞

------

- B: MPTask 开始运行。在这里故意打印，这样才可以观察到 flagMPTaskRun 的脉冲。
- MP Delay : MPTask 阻塞
- C: LPTask 开始运行，获得二进制信号量，然后故意打印很多字符
- D: HP Delay 时间到，HPTask 恢复运行，它无法获得二进制信号量，一直阻塞等待
- E: MP Delay 时间到，MPTask 恢复运行，它比 LPTask 优先级高，一直运行。导致 LPTask 无法运行，自然无法释放二进制信号量，于是 HPTask 用于无法运行。

总结:

- LPTask 先持有二进制信号量，
- 但是 MPTask 抢占 LPTask，使得 LPTask 一直无法运行也就无法释放信号量
- 导致 HPTask 任务无法运行
- 优先级最高的 HPTask 竟然一直无法运行！

示例代码：

![image-20260319013829019](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260319013829019.png)

思路：A先运行，然后delay，给B之后B也delay，到C之后C拿了锁之后也delay，但是delay 的时间长，这个时候A的delay结束，但是C有可能不能开锁，到A的时间不能运行，因为没有开锁，阻塞状态任务给了B运行。

程序运行的时序图如下：

![image-20260319013122426](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260319013122426.png)

如果能提升LPTask任务的优先级，让它能尽快运行、释放锁，“优先级反转"的问题不就解决了吗？

因此提出了**优先级继承**

代码在之前的基础上做了一点修改

```c
int main( void )
{
    prvSetupHardware();

    /* 创建互斥量/二进制信号量 */
    //xLock = xSemaphoreCreateBinary();
    xLock = xSemaphoreCreateMutex();
```

帮助理解：

- 假设持有互斥锁的是任务 A，如果更高优先级的任务 B 也尝试获得这个锁
- 任务 B 说：你既然持有宝剑，又不给我，那就继承我的愿望吧
- 于是任务 A 就继承了任务 B 的优先级
- 这就叫：优先级继承
- 等任务 A 释放互斥锁时，它就恢复为原来的优先级
- 互斥锁内部就实现了优先级的提升、恢复

运行时序图如下图所示：

- A: HPTask 执行`xSemaphoreTake(xLock, portMAX_DELAY);`，它的优先级被 LPTask 继承
- B: LPTask 抢占 MPTask，运行
- C: LPTask 执行`xSemaphoreGive(xLock);`，它的优先级恢复为原来值
- D: HPTask 得到互斥锁，开始运行
- 互斥锁的 "优先级继承"，可以减小 "优先级反转" 的影响

![image-20260319013750461](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260319013750461.png)
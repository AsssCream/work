### Day3 钩子函数 + 任务调度算法

#### 1. 空闲任务和钩子函数

任务的清理工作和回收工作是放在空闲任务里面的

像下面这样的自我消灭删除任务的方法会让CPU没有时间去处理内存调度

```c
void myTask1(void *pamas)
{
	while (1)
	{
		printf("1");
		xTask2Return = xTaskCreate(myTask2,"Task2", 100, NULL, 2, &xHandleTask2);
		if(xTask2Return != pdPASS) printf("xTask2Create error!\r\n");
	}
}
void myTask2(void *pamas)
{
	while (1)
	{
		printf("2");
		vTaskDelay(2);
		vTaskDelete(NULL);
	}
}
```

处理方式有两种：

① 只要有任务主动让出CPU（延时或者阻塞）
```c
void myTask1(void *pamas)
{
	while (1)
	{
		printf("1");
		xTask2Return = xTaskCreate(myTask2,"Task2", 100, NULL, 2, &xHandleTask2);
		vTaskDelay(10);
		if(xTask2Return != pdPASS) printf("xTask2Create error!\r\n");
	}
}
```

② 开启钩子函数（这个时候上面的task1和空闲任务要同一个优先级，不然空闲函数执行不了）

![image-20260315132642929](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260315132642929.png)

```c
void vApplicationIdleHook( void )
{
	int xTask1runflag = 0;
	int xTask2runflag = 0;
	int xTask3runflag = 0;
	printf("Idle Task\r\n");
}
```
#### 2. 调度策略

##### ① 可否抢占？（配置项：`configUSE_PREEMPTION`）

控制高优先级任务是否能优先执行。

- **可以：可抢占调度（Pre-emptive）**

  - 高优先级的就绪任务会**立即抢占**当前任务并执行。
  - 后续可进一步细分其他调度规则。
  
- **不可以：合作调度模式（Co-operative Scheduling）**

  - 当前任务执行时，即使更高优先级任务就绪，也**无法立即运行**，必须等待当前任务主动让出 CPU 资源。
  
  - 同优先级任务也必须等待：更高优先级任务都无法抢占，同优先级任务自然也需排队等待。
  

![image-20260315142117660](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260315142117660.png)


------

##### ② 可抢占前提下，同优先级任务是否轮流执行？（配置项：`configUSE_TIME_SLICING`）

在开启抢占的基础上，控制同优先级任务的调度方式。

- **轮流执行：时间片轮转（Time Slicing）**

  - 同优先级任务按时间片轮流占用 CPU，每个任务执行一个时间片后切换到下一个任务。（公平分配）

- **不轮流执行（without Time Slicing）**

  - 当前任务会**持续执行**，直到主动放弃 CPU，或者被更高优先级任务抢占。
  

![image-20260315142138352](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260315142138352.png)

------

##### ③、「可抢占 + 时间片轮转」前提下，空闲任务是否让步于用户任务？（配置项：`configIDLE_SHOULD_YIELD`）

在同时开启抢占和时间片轮转的基础上，进一步细化空闲任务的调度行为。

- **空闲任务让步：空闲任务优先级更低**

  - 空闲任务每执行一次循环，就会主动检查是否有就绪的用户任务，并**主动让位**给用户任务执行。

- **空闲任务不让步：空闲任务与用户任务平等**

- 空闲任务和用户任务按时间片**轮流执行**，没有优先级差异，地位平等。

![image-20260315142206559](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260315142206559.png)

#### 3. 同步互斥与通信

① 有缺陷的同步

```c
void Task1Function(void * param)
{
	while (1)
	{
		volatile int i;
		
		for(i = 0; i < 10000000; i++)
		{
			sum++;
		}
		Task2runflag = 1;
		
		vTaskDelete(NULL);
	}
}

void Task2Function(void * param)
{
	while (1)
	{
        // 这种一直等待判断的方式会一直占用着CPU的资源，在不执行的时候应该让Task2进入休眠状态或者当前任务进入阻塞状态
		if(Task2runflag == 1)
			printf("sum = %d", sum);
	}
}
```

② 有缺陷的互斥
```c
void TaskGenericFunction(void * param)
{
	while (1)
	{
        //但是这个判断还是有缺陷的，因为在判断和让标志位置1之间是有时间间隔的
		if(!Task34runflag)
		{
			Task34runflag = 1;
			printf("%s\r\n",(char *)param);
			Task34runflag = 0;			
			vTaskDelay(1); //这个延时必须要加在后面 如果加在flag的前面就和没加延时没啥区别
		}
	}
}

// 主函数
xTaskCreate(TaskGenericFunction, "Task3", 100, "task3 is running", 1, NULL);
xTaskCreate(TaskGenericFunction, "Task4", 100, "task4 is running", 1, NULL);
```






































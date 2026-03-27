### Day2 RTOS点亮LED + 数据类型 + 创建任务

#### 1. ⭐️⭐️⭐️⭐️⭐️ 调试错误：debug*** error 65: access violation at 0x4002100C : no 'write' permission

解决方法：https://blog.csdn.net/djdh548di/article/details/148932966

debug目录下：

Dialog DLL默认是DARMSTM.DLL

Parameter默认是-pSTM32F103ZE

------

#### 2. F103ZET6的灯是在PB5上面(蓝色)
``` c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE );	
   
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_5;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
   
GPIO_Init( GPIOB, &GPIO_InitStructure ); 

   // 主函数
void myTask2(void *pamas)
{
	while (1)
	{
		GPIO_WriteBit(GPIOB, GPIO_Pin_5, Bit_RESET);
		delay_ms(1000);
		GPIO_WriteBit(GPIOB, GPIO_Pin_5, Bit_SET);
		delay_ms(1000);	
	}
}
```

------

#### 3. 数据类型分析和变量名定义：

![image-20260314115507905](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260314115507905.png)

​	变量名：

![image-20260314115553080](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260314115553080.png)

​	函数名：

![image-20260314115613783](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260314115613783.png)

​	宏的名：

![image-20260314115652348](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260314115652348.png)

------

#### 4. 指针和句柄的联系

我们平时说的handle句柄其实是指针，指的是对底层硬件实例的指针的引用（e.g. 多个进程都要用到声卡，显卡，但是不可能给每一个进程都分配相同的空间去控制）是为了更高效地管理资源。任务调度里面的TCB结构体可以动态分配可以静态分配。在执行过程中会预先分配好空间给这个结构体。

   视频拓展：
   【句柄与指针，傻傻分不清】 https://www.bilibili.com/video/BV1Vb4y1d7rp/?share_source=copy_web&vd_source=1022ddc0d0b28239e5c77177c0f844a5
   指针是放在栈里面的，是分配的，而指针对应的malloc分配出来的空间是在堆里面的。

------

#### 5. 静态任务配置：

   流程图：
   ![image-20260314132902608](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260314132902608.png)

   代码：

   ```c
   StackType_t xTask3Stack[100];
   
   StaticTask_t xTask3TCB;
   
   StackType_t xIdleTask3Stack[100];
   StaticTask_t xIdleTask3TCB;
   
   
   // vApplicationGetIdleTaskMemory( &pxIdleTaskTCBBuffer, &pxIdleTaskStackBuffer, &ulIdleTaskStackSize );
   
   void vApplicationGetIdleTaskMemory( StaticTask_t ** ppxIdleTaskTCBBuffer,
   										   StackType_t ** ppxIdleTaskStackBuffer,
   										   uint32_t * pulIdleTaskStackSize )
   {
   	* ppxIdleTaskTCBBuffer = &xIdleTask3TCB;
   	* ppxIdleTaskStackBuffer = xIdleTask3Stack;
   	* pulIdleTaskStackSize = 100;
   }
   
   // 主函数
   xTaskCreateStatic(myTask3,"Task3", 100, NULL, 1, xTask3Stack, &xTask3TCB);
   ```

------

#### 6. RTOS优先级考虑：

RTOS里面设置优先级，如果都是同等的优先级就会交错运行，如果是不同的优先级就会先运行优先级高的，如果优先级高的任务不释放就不会运行低优先级的任务。

------

#### 7. 设置一个通用的函数：TaskGenericFunction

```c
void TaskGenericFunction(void *parm)
{
	int val = (int)parm; //这里注意是要类型转换 void* 不能直接赋值给int 这个parm是一个指针变量
	while (1)
	{
		printf("%d",val);
	}	
}

//主函数
xTaskCreate(TaskGenericFunction,"Task4", 100, (void *)4, 1, NULL);
xTaskCreate(TaskGenericFunction,"Task5", 100, (void *)5, 1, NULL);
```
------

#### 8. 在删除任务之后如果继续删除的话会报错：

```c
// 这个是错误的代码！
for(i = 0; i < 100; i++)
{
	printf("111");
	vTaskDelete(xHandleTask1);  // 循环100次，重复删除Task1！
}
```
------

#### 9. 栈地址是从高到低写入的，如果分配的栈空间不够的话就会崩溃

#### （一定要思考栈空间选多大）：

答疑：内存里是存text，data， bss， 堆， 栈， 创建的每个任务都有自己的栈，这个栈实际上是在堆上。

![image-20260314203416746](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260314203416746.png)

------

#### 10. 任务状态转换图
![image-20260314210225639](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260314210225639.png)
每一个任务其实都在一个链表里面

------

#### 11.  ⭐️⭐️⭐️⭐️⭐️ 晶振的设置！

![image-20260314231618837](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260314231618837.png)

正确配置：8MHz × 9 = 72MHz 
SysTick 重载值 = 72MHz / 1000 - 1 = 71999 → 1ms 触发一次中断 → `xTaskGetTickCount()` 1tick=1ms

------

#### 12. 任务之间的影响（代码）

```c
void myTask1(void *pamas)
{
	TickType_t t_start = xTaskGetTickCount(); //得到原始时间

	TickType_t t;
		
	int flag = 0;
	
	while (1)
	{
		
		t = xTaskGetTickCount();// 得到现在时间
		
		task1runflag = 1;
		task2runflag = 0;
		task3runflag = 0;
		printf("1");
		
        // 让任务三中断10ms 
		if(!flag && (t > t_start + 10))
		{
			vTaskSuspend(xHandleTask3);
			flag = 1;
		}
		

		if(t > t_start + 20)
		{
			vTaskResume(xHandleTask3);
		}
			
	}
}

void myTask2(void *pamas)
{
	int i = 0;
	
	while (1)
	{
		task1runflag = 0;
		task2runflag = 1;
		task3runflag = 0;

		printf("2");	
		
        // 阻塞10ms
		vTaskDelay(10);

	}
}
```

-----

#### 13. vTaskDelay和vTaskDelayUntil的区别

![image-20260315002335596](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260315002335596.png)

前者是每一次的延时时间是固定的
后者是进去和出来的时间都是固定的（周期是一样的）

做测试时候要注意把测试的任务优先级设高，不然其他任务的运行会影响当前的任务
代码如下：

```c
// void vTaskDelay( const TickType_t xTicksToDelay );
// BaseType_t xTaskDelayUntil( TickType_t * const pxPreviousWakeTime, const TickType_t xTimeIncrement );

void myTask1(void *pamas)
{
	TickType_t t_start = xTaskGetTickCount(); //得到原始时间
	
	int i;
	
	int j;
	
	while (1)
	{
		// 注意 这个标志位不能放在循环的后面
		task1runflag = 1;
		task2runflag = 0;
		task3runflag = 0;
		
		for(i = 0; i < rands[j]; i++)
		{
			printf("1"); // 其实这里已经是一个用串口打印去控制高电平的持续时间
		}

		if(j++ == 5)
			j = 0;
		
		
#if 0
		vTaskDelay(20);
#else
		vTaskDelayUntil(&t_start, 20);
#endif
			
	}
}

```

### Day1 修改串口重定义 + 第一个RTOS程序 + 初步理解

```c
int fputc( int ch, FILE *f )
{
	USART_TypeDef* USARTx = USART1;
	while((USART1->SR & (1<<7)) == 0); // SR状态寄存器 TXE: Transmit data register empty
    // while 循环写在前面的话就可以避免串口死等发送的情况，先发送等下一次来的时候再判断就好
	USART1 ->DR = ch; // DR寄存器可读可写
	return ch;
}
```

FreeRtos核心就是以下这几个文件

![image-20260313112043009](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260313112043009.png)

进入keil的仿真程序之后，如果想看串口情况的话，就直接在view -> serial windows ->uart

RTOS里面最重要的就是下面这个代码

```c
BaseType_t xTaskCreate(
    TaskFunction_t pxTaskCode,  // 任务函数
    const char * const pcName,  // 任务名
    const configSTACK_DEPTH_TYPE usStackDepth, // 栈大小
    void * const pvParameters,  // 任务参数 这个就是任务函数里面输入的参数 如果没有的话可以设置为NULL
    UBaseType_t uxPriority,     // 优先级
    TaskHandle_t * const pxCreatedTask // 任务句柄（输出参数） 如果不用输出的话可以设置为NULL
);
```

![image-20260313125655717](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260313125655717.png)

以下是部分代码

```c
// 这里是指函数指针的类型
// 在xTaskCreate里面已经声明了 TaskFunction_t pxTaskCode
// 就代表了pxTaskCode是一个函数指针，最后执行是在这个函数的地址
// 里面传入的参数就是xTaskCreate里面的第四个参数
// 因为现在我不用传参，所以第四个参数为NULL
void myTask1(void *pamas)
{
	while (1)
	{
		printf("1");
	}
}

// main函数里面就是
xTaskCreate(myTask1,"Task1", 100, NULL, 1, &xHandleTask1);
```

-----

### RTOS的初步理解

① 裸机缺点：无法很好的解决任务A，B的相互影响（轮询模式）；无法让高优先级函数立刻执行（可以放在中断里面，但是如果没有中断呢）；若A，B有依赖，效率低。现在用到网络的，AI都要用到RTOS，RTOS可以实现任务的切换，可以看到就像两个程序一起在执行

② RTOS可以有一个简单的理解就是保存现场，恢复现场：

保存现场：把当前执行任务寄存器的值存入栈

恢复现场：把栈里面值拿出来给CPU寄存器

③ 高优先级任务：间歇性运行（阻塞状态），数据易丢失，要处理紧急事务（WiFi任务：等待UART数据，要读UART）

低优先级：可以一直运行，不紧急

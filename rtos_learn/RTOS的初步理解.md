### 写代码要学会抽象

例子：点灯
第一种情况是直接用配置引脚电平的方式去写函数。但是不能避免团队里面有人不会硬件

第二种情况是封装成LED_Control()函数。但是在while循环里面一直判断是哪一个灯，容易造成屎山代码

第三种情况就是用结构体链表的形式封装 ⭐️⭐️⭐️⭐️⭐️

#### 1. 先定义LED结构体

   这里面包含名称的指针，控制函数指针，链表下一个节点的指针

   ```c
struct led_device
{
    char *name;                  // LED名称（唯一标识）
    int (*led_control)(int state);// LED控制函数指针（state=0灭，1亮）
    struct led_device *ptNext;   // 链表下一个节点
};
   ```

#### 2. 再定义全局链表，想象最后是把每一个部分通过结构体指针给连起来

   ```c
struct led_device *g_leds = NULL;
   ```

#### 3. 实现注册函数

   ```c
void register_led(struct led_device *p)
{
    if (p == NULL) return; // 空指针直接返回
    
    // 情况1：链表为空，新节点作为头节点
    if (g_leds == NULL)
    {
        g_leds = p; //g_leds 是全局变量
        p->ptNext = NULL; // 新节点的下一个为空
        return;
    }
    
    // 情况2：链表已有节点，找到最后一个节点，尾插新节点
    struct led_device *temp = g_leds;
    while (temp->ptNext != NULL) // 遍历到链表末尾
    {
        temp = temp->ptNext;
    }
    temp->ptNext = p; // 把新节点挂到末尾
    p->ptNext = NULL; // 新节点的下一个为空
}
   ```

#### 4. 实现查找函数，也就是操作点灯函数

   ```c
struct led_device *get_led(char *name)
{
    if (name == NULL || g_leds == NULL) return NULL;
    
    struct led_device *temp = g_leds;
    // 遍历链表，匹配名字
    while (temp != NULL)
    {
        // 字符串比较，找到匹配的LED
        if (strcmp(temp->name, name) == 0)
        {
            return temp; // 返回找到的LED结构体指针
        }
        temp = temp->ptNext; // 找下一个
    }
    return NULL; // 没找到返回空
}
   ```

#### 5. 加上两个简单的判断函数 并且定义结构体

   ```c
   // LED0的具体控制函数（假设LED0接PA0口）
   int led0_control(int state)
   {
    // state=0：灭，state=1：亮（调用HAL库的写引脚函数）
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_0, state ? GPIO_PIN_SET : GPIO_PIN_RESET);
    return 0; // 返回0表示成功
   }
   // LED_IIC的具体控制函数（假设是IIC扩展的LED，逻辑不同）
   int led_iic_control(int state)
   {
       // 这里是IIC总线控制LED的逻辑（示例）
       if (state == 1)
       {
           // 发送IIC指令：点亮IIC扩展的LED
           printf("IIC LED ON\n");
       }
       else
       {
           // 发送IIC指令：熄灭IIC扩展的LED
           printf("IIC LED OFF\n");
       }
       return 0;
   }
   
   // 定义LED0的结构体（全局变量，初始化时赋值）
   struct led_device g_tled0 = {
       .name = "led0",          // 名字：led0
       .led_control = led0_control, // 绑定LED0的控制函数
       .ptNext = NULL           // 初始下一个为空
   };
   
   // 定义IIC LED的结构体
   struct led_device g_tlediic = {
       .name = "led_iic",
       .led_control = led_iic_control,
       .ptNext = NULL
   };

   ```

#### 6. 初始化函数与主函数

   ```c
   void leds_init(void)
   {
       // 把LED0和IIC LED注册到全局链表（录入花名册）
       register_led(&g_tled0);
       register_led(&g_tlediic);
       
       // 若新增100个LED，只需要：
       // 1. 写该LED的control函数；
       // 2. 定义结构体并赋值；
       // 3. 调用register_led注册；
   }
   
   int main(void)
   {
       // 1. 硬件初始化（HAL库基础初始化）
       HAL_Init();
       SystemClock_Config();
       MX_GPIO_Init(); // 初始化PA0口（LED0硬件）
       
       // 2. LED设备初始化（注册所有LED到链表）
       leds_init();
       
       // 3. 业务逻辑：控制LED0亮
       struct led_device *p_led0 = get_led("led0");
       if (p_led0 != NULL) // 先判断是否找到，避免空指针
       {
           p_led0->led_control(1); // 调用LED0的控制函数，亮灯
       }
       
       // 4. 控制IIC LED灭
       struct led_device *p_led_iic = get_led("led_iic");
       if (p_led_iic != NULL)
       {
           p_led_iic->led_control(0); // 调用IIC LED的控制函数，灭灯
       }
       
       while (1)
       {
           // 循环逻辑...
       }
   }

### 编译错误处理方法总结

   ```

L6200E: Symbol fputc multiply defined (by main.o and serial.o)

意味着 `fputc` 函数在 `main.c` 和 `serial.c` 两个文件中都被定义了
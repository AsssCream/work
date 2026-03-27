### Source Insight使用流程

1. 点击Project -> New Project

2. 找到工程文件，把路径复制到第二行，然后修改文件名字

   ![image-20260313101650914](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260313101650914.png)

3. 点击OK，然后把所有的文件添加进去

![image-20260313101915111](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260313101915111.png)

![image-20260313102011128](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260313102011128.png)

4. 然后同步一下 Project->Synchronize Files

![image-20260313102245084](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260313102245084.png)

5. 在里面查找汇编文件

![image-20260313102726938](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260313102726938.png)

6. 快速注释掉代码

   e.g. 比如我要删掉vTimer2IntHandler相关的内容，避免他影响到了代码

   ① 首先用ctrl + f在汇编文件里面查找你要注释掉的内容，然后在汇编文件里面操作

   ```
   DCD     0 ; vTimer2IntHandler         ; TIM2
   ```

   ② 然后删掉

   ```
   IMPORT vTimer2IntHandler
   ```

7. 快速查找相关文件 ctrl + /  

![image-20260313103551263](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260313103551263.png)

8. 全局搜索

![image-20260313104230904](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260313104230904.png)

9. shift + tab是左移 返回上一步是ctrl + tab 右移是tab 左移是shift+tab
10. 想看到下面的提示框，可以直接在工具栏的view -> Panels -> Context Windows
11. F8 是高亮，F4 是在查找之后跳下一个
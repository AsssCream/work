### 安装过程

csdn文章：https://blog.csdn.net/m0_65152767/article/details/135316846?fromshare=blogdetail&sharetype=blogdetail&sharerId=135316846&sharerefer=PC&sharesource=weixin_52806936&sharefrom=from_link

#### 1. 设置用户签名

git config --global user.name "your name"
git config --global user.email "your email"

检查：$ git config --list

#### 2. 相关语法（在对应的文件夹操作）
① $ git init （如果看不到,git的文件夹就代表隐藏了）

② $ git status （检查本地库状态）

③ $ vim hello.txt 新增文件 ：

**编辑完文件内容之后 按一下键盘左上角的 `Esc` 键**

**按住 `Shift` 键 输入冒号 `:`**

**输入 `wq` 然后按回车（Enter）键**

④ $ git add hello.txt  将工作区的文件添加到暂存区

⑤ $ git commit -m "日志信息" 文件名  将暂存区的文件提交到本地库 这个就是记录是第几次修改

一般每次修改完都用 ②④②⑤

⑥ $ git add .   批量添加修改文件到暂存区

⑦ $ git reflog  查看版本信息

⑧ $ git reset --hard <commit> 后面这个<commit>是指最前面的编号 e.g. $ git reset --hard f6c1c01

⑨ $ cat hello.txt 直接查看文件里面的内容

⑩ $ git log --oneline 这样的话就可以直接查看简短的日志

⑪ $ git config --global core.quotepath false Git 默认会把中文转码显示

⑫ $ git pull 同步最新

#### 3. 和github建立联系

流程：$ git add .  -> $ git commit -m "日志信息" 文件名 -> $ git push

如果出现错误，这是因为在 GitHub 网页端做了某些操作，导致云端多出了一些本地没有的记录。
![image-20260315194121809](C:\Users\killa\AppData\Roaming\Typora\typora-user-images\image-20260315194121809.png)

解决方法：$ git pull origin main 先同步

#### 4. SSL报错解决办法

① 先查看现在的远程地址：git remote -v    **是 `https` 开头的，这就是报错根源。**

② 直接改成 SSH 地址  git remote set-url origin git@github.com:AsssCream/work.git

③ 然后直接 git push

这个时候如果不成功，是因为还没给 Git 配置 **SSH 公钥**

首先生成 SSH 密钥 ：ssh-keygen -t ed25519 -C "github"（一路回车）

再查看并复制公钥 cat ~/.ssh/id_ed25519.pub 

（ 输出一大段以 `ssh-ed25519` 开头的字符串，**全部选中 → 右键复制 **）

最后 去 GitHub 粘贴公钥 https://github.com/settings/ssh/new

1. Title 随便填：`mygit`

2. Key 类型选：`Authentication Key`

3. Key 框里：**粘贴你刚才复制的那一大段**

4. 点下面绿色按钮：**Add SSH key**

5. 回到 Git Bash，再次 push

![image-20260327140418745](C:\Users\lhh\AppData\Roaming\Typora\typora-user-images\image-20260327140418745.png)

如果第二次还是不行的话

切换到 443 端口 SSH：

1. git remote set-url origin ssh://git@ssh.github.com:443/AsssCream/work.git

2. git push

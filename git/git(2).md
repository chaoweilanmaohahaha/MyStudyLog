# git常用命令和基本原理

##### git clone：

为啥要从这个命令开始记起，因为这是我接触git的第一个常用的命令，也就是从远程仓库中将项目克隆到本地仓库中。

这相当于是获得了仓库中文件的一个拷贝，当然获得这样一个拷贝需要的就是一个远程仓库的地址（URL）

###### example：

git clone http://www.kernel.org/pub/scm/git/git.git

上面的的命令中采用的是http协议，当然git提供了多种协议，也可以使用ssh协议，具体有什么区别和协议本身有关吧。

ps：如果要Git源代码的话可以使用git://协议访问。



##### git init：

初始化本地的版本库，这个命令会在当前的文件夹下生成一个.git隐藏文件，那么这个.git文件究竟是个啥？

![TIM图片20190406165144](C:\Users\lenovo\Desktop\TIM图片20190406165144.png)

上面这张图是我打开.git文件夹后所带的文件和文件夹，分别是：

###### hooks：

存放这git中的一些shell脚本文件。

###### info：

存放仓库信息。

###### logs：

保存更新的引用记录。

###### objects:

存放所有的git对象。

###### refs：

保存当前最后一次提交的哈希值。

###### config：

git的配置文件。

###### index:

暂存区。

###### HEAD:

映射到ref引用，找到下一次commit的前一次哈希值


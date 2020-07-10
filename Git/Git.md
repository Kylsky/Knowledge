# Git

## 什么是Git？

Git 是一个开源的分布式版本控制系统，通过分支管理项目的各个版本



## Git安装

### **安装依赖包**

yum install curl-devel expat-devel gettext-devel  openssl-devel zlib-devel

### **安装git**

yum -y install git-core



## git配置

### **用户信息**

git config --global user.name "kyle"

git config --global user.email 752051085@qq.com

使用global会默认对所有项目生效，若想使特定项目使用特定的配置，去掉--global即可，将会保存在当前项目的git/config下



## Git一般工作流程

1.克隆git资源作为工作目录

2.在克隆的资源上添加或修改文件

3.更新其他人的资源，若有冲突则解决后更新

4.在提交前查看修改

5.提交修改



## Git常用命令

### **git init**

在当前目录下生成git仓库

### **git init <目录>**

在指定目录下生成git仓库

### **git add test.txt** 

将文件提交到缓存（暂存区）

### **git commit -m "第一次提交"**

将暂存区的文件提交到当前分支的仓库中，提交前需要先配置git config

**-a**

跳过add缓存步骤，直接提交到仓库中

**-m**

提交的日志

### **git clone <repo> <directory>**

从指定仓库拉取内容到指定文件夹

### **git status**

查看所在分支，以及在上次提交之后是否有修改

### git diff

查看执行git status的结果的详细信息（显示尚未缓存的改动）

--**cached**

查看已缓存的改动

**HEAD**

查看已缓存和未缓存的所有改动

**--stat**

只显示摘要

### git reset HEAD

取消已经缓存的内容

### git rm <file>

若使用正常操作删除文件，那么运行git status时会产生Changes not staged for commit提示，要从git中移除某个文件，必须要从已经跟踪的文件清单中移除

**-f**

若缓存区已有该文件，则需要强制删除

**--cached**

只想删除缓存区而不想删除工作目录中的文件

**-r**

递归删除

### **git mv**

用于移动或重命名一个缓存区中的文件、目录、软连接，并更新工作目录



## Git分支管理

### *git rebase

```
git rebase [-i | --interactive] [<options>] [--exec <cmd>]
	[--onto <newbase> | --keep-base] [<upstream>] [<branch>]
git rebase [-i | --interactive] [<options>] [--exec <cmd>] [--onto <newbase>]
	--root [<branch>]
git rebase (--continue | --skip | --abort | --edit-todo )
```

参考链接：https://www.cnblogs.com/596014054-yangdongsheng/p/10445057.html

**rebase用来做什么？**

rebase和merge有相同的功能——整合来自不同分支的修改。

文档解释：forward-port local commits to the updated upstream head

即git rebase 用来将本地提交更新到最新的上游HEAD指针处

**rebase和merge的区别？**

merge会产生历史的分叉，而rebase可以使其成为一条直线

**rebase如何将历史改变成一条直线？**

原理是首先找这两个分支的最近共同祖先 ，然后对比当前分支相对于该祖先的历次提交，提取相应的修改并存为临时文件，然后将当前分支指向目标基底 , 最后以此将之前另存为临时文件的修改依序应用。

**官方文档的解释**

**-i|--interactive**	弹出交互式界面

**options**	 选项，一般是一个区间

**--exec <cmd>**	

**--onto<newbase>|--keep-base**	重定向到一个新的基底，或保持当前基底

**< upstream>**	若upstream没有被明确指定则会使用branch.<name>.remote和branch.<name>.merge选项。若当前不处于任何分支或当前分支没有上游，则rebase会被取消。所有在当前分支中提交但不在<upstream>中的更改都保存到一个临时区域。

**< branch>**	若branch明确制定了，则git rebase会先调用git checkout <branch>，否则会停留在当前分支

**--root [<branch>]**	

**--continue**	解决冲突后继续rebase

**--skip**	跳过冲突

**--abort**	取消rebase，移除.git/rebase-apply工作文件

**--edit-todo**	在交互式rebase时编辑todo事宜

### git branch

无参数时，会列出本地的分支

**< branchName>**

加上参数后，会创建一个新的分支

**-d < branchName>**

删除指定分支

### git checkout (branchName)

切换分支。当切换分支的时候，Git 会用该分支的最后提交的快照替换你的工作目录的内容， 所以多个分支不需要多个目录。

**-b <branchName>**

创建并切换到一条新的分支

### git merge

合并分支。主要分以下几种情况

![fast-forward](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/fast-forward1.jpg)

👆由于当前 master 分支所指向的提交是你当前提交（dev的提交）的直接上游，所以 Git 只是简单的将 master 指针向前移动。 Git 在合并两者的时候，只会简单的将指针向前推进（指针右移），因为这种情况下的合并操作没有需要解决的分歧——这就叫做 快进（fast-forward）

![fast-forward](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/no-fast.jpg)

👆在 master 分支和 dev 分支的公共祖先 B2 后，master 和 dev 的提交是对不同文件或者同一文件的不同部分进行了修改，Git 也可以直接合并它们

![fast-forward](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/conflict.jpg)

👆master和dev修改了相同文件并导致了冲突。任何因包含合并冲突而有待解决的文件，都会以未合并状态标识出来。 Git 会在有冲突的文件中加入标准的冲突解决标记，这样你可以打开这些包含冲突的文件然后手动解决冲突。 

## Git提交历史

### git log

查看git提交历史，主要内容为commit（提交编号）、author（作者）、Date（时间）

**--oneline**

查看简介版本

**--graph**

开启拓扑图选项

**--reverse**

逆向显示所有日志

**--author=<name>**

根据名字查看日志



## Git标签

### git tag

看查所有标签

**-a <tagName> <commitCode>**

为指定的提交打上标签，若commitCode省略，则默认为上一次提交打上标签

**-m <comment>**

指定标签信息

**-s <tagName>**

PGB签名标签



## Git远程仓库

### git remote

查看所有远程仓库

**-v**

看到每个别名的实际链接地址

### git remote add [shortname] [url]

添加一个新的远程仓库，指定一个简单得名字，如origin，以便将来引用。

若将github作为远程仓库，则需要生成ssh key并添加到对应github账号,命令如下：

**ssh-keygen -t rsa -C "youremail@example.com"**

生成的.ssh文件夹子在windows下位于C:\Users\Administrator\.ssh，在linux下位于~/.ssh，找到id_rsa,用文本编辑器打开，复制所有内容。在github的Settins->SSH and PGP keys中添加新的SSH密钥。

添加后，使用ssh -T git@github.com验证

### git push -u [shortname] [branch]

将当前分支的仓库提交到shortname对应的远程仓库地址的指定分支上，如：

git push -u origin master

**-u**

使用-u，使得创建一个config，将本地仓库分支对应到远程仓库分支，这样之后的push和pull操作就会知道应该从远程的哪个分支执行，就只需要输入git push和git pull即可

### **git fetch**

从远程仓库下载新分支与数据。git fetch是将远程主机的最新内容拉到本地，用户在检查了以后决定是否合并到工作本机分支中。

**[alias**]

指定远程仓库别名

### **git merge [alias] [branch]**

从指定远程仓库的指定分支合并到当前分支

### git push [alias] [branch]

将[branch] 分支推送成为 [alias] 远程仓库上的 [branch] 分支

### git remote rm [alias]

删除远程仓库

### git pull [alias] [branch]

git pull 则是将远程主机的最新内容拉下来后直接合并，即：git pull = git fetch + git merge，这样可能会产生冲突，需要手动解决。



## 总结

只是一个比较简单的整理，更多复杂的操作可以到git --help里查看，以后遇到了也会做补充。
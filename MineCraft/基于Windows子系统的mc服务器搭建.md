# 1.前言

之前也在阿里云、华为云等云服务器搭过minecraft的服务器，只不过云服务器的性能实在是不咋地（主要是没钱提高配置），因此这次选择用换下来的旧笔记本来发挥余热，打算安装windows子系统作为mc的服务器。需要的items主要如下：

```
1.一台windows主机，我的配置是16G内存+几百G的磁盘空间
2.natapp内网穿透工具
3.一点小钱
```

本片主要内容从下一节展开

# 2.安装Linux子系统

## 2.1 开启开发者模式

开始菜单点击设置

![image-20210318155032933](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210318155032933.png)

点击最后一个：更新和安全，找到开发者选项，选中开发人员模式

![image-20210318155130836](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210318155130836.png)

这一步骤完成后电脑将会开启开发人员模式的相关配置，需要根据电脑的性能等待若干时间，别着急



## 2.2 启用Linux子系统

还是老地方，找到应用

![image-20210318155303023](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210318155303023.png)

选择程序和功能：

![image-20210318155329176](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210318155329176.png)

点击启用或关闭windows功能

![image-20210318155359929](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210318155359929.png)

这里有可能会显示英文，因此有几率需要考验你是否拥有扎实的初中英语水平，打个勾点确定即可：

![image-20210318155459322](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210318155459322.png)



## 2.3 windows商店下载系统

找到microsoft store，搜索linux，出现下图，如果没有，请检查之前的配置是否完成。这里我选择的是unbuntu

![image-20210318155727920](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210318155727920.png)

点击安装，安装后启动，启动以后会弹出一个黑色的框框，会花一点时间进行安装操作。

安装完毕后，会提示创建用户，这一步就自己搞一下好了。



# 3.SSH&XShell

如果你是一个没什么编程基础的小白，那么就跳过这一步吧~

```
#卸载ssssh server
sudo apt-get remove openssh-server

#安装ssh server
sudo apt-get install openssh-server

#修改配置文件
sudo vim /etc/ssh/sshd_config
#修改内容如下
#默认是22，windows有自己的ssh服务，所以这里要改一下
Port 222 
UsePrivilegeSeparation no
PasswordAuthentication yes
# 这里改成你登陆WSL用的账号，比如我是Kyle
AllowUsers  Kyle

#启动ssh
sudo service ssh --full-restart
```

至此，就可以通过xshell连接子系统了。



# 4.安装jdk

由于minecraft是需要运行在java环境下的，因此得安装个java，本次安装就拿比较稳定的jdk8

```
sudo apt-get update
sudo apt-get install openjdk-8-jdk
#检查java版本
java -version
```



# 5.下载服务端程序

这里丢个链接，直接下载即可

```
wget https://launcher.mojang.com/v1/objects/1b557e7b033b583cd9f66746b7a9ab1ec1673ced/server.jar
```

## 5.1 第一次启动

使用下面的命令第一次启动,第一次会启动失败，在提示启动失败后ctrl+c关闭。

```
java -Xmx1024M -Xms1024M -jar server.jar nogui
```



## 5.2 编辑eula.txt

```
vi eula.txt
#修改false->true
eula=true
```



## 5.3 编辑server.properties

```
vi server.properties
#修改online-mnode=false
online-mode=false
```



## 5.4 第二次启动

```
java -Xmx1024M -Xms1024M -jar server.jar nogui
```

这次启动后，就能在本地访问服务器了，但是如果想让服务器迎接更多的小伙伴，那么请看下一步



# 6.natapp内网穿透

## 6.1 首先

内网穿透其实也不难，首先，先从

```
natapp.cn
```

这个地址上注册一个账号，然后下载一个window客户端



## 6.2  其次

花点小钱（我买的是9元1月）买个隧道，配置如下：

![image-20210318163815573](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210318163815573.png)

协议指定tcp，远程端口随便选一个即可，尽量在1024-65535，本地端口这边就必须写25565拉



## 6.3 最后

购买完成后，隧道会有一个authtoken提供，此时打开natapp的windows客户端，输入：

```
natapp -authtoken=xxxxxxxx(这里填写你的authtoken)
```



如果出现以下的提示，那么恭喜，配置就顺利完成啦！

![image-20210318163327954](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20210318163327954.png)


# Dokcer安装Jenkins

## 一、启动Docker

```
systemctl start docker
```



## 二、pull jenkins最新版本镜像

下载镜像

```
docker pull jenkins/jenkins:lts
```

查看镜像

```
docker images
```

通过上面的操作获取image_id,查看镜像的详细信息

```
docker inspect ${image_id}
```



## 三、创建jenkins目录

创建文件夹

```
mkdir /opt/jenkins/
```

设置归属用户id

```
chown -R 1000:1000 /opt/jenkins/
```



## 四、启动jenkins

最后的jenkins:lts指的是jenkins版本，可能会不一样，可以在docker images中查看

```
docker run -d -p 8888:8080 -p 50005:50000 --name jenkins2 -e JENKINS_OPTS="--prefix=/jenkins" -e JENKINS_ARGS="--prefix=/jenkins" -e JAVA_OPTS=-Duser.timezone=Asia/Shanghai --add-host=git.wenlvcloud.com:61.174.54.88 --add-host=repo.ctbiyi.com:140.249.193.206 --privileged=true -v /home/docker_data/jenkins2:/var/jenkins_home jenkins/jenkins:2.164.3 
```

容器启动后会有一个id，之后会用到，比如在这里我的id是

```
1a2a267133a7eb239379e87d57e9732fbf494534cb9f8bd4570d676d2199d819
```



## 五、查看服务启动情况

```
docker ps | grep jenkins
```

发现jenkins已经启动了



## 六、访问Jenkins并解锁

jenkins启动后，来访问一下他的默认端口8080，如果是本地运行的，则直接在浏览器输出localhost:8080,若是远程主机，则是http://${ip_addr}:8080,看一下访问界面，发现要解锁，则回到命令行，输入以下指令进入容器交互界面，这里就要用到之前的id了

```
docker exec -it 1a2a267133a7eb239379e87d57e9732fbf494534cb9f8bd4570d676d2199d819 /bin/bash
```

进入交互界面后，在输入

```
cat /var/jenkins_home/secrets/initialAdminPassword
//得到密码
fbd8af5c3*******b5a33a199e8152fd
```

将密码输入到浏览器的输入框中就能登陆进去了~另外，退出容器使用**Ctrl+D**或者**输入exit**即可



## 七、插件安装

在浏览器登录进入Jenkins后，首先会让你安装插件，由于我也不知道有啥插件，因此就选了安装推荐的插件，点进去之后发现出现了错误：

```
No such plugin: cloudbees-folder
```

查了一下发现是缺少了插件，因此定位到Jenkins的插件文件夹中，

```
/opt/jenkins/war/WEB-INF/detached-plugins
```

并下载对应插件

```
wget http://ftp.icm.edu.pl/packages/jenkins/plugins/cloudbees-folder/latest/cloudbees-folder.hpi
```

然后重启容器

```
docker restart 1a2a267133a7eb239379e87d57e9732fbf494534cb9f8bd4570d676d2199d819
```

再刷新浏览器

![img](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/jenkinsPage.jpg)

发现直接进入主界面了，之前的推荐插件安装直接被跳过了，看来只能手动装了。(另外我在另一台服务器上并不会出现这个问题，所以祝你好运吧)



## 八、时间校准

点击系统管理，选择执行脚本命令：

打开 【系统管理】->【脚本命令行】运行下面的命令

```
System.setProperty('org.apache.commons.jelly.tags.fmt.timeZone', 'Asia/Shanghai')
```



## 九、其他

还可以使用dockerfile和docker-compose的方法通过构建docker镜像来安装jenkins，这里就不过多介绍了，贴个链接吧

```
https://www.cnblogs.com/stulzq/p/8627360.html
```


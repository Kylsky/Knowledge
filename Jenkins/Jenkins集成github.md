# Jenkins集成github

## 一、生成ssh

```
ssh-keygen -t rsa -b 4096 -C "752051085@qq.com"
cd ~/.ssh
#把显示出来的公钥保存到github上
cat id_rsa.pub
```



## 二、设置带权限的access_token

在github上的Settings->Developers settings->Personal access tokens->Create new token

选择repo和admin:repo_hook后生成，这个access只能看到一次，所以要保存好

```
a44f22cf13********4d874f8472c 
```

进入系统设置创建对应凭据

![img](http://kylescloud.top/site/pic/Jenkins1.png)

找到Github Server选项，添加一个凭据，添加之后记得勾选管理hook

![img](http://kylescloud.top/site/pic/Jenkins2.png)

将生成的access_token复制到Secret一栏，注意红框里的内容，描述可以自己写

![image-20200415152425737](http://kylescloud.top/site/pic/Jenkins3.png)



## 三、安装插件

去这里安装git、github、maven插件，安装完成可能需要重启一下jenkins,安装插件会要好一会儿，先去睡一觉吧~

![image-20200415152425737](http://kylescloud.top/site/pic/Jenkins4.png)



## 四、配置环境

进入系统管理->全局工具配置，对于jdk、git、maven等环境都可以选择自动安装，安装完成后在容器中通过where is命令查找路径后配置，如where is git

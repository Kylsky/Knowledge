# es安装head插件

head插件是一个es的可视化管理工具，在安装7.6.2版本的es对应head插件，发现要先安装node.js

## 一、安装nodejs

```
1. curl --silent --location https://rpm.nodesource.com/setup_10.x | sudo bash -

2. yum -y install nodejs
```



## 二、下载并安装head插件

由于插件位于github，因此**预装一下git**

```
yum install -y git
```

**下载插件**

```
git clone git://github.com/mobz/elasticsearch-head.git
```

**安装**

```
npm config set registry=http://registry.npm.taobao.org/
cd elasticsearch_head
npm install
```



## 三、启动插件并访问

```
npm run start
#http://localhost:9100
```

由于浏览器的跨域问题，需要在head界面上将localhost修改为指定ip，或者在elasticsearch_head配置文件中的app.js修改第4374行的localhost为指定ip
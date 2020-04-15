# Docker安装gitlab

jenkins通常搭配gitlab进行持续集成，因此来看一下Docker如何安装gitlab

## 一、拉取镜像

```undefined
docker pull gitlab/gitlab-ce
```



## 二、创建容器

这里需要注意第三行的几个端口，比如--publish 10080:80，表示的是将容器中的80端口映射到主机的10080端口，前者表示主机端口，后者表示容器端口，为了防止端口冲突，这里把主机的代理端口改成自定义的。

```
sudo docker run --detach \
--hostname gitlab.bill.com \
--publish 4443:443 --publish 10080:80 --publish 10022:22 \
--name gitlab \
--restart always \
--volume ~/gitlab/config:/etc/gitlab \
--volume ~/gitlab/logs:/var/log/gitlab \
--volume ~/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce:latest
```

容器创建并启动后，会返回一个容器id

```
4a4bff953c9a4d81b87e636d363cfceb239e3000b0b365fa8803b105c41df2db
```



## 三、开放端口

如果实在云服务器创建的容器，则需要开放一下端口，本地可以跳过这一步



## 四、访问页面

```
ip:10080
```

进入后会先让root用户修改密码，之后可以用root+密码进行登录



## 五、配置gitlab邮件

要能充分使用gitlab, 必须配置邮件发送功能, 修改配置文件 gitlab.rb (启动镜像后产生的文件), 这里配置QQ邮箱，可以在自己的qq邮箱的设置->账号中操作

```
vim /home/software/gitlab/etc/gitlab.rb
```

在文件末尾加上下面的内容,相关内容填写自己的即可

```
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "example@qq.com"
gitlab_rails['smtp_password'] = "授权码"
gitlab_rails['smtp_domain'] = "smtp.qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
gitlab_rails['gitlab_email_from'] = 'example@qq.com'
```



## 六、在容器中更新配置

```
docker exec -it gitlab /bin/bash
```

在容器中执行

```
gitlab-ctl reconfigure 
```

测试邮件

```
gitlab-rails console
```

在出现的命令行中输入

```
Notify.test_email('752051085@qq.com', 'Message Subject', 'Message Body').deliver_now     # 发送测试邮件
```

如果没有出现问题的话就能在邮箱收到邮件啦~



## 七、配置项目路径

如果映射的主机端口是80，那么则不用修改，否则修改以下文件

```
vim /home/software/gitlab/data/gitlab-rails/etc/gitlab.yml
```

到文件开头可以找到production->gitlab->port，改成主机对应的端口即可,这里对应改成10080

之后进入gitlab容器使用命令重启即可(注意不要使用reconfigure命令)

```
docker exec -it gitlab /bin/bash
gitlab-ctl restart
```



## 八、总结

以上就是docker搭建gitlab的常规操作，另外抱怨一下gitlab占用的内存实在是太多了，启动以后4个g的华为云1半的内存都给吃了，实在是难顶……

之后会详细讲一下gitlab与jenkins整合。不知道服务器顶不顶得住我之后一波又一波的操作。
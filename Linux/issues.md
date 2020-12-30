## 1.Linux ssh无法访问原因

### 问题：

```
/var/empty/sshd must be owned by root and not group or world-writable
```



### 解决方法：

```shell
chmod 744 /var/empty/sshd
```


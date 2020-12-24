## 1.RBD失败策略

### 异常信息：

```
MISCONF Redis is configured to save RDB snapshots, but it is currently not able to persist on disk. Commands that may modify the data set are disabled, because this instance is configured to report errors during writes if RDB snapshotting fails (stop-writes-on-bgsave-error option). Please check the Redis logs for details about the RDB error.
```



### 解决方法：

```
修改配置文件中stop-writes-on-bgsave-error为no

or

执行：
./redis-cli 
127.0.0.1:6379> config set stop-writes-on-bgsave-error no
```




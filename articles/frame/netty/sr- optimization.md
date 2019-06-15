## Netty高并发性能调优
### 单机百万连接调优
- 如何模拟百万连接
- 突破局部文件句柄限制
```
ulimit -n
vi /etc/security/limits.conf
# 添加如下
$ hard nofile 1000000
$ soft nofile 1000000
```
- 突破全局文件句柄限制
```
cat proc/sys/fs/file-max

echo 20000 > proc/sys/fs/file-max

vi /etc/systs.conf
# 添加如下
fs.file-max=1000000
systl -p /etc/systl.conf
```

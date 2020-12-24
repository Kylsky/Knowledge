# NFS（Network File System）

NFS主要是采用远程过程调用RPC机制实现文件传输



yum install -t nfs-utils rpcbind



![image-20201216090625472](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201216090625472.png)



服务端：

mkdir -p /usr/local/data/www-data

vi /etc/exports

/usr/local/data/www-data 192.168.10.10/24(rw sync)

systemctl start nfs.service

systemctl start rpcbind.service

systemctl enable nfs.service

systemctl enable rpcbind.service

exportfs

![image-20201216093431657](http://kyle-pic.oss-cn-hangzhou.aliyuncs.com/img/image-20201216093431657.png)



客户端：

yum install -y nfs-utils

showmount -e 192.168.10.10

mkdir -p /usr/local/data/www-data

mount 192.168.10.10:/usr/local/data/www-data /mnt

cd /mnt 

ll

tips:文件并不真正存在与客户端，而是通过远程rpc映射到了客户端上


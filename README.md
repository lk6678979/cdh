# cdh6.2安装
## 1. 下载安装文件
* ClouderaManager下载地址(全部下载）
https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPMS/x86_64/  
![](https://github.com/lk6678979/image/blob/master/cdh/down-1.png)  
* CDH6.2.0安装包地址：https://archive.cloudera.com/cdh6/6.2.0/parcels/  
由于我们的操作系统为CentOS7，需要下载以下文件：
![](https://github.com/lk6678979/image/blob/master/cdh/down-2.png)  
## 2. 关闭防火墙和iptables
```shell
systemctl stop firewalld.service
systemctl stop iptables.service
systemctl disable firewalld.service
systemctl disable iptables.service
```
## 3. 关闭SELinux

```shell
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```
## 4. 所有服务器修改hostname
1. sudo vi /etc/sysconfig/network修改主机名：
```shell
HOSTNAME=node1.cdh.com
```
2. vi /etc/hostname修改为：node1.cdh.com
## 5. 修改host
vi /etc/host,添加：
```shell
192.168.10.41 node1.cdh.com
192.168.10.42 node2.cdh.com
192.168.10.43 node3.cdh.com
```
快速添加指令：
```shell
cat << EOF >> /etc/hosts 
192.168.10.41 node1.cdh.com
192.168.10.42 node2.cdh.com
192.168.10.43 node3.cdh.com
EOF
```
## 6. 配置SSH免密码登陆,需要输入一次密码
* 所有服务器，ssh-copy不用copy自己这台服务器，只用copy到其他2台
```shell
cd /root/.ssh
ssh-keygen -t rsa
ssh-copy-id -i root@node1.cdh.com
ssh-copy-id -i root@node2.cdh.com
ssh-copy-id -i root@node3.cdh.com
```

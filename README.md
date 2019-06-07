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
## 7. 安装JDK1.8

建议使用 **/usr/java/jdk1.8** 作为 **JAVA_HOME**,因为YARN等组件默认使用这个目录为 **JAVA_HOME**，直接配置到这里可以避免很多麻烦。
假设jdk的tar已经拷贝到服务器的 **/usr/java** 目录下 ：

```
tar zxvf jdk-8u152-linux-x64.tar.gz
mkdir jdk1.8
mv jdk1.8.0_152/* jdk1.8/
```
* 配置环境变量：
```
cat << EOF >> /etc/profile
export JAVA_HOME=/usr/java/jdk1.8
export PATH=\$JAVA_HOME/bin:\$PATH
export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar
EOF
source /etc/profile
echo "JAVA_HOME=/usr/java/jdk1.8" >> /etc/environment
```
## 8. 安装和配置mysql数据库

* 首先删除自带的MariaDB：

```
yum erase -y mariadb mariadb-libs
```

* 安装Mysql，因为依赖关系，这里必须按照这个顺序安装(自行下载安装rpm文件)：

```
rpm -ivh mysql-community-common-5.7.20-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.20-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.20-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.20-1.el7.x86_64.rpm
```

* 设置Mysql：

```
systemctl enable mysqld.service
service mysqld start
grep 'temporary password' /var/log/mysqld.log
```

* 执行完毕之后会有类似如下显示（临时生成的密码会有区别）：

```
2017-12-17T11:26:18.937718Z 1 [Note] A temporary password is generated 
for root@localhost: LgEu(D(<Y9Q?
```

* 根据上面查找到的密码登录mysql

```
mysql -uroot -p
```

* 以下是mysql命令行：

修改密码，密码为：taima@123ABC，必须包含大小写字母、数字和符号
```
alter user root@localhost identified by 'Sziov@2019';
#授权用户root使用密码passwd从任意主机连接到mysql服务器
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'Sziov@2019' WITH GRANT OPTION;
flush privileges;
```
为ActiveMonitor和Hive创建数据库：
```
create database amon DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database oozie DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
quit;
```

## 9. 创建/usr/share/java目录，将mysql-jdbc包放过去（所有节点）,mysql包自行下载
```shell
mkdir -p /usr/share/java
mv /opt/mysql-j/mysql-connector-java-5.1.44.jar /usr/share/java/
#mysql-connector-java-5.1.34.jar 一定要命名为mysql-connector-java.jar
mv /usr/share/java/mysql-connector-java-5.1.44.jar /usr/share/java/mysql-connector-java.jar 
```

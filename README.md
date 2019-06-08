# cdh6.2安装
## 1. 下载安装文件
* ClouderaManager下载地址(全部下载）
https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPMS/x86_64/  
* 全部文件下载地址：
```
https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPMS/x86_64/cloudera-manager-agent-6.2.0-968826.el7.x86_64.rpm  
https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPMS/x86_64/cloudera-manager-daemons-6.2.0-968826.el7.x86_64.rpm  
https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPMS/x86_64/cloudera-manager-server-6.2.0-968826.el7.x86_64.rpm  
https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPMS/x86_64/cloudera-manager-server-db-2-6.2.0-968826.el7.x86_64.rpm  
https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPMS/x86_64/enterprise-debuginfo-6.2.0-968826.el7.x86_64.rpm  
https://archive.cloudera.com/cm6/6.2.0/redhat7/yum/RPMS/x86_64/oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm  
https://archive.cloudera.com/cm6/6.2.0/allkeys.asc
```
![](https://github.com/lk6678979/image/blob/master/cdh/down-1.png)  
* CDH6.2.0安装包地址：https://archive.cloudera.com/cdh6/6.2.0/parcels/  
由于我们的操作系统为CentOS7，需要下载以下文件：
* 全部文件下载地址：
```
https://archive.cloudera.com/cdh6/6.2.0/parcels/CDH-6.2.0-1.cdh6.2.0.p0.967373-el7.parcel  
https://archive.cloudera.com/cdh6/6.2.0/parcels/CDH-6.2.0-1.cdh6.2.0.p0.967373-el7.parcel.sha1  
https://archive.cloudera.com/cdh6/6.2.0/parcels/CDH-6.2.0-1.cdh6.2.0.p0.967373-el7.parcel.sha256,注意把sha256后缀的文件名修改为sha  
https://archive.cloudera.com/cdh6/6.2.0/parcels/manifest.json  
```
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
## 7. 安装和配置mysql数据库

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

修改密码，密码为：Owp@2019，必须包含大小写字母、数字和符号
```
alter user root@localhost identified by 'Owp@2019';
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

## 8. 创建/usr/share/java目录，将mysql-jdbc包放过去
* 所有节点都需要执行,mysql包自行下载
```shell
mkdir -p /usr/share/java
mv /opt/mysql-j/mysql-connector-java-5.1.44.jar /usr/share/java/
#mysql-connector-java-5.1.34.jar 一定要命名为mysql-connector-java.jar
mv /usr/share/java/mysql-connector-java-5.1.44.jar /usr/share/java/mysql-connector-java.jar 
```
## 9. 安装httpd和createrepo
```shell
yum -y install httpd createrepo
yum install httpd
service httpd start
#设置httpd服务开机自启
systemctl enable httpd.service
```
## 10 设置swap(所有节点）
```
echo vm.swappiness = 0 >> /etc/sysctl.conf
sysctl -w vm.swappiness=0
```
## 11. 禁用透明页(所有节点）
```
cat << EOF >> /etc/rc.d/rc.local
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
EOF
chmod +x /etc/rc.d/rc.local
```
## 12. 将下载的cm和cdh文件上传到linux目录，例如如下2个目录
```shell
#把 CDH 6.2 的三个文件放到/root/cdh 6.2中，并且注意把sha256后缀的文件名修改为sha
mkdir -p /root/cdh6.2
# 把 Cloudera Manager 6.2 的7个文件放到/root/cm6.2中
mkdir -p /root/cm6.2
```
## 13. 配置本地repo源 
### 13.1 配置Cloudera Manager包yum源（manager节点）
* 将Cloudera Manager安装需要的5个rpm包以及一个asc文件上传到linux服务器(步骤13创建的目录/root/cm6.2)，执行createrepo命令生成rpm元数据。
```shell
#进入目录
cd /root/cm6.2/
#创建repodata： 
yum install createrepo
#（注意此命令的最后带一个点） 最终 cm6.2目录下多了一个repodata目录
createrepo . 
``` 
### 13.2 配置Web服务器
* 将 cdh6.2目录 和 cm6.1目录 移动到/var/www/html目录下, 使得用户可以通过HTTP访问这些rpm包。
```shell
cd /root
mkdir -p /var/www/html
mv ./cdh6.2/ ./cm6.2/ /var/www/html
``` 
### 14.3 使得用户可以通过HTTP访问这些/var/www/html目录下的文件
* 192.168.10.41是本机地址
```shell
cat << EOF >> /etc/yum.repos.d/os.repo
[osrepo]
name=os_repo
baseurl=http://192.168.10.41/cm6.2
enabled=true
gpgcheck=false
EOF
sudo yum repolist
```
* 修改 /etc/httpd/conf/httpd.conf 配置文件，在`<IfModule mime_module>`中修改以下内容,把`AddType application/x-gzip .gz .tgz` 修改为 `AddType application/x-gzip .gz .tgz .parcel`
```shell
vim /etc/httpd/conf/httpd.conf	
AddType application/x-gzip .gz .tgz .parcel
```
* 重启httpd服务 
```shell
systemctl restart httpd
```
* 访问我们配置好的cdh资源  
http://192.168.10.41/cdh6.2/  
http://192.168.10.41/cm6.2/  
![](https://github.com/lk6678979/image/blob/master/cdh/cdh6.2.jpg)  
![](https://github.com/lk6678979/image/blob/master/cdh/cm6.2.jpg) 
* 制作Cloudera Manager的repo源（192.168.10.41是本机地址）  
```shell
cat << EOF >> /etc/yum.repos.d/cm.repo
[cmrepo]
name = cm_repo
baseurl = http://192.168.10.41/cm6.2
enable = true
gpgcheck = false
EOF
sudo yum repolist
```
* 重启httpd服务  
```shell
systemctl restart httpd
```

## 14. 安装JDK
* 所有节点，请装官方提供的JDK：oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm，默认装在/usr/java/jdk1.8.0_181-cloudera目录
### 14.1 第一种方式：
* 因为已经配置好repo仓库所以yum时会到192.168.88.100/cm6.2目录下找到oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm进行安装
```shell
yum -y install oracle-j2sdk1.8-1.8.0+update181-1.x86_64
```
### 14.2 第二种方式
* 直接使用 rpm -ivh 命令安装 rpm 文件的方式
```
cd /var/www/html/cm6.2
rpm -qa | grep java # 查询已安装的java
yum remove java* # 卸载
rpm -ivh oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
```
### 14.3 配置环境变量：
```
cat << EOF >> /etc/profile
export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
export PATH=\$JAVA_HOME/bin:\$PATH
export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar
EOF
source /etc/profile
echo "JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera" >> /etc/environment
```
### 5 初始化数据库
```
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql -uroot -pOwp@2019 cm cm
```
然后手动输出SCM账号的密码Owp@2019
最后会显示
```
All done, your SCM database is configured correctly!
```
* 说明：必须保证/usr/share/java目录中已经存在 mysql-connector-java.jar文件或者软连接：ln -s mysql-connector-java-5.1.46.jar mysql-connector-java.jar

# cdh6.2安装
## 1. 关闭防火墙和iptables(所有节点）
```shell
systemctl stop firewalld.service
systemctl disable firewalld.service
```
## 2. 关闭SELinux(所有节点）
```shell
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```
## 3. 设置swap(所有节点）
```
echo vm.swappiness = 0 >> /etc/sysctl.conf
sysctl -w vm.swappiness=0
```
## 4. 禁用透明页(所有节点）
```
cat << EOF >> /etc/rc.d/rc.local
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled
EOF
chmod +x /etc/rc.d/rc.local
```
## 5. 所有服务器修改hostname
1. sudo vi /etc/sysconfig/network修改主机名：
```shell
HOSTNAME=node1.cdh.com
```
2. vi /etc/hostname修改为：node1.cdh.com
## 6. 修改host
vi /etc/host,添加：
```shell
192.168.10.41 node1.cdh.com
192.168.10.42 node2.cdh.com
192.168.10.43 node3.cdh.com
```
* 快速添加指令：
```shell
cat << EOF >> /etc/hosts 
192.168.10.41 node1.cdh.com
192.168.10.42 node2.cdh.com
192.168.10.43 node3.cdh.com
EOF
```
## 7. 配置SSH免密码登陆,需要输入一次密码
* 所有服务器，ssh-copy不用copy自己这台服务器，只用copy到其他2台
```shell
cd /root/.ssh
ssh-keygen -t rsa
ssh-copy-id -i root@node1.cdh.com
ssh-copy-id -i root@node2.cdh.com
ssh-copy-id -i root@node3.cdh.com
```
### 8. 在每个服务器创建hdfs用户，并赋予root权限
```
adduser hdfs
vim /etc/sudoers 
#修改 /etc/sudoers 文件，找到下面一行，把前面的注释（#）去掉
%wheel    ALL=(ALL)    ALL
#然后修改用户，使其属于root组（wheel），命令如下：
usermod -g root hdfs
#注意，CHD会去检测hdfs用户是否属于自己的组（hdfs）和hadoop，所有装好cdh后
usermod -G root,hadoop,hdfs hdfs
```
# ---------记得一定要重启服务器reboot---------
## 9. 下载安装文件
### 9.1 找一台可以上网的电脑，根据官网地址下载依赖包，配置 repos服务
https://www.cloudera.com/documentation/enterprise/upgrade/topics/cm_ig_create_local_package_repo.html  
### 9.2 安装httpd和createrepo
```shell
yum -y install httpd createrepo
yum install httpd
service httpd start
#设置httpd服务开机自启
systemctl enable httpd.service
```
* 修改 /etc/httpd/conf/httpd.conf 配置文件，在`<IfModule mime_module>`中修改以下内容,把`AddType application/x-gzip .gz .tgz` 修改为 `AddType application/x-gzip .gz .tgz .parcel`
```shell
vim /etc/httpd/conf/httpd.conf	
AddType application/x-gzip .gz .tgz .parcel
```
### 9.3 下载依赖包
### 9.3.1 Cloudera Manager 6  
```
sudo mkdir -p /var/www/html/cloudera-repos
sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/cm6/6.2.0/redhat7/ -P /var/www/html/cloudera-repos
sudo wget https://archive.cloudera.com/cm6/6.2.0/allkeys.asc -P /var/www/html/cloudera-repos/cm6/6.2.0/
```
```
sudo chmod -R ugo+rX /var/www/html/cloudera-repos/cm6
```
### 9.3.2 CDH 6
```
sudo mkdir -p /var/www/html/cloudera-repos
sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/cdh6/6.2.0/redhat7/ -P /var/www/html/cloudera-repos
```
```
sudo wget --recursive --no-parent --no-host-directories https://archive.cloudera.com/gplextras6/6.2.0/redhat7/ -P /var/www/html/cloudera-repos
```
```
sudo chmod -R ugo+rX /var/www/html/cloudera-repos/cdh6
sudo chmod -R ugo+rX /var/www/html/cloudera-repos/gplextras6
```
### 9.4 在1.3下载的安装包放到集群服务器中任意一台中，目录/var/www/html/cloudera-repos  
并执行指令：
```
sudo chmod -R ugo+rX /var/www/html/cloudera-repos/cm6
sudo chmod -R ugo+rX /var/www/html/cloudera-repos/cdh6
sudo chmod -R ugo+rX /var/www/html/cloudera-repos/gplextras6
```
### 9.5 创建临时内部存储库
* 可以快速创建临时远程存储库以一次性部署程序包。Cloudera建议使用运行Cloudera Manager的相同主机或网关主机。
* 此示例使用Python SimpleHTTPServer作为Web服务器来托管在/var/www/html等目录，但您可以使用其他目录。
* 确定系统未侦听的端口。此示例使用端口8900。
* 在/var/www/html目录中启动Python SimpleHTTPServer：
```shell
cd /var/www/html 
python -m SimpleHTTPServer 8900
```
```
在0.0.0.0端口8900上提供HTTP服务...
```
* 访问存储库URL,http://<web_server>:8900/cloudera-repos/，在浏览器中验证您下载的文件是否存在

### 9.5 每台机器上都配置repo源
#### 9.5.1 删除CenteOS 自带的源，因为你无法访问外网，不删除会报错
```
rm -rf /etc/yum.repos.d/Centos*
```
#### 9.5.2 创建 /etc/yum.repos.d/cloudera-repo.repo(里面的IP对应的是你的rpm包的服务器ip)
```
cat << EOF >> /etc/yum.repos.d/cloudera-repo.repo
[cloudera-repo]
name=cloudera-repo
baseurl=http://192.168.10.41:8900/cloudera-repos/cm6/6.2.0/redhat7/yum/
enabled=1
gpgcheck=0 
EOF
```
#### 9.5.3 添加一个CDH组件的包源
```
cat << EOF >> /etc/yum.repos.d/cloudera-repo-cdh.repo
[cloudera-repo-cdh]
name=cloudera-repo-cdh
baseurl=http://192.168.10.41:8900/cloudera-repos/cdh6/6.2.0/redhat7/yum/
enabled=1
gpgcheck=0
EOF
```
* 重启httpd
```
systemctl restart httpd
````
#### 9.5.4 清理 repos 源
```
yum clean all
```
### 10. 安装JDK(所有节点)
```
yum install oracle-j2sdk1.8
```
* 配置环境变量
```
cat << EOF >> /etc/profile
export JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
export PATH=\$JAVA_HOME/bin:\$PATH
export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar
EOF
source /etc/profile
```
### 11. 安装Cloudera Manager Server(仅在manage节点安装)
```
yum install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server
```
## 12. 安装和配置mysql数据库
### 12.1 首先删除自带的MariaDB：
```
yum erase -y mariadb mariadb-libs
```
### 12.2 安装Mysql，因为依赖关系，这里必须按照这个顺序安装(自行下载安装rpm文件)：
```
yum install libaio
yum install libnuma*
rpm -ivh mysql-community-common-5.7.20-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.20-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.20-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.20-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-compat-5.7.20-1.el7.x86_64.rpm
```
### 12.3 设置Mysql：
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
### 12.4 根据上面查找到的密码登录mysql
```
mysql -uroot -p
```
* 以下是mysql命令行：
修改密码，密码为：Owp@2019，必须包含大小写字母、数字和符号
```
alter user root@localhost identified by 'Owp@2019';
#授权用户root使用密码passwd从任意主机连接到mysql服务器
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'Owp@2019' WITH GRANT OPTION;
flush privileges;
```
### 12.5 新建cloudera manager 需要的用户和库
```
# 为 Cloudera Software 创建数据库
CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'Owp@2019';
GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'Owp@2019';
GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'Owp@2019';
GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'Owp@2019';
GRANT ALL ON metastore.* TO 'metastore'@'%' IDENTIFIED BY 'Owp@2019';
GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY 'Owp@2019';
GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY 'Owp@2019';
GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY 'Owp@2019';
GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'Owp@2019';
flush privileges;
quit;
```
## 13. 创建/usr/share/java目录，将mysql-jdbc包放过去
* 所有节点都需要执行,mysql包自行下载
```shell
mkdir -p /usr/share/java
cp /opt/mysql-j/mysql-connector-java-5.1.44.jar /usr/share/java/
#mysql-connector-java-5.1.34.jar 一定要命名为mysql-connector-java.jar
mv /usr/share/java/mysql-connector-java-5.1.44.jar /usr/share/java/mysql-connector-java.jar 
```

### 14 初始化数据库
* 运行下面的脚本，告诉cloudera 你建立的数据库用户密码等信息
```
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql scm scm 'Owp@2019'
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql amon amon 'Owp@2019'
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql rman rman 'Owp@2019'
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql hue hue 'Owp@2019'
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql metastore metastore 'Owp@2019'
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql sentry sentry 'Owp@2019'
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql nav nav 'Owp@2019'
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql navms navms 'Owp@2019'
/opt/cloudera/cm/schema/scm_prepare_database.sh mysql oozie oozie 'Owp@2019'
```
```
All done, your SCM database is configured correctly!
```
* 说明：必须保证/usr/share/java目录中已经存在 mysql-connector-java.jar文件或者软连接：ln -s mysql-connector-java-5.1.46.jar mysql-connector-java.jar

### 14.3 启动Cloudera Manager Server
```shell
systemctl start cloudera-scm-server
ps -ef | grep cloudera-scm-server 查看是否启动
systemctl status cloudera-scm-server 显示 Active: active (running) 
```	
* 启动需要一点时间，可以通过日志，查看进度
```
tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log
```
### 14.4 检查端口是否监听
```shell
yum install net-tools 安装 netstat
netstat -lnpt | grep 7180 要等一段时间启动完全启动成功后，才能看到端口被使用，然后才能真正访问到CM的登录网页
#显示 tcp 0  0 0.0.0.0:7180  0.0.0.0:*  LISTEN  68289/java
```
## 15 CDH安装
### 15.1 通过 192.168.10.41:7180/cmf/login 访问 CM(admin/admin)
### 15.2 登陆界面
![](https://github.com/lk6678979/image/blob/master/cdh/login.jpg) 
### 15.2 欢迎界面（开始安装）
![](https://github.com/lk6678979/image/blob/master/cdh/welcome.jpg)  
### 15.3 选择版本（我们选择免费版）
![](https://github.com/lk6678979/image/blob/master/cdh/choose.jpg)   
### 15.4 安装首页
![](https://github.com/lk6678979/image/blob/master/cdh/install-free-1.jpg) 
### 15.5 选择本地存储库，也就是我们之前已创建好的库
![](https://github.com/lk6678979/image/blob/master/cdh/local-1.png) 
![](https://github.com/lk6678979/image/blob/master/cdh/local-1.png) 
### 15.6 JDK选项
![](https://github.com/lk6678979/image/blob/master/cdh/jdk.jpg) 
### 15.7 提供SSH登陆凭证（我们这里使用root密码）
![](https://github.com/lk6678979/image/blob/master/cdh/root.jpg)
### 15.8 安装Agent(时间较长)
![](https://github.com/lk6678979/image/blob/master/cdh/install-agent-1.png)
### 15.9 执行检查
![](https://github.com/lk6678979/image/blob/master/cdh/check.png)
### 15.10 检查完成
![](https://github.com/lk6678979/image/blob/master/cdh/check-complete.png)
### 15.11 选择需要安装的组件
![](https://github.com/lk6678979/image/blob/master/cdh/choose-install.png)
![](https://github.com/lk6678979/image/blob/master/cdh/choose-install-2.png)
### 15.12 组件安装配置
![](https://github.com/lk6678979/image/blob/master/cdh/config-1.png)
![](https://github.com/lk6678979/image/blob/master/cdh/config-2.png)
### 15.1.13 配置hive数据库（我们在上面已创建数据库metastore，账号：metastore，密码：Owp@2019)
![](https://github.com/lk6678979/image/blob/master/cdh/config-databse-hive.png)
### 15.1.14 检查其他配置
![](https://github.com/lk6678979/image/blob/master/cdh/conifg-all.png)
### 15.1.1 开始执行
![](https://github.com/lk6678979/image/blob/master/cdh/start-install.png)
### 15.16 组件安装完成
![](https://github.com/lk6678979/image/blob/master/cdh/install-ok.png)
### 15.17 HDFS安装HA模式
![](https://github.com/lk6678979/image/blob/master/cdh/hdfs-ha.png)

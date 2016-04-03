# **CM5.6.0 安装流程**
## X. 离线CM-CDH Parcel包安装
参考: http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_c.html
## Y. 在线自动安装
参考: http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_a.html

### 系统要求 <br>
**OS版本:** Centos 6.5 X 64 <br>
**CM版本**: 5.6.0 <br>
CM下载地址: <br>
http://www.cloudera.com/documentation/enterprise/latest/topics/cm_vd.html#cmvd_topic_1<br>
CM parcel下载地址:<br>
https://archive.cloudera.com/cm5/cm/5/cloudera-manager-el6-cm5.6.0_x86_64.tar.gz<br>

**CDH下载地址**: 5.6.0 <br>
download site: https://archive.cloudera.com/cdh5/parcels/5.6.0/ <br>
- CDH-5.6.0-1.cdh5.6.0.p0.45-el5.parcel.sha1
- CDH-5.6.0-1.cdh5.6.0.p0.45-el5.parcel
- manifest.jason

### 安装准备 <br>
**1). 产品需求 ** <br>
**存储** <br>
*CM服务器主机*<br>
/var 文件夹下至少5G<br>
/usr 文件夹下至少500M

对于parcels安装, 所需空间由安装的parcel包的数量决定，但只有一个parcel会被下载到CM服务器。CM服务器上本地parcel库的中的parcel包大小：<br>
- CDH 4.6 - 700 MB/parcel; <br>
- CDH 5 (包括Impala和Search) - 1.5 GB/parcel (压缩后), 2 GB/parcel (解压后) <br>
- Cloudera Impala - 200 MB/parcel <br>
- Cloudera Search - 400 MB/parcel <br>

*CM服务* <br>
主机监控和服务监控数据库储存位置在/var， 确保至少20G空间。

*Agents* <br>
在Agent主机上加压后的parcel所需空间为下载到CM服务器上的parcel包的3倍。默认parcel的目录为/opt/cloudera/parcels。

**RAM**<br>
一般为4 GB。（在使用Oracle数据库时必须）

**Python**<br>
CDH 4需要Python 2.4以上, CDH 5需要Python 2.6或2.7.

**Perl** <br>
CM需要perl.

**2). 网络和安全**<br>
***/etc/hosts文件***<br>
 - 包含集群所有主机的主机名和IP地址
 - **主机名不允许出现大写字母**
 - 不包含重复主机名
例如：

| ip | hostname |
| :------------- | :------------- |
| 127.0.0.1       | localhost       |
| 192.168.0.11 | server001 |
| 192.168.0.12 | server002 |
| 192.168.0.13 | server003 |
| 192.168.0.14 | server004 |
| 192.168.0.15 | server005 |

对于Centos系统，设置每个节点的/etc/sysconfig/network文件，修改主机名file。
``` shell
vi /etc/sysconfig/network   # modify host name for each host
# NETWORKING=yes
# HOSTNAME=server003
# NETWORKING_IPV6=no  ##---- Note: IPV6 must be disabled
# GATEWAY=192.168.0.1

reboot    #make this change work

vi /etc/hosts # combine IP addresses with hostnames in CM manager server
# 127.0.0.1 localhost
# 192.168.0.12 server002
# 192.168.0.13 server003

scp /etc/hosts server002:/etc #copy this host file to other cluster nodes
```

***设置节点之间的免密码登陆*** <br>
``` shell
ssh-keygen # generate public password key
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
## repeat previous 2 steps in each node

ssh-copy-id -i server003 # copy this public key to CM server
chmod 600 ~/.ssh/authorized_keys #authorize the key file
scp ~/.ssh/authorized_keys server003:~/.ssh/ #copy this key file to other nodes
service sshd restart #restart ssh service to make this change work
```
***关闭防火墙和SELinux服务防止意外错误***<br>
注：7180端口必须打开，因为CM默认通过此端口进行集群访问 <br>
```shell
service iptables stop #turn off the
setenforce 0 #disable the SELinux
vi /etc/sysconfig/selinux
# SELINUX=disabled ##permanantly turn off the selinux, work after reboot
```
**可能出现的错误: ssh connect Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password) or No supported authentication methods available (server sent public key)***<br>

**解决方案**<br>
- 确认 **~/.ssh** 目录和其内部文件的权限。  (**home directory ~**, **~/.ssh directory** 和 **~/.ssh/authorized_keys** 文件必须是用户 **可写** 状态: rwx------ 和 rwxr-xr-x均可, 不推荐rwxrwx---。（**700 or 755**, not 775).)
```shell
chmod 755 ~ ~/.ssh/
```
- **~/.ssh/authorized_keys** 文件必须是可读（至少400），由于还需要添加其他节点的公共密钥，因此需要可写 (**600**)。
```shell
chmod 600 ~/.ssh/*
```
- 确认SELinux是否关闭
- 确认 home 目录的所述用户 （CM服务器上必须为root：root）
```shell
chown -R root:root /root/.ssh/
```
- 修改/etc/ssh/sshd_config，将 **PubkeyAuthentication** 状态设置为 yes
```shell
vi /etc/ssh/sshd_config
# uncomment PubkeyAuthentication yes
```

**3). 同步所有节点的时间** <br>
**方法:** 先同步CM安装节点与对时节点; 再将其他节点与CM节点同步<br>
**工具:** ntp

```shell
yum install ntp
chkconfig ntpd on #make ntp run on boot
chkconfig --list ntpd #check if ntp works correctly
# ntpd            0:关闭  1:关闭  2:启用  3:启用  4:启用  5:启用  6:关闭

## configure the ntp server in CM server
ntpdate -u 110.75.186.248 #just in case the time lag is too big
vi /etc/ntp.conf
# driftfile /var/lib/ntp/drift
# restrict 127.0.0.1
# restrict -6 ::1
# restrict default nomodify notrap
# server 110.75.186.248 prefer
# includefile /etc/ntp/crypto/pw
# keys /etc/ntp/keys

service ntpd start #start ntpd service

# configure the ntp server in other cluster nodes
ssh server002
vi /etc/ntp.conf
# driftfile /var/lib/ntp/drift
# restrict 127.0.0.1
# restrict -6 ::1
# restrict default kod nomodify notrap nopeer noquery
# restrict -6 default kod nomodify notrap nopeer noquery
## the CM server ip
# server server003
# includefile /etc/ntp/crypto/pw
# keys /etc/ntp/keys

ntpdate -u server003
service ntpd start
```
**4). CM服务器上安装并配置MYSQL数据库服务**
```shell
yum install mysql-server
chkconfig mysqld on #start sql on boot
service mysqld start
mysqladmin -u root password 'feelwx'

mysql -uroot -pfeelwx #initial a mysql database
```

```mysql
create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database amon DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
create database oozie;
# give privileges to root user -- user name is root and password is feelwx.
grant all privileges on *.* to 'root'@'server003' identified by 'feelwx' with grant option;
flush privileges;
```
所有MYSQL可提供的数据库类型：

| Role | Database     | User | Password |
| :------------- | :-------------: | :-------------: | :------------- |
| Activity Monitor     | amon       | root | feelwx |
| Reports Manager | rman | root | feelwx |
| Hive Metastore Server | metastore | root | feelwx |
| Sentry Server | 	sentry | root | feelwx |
| Cloudera Navigator Audit Server | nav | root | feelwx |
| Cloudera Navigator Metadata Server | navms | root | feelwx |

**可能的报错: 在CM配置网页的集群数据库设置时: Logon denied for user/password. Able to find the database server and database, but logon request was rejected** <br>
解决方法: <br>
这可能是由于数据库名所属的用户名不是root，可以通过删除 hive, amon, hue, oozie数据库然后重新安装解决。 <br>
```mysql
drop database hive;
drop database oozie;
drop database amon;
drop database hue;
```

***配置 mysql option 文件 (官方配置推荐):*** http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_mysql.html#cmig_topic_5_5_2_unique_1
```shell
sudo service mysqld stop #incase the mysql is running
## Move old InnoDB log files /var/lib/mysql/ib_logfile0 and /var/lib/mysql/ib_logfile1 out of /var/lib/mysql/ to a backup location.
sudo mkdir /var/lib/mysql/ib_logfile_backup
mv /var/lib/mysql/ib_logfile0 /var/lib/mysql/ib_logfile1 /var/lib/mysql/

## update mysql my.calcNormFactors
vi /etc/my.cnf
--------------------------------------------------------------
[mysqld]
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
# symbolic-links = 0

key_buffer = 16M
key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1

max_connections = 550
#expire_logs_days = 10
#max_binlog_size = 100M

#log_bin should be on a disk with enough free space. Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your system
#and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log

# For MySQL version 5.1.8 or later. Comment out binlog_format for older versions.
binlog_format = mixed

read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M

# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

sql_mode=STRICT_ALL_TABLES
-----------------------------------------------------
```
***将mysql设置为开机启动.*** <br>
以下的例子中，root用户的密码为空。
```shell
sudo /sbin/chkconfig mysqld on
sudo /sbin/chkconfig --list mysqld
## mysqld          0:off   1:off   2:on    3:on    4:on    5:on    6:off

sudo service mysqld start #start mysql

#Set the MySQL root password
sudo /usr/bin/mysql_secure_installation
[...]
Enter current password for root (enter for none):
OK, successfully used password, moving on...
[...]
Set root password? [Y/n] y
New password:
Re-enter new password:
Remove anonymous users? [Y/n] Y
[...]
Disallow root login remotely? [Y/n] N
[...]
Remove test database and access to it [Y/n] Y
[...]
Reload privilege tables now? [Y/n] Y
All done!
```
***安装 MySQL JDBC 驱动*** <br>
将JDBC 驱动安装到需要一下服务的节点：Cloudera Manager Server，Activity Monitor, Reports Manager, Hive Metastore Server, Sentry Server, Cloudera Navigator Audit Server, and Cloudera Navigator Metadata Server roles.

### cloudera manager 5.6.0的安装
#### X 方案
***创建用户账户*** <br>
当使用tarballs安装Cloudera Manager时，必须在各节点手动创建用户账户。 默认CM服务器和其服务使用的账户名为：cloudera-scm。
```shell
sudo useradd --system --home=/opt/cm-5.6.0/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
```
***创建Cloudera Manager Server本地数据储存目录*** <br>
新建文件夹/var/lib/cloudera-scm-server，设置其所属用户为cloudera-scm。
```shell
sudo mkdir /var/log/cloudera-scm-server
sudo chown cloudera-scm:cloudera-scm /var/log/cloudera-scm-server
```
***配置Cloudera Manager Agents***<br>
在各个Cloudera Manager Agent节点上设置Cloudera Manager Agent将其指向Cloudera Manager Server所在的主机。配置信息位于 **/opt/cm-5.6.0/etc/cloudera-scm-agent/config.ini**:

```shell
vi /opt/cm-5.6.0/etc/cloudera-scm-agent/config.ini
# server_host=server003
# server_port=7182 #(default)
```
***配置 Custom Cloudera Manager 用户和 Custom 目录*** <br>
Cloudera Manager会在 **/var/log** 和 **/var/lib** 下默认创建以下文件夹: <br>
- /var/log/cloudera-scm-headlamp
- /var/log/cloudera-scm-firehose
- /var/log/cloudera-scm-alertpublisher
- /var/log/cloudera-scm-eventserver
- /var/lib/cloudera-scm-headlamp
- /var/lib/cloudera-scm-firehose
- /var/lib/cloudera-scm-alertpublisher
- /var/lib/cloudera-scm-eventserver
- /var/lib/cloudera-scm-server


 如果这些文件夹已经存在，Cloudera Manager installer对其不做任何改变。 **如果这些文件不属于cloudera-scm用户，Cloudera Manager将无法在这些目录下读写文件**, 因此为了保证CM服务能正常运行，确认这些目录为cloudera-scm所有。
```shell
 sudo chown -R cloudera-scm:cloudera-scm /var/log/cloudera-scm-* /var/lib/cloudera-scm-*
```

***创建parcel目录*** <br>
在Cloudera Manager Server主机上:
```shell
sudo mkdir -p /opt/cloudera/parcel-repo
sudo chown cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo
```
在其他各个节点:
```shell
sudo mkdir -p /opt/cloudera/parcels
sudo chown cloudera-scm:cloudera-scm /opt/cloudera/parcels
```

***安装Cloudera Manager Server and Agents***
解压复制CM tarballs到目标安装目录.
```shell
sudo tar -xzf cloudera-manager*.tar.gz -C /opt #cm-5.6.0 directory
```

***初始化 CM 数据库***
```shell
cp mysql-connector-java-5.1.38/mysql-connector-java-5.1.38 /opt/cm-5.6.0/share/cmf/lib/
/opt/cm-5.6.0/share/cmf/schema/scm_prepare_database.sh mysql cm -hlocalhost -uroot -pfeelwx --scm-host localhost scm scm scm
# 格式是:scm_prepare_database.sh 数据库类型 数据库 服务器 用户名 密码 –scm-host Cloudera_Manager_Server所在的机器
```
***下载 CDH parels***
```shell
cp CDH-5.1.3-1.cdh5.1.3.p0.12-el6.parcel CDH-5.1.3-1.cdh5.1.3.p0.12-el6.parcel.sha1 manifest.json /opt/cloudera/parcel-repo/
mv CDH-5.1.3-1.cdh5.1.3.p0.12-el6.parcel.sha1 CDH-5.1.3-1.cdh5.1.3.p0.12-el6.parcel.sha # or CM will download CDH parcels again
```
配置 **/opt/cm-5.6.0/etc/cloudera-scm-agent/config.ini** == > 设置主机名（#server_host=server003） <br>
复制改配置文件到集群各节点: <br>
```shell
scp -r /opt/cm-5.6.0 server002:/opt/
/opt/cm-5.6.0/etc/init.d/cloudera-scm-server start
/opt/cm-5.6.0/etc/init.d/cloudera-scm-agent start
```

***开始运行Cloudera Manager Server*** <br>
**注意: 当你开始运行Cloudera Manager Server 和 Agents时, Cloudera Manager 假定HDFS 和 MapReduce没有运行. 如果这些服务已经运行，请先关闭。**

设置CM server开机启动 （root用户）: <br>
```shell
sudo /opt/cm-5.6.0/etc/init.d/cloudera-scm-server start
```

**可能的报错: cloudera-scm-server is dead and pid file exists** <br>
检查日志文件: **/var/log/cloudera-scm-server/cloudera-scm-server.log** <br>
原因:
- SELinux 或防火墙被开启==> 关闭 SELinux 和防火墙(同前)
- postgresql没有正常启动 ==> 重启 postgresql
解决方法:
```shell
rm /var/run/cloudera-scm-server.pid
service cloudera-scm-server-db stop
/etc/rc.d/init.d/postgresql restart
service cloudera-scm-server-db start
service cloudera-scm-server start
```

***设置Cloudera Manager Server开机自动运行:***
```shell
cp /opt/cm-5.6.0/etc/init.d/cloudera-scm-server /etc/init.d/cloudera-scm-server #this is the dir containing service running with boot
chkconfig cloudera-scm-server on
```

***开始运行Cloudera Manager Agents***
```shell
sudo tarball_root/etc/init.d/cloudera-scm-agent start
```
***设置Cloudera Manager Agents开机自动运行:***
```shell
cp /opt/cm-5.6.0/etc/init.d/cloudera-scm-agent /etc/init.d/cloudera-scm-agent
chkconfig cloudera-scm-agent on
```


#### Y 方案
使用自动安装，CM主机需满足以下要求:
- 以root用户权限进入Cloudera Manager Server 主机或与其他节点可以免密码登陆的节点。
- 允许Cloudera Manager Server主机拥有通过相同端口登陆各节点的SSH权限。
- 所有节点必须连接standard package repositories 和  archive.cloudera.com 或 同一个本地包含所有安装文件的repository
**注意: 若日后需要迁移CM主机，不太推荐此方法.**

***下载CM installer* <br>***
```shell
wget https://archive.cloudera.com/cm5/installer/latest/cloudera-manager-installer.bin
chmod u+x cloudera-manager-installer.bin
sudo ./cloudera-manager-installer.bin
```
***登陆CM admin界面* <br>***
进入 **http://Server host:7180** (192.168.0.13:7180 或首先修改 **C:/Windows/System32/drivers/hosts**， 将 192.168.0.13 server003 信息输入后用server003:7180地址登陆)。

默认用户名和密码均为admin。 <br>
**Cloudera Manager 不支持修改 admin 用户名，但可以改密码**。可以通过新建用户然后将管理员权限赋予此用户后删除admin用户达到改用户名的效果。

***使用Cloudera Manager Wizard自动安装和配置*** <br>
- 选择CM安装版本
- 通过主机名或IP地址搜索需要安装的主机
- 通过SSH连接各主机安装Cloudera Manager Agent等
- 安装CDH和其他服务
- 配置CDH和其他服务
(基本采用默认配置)

选择CM server主机

选择安装运行CDH的主机

搜索安装主机名的案例：

| Range definition | matching hosts     |
| :------------- | :------------- |
| 10.1.1.[1-4]       | 10.1.1.1, 10.1.1.2, 10.1.1.3, 10.1.1.4       |
| host[1-3].company.com | host1.company.com, host2.company.com, host3.company.com |

***安装的软件和方法*** <br>
1. 选择repository的方式。
  - 使用Parcels
  - 指定安装的parcel所在的Parcel 目录或Local Parcel Repository路径。

  - 使用Cloudera Manager安装软件

2. 在各节点上安装Oracle Java SE Development Kit (JDK)。

**注意: 若其中一个节点处于bad health状态 (red alert), 检查/var/log/cloudera-scm-agent/cloudera-scm-agent.log (heartbeat log file)**
此处遇到的错误信息为：Failed to receive heartbeat from agent, 由/etc/hosts配置不正确所导致的。

解决方案:<br>
- 运行 ```uname -a``` 检查主机名 (```s python -c 'import socket; print  socket.getfqdn(),socket.gethostbyname(socket.getfqdn())' ``` 可以获得主机名和IP地址)
- 运行 ```/sbin/ifconfig```注意eth0的地址
- 运行 ``` host -v -t A `hostname` ``` 检查是否与之前输出的主机名和IP地址一致。

参考: http://www.cloudera.com/documentation/cdh/5-0-x/CDH5-Installation-Guide/cdh5ig_networknames_configure.html

### 添加服务
选择所有服务： **HDFS, YARN (includes MapReduce 2), ZooKeeper, Oozie, Hive, Hue, HBase, Impala, Solr, Spark, and Key-Value Store Indexer**

- **The Flume service can be added only after your cluster has been set up.**


### 配置服务器设置
1. 选择数据库类型
- 使用默认 **内置数据库 (postgresql)** （注意保存自动生成的密码）
- 选择外部数据库。输入数据库类型，名字，用户和密码。
2. 点击测试连接确认CM是否能与数据库正常通信。

**可能的报错: log4j:ERROR setFile(null,false) call failed.**
原因:<br>
检查/var/log/cloudera-scm-agent/cloudera-scm-agent.log: /var/log/cloudera-scm-eventserver/mgmt-cmf-mgmt-EVENTSERVER-server003.log.out不存在.

解决方案:<br>
检查 cloudera-scm-eventserver 目录所属的用户是否属于cloudera-scm
```shell
sudo chown cloudera-scm:cloudera-scm /var/log//var/log/cloudera-scm* /var/lib//var/log/cloudera-scm*
```

***修改默认密码*** <br>
点击右上方导航栏用户名选择 Change Password. <br>
'admin' to 'feelwx'

***配置MySQL for Oozie*** <br>
创建Oozie 数据库和 Oozie MySQL 用户

```mysql
mysql -u root -p
Enter password: feelwx

mysql> create database oozie;
Query OK, 1 row affected (0.03 sec)

mysql>  grant all privileges on oozie.* to 'oozie'@'localhost' identified by 'oozie';
Query OK, 0 rows affected (0.03 sec)

mysql>  grant all privileges on oozie.* to 'oozie'@'%' identified by 'oozie';
Query OK, 0 rows affected (0.03 sec)

mysql> exit
Bye
```
***添加MySQL JDBC驱动 到 Oozie*** <br>
目标目录/opt/cloudera/parcels/CDH/lib/oozie/lib/ <br>

### 测试安装
检查主机 Heartbeats: By default, every Agent must heartbeat successfully every 15 seconds. 记录文件： **/var/log/cloudera-scm-agent/cloudera-scm-agent.log**

运行MapReduce任务: 若采用Parcel安装，运行```sudo -u hdfs hadoop jar /opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar pi 10 100 ```<br>
运行结果可以在CM页面点击查看： Clusters > ClusterName > yarn Applications

**注意: 如果你在重装CM之前没有停止老版本CM的运行, /var/lib/alternatives 文件将同时出现新旧两个条目，使 /etc/alternatives/ 下的服务链接指向旧版本的CDHparcel目录，导致服务无法正常运行。此处我们遇到的错误为：hadoop command is not found**

解决方法:
- 检查目录/var/lib/alternatives: ```cat /var/lib/alternatives/sqoop```
- 移除/var/lib/alternatives/sqoop-import 和 /var/lib/alternatives/sqoop: ```rm /var/lib/alternatives/sqoop-import /var/lib/alternatives/sqoop```
- 移除所有/var/lib/alternatives/下cloudera相关的文件: ```for x in `grep -l cloudera /var/lib/alternatives/*`; do  rm $x; done```
- 重启 cloudera-scm-agent: ```service cloudera-scm-agent restart```

***测试Hue*** <br>
Hue is a graphical user interface that allows you to interact with your clusters by running applications that let you browse HDFS, manage a Hive metastore, and run Hive, Impala, and Search queries, Pig scripts, and Oozie workflows.
- In the Cloudera Manager Admin Console Home > Status tab, click the Hue service.
- Click the Hue Web UI link, which opens Hue in a new window.
- Log in with the credentials, username: hdfs, password: hdfs.
- Choose an application in the navigation bar at the top of the browser window.

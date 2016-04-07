# **CM5.6.0 installation protocol**
## X. [Official path C: manual installation using cloudera tarballs](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_c.html)
## Y. [Official path A: automated installation](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_a.html)

---
### System requirement <br>
**OS:** Centos 6.5 X 64 <br>
**CM**: [5.6.0](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_vd.html#cmvd_topic_1)<br>
[tarball file download](https://archive.cloudera.com/cm5/cm/5/cloudera-manager-el6-cm5.6.0_x86_64.tar.gz)<br>

**CDH**: [5.6.0](https://archive.cloudera.com/cdh5/parcels/5.6.0/) <br>
- CDH-5.6.0-1.cdh5.6.0.p0.45-el5.parcel.sha1
- CDH-5.6.0-1.cdh5.6.0.p0.45-el5.parcel
- manifest.jason
---
### Preparation <br>
**1). Resource requirement** <br>
**Disk Space** <br>
*Cloudera Manager Server*<br>
- 5 GB on the partition hosting /var
- 500 MB on the partition hosting /usr

For parcels, the space required depends on the number of parcels you download to the Cloudera Manager Server and distribute to Agent hosts. You can download multiple parcels of the same product, of different versions and builds. If you are managing multiple clusters, only one parcel of a product/version/build/distribution is downloaded on the Cloudera Manager Server—not one per cluster. In the local parcel repository on the Cloudera Manager Server, the approximate sizes of the various parcels are as follows: <br>
- CDH 4.6 - 700 MB per parcel; <br>
- CDH 5 (which includes Impala and Search) - 1.5 GB per parcel (packed), 2 GB per parcel (unpacked) <br>
- Cloudera Impala - 200 MB per parcel <br>
- Cloudera Search - 400 MB per parcel <br>

*Cloudera Management Service* <br>
The Host Monitor and Service Monitor databases are stored on the partition hosting /var. Ensure that you have at least 20 GB available on this partition.

*Agents* <br>
On Agent hosts each unpacked parcel requires about three times the space of the downloaded parcel on the Cloudera Manager Server. By default unpacked parcels are located in /opt/cloudera/parcels.

**RAM**<br>
4 GB is recommended for most cases and is required when using Oracle databases.

**Python**<br>
Cloudera Manager and CDH 4 require Python 2.4 or later, but Hue in CDH 5 and package installs of CDH 5 require Python 2.6 or 2.7.

**Java**
**Note: remove preinstalled openJDK and reinstall JDK 1.7.X or upper version for CDH 5.6.0 installation**<br>
```shell
rpm -qa | grep "java|jdk"
#eg: rpm -e --nodeps java-1.5.0-gcj-1.5.0.0-29.1.el6.x86_64
rpm -ivh jdk-7u79-linux-x64.rpm
echo "JAVA_HOME=/usr/java/latest/" >> /etc/environment #add to global variable
```
**Perl** <br>
Cloudera Manager requires perl.

**2). Networking and security requirement**<br>
***Format /etc/hosts file***<br>
Cluster hosts must have a working network name resolution system and correctly formatted /etc/hosts file.

 /etc/hosts:
 - Contain consistent information about hostnames and IP addresses across all hosts
 - **Not contain uppercase hostnames**
 - Not contain duplicate IP addresses

| ip | hostname |
| :------------- | :------------- |
| 127.0.0.1       | localhost       |
| 192.168.0.11 | server001 |
| 192.168.0.12 | server002 |
| 192.168.0.13 | server003 |
| 192.168.0.14 | server004 |
| 192.168.0.15 | server005 |

For RHEL and CentOS, the /etc/sysconfig/network file on each host must contain the hostname you have just set (or verified) for that host.
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
***Set cluster password-less sudo permission (switch to other node without typing password)*** <br>
The Cloudera Manager Server must have SSH access to the cluster hosts when you run the installation or upgrade wizard. You must log in using a root account or an account that has **password-less sudo permission**. For authentication during the installation and upgrade procedures, you must either enter the password or upload a public and private key pair for the root or sudo user account. If you want to use a public and private key pair, the public key must be installed on the cluster hosts before you use Cloudera Manager.

``` shell
ssh-keygen # generate public password key
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
## repeat previous 2 steps in each node

ssh-copy-id -i server003 # copy this public key to CM server
chmod 600 ~/.ssh/authorized_keys #authorize the key file
scp ~/.ssh/authorized_keys server003:~/.ssh/ #copy this key file to other nodes
service sshd restart #restart ssh service to make this change work
```
***Turn off firewall to avoid communication errors (every node)***<br>
No blocking by iptables or firewalls; port 7180 must be open because it is used to access Cloudera Manager after installation. <br>
```shell
service iptables stop #turn off the
setenforce 0 #disable the SELinux
vi /etc/sysconfig/selinux
# SELINUX=disabled ##permanantly turn off the selinux, work after reboot
```
**[Potetnial Error:](http://ubuntuforums.org/showthread.php?t=1932058) ssh connect Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password) or No supported authentication methods available (server sent public key)***

**Solutions**<br>
- Make sure the permissions on the **~/.ssh** directory and its contents are proper (Your **home directory ~**, **your ~/.ssh directory** and the **~/.ssh/authorized_keys** file on the remote machine must be **writable** only by you: rwx------ and rwxr-xr-x are fine, but rwxrwx--- is no good¹, even if you are the only user in your group (if you prefer numeric modes: **700 or 755**, not 775).)
`chmod 755 ~ ~/.ssh/`

- Your **~/.ssh/authorized_keys** file (on the remote machine) must be readable (at least 400), but you'll need it to be also writable (**600**) if you will add any more keys to it; Your private key file (on the local machine) must be readable and writable only by you: rw-------, i.e. 600.
`chmod 600 ~/.ssh/*`

- Make sure SELinux is off
- Check the ownership on home directory on the server system.
`chown -R root:root /root/.ssh/`

- Check configuration of /etc/ssh/sshd_config and set **PubkeyAuthentication** to yes
```shell
vi /etc/ssh/sshd_config
# uncomment PubkeyAuthentication yes
```

**3). Synchronize time of all nodes** <br>
**Method:** synchronize CM server node with remote server; then synchronize other nodes with the CM server <br>
**Tool:** ntp

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
**4). Install and configure mysql-server on CM server**
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
All available databases

| Role | Database     | User | Password |
| :------------- | :-------------: | :-------------: | :------------- |
| Activity Monitor     | amon       | root | feelwx |
| Reports Manager | rman | root | feelwx |
| Hive Metastore Server | metastore | root | feelwx |
| Sentry Server | 	sentry | root | feelwx |
| Cloudera Navigator Audit Server | nav | root | feelwx |
| Cloudera Navigator Metadata Server | navms | root | feelwx |

**Potential Error: you may meet error during database setup: Logon denied for user/password. Able to find the database server and database, but logon request was rejected** <br>
Solutions: <br>
Try delete the database: hive, amon, hue, oozie and recreate them with previous code <br>
```mysql
drop database hive;
drop database oozie;
drop database amon;
drop database hue;
```

***Configure mysql option file ([official mysql configuration file](http://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_mysql.html#cmig_topic_5_5_2_unique_1)):***
```shell
sudo service mysqld stop #incase the mysql is running
## Move old InnoDB log files /var/lib/mysql/ib_logfile0 and /var/lib/mysql/ib_logfile1 out of /var/lib/mysql/ to a backup location.
sudo mkdir /var/lib/mysql/ib_logfile_backup
mv /var/lib/mysql/ib_logfile0 /var/lib/mysql/ib_logfile1 /var/lib/mysql/

## update mysql my.calcNormFactors
vi /etc/my.cnf
```
---
```
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
```
---

***start mysql on boot.*** <br>
 In the following example, the current root password is blank.
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
***Installing the MySQL JDBC Driver*** <br>
Install the JDBC driver on the Cloudera Manager Server host, as well as hosts to which you assign the Activity Monitor, Reports Manager, Hive Metastore Server, Sentry Server, Cloudera Navigator Audit Server, and Cloudera Navigator Metadata Server roles.

---
### Installation of CM
#### X strategy
***Create Users*** <br>
The Cloudera Manager Server and managed services require a user account to complete tasks. When installing Cloudera Manager from tarballs, you **must create this user account on all hosts manually**. Because Cloudera Manager Server and managed services are configured to use the user account cloudera-scm by default, creating a user with this name is the simplest approach.
`sudo useradd --system --home=/opt/cm-5.6.0/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm`

***Create the Cloudera Manager Server Local Data Storage Directory*** <br>
Create the following directory: /var/lib/cloudera-scm-server.<br> Change the owner of the directory so that the cloudera-scm user and group have ownership of the directory
```shell
sudo mkdir /var/log/cloudera-scm-server
sudo chown cloudera-scm:cloudera-scm /var/log/cloudera-scm-server
```
***Configure Cloudera Manager Agents***<br>
On every Cloudera Manager Agent host, configure the Cloudera Manager Agent to point to the Cloudera Manager Server by setting the following properties in the **/opt/cm-5.6.0/etc/cloudera-scm-agent/config.ini** configuration file:

By default, a tarball installation has a var subdirectory where state is stored. In a non-tarball installation, state is stored in **/var.** Cloudera recommends that you reconfigure the tarball installation to use an external directory as the /var equivalent (/var or any other directory outside the tarball) so that when you upgrade Cloudera Manager, the new tarball installation can access this state. Configure the installation to use an external directory for storing state by editing /opt/cm-5.6.0/etc/default/cloudera-scm-agent and setting the CMF_VAR variable to the location of the /var equivalent. If you do not reuse the state directory between different tarball installations, duplicate Cloudera Manager Agent entries can occur in the Cloudera Manager database.
```shell
vi /opt/cm-5.6.0/etc/cloudera-scm-agent/config.ini
# server_host=server003
# server_port=7182 #(default)
```
***Configuring for a Custom Cloudera Manager User and Custom Directories*** <br>
By default, Cloudera Manager creates the following directories in **/var/log** and **/var/lib**: <br>
- /var/log/cloudera-scm-headlamp
- /var/log/cloudera-scm-firehose
- /var/log/cloudera-scm-alertpublisher
- /var/log/cloudera-scm-eventserver
- /var/lib/cloudera-scm-headlamp
- /var/lib/cloudera-scm-firehose
- /var/lib/cloudera-scm-alertpublisher
- /var/lib/cloudera-scm-eventserver
- /var/lib/cloudera-scm-server


 Cloudera Manager installer makes no changes to any directories that already exist. **Cloudera Manager cannot write to any existing directories for which it does not have proper permissions**, and if you do not change ownership, Cloudera Management Service roles may not perform as expected.
`sudo chown -R cloudera-scm:cloudera-scm /var/log/cloudera-scm-* /var/lib/cloudera-scm-*`

**Alternative**: Use alternate directories: <br>
```shell
mkdir /var/cm_logs/cloudera-scm-headlamp
chown cloudera-scm /var/cm_logs/cloudera-scm-headlamp
```
In **CM console:** <br>
Select Clusters > Cloudera Management Service <br>
Select Scope > role name. <br>
Click the Configuration tab. <br>
Enter a term in the Search field to find the settings to be changed. For example, you might enter /var or directory. <br>
Update each value with the new locations for Cloudera Manager to use. <br>

***Create Parcel Directories*** <br>
On the Cloudera Manager Server host:
```shell
sudo mkdir -p /opt/cloudera/parcel-repo
sudo chown cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo
```
On each cluster host:
```shell
sudo mkdir -p /opt/cloudera/parcels
sudo chown cloudera-scm:cloudera-scm /opt/cloudera/parcels
```

***Install the Cloudera Manager Server and Agents***
Tarballs contain both the Cloudera Manager Server and Cloudera Manager Agent in a single file. Copy the tarballs and unpack them on all hosts on which you intend to install Cloudera Manager Server and Cloudera Manager Agents.
`sudo tar -xzf cloudera-manager*.tar.gz -C /opt #cm-5.6.0 directory`

***Install MYSQL database for CM***
Download JDBC driver and put mysql-connector-java-5.1.38-bin.jar to /opt/cm-5.6.0/share/cmf/lib/.

***Initialize CM database***
```shell
cp mysql-connector-java-5.1.38/mysql-connector-java-5.1.38 /opt/cm-5.6.0/share/cmf/lib/
/opt/cm-5.6.0/share/cmf/schema/scm_prepare_database.sh mysql cm -hlocalhost -uroot -pfeelwx --scm-host localhost scm scm scm
# 格式是:scm_prepare_database.sh 数据库类型 数据库 服务器 用户名 密码 –scm-host Cloudera_Manager_Server所在的机器
```
***Download CDH parels for installation***
```shell
cp CDH-5.1.3-1.cdh5.1.3.p0.12-el6.parcel CDH-5.1.3-1.cdh5.1.3.p0.12-el6.parcel.sha1 manifest.json /opt/cloudera/parcel-repo/
mv CDH-5.1.3-1.cdh5.1.3.p0.12-el6.parcel.sha1 CDH-5.1.3-1.cdh5.1.3.p0.12-el6.parcel.sha # or CM will download CDH parcels again
```
Configure **/opt/cm-5.6.0/etc/cloudera-scm-agent/config.ini** == > set #server_host=server003 (CM server hostname) <br>
Copy this configuration file to other nodes and then start server and agent service: <br>
```shell
scp -r /opt/cm-5.6.0 server002:/opt/
/opt/cm-5.6.0/etc/init.d/cloudera-scm-server start
/opt/cm-5.6.0/etc/init.d/cloudera-scm-agent start
```

***Start the Cloudera Manager Server*** <br>
**Note: When you start the Cloudera Manager Server and Agents, Cloudera Manager assumes you are not already running HDFS and MapReduce. If these services are running: Shut down HDFS and MapReduce.**

start CM as root: <br>
`sudo /opt/cm-5.6.0/etc/init.d/cloudera-scm-server start`

As another user: <br>
If you run as another user, ensure the user you created for Cloudera Manager owns the location to which you extracted the tarball including the newly created database files.
```shell
sudo chown -R cloudera-scm:cloudera-scm /opt/cloudera-manager
sudo -u cloudera-service tarball_root/etc/init.d/cloudera-scm-server start  # if you want to run CM as cloudera-service
```
Edit the configuration files so the script internally changes the user. <br>
- Remove the following line from **/opt/cm-5.6.0/etc/default/cloudera-scm-server**: `export CMF_SUDO_CMD=" "`
- Change the user and group in /opt/cm-5.6.0/etc/init.d/cloudera-scm-server to the user you want the server to run as.: <br>
```
# USER=cloudera-service
# GROUP=cloudera-service
sudo tarball_root/etc/init.d/cloudera-scm-server start # run as root
```

**[Potential Error:](https://community.cloudera.com/t5/Cloudera-Manager-Installation/cloudera-scm-server-is-dead-and-pid-file-exists/td-p/22701) cloudera-scm-server is dead and pid file exists** <br>

Check the log file: **/var/log/cloudera-scm-server/cloudera-scm-server.log** <br>
Reasons:
- SELinux or firewall is on ==> turn off SELinux and firewall (as previous)
- postgresql didn't work correctly ==> restart postgresql
Solutions:
```shell
rm /var/run/cloudera-scm-server.pid
service cloudera-scm-server-db stop
/etc/rc.d/init.d/postgresql restart
service cloudera-scm-server-db start
service cloudera-scm-server start
```

***To start the Cloudera Manager Server automatically after a reboot:***
```shell
cp /opt/cm-5.6.0/etc/init.d/cloudera-scm-server /etc/init.d/cloudera-scm-server #this is the dir containing service running with boot
chkconfig cloudera-scm-server on
```
On the Cloudera Manager Server host, open the **/etc/init.d/cloudera-scm-server** file and change the value of CMF_DEFAULTS from ${CMF_DEFAULTS:-/etc/default} to tarball_root/etc/default.

***Start the Cloudera Manager Agents***
`sudo tarball_root/etc/init.d/cloudera-scm-agent start`

***To start the Cloudera Manager Agents automatically after a reboot:***
```shell
cp /opt/cm-5.6.0/etc/init.d/cloudera-scm-agent /etc/init.d/cloudera-scm-agent
chkconfig cloudera-scm-agent on
```
---
#### Y strategy
To use this method, server, and cluster hosts must satisfy the following requirements:
- Provide the ability to log in to the Cloudera Manager Server host using a root account or an account that has **password-less sudo permission**.
- Allow the Cloudera Manager Server host to have **uniform SSH access** on the same port to all hosts. See Networking and Security Requirements for further information.
- All hosts must have access to standard package repositories and either archive.cloudera.com or a local repository with the required installation files
**Note: it is not recommended for production deployments because its not intended to scale and may require database migration as your cluster grows.**

***Download CM installer* <br>***
```shell
wget https://archive.cloudera.com/cm5/installer/latest/cloudera-manager-installer.bin
chmod u+x cloudera-manager-installer.bin
sudo ./cloudera-manager-installer.bin
```
***login in CM admin console* <br>***
In a web browser, enter **http://Server host:7180** ([login](http://192.168.0.13:7180) or first modify **C:/Windows/System32/drivers/hosts** with 192.168.0.13 server003 then login in as server003:7180), where Server host is the fully qualified domain name or IP address of the host where the Cloudera Manager Server is running.

Log into Cloudera Manager Admin Console. The default credentials are: Username: admin Password: admin. <br>
**Cloudera Manager does not support changing the admin username for the installed account**. You can change the password using Cloudera Manager after you run the installation wizard. Although you cannot change the admin username, you can add a new user, assign administrative privileges to the new user, and then delete the default admin account.

***Use the Cloudera Manager Wizard for Software Installation and Configuration*** <br>
- Select the edition of Cloudera Manager to install
- Find the cluster hosts you specify via hostname and IP address ranges
- Connect to each host with SSH to install the Cloudera Manager Agent and other components
- Optionally install the Oracle JDK on the cluster hosts.
- Install CDH and managed service packages or parcels
- Configure CDH and managed services automatically and start the services
(Almost all use the default settings)

Choose Cloudera Manager Hosts

Choose which hosts will run CDH and managed services:

To enable Cloudera Manager to automatically discover hosts on which to install CDH and managed services, enter the cluster hostnames or IP addresses. You can also specify hostname and IP address ranges.

| Range definition | matching hosts     |
| :------------- | :------------- |
| 10.1.1.[1-4]       | 10.1.1.1, 10.1.1.2, 10.1.1.3, 10.1.1.4       |
| host[1-3].company.com | host1.company.com, host2.company.com, host3.company.com |

***Choose Software Installation Method and Install Software*** <br>
1. Select the repository type to use for the installation. In the Choose Method section select one of the following:
  - Use Parcels: Choose the parcels to install. The choices you see depend on the repositories you have chosen – a repository may contain multiple parcels. Only the parcels for the latest supported service versions are configured by default.
  - To specify the Parcel Directory or Local Parcel Repository Path, add a parcel repository: Parcel Directory and Local Parcel Repository Path - Specify the location of parcels on cluster hosts and the Cloudera Manager Server host.
  - Click OK. Parcels available from the configured remote parcel repository URLs are displayed in the parcels list. If you specify a URL for a parcel version too new to be supported by the Cloudera Manager version, the parcel is not displayed in the parcel list.

  - If you are using Cloudera Manager to install software, select the release of Cloudera Manager Agent. You can choose either the version that matches the Cloudera Manager Server you are currently using or specify a version in a custom repository. If you opted to use custom repositories for installation files, you can provide a GPG key URL that applies for all repositories.

2. Select Install Oracle Java SE Development Kit (JDK) to allow Cloudera Manager to install the JDK on each cluster host. If you have already installed the JDK, do not select this option.

3. Specify host installation properties:
Select root or enter the username for an account that has password-less sudo permission. --> root & feelwx

4.  Cloudera Manager performs the following:
Parcels - installs the Oracle JDK and the Cloudera Manager Agent packages and starts the Agent. Click Continue. During parcel installation, progress is indicated for the phases of the parcel installation process in separate progress bars. If you are installing multiple parcels, you see progress bars for each parcel. When the Continue button at the bottom of the screen turns blue, the installation process is completed.

**[Potential Error:](http://www.cloudera.com/documentation/cdh/5-0-x/CDH5-Installation-Guide/cdh5ig_networknames_configure.html) if one node is in bad health (red alert), check the /var/log/cloudera-scm-agent/cloudera-scm-agent.log file for error information (heartbeat log file)**
Like here I suffered this error: Failed to receive heartbeat from agent, this is caused by improper assignment in **/etc/hosts** file.

Solutions:<br>
The canonical name of **each host in /etc/hosts must be the FQDN** (for example myhost-1.mynet.myco.com), not the unqualified hostname (for example myhost-1). The canonical name is the first entry after the IP address. <br>
If you are using DNS, storing this information in /etc/hosts is not required, but it is good practice. <br>
Make sure the **/etc/sysconfig/network** file on each system contains the hostname you have just set (or verified) for that system, for example myhost-1.<br>
Check that this system is consistently identified to the network:<br>
- Run `uname -a` and check that the hostname matches the output of the hostname command. (`python -c 'import socket; print  socket.getfqdn(),socket.gethostbyname(socket.getfqdn())' ` could provide both hostname and its IP address)
- Run `/sbin/ifconfig` and note the value of inet addr in the eth0 entry
- Run `host -v -t A 'hostname' ` and make sure that hostname matches the output of the hostname command, and has the same IP address as reported by ifconfig for eth0
---
### Add Services
In the first page of the Add Services wizard, choose the combination of services to install and whether to install Cloudera Navigator: <br>
All Services - **HDFS, YARN (includes MapReduce 2), ZooKeeper, Oozie, Hive, Hue, HBase, Impala, Solr, Spark, and Key-Value Store Indexer**
- Some services depend on other services; for example, HBase requires HDFS and ZooKeeper. Cloudera Manager tracks dependencies and installs the correct combination of services.
- In a Cloudera Manager deployment of a CDH 5 cluster, the YARN service is the default MapReduce computation framework. Choose Custom Services to install MapReduce, or use the Add Service functionality to add MapReduce after installation completes.
- **The Flume service can be added only after your cluster has been set up.**
- Customize the assignment of role instances to hosts. The wizard evaluates the hardware configurations of the hosts to determine the best hosts for each role. The wizard assigns all worker roles to the same set of hosts to which the HDFS DataNode role is assigned. You can reassign role instances if necessary. <br>
Click a field below a role to display a dialog containing a list of hosts. If you click a field containing multiple hosts, you can also select All Hosts to assign the role to all hosts, or Custom to display the pageable hosts dialog.
---
### Configure Database Settings
1. Choose the database type:
- Keep the default setting of Use **Embedded Database (postgresql)** to have Cloudera Manager create and configure required databases. Record the auto-generated passwords.
- Select Use Custom Databases to specify external database host, enter the database type, database name, username, and password for the database that you created when you set up the database. Here we use the **Mysql database** with username root and password feelwx
2. Click Test Connection to confirm that Cloudera Manager can communicate with the database using the information you have supplied.

Review Configuration Changes and Start Services<br>
Review the configuration changes to be applied. Confirm the settings entered for file system paths.  <br>
The wizard starts the services.<br>

**Potential Error: log4j:ERROR setFile(null,false) call failed.**
Reasons:<br>
Check the /var/log/cloudera-scm-agent/cloudera-scm-agent.log and get this informaiton: /var/log/cloudera-scm-eventserver/mgmt-cmf-mgmt-EVENTSERVER-server003.log.out not exist.

Solutions:<br>
1. check the user of cloudera-scm-eventserver and it should belong to cloudera-scm
2. assign the cloudera-scm* directories to user cloudera-scm<br>

  `sudo chown cloudera-scm:cloudera-scm /var/log//var/log/cloudera-scm* /var/lib//var/log/cloudera-scm*`

***Change the Default Administrator Password*** <br>
Click the logged-in username at the far right of the top navigation bar and select Change Password. <br>
**'admin' to 'feelwx'**

***Configuring MySQL for Oozie*** <br>
Create the Oozie Database and Oozie MySQL User

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
***Add the MySQL JDBC Driver JAR to Oozie*** <br>
Copy or symbolically link the MySQL JDBC driver JAR into following directories: <br>
For installations that use parcels: `/opt/cloudera/parcels/CDH/lib/oozie/lib/` <br>

---
### Test the Installation
Checking Host Heartbeats: By default, every Agent must heartbeat successfully every 15 seconds. A recent value for the Last Heartbeat means that the Server and Agents are communicating successfully. view file **/var/log/cloudera-scm-agent/cloudera-scm-agent.log**

Running a MapReduce Job: Parcel -- `sudo -u hdfs hadoop jar /opt/cloudera/parcels/CDH/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar pi 10 100 `<br>
Depending on whether your cluster is configured to run MapReduce jobs on the YARN or MapReduce service, view the results of running the job by selecting one of the following from the top navigation bar in the Cloudera Manager Admin Console: **Clusters > ClusterName > yarn Applications**

**[Potential Error:](https://community.cloudera.com/t5/Cloudera-Manager-Installation/Incorrect-symlinks-being-created-by-cloudera-scm-agent/td-p/18092) if you reinstall without deactivating the old cluster/parcel first, the /var/lib/alternatives files for cloudera have two entries in them, one for the old parcel and one for the new parcel, thus causing the service installs in /etc/alternatives/ point to previous CDH parcel directory. Here we meet the error that the hadoop command is not found**<br>

Reasons: <br>
For Cloudera-scm-agent runs a tool called /usr/lib64/cmf/service/common/alternatives.sh to generate /etc/alternatives and symlinks to /usr/bin/\*. This bash script executes update-alternatives based on the PARCELS_DIR and PARCEL_DIRNAME variables. There are files in /var/lib/alternatives/ which seem to be used as overrides for the update-alternatives tool. <br>

Solutions:
- Check the /var/lib/alternatives: `cat /var/lib/alternatives/sqoop`
- Remove the /var/lib/alternatives/sqoop-import file and /var/lib/alternatives/sqoop: `rm /var/lib/alternatives/sqoop-import /var/lib/alternatives/sqoop`
- Remove all cloudera related /var/lib/alternatives/ files after a botched install if you do not deactivate the parcel prior to reinstall: `for x in `grep -l cloudera /var/lib/alternatives/\*; do  rm $x; done`
- Restart cloudera-scm-agent the proper symlink is created: `service cloudera-scm-agent restart`

***Testing with Hue*** <br>
Hue is a graphical user interface that allows you to interact with your clusters by running applications that let you browse HDFS, manage a Hive metastore, and run Hive, Impala, and Search queries, Pig scripts, and Oozie workflows.
- In the Cloudera Manager Admin Console Home > Status tab, click the Hue service.
- Click the Hue Web UI link, which opens Hue in a new window.
- Log in with the credentials, username: hdfs, password: hdfs.
- Choose an application in the navigation bar at the top of the browser window.


***Install flume (all nodes)*** <br>
1. Import [cdh5 repo](http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/cloudera-cdh5.repo) to /etc/yum.repos.d
```shell
cd /etc/yum.repos.d
wget http://archive.cloudera.com/cdh5/redhat/6/x86_64/cdh/cloudera-cdh5.repo #(flume1.6.0)
```

2. Installation using packages
```shell
sudo yum install flume-ng
sudo yum install flume-ng-agent
sudo yum install flume-ng-doc
```

3. Configuration
```shell
cp /etc/alternatives/flume-ng-conf/flume-conf.properties.template /etc/alternatives/flume-ng-conf/flume.conf
cp /etc/alternatives/flume-ng-conf/flume-env.sh.template /etc/alternatives/flume-ng-conf/flume-env.sh
```

4. Verify installation
```flume-ng help
#should appear something like this:
Usage: /usr/bin/flume-ng <command> [options]...

commands:
  help                  display this help text
  agent                 run a Flume agent
  avro-client           run an avro Flume client
  version               show Flume version info

global options:
  --conf,-c <conf>      use configs in <conf> directory
  --classpath,-C <cp>   append to the classpath
  --dryrun,-d           do not actually start Flume, just print the command
  --Dproperty=value     sets a JDK system property value

agent options:
  --conf-file,-f <file> specify a config file (required)
  --name,-n <name>      the name of this agent (required)
  --help,-h             display help text

avro-client options:
  --rpcProps,-P <file>  RPC client properties file with server connection params
  --host,-H <host>      hostname to which events will be sent (required)
  --port,-p <port>      port of the avro source (required)
  --dirname <dir>       directory to stream to avro source
  --filename,-F <file>  text file to stream to avro source [default: std input]
  --headerFile,-R <file> headerFile containing headers as key/value pairs on each new line
  --help,-h             display help text

  Either --rpcProps or both --host and --port must be specified.

Note that if <conf> directory is specified, then it is always included first
in the classpath.
```

5. Run flume
```shell
sudo service flume-ng-agent start
or
/usr/bin/flume-ng agent -c <config-dir> -f <config-file> -n <agent-name>
# /usr/bin/flume-ng agent -c /etc/flume-ng/conf -f /etc/flume-ng/conf/flume.conf -n agent
```

6. Test
- Set up an example conf for flume:
  ```shell
  vi example.conf
  # example.conf: A single-node Flume configuration

  # Name the components on this agent
  a1.sources = r1
  a1.sinks = k1
  a1.channels = c1

  # Describe/configure the source
  a1.sources.r1.type = netcat
  a1.sources.r1.bind = localhost
  a1.sources.r1.port = 44444

  # Describe the sink
  a1.sinks.k1.type = logger

  #  Use a channel which buffers events in memory
  a1.channels.c1.type = memory
  a1.channels.c1.capacity = 1000
  a1.channels.c1.transactionCapacity = 100

  # Bind the source and sink to the channel
  a1.sources.r1.channels = c1
  a1.sinks.k1.channel = c1
  ```

- Start flume
`/bin/flume-ng agent --conf conf --conf-file example.conf --name a1 -Dflume.root.logger=INFO,console`

- Open another console and run ```telnet localhost 44444```
  like this:
 ```shell
 Trying 127.0.0.1...
 Connected to localhost.localdomain (127.0.0.1).
 Escape character is '^]'.
 Hello world! <ENTER>
 OK
 ```
 In flume running terminal, the following infomation should be found.
 ```shell
 12/06/19 15:32:19 INFO source.NetcatSource: Source starting
 12/06/19 15:32:19 INFO source.NetcatSource: Created serverSocket:sun.nio.ch.ServerSocketChannelImpl[/127.0.0.1:44444]
 12/06/19 15:32:34 INFO sink.LoggerSink: Event: { headers:{} body: 48 65 6C 6C 6F 20 77 6F 72 6C 64 21 0D          Hello world!. }
 ```
**Note: flume log files are stored in `/var/log/flume-ng`**

7. Integrate flume service into CM console
On the Home > Status tab, click ComboBOx to the right of the cluster name and select Add a Service. A list of service types display. You can add one type of service at a time.

![add service](https://github.com/CasiaFan/BigDataAnalysis-Spark/blob/master/CM-install/CM%20issue/CM-add-kafka-flume-service.jpg)

***Install Kafka (all nodes)***<br>
**A. install parcels and add service in CM console. (recommend)**
In Cloudera Manager, download the Kafka parcel, distribute the parcel to the hosts in your cluster, and then activate the parcel. Then add service into CM like mentioned above. The default kafka verison is the most updated: **2.0.1-1.2.0.1.p0.5**

![parcel install](https://github.com/CasiaFan/BigDataAnalysis-Spark/blob/master/CM-install/CM%20issue/CM.kafka-parcel-install.jpg)

**B. Install parcels and add service in CM console.**<br>
- Select kafka version to be installed. Here we choose 0.8.2.0+kafka1.4.0+127
```shell
cd /etc/yum.repos.d
wget http://archive.cloudera.com/kafka/redhat/6/x86_64/kafka/cloudera-kafka.repo
```

- Installation
```shell
sudo yum clean all
sudo yum install kafka
sudo yum install kafka-server
```
- Edit `/etc/kafka/conf/server.properties`
Ensure the broker.id is unique for each node and broker in Kafka cluster, and zookeeper.connect points to same ZooKeeper for all nodes and brokers.

~~Here, we set the Zookeper as same as the node of SM server: server003:2181 (2181 is the default port)~~

We just use the default setting and modify them in CM console.

- Start Kafka
`sudo service kafka-server start`

- Verify if all nodes are correctly registered to the same ZooKeeper
```shell
zookeeper-client
ls /brokers/ids #see all of the IDs for the brokers you have registered in your Kafka cluster.
get /brokers/ids/<ID> #discover to which node a particular ID is assigned
```

---
### Sevral issues during CM console configuration
**1. [Wrong Java version used in CDH:](http://stackoverflow.com/questions/28854722/how-to-change-the-version-of-java-that-cdh-uses)** <br>
modify `/etc/default/bigtop-utils file`<br>
`export JAVA_HOME=/usr/java/latest/ # export JAVA_HOME`

**2. See missing and underreplicated blocks: [Access denied for user root. Superuser privilege is required Fsck on path '/' FAILED (hdfs fsck -list-corruptfileblocks)](https://community.cloudera.com/t5/Storage-Random-Access-HDFS/hdfs-fsck-command-issue/td-p/25870)**:<br>
```shell
runuser -l  userNameHere -c 'command'
runuser -l  userNameHere -c '/path/to/command arg1 arg2'
```

**3. Kafka: fail to start during activation. [Java.lang.outOfMemory Error](https://community.cloudera.com/t5/Storage-Random-Access-HDFS/hdfs-fsck-command-issue/td-p/25870)**

![java error](https://github.com/CasiaFan/BigDataAnalysis-Spark/blob/master/CM-install/CM%20issue/kafka-CM-install-error.java.outofmem.console.jpg)

log file: <br>

![error log](https://github.com/CasiaFan/BigDataAnalysis-Spark/blob/master/CM-install/CM%20issue/kafka-CM-install-error.java.outofmem%2Cserver.log.jpg)

Solution:
Configure the java heaper size of broker:<br>

![java size config](https://github.com/CasiaFan/BigDataAnalysis-Spark/blob/master/CM-install/CM%20issue/broker-heap-size-configuration.jpg)


**4. HDFS error: Percentage under replicated blocks: 100.00%. Critical threshold: 40.00%. [1](https://community.cloudera.com/t5/Cloudera-Manager-Installation/Percentage-under-replicated-blocks-100-00/td-p/26133)[2](http://stackoverflow.com/questions/35464059/cloudera-manager-hdfs-under-replicated-blocks)**<br>

![underreplicated error](https://github.com/CasiaFan/BigDataAnalysis-Spark/blob/master/CM-install/CM%20issue/hdfs.underreplicated-blockes.error.jpg)

Your default replication factor is probably 3. Since you only have one node, that's impossible to satisfy.  On single-node clusters, you should reduce the replication factor to 1 in HDFS configuration in Cloudera Manager. After modification, you may need to restart cloudera agent service; if it doesn't work, then run `hadoop fsck /|egrep -v '^\.+$'|grep -i replica`.

![replication factor config](https://github.com/CasiaFan/BigDataAnalysis-Spark/blob/master/CM-install/CM%20issue/hdfs.replicated-block-config.jpg)

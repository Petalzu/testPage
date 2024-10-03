---
title: Cloud Computing & Big Data
date: 2024-06-04 15:53:44
updated: 2024-06-06 16:07:00
tags: [云计算, 大数据, hadoop, ceph, spark, openstack, docker]
categories: 
  - 笔记
  - 计算
toc: true
---
## 前言
云计算是一种基于互联网的计算方式，通过互联网提供的各种服务，包括计算、存储、网络、数据库、分析等，用户可以根据需求随时随地使用这些服务。云计算的优势在于灵活性、可扩展性、高可用性、安全性等，可以帮助用户降低成本、提高效率、提升竞争力。

本文记录了一些云计算相关内容实验和学习笔记。
<!-- more -->
## ssh免密认证
ssh key认证的基本原理是生成密钥对，将公钥id_rsa.pub内容放到认证文件authorized_keys中，使用私钥id_rsa登录服务器。

1、生成密钥对

ssh-keygen

2、将公钥放到要目标认证节点的认证文件中

#ssh-copy-id root@node01
#ssh-copy-id root@node02
#ssh-copy-id root@node03


3、如果要全部相互免认证，则创建一个config文件，内容如下：

#vi \~/.ssh/config

Host *
StrictHostKeyChecking no

#cd \~/.ssh/
#cat id_rsa.pub \>\> authorized_keys
#chmod 600 config authorized_keys
#cd \~/.ssh
#scp authorized_keys config id_rsa node01:\~/.ssh/
#scp authorized_keys config id_rsa node02:\~/.ssh/
#scp authorized_keys config id_rsa node03:\~/.ssh/


### Setup passphraseless ssh

#ssh-keygen -t rsa -P '' -f \~/.ssh/id_rsa
#cat \~/.ssh/id_rsa.pub \>\> \~/.ssh/authorized_keys
#chmod 0600 \~/.ssh/authorized_keys

#vi \~/.ssh/config

Host *
    StrictHostKeyChecking no

#chmod 600 \~/.ssh/config



## ceph cluster manual setup
### 一、基础配置
准备至少三台虚拟机，每台虚拟机除了系统盘还需至少一个数据盘，两块网卡
- 0、最小化安装操作系统
- 1、配置网络，使用固定IP地址
- 2、配置时区和时钟同步
- 3、配置主机名和主机名解析
- 4、配置ssh密钥认证和免密认证
- 5、关闭防火墙和不需要的服务
- 6、基本的文件系统优化，生产环境还需对磁盘和网卡进行优化
- 7、升级系统到最新
- 8、添加所需软件仓库(centos7需要添加yum源，只能支持到N版本，openEuler和ubuntu都不需要添加除非使用更新版本)

手动安装部署分布式存储ceph集群
所有节点安装ceph，以openEuler为例：

#yum install -y ceph   （Uubntu：#apt update && apt install -y ceph）

### 二、mon配置
1. 登录到node01，查看ceph目录是否已经生成

#ls /etc/ceph
rbdmap 

2. 生成ceph配置文件

#touch /etc/ceph/ceph.conf

3. 执行uuidgen命令，得到一个唯一的标识，作为ceph集群的ID

#uuidgen
79e959e7-a534-46ef-94b7-eaee96e4c4ee

4. 手动配置ceph.conf，添加基础内容

#vi /etc/ceph/ceph.conf

[global]
fsid = 79e959e7-a534-46ef-94b7-eaee96e4c4ee
mon initial members = node01
mon host = 192.168.1.101
#存储网络和集群网络地址段
public network = 192.168.1.0/24
cluster network = 192.168.2.0/24
#启用集群认证
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
#设置副本数，默认为3
osd pool default size = 3
#PG处于degraded(降级)状态不影响其IO能力，min_size是一个PG能接受IO的最小副本数，默认是2
osd pool default min size = 1
#新建pool默认pg和pgp，实验环境我们设置较小，实际环境要根据节点和osd数进行计算
osd pool default pg num = 64
osd pool default pgp num = 64
#在 CRUSH 规则中chooseleaf使用的存储桶类型，1表示host
osd crush chooseleaf type = 1
osd_mkfs_type = xfs
max mds = 5
mds max file size = 100000000000000
mds cache size = 1000000
#设置osd节点down后900s，把此osd节点逐出ceph集群，把之前映射到此节点的数据映射到其他节点。
mon osd down out interval = 900
[mon]
#把时钟偏移设置成0.5s，默认是0.05s,由于ceph集群中存在异构PC，导致时钟偏移总是大于0.05s，为了方便同步直接把时钟偏移设置成0.5s
mon clock drift allowed = .50

5. 为集群创建密钥环、并生成监视器密钥

#ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'

6. 生成管理员密钥环，生成 client.admin 用户并加入密钥环。

#sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'

7. 生成一个引导-osd密钥环，生成一个client.bootstrap-osd用户并将用户添加到密钥环中

#sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd'

8. 将生成的密钥添加到ceph.mon.keyring

#sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
#sudo ceph-authtool /tmp/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring

9. 使用主机名、主机IP地址(ES)和FSID生成monmap。把它保存成/tmp/monmap

#monmaptool --create --add node01 192.168.1.101 --fsid 79e959e7-a534-46ef-94b7-eaee96e4c4ee /tmp/monmap
*此处主机将命令行中主机名、IP地址和fsid更换为实际所用的

10. 创建一个默认的数据目录，默认名称为ceph-主机名，当前主机名为node01，所以默认目录名为ceph-node01

#sudo -u ceph mkdir /var/lib/ceph/mon/ceph-node01

11. 修改ceph.mon.keyring属主和属组为ceph

#chown ceph.ceph /tmp/ceph.mon.keyring

12. 始化mon

#sudo -u ceph ceph-mon --mkfs -i node01 --monmap /tmp/monmap --keyring /tmp/ceph.mon.keyring

*初始化完毕查看/var/lib/ceph/mon/ceph-node01下是否生成文件，如果没有生成则是文件夹权限问题，需要将文件夹属主改成ceph，以及ceph.mon.keyring属主问题。

#ls /var/lib/ceph/mon/ceph-node01/
keyring  kv_backend  store.db

13. 为了防止重新被安装创建一个空的done文件
#sudo touch /var/lib/ceph/mon/ceph-node01/done

14. 启动mon
#systemctl start ceph-mon@node01    （Ubuntu因为安装完已经启动服务，用restart重启服务使得配置生效）

15. 查看状态
#systemctl status ceph-mon@node01

16. 设置开机启动
#systemctl enable ceph-mon@node01


17. 查看集群情况

[root@node01 \~]#ceph -s
  cluster:
    id:     79e959e7-a534-46ef-94b7-eaee96e4c4ee
    health: HEALTH_WARN
            mon is allowing insecure global_id reclaim
            1 monitors have not enabled msgr2
  services:
    mon: 1 daemons, quorum node01 (age 27s)
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:

存在health告警，运行如下命令消除：

#ceph config set mon auth_allow_insecure_global_id_reclaim false --force
#ceph mon enable-msgr2


18. 将/etc/ceph下文件拷贝到其他集群主机
#scp /etc/ceph/* root@node02:/etc/ceph/
#scp /etc/ceph/* root@node03:/etc/ceph/
拷贝过去以后其他节点也就可以执行ceph -s查看集群情况了。
第一次安装如果出现key错误就重新开始。

### # 添加其他mon
（mon节点推荐为3个，如果添加了node02,那么node03也要添加一下，配置文件中mon host也需要增加新mon地址）
1. 登录新监视器主机创建默认目录
#ssh node02
#sudo -u ceph mkdir /var/lib/ceph/mon/ceph-node02
2. 在临时目录获取监视器密钥环
#ceph auth get mon. -o /tmp/ceph.mon.keyring
3. 获取监视器运行图
#ceph mon getmap -o /tmp/ceph.mon.map
4. 修改ceph.mon.keyring属主和属组为ceph
#chown ceph.ceph /tmp/ceph.mon.keyring
5. 初始化mon
#sudo -u ceph ceph-mon --mkfs -i node02 --monmap /tmp/ceph.mon.map --keyring /tmp/ceph.mon.keyring
6. 为了防止重新被安装创建一个空的done文件
#sudo touch /var/lib/ceph/mon/ceph-node02/done
7. 启动服务，它会自动加入集群
#systemctl start ceph-mon@node02           （Ubuntu因为安装完已经启动服务，用restart重启服务使得配置生效）
#systemctl status ceph-mon@node02
#systemctl enable ceph-mon@node02
9. 查看集群
#ceph -s

*如果因添加node2导致集群出错，需要到node01去先修改配置，再从1开始添加新的mon，node01步骤如下：
#systemctl stop ceph-mon@node01
#monmaptool /tmp/monmap --rm node02
#ceph-mon -i node01 --inject-monmap /tmp/monmap
#systemctl start ceph-mon@node01
### # 添加其他mon(03)
1. 登录新监视器主机创建默认目录
#ssh node03
#sudo -u ceph mkdir /var/lib/ceph/mon/ceph-node03
2. 在临时目录获取监视器密钥环
#ceph auth get mon. -o /tmp/ceph.mon.keyring
3. 获取监视器运行图
#ceph mon getmap -o /tmp/ceph.mon.map
4. 修改ceph.mon.keyring属主和属组为ceph
#chown ceph.ceph /tmp/ceph.mon.keyring
5. 初始化mon
#sudo -u ceph ceph-mon --mkfs -i node03 --monmap /tmp/ceph.mon.map --keyring /tmp/ceph.mon.keyring
6. 为了防止重新被安装创建一个空的done文件
#sudo touch /var/lib/ceph/mon/ceph-node03/done
7. 启动服务，它会自动加入集群
#systemctl start ceph-mon@node03            （Ubuntu因为安装完已经启动服务，用restart重启服务使得配置生效）
#systemctl status ceph-mon@node03
#systemctl enable ceph-mon@node03
9. 查看集群
#ceph -s

修改配置文件，增加所有mon节点的IP地址

#vi /etc/ceph/ceph.conf

[global]
mon host = 192.168.1.101,192.168.1.102,192.168.1.103

将配置文件scp到其他节点：(且勿向本节点scp，否则可能会导致文件为空)

[root@node01 \~]#scp /etc/ceph/ceph.conf node02:/etc/ceph/
[root@node01 \~]#scp /etc/ceph/ceph.conf node03:/etc/ceph/

重启各个节点的服务使得配置生效(Linux配置文件更改后必须重启相应服务才能生效)

[root@node01 \~]#systemctl daemon-reload && systemctl restart ceph-mon@node01
[root@node02 \~]#systemctl daemon-reload && systemctl restart ceph-mon@node01
[root@node03 \~]#systemctl daemon-reload && systemctl restart ceph-mon@node01

### 三、安装OSD
准备存储磁盘，如vdb。系统盘为vda。

#ssh node01
#lsblk
#ls /dev/vd*

/dev/vdb如果为全新盘可直接用来准备OSD

#sudo ceph-volume lvm create --data /dev/vdb

OSD编号默认从0开始增加            

#systemctl start ceph-osd@0
#systemctl status ceph-osd@0
#systemctl enable ceph-osd@0

注意事项：
1、使用的硬盘必须先利用dd命令把硬盘的前512K填充为0,直接干掉分区信息。
#dd if=/dev/zero of=/dev/vdb bs=512K count=1
注意,一定要再三检查目标硬盘是否是期望的硬盘,如果操作错了硬盘,分区表直接就没了。
2、OSD依赖于cluster network，cluster网络不通会导致OSD启动失败。

其他节点需要提前导出/var/lib/ceph/bootstrap-osd/ceph.keyring，或从node01上拷贝。
#ceph auth get  client.bootstrap-osd -o /var/lib/ceph/bootstrap-osd/ceph.keyring

所有节点添加完毕查看

#ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         0.29306  root default
-3         0.09769      host node01
 0    hdd  0.09769          osd.0        up   1.00000  1.00000
-5         0.09769      host node02
 1    hdd  0.09769          osd.1        up   1.00000  1.00000
-7         0.09769      host node03
 2    hdd  0.09769          osd.2        up   1.00000  1.00000


#ceph -s
  cluster:
    id:     79e959e7-a534-46ef-94b7-eaee96e4c4ee
    health: HEALTH_WARN
            no active mgr
  services:
    mon: 3 daemons, quorum node01,node02,node03 (age 33m)
    mgr: no daemons active
    osd: 3 osds: 3 up (since 3m), 3 in (since 13m)
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:

生产环境中buluestore db和wal日志可以使用SSD或NVME固态盘以提高性能。创建OSD命令如下：

 ceph-volume lvm prepare --data {vg/lv} --block.wal {partition} --block.db {/path/to/device}     

其中：

--data        sata        HDD  big disk   数据盘，sata接口，存储容量较大
--block.db    sas/PCI-E   SSD  bigpart    数据db盘，ssd或NVME，默认数据盘4%，一般大
--block.wal   sas/PCI-E   SSD  smallpart  数据日志盘，ssd或NVME，默认数据盘1%，较小。

例如

#sudo ceph-volume lvm prepare --data /dev/sdc --block.wal /dev/sdg2 --block.db /dev/sdg1
#sudo ceph-volume lvm activate --all
#sudo ceph-volume lvm list

显示osd.num，num后面会用到。

#sudo ceph-volume lvm activate 0   #如果上一步显示已activate successful可跳过

### 四、安装mgr服务
mgr可以安装在任意一个节点上，通常和mon安装在一起，可以安装两个mrg，但任何时候只有一个节点的mgr为active，另一个处于standbys。但active的节点出现故障时，standbys立即切换为actvie以防止服务中断。出故障的mgr恢复后就成为standbys，并不会切回之前的角色。
安装mgr服务步骤如下：
1. 登录到节点，创建密钥环
#ssh node01
#ceph auth get-or-create mgr.node01 mon 'allow profile mgr' osd 'allow *' mds 'allow *'
2. 创建mgr目录
#sudo -u ceph mkdir /var/lib/ceph/mgr/ceph-node01/
3. 导出密钥环到mgr目录下
#ceph auth get mgr.node01 -o /var/lib/ceph/mgr/ceph-node01/keyring
4. 启动服务
#systemctl start ceph-mgr@node01
#systemctl status ceph-mgr@node01
#systemctl enable ceph-mgr@node02

查看集群状态
[root@node01 \~]#ceph -s
  cluster:
    id:     79e959e7-a534-46ef-94b7-eaee96e4c4ee
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum node01,node02,node03 (age 41m)
    mgr: node01(active, since 13s)
    osd: 3 osds: 3 up (since 12m), 3 in (since 22m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   15 MiB used, 300 GiB / 300 GiB avail
    pgs:     1 active+clean

在其他节点上安装mgr：
#ceph auth get-or-create mgr.#name mon 'allow profile mgr' osd 'allow *' mds 'allow *'
#sudo -u ceph mkdir /var/lib/ceph/mgr/ceph-#name
#ceph auth get mgr.#name-o /var/lib/ceph/mgr/ceph-#namekeyring
#systemctl start ceph-mgr@#name#systemctl status ceph-mgr@#name#systemctl enable ceph-mgr@#name
安装danshboard（dashboard只能安装在mgr节点，即使有两个mrg，只有actve的mgr上的生效）：

#yum install -y ceph-mgr-dashboard   (Ubuntu使用apt命令安装ceph-mgr-dashboard包即可)

mgr启用dashboard模块

#ceph mgr module enable dashboard

自创建证书

#ceph dashboard create-self-signed-cert
#openssl req -new -nodes -x509   -subj "/O=IT/CN=ceph-mgr-dashboard" -days 3650   -keyout dashboard.key -out dashboard.crt -extensions v3_ca
#ceph config-key set mgr/dashboard/crt -i dashboard.crt
#ceph config-key set mgr/dashboard/key -i dashboard.key

如果没有证书关闭证书认证

#ceph dashboard set-rgw-api-ssl-verify False
如果有证书则导入

#ceph config-key set mgr/dashboard/crt -i dashboard.crt
#ceph config-key set mgr/dashboard/key -i dashboard.key

指定IP和端口（IP为active mgr的导致，端口可以任意指定，只要是没有使用的，例如http端口使用8080，https端口使用8443）

#ceph config set mgr mgr/dashboard/server_addr     #IP
#ceph config set mgr mgr/dashboard/server_port     #PORT
#ceph config set mgr mgr/dashboard/ssl_server_port #SSLPORT

启用证书认证时dashboard使用https协议，关闭证书认证时使用http协议。

#ceph config set mgr mgr/dashboard/server_addr  192.168.1.101
#ceph config set mgr mgr/dashboard/server_port  8080
#ceph config set mgr mgr/dashboard/ssl_server_port 8443

查看

#ceph config dump


查看用户role

#ceph dashboard ac-role-show

查看用户

#ceph dashboard ac-user-show 

创建用户使用 ceph dashboard ac-user-create命令，使用格式如下：

#ceph dashboard ac-user-create \<username> -i \<file-containing-password\> administrator

首先创建一个用户密码文件，文件内容为用户密码，然后创建管理员角色用户

#vi adminpassword
#ceph dashboard ac-user-create admin  -i adminpassword administrator
{"username: "admin", "password": "#2b#12#TeQxJA9D7ZE9bmfCFz5id.EwVHZJdx4GvSO.LBFsctoAYHF0XPTKW", "roles": ["administrator"], "name: null, "email": null, "lastUpdate": 1705570543, "enabled": true, "pwdExpirationDate": null, "pwdUpdateRequired": false}

mgr服务重新加载配置

#systemctl reload ceph-mgr@node01

查看端口，看看服务是否已经启动

#netstat -an | grep 8443
tcp6       0      0 :::8443                 :::*                    LISTEN

服务启动后通过浏览器访问dashboard：
https://#IP:#PORT



## ubuntu2204安装hadoop-3.3.6
准备好三个Linux虚拟机（一个节点只能使用伪集群），本例中使用了ubuntu2204LTS
本例子中三个节点为node01、node02、node03
### 一、完成准备工作
1. 网络配置

#ls /etc/netplan/
#vi /etc/netplan/50-cloud-init.yaml
network:
    version: 2
    ethernets:
        ens3:
            dhcp4: false
            match:
                macaddress: fa:16:3e:b1:41:1c
            set-name ens3
            addresses: [192.168.1.101/24]
            routes:
              - to: default
                via: 192.168.1.1
                metric: 100
            nameervers:
              addresses: [202.201.0.131,202.201.0.132]
#netplan apply

2. 设置ssh免密认证，方便scp文件

#ssh-keygen -t rsa -P '' -f \~/.ssh/id_rsa
#cat \~/.ssh/id_rsa.pub \>\> \~/.ssh/authorized_keys
#chmod 0600 \~/.ssh/authorized_keys
#vi \~/.ssh/config
Host *
    StrictHostKeyChecking no
#chmod 600 \~/.ssh/config


3. 主机名设置和解析

#设置主机名
hostnametl set-hostnamenode01
#查看IP地址
ip add
#添加主机名解析
vi /etc/hosts

192.168.1.101  node01
192.168.1.102  node02
192.168.1.103  node03
#验证
ping node01
ping node02
ping node03
#将解析文件scp到其他节点
scp /etc/hosts node02:/etc/
scp /etc/hosts node03:/etc/


4. 设置时区和时钟同步

#timedatectl set-timezone Asia/Shanghai
NTP客户端可使用ubuntu自带的systemd-timesyncd
#systemctl status systemd-timesyncd
#vi /etc/systemd/timesyncd.conf
[Time]
NTP= ntpservername#systemctl restart systemd-timesyncd
#timedatectl timesync-status

或者安装chrony作为ntp客户端
#apt update && apt install chrony -y
#vi /etc/chrony/chrony.conf
#pool 0.ubuntu.pool.ntp.org iburst maxsources 1
#pool 1.ubuntu.pool.ntp.org iburst maxsources 1
#pool 2.ubuntu.pool.ntp.org iburst maxsources 2 
pool ntpservername       iburst maxsources 2
#systemctl restart chrony
#chronyc sources


5. 优化系统关闭不需要的服务以节省资源

文件系统优化
#vi /etc/sysctl.conf
fs.file-max = 20480000
fs.nr_open= 10240000
#sysctl -p
#vi /etc/security/limits.conf
* soft     nproc          102400
* hard     nproc          104800
* soft     nofile         102400
* hard     nofile         104800
root soft     nproc          102400
root hard     nproc          104800
root soft     nofile         102400
root hard     nofile         104800
#reboot

关闭不需要的服务
#systemctl set-default multi-user.target 
#systemctl disable cloud-config.service cloud-final.service cloud-init-local.service cloud-init.service cloud-init-hotplugd.socket cloud-config.target cloud-init.target  multipathd.service multipathd.socket iscsid.socket  apparmor.service systemd-resolved.service ufw.service fwupd.service  graphical.target cron

Ubuntu自带一个systemd-resolved.service服务，/etc/resolv.conf是个链接文件，停止该服务需要删除该链接文件并新建。
#rm /etc/resolv.conf
#vi /etc/resolv.conf
nameerver 
设置一个网络可达服务可用的DNS


6. 升级系统

apt update && apt upgrade -y


7. 创建所需用户并做ssh免密认证(直接使用root账户也可以，一般不推荐，注意/home磁盘分区大小)

#adduser hadoop
切换到hadoop用户，并为hadoop用户做ssh免密认证。
#sudo su - hadoop
hadoop@node01:\~#
#ssh-keygen -t rsa -P '' -f \~/.ssh/id_rsa
#cat \~/.ssh/id_rsa.pub \>\> \~/.ssh/authorized_keys
#chmod 0600 \~/.ssh/authorized_keys
#vi \~/.ssh/config
Host *
    StrictHostKeyChecking no
#chmod 600 \~/.ssh/config


### 二、jdk8安装
在hadoop用户下完成jdk8的安装配置：

#wget /jdk-8u401-linux-x64.tar.gz
#tar -zxvf jdk-8u401-linux-x64.tar.gz
#./jdk1.8.0_401/bin/java -version

在.bashrc文件中添加jdk8环境变量

#vi .bashrc
#jdk8 path config
export JAVA_HOME=/home/hadoop/jdk1.8.0_401
export PATH=#JAVA_HOME/bin:#PATH
export CLASSPATH=.:#JAVA_HOME/jre/lib:#JAVA_HOME/lib:#JAVA_HOME/lib/tools.jar
#source .bashrc
#java -version
java version "1.8.0_401"
Java(TM) SE Runtime Environment (build 1.8.0_401-b10)
Java HotSpot(TM) 64-Bit Server VM (build 25.401-b10, mixed mode)

将jdk8包和修改后的环境变量scp到其他两个节点\

#scp -r jdk1.8.0_401/  node02:/home/hadoop/
#scp -r jdk1.8.0_401/  node03:/home/hadoop/
#scp .bashrc node02:/home/hadoop/
#scp .bashrc node03:/home/hadoop/


### 三、hadoop安装配置
下载hadoop-3.3.6二进制包，解压

#wget /hadoop-3.3.6.tar.gz
#tar -zxvf hadoop-3.3.6.tar.gz

设置haodoop环境变量

#vi \~/.bashrc
#hadoop path config
export HADOOP_HOME=/home/hadoop/hadoop-3.3.6
export PATH=#PATH:#HADOOP_HOME/bin:#HADOOP_HOME/sbin
export HADOOP_MAPRED_HOME=#HADOOP_HOME
export HDFS_NAMEODE_USER=hadoop
export HDFS_DATANODE_USER=hadoop
export HDFS_SECONDARYNAMEODE_USER=hadoop
export YARN_RESOURCEMANAGER_USER=hadoop
export YARN_NODEMANAGER_USER=hadoop

hadoop@node01:\~#source \~/.bashrc
hadoop@node01:\~#hadoop version
Hadoop 3.3.6
Source code repository https://github.com/apache/hadoop.git -r 1be78238728da9266a4f88195058f08fd012bf9c
Compiled by ubuntu on 2023-06-18T08:22Z
Compiled on platform linux-x86_64
Compiled with protoc 3.7.1
From source with checksum 5652179ad55f76cb287d9c633bb53bbd
This command was run using /home/hadoop/hadoop-3.3.6/share/hadoop/common/hadoop-common-3.3.6.jar


修改hadoop配置文件：

1. etc/hadoop/hadoop-env.sh
#cd #HADOOP_HOME
#vi etc/hadoop/hadoop-env.sh
export JAVA_HOME=/home/hadoop/jdk1.8.0_401
export HADOOP_HOME=/home/hadoop/hadoop-3.3.6
export HADOOP_CONF_DIR=#{HADOOP_HOME}/etc/hadoop

2. etc/hadoop/core-site.xml:
#vi etc/hadoop/core-site.xml
```xml
<configuration>
<property>
<name>fs.defaultFS</name>
<value>hdfs://node01:9000</value>
</property>
<property>
<name>hadoop.tmp.dir</name>
<value>/home/hadoop/hadoop-3.3.6/tmp</value>
</property>
</configuration>
```
创建tmp目录
#mkdir /home/hadoop/hadoop-3.3.6/tmp

3. etc/hadoop/hdfs-site.xml
#vi etc/hadoop/hdfs-site.xml
```xml
<configuration>
<property>
<name>dfs.nameode.http-address</name>
<value>node01:9870</value>
</property>
<property>
<name>dfs.nameode.namedir</name>
<value>/home/hadoop/hadoop-3.3.6/dfs/name/value>
</property>
<property>
<name>dfs.datanode.data.dir</name>
<value>/home/hadoop/hadoop-3.3.6/dfs/data</value>
</property>
<property>
<name>dfs.replication</name>
<value>3</value>
</property>
<property>
<name>dfs.permissions.enabled</name>
<value>false</value>
</property>
</configuration>
```
当前副本数为3，一个节点需要将dfs.replication值设置为1

4. etc/hadoop/mapred-site.xml
#vi etc/hadoop/mapred-site.xml
```xml
<configuration>
<property>
<name>mapreduce.framework.name/name>
<value>yarn</value>
</property>
<property>
<name>mapreduce.application.classpath</name>
<value>#HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:#HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
</property>
<property>
<name>mapreduce.admin.user.env</name>
<value>HADOOP_MAPRED_HOME=/home/hadoop/hadoop-3.3.6</value>
</property>
<property>
<name>yarn.app.mapreduce.am.env</name>
<value>HADOOP_MAPRED_HOME=/home/hadoop/hadoop-3.3.6</value>
</property>
</configuration>
```

5. etc/hadoop/yarn-site.xml
#vi etc/hadoop/yarn-site.xml
```xml
<configuration>
<property>
<name>yarn.nodemanager.aux-services</name>
<value>mapreduce_shuffle</value>
</property>
<property>
<name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
<value>org.apache.hadoop.mapred.ShuffleHandler</value>
</property>
<property>
<name>yarn.resourcemanager.hostname/name>
<value>node01</value>
</property>
<property>
<name>yarn.nodemanager.env-whitelist</name>
<value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
</property>
<property>
<name>yarn.nodemanager.vmem-check-enabled</name>
<value>false</value>
</property>
</configuration>
```

6. etc/hadoop/workers 
#vi etc/hadoop/workers
node01
node02
node03

只有一个节点的在workers只写当前节点名称。实际使用的是IP地址，这些主机名我们都已经做过解析了，否则直接写节点IP地址。


将hadoop和修改后的环境变量文件.bashrc拷贝到其他两个节点的用户目录下：

#cd
hadoop@node01:\~# scp -r hadoop-3.3.6 node02:/home/hadoop/
hadoop@node01:\~# scp -r hadoop-3.3.6 node03:/home/hadoop/
hadoop@node01:\~# scp .bashrc node02:/home/hadoop/
hadoop@node01:\~# scp .bashrc node03:/home/hadoop/


初始文件系统
hadoop@node01:\~# hdfs nameode -format
启动分布式文件系统hdfs服务：
hadoop@node01:\~# start-dfs.sh
第一次启动会创建数据目录和日志目录

查看进程：

hadoop@node01:\~#jps
129953 Jps
129539 DataNode
129396 Nameode
129772 SecondaryNameode

hadoop@node01:\~#ssh node02
hadoop@node02:\~#jps
28615 Jps
28475 DataNode
hadoop@node02:\~#exit

hadoop@node01:\~#ssh node03
hadoop@node03:\~#jps
21988 DataNode
22138 Jps
hadoop@node03:\~#exit


会看到node01上有Nameode、SecondaryNameode和DataNode，node02和node03上只有DataNode。hdfs服务正常。

此时可以通过浏览器访问http://node01:9870/
（windows客户端可在c:\Windows\system32\drivers\etc\hosts添加解析）

启动资源管理服务yarn：

hadoop@node01:\~#start-yarn.sh

hadoop@node01:\~#jps
52179 SecondaryNameode
51942 DataNode
52679 NodeManager
51803 Nameode
52526 ResourceManager
hadoop@node02:\~#jps
10594 Jps
10425 NodeManager
10173 DataNode
hadoop@node03:\~#jps
4673 Jps
4254 DataNode
4511 NodeManager


会看到node01上启动了ResourceManager和NodeManager，node02和node03上启动了NodeManager。
资源管理默认web页面为http://node01:8088/

分布式文件系统hdfs目录结构和使用类似于Linux文件系统，根目录下可创建用户目录，每个用户会有一个默认目录。
我们给当前用户创建默认目录：
hadoop@node01:\~#hdfs dfs -mkdir /user
hadoop@node01:\~#hdfs dfs -mkdir /user/hadoop
创建input测试目录，上传测试文件：
hadoop@node01:\~#hdfs dfs -mkdir input
hadoop@node01:\~#cd #HADOOP_HOME
hadoop@node01:\~/hadoop-3.3.6#hdfs dfs -put etc/hadoop/*.xml input
hadoop@node01:\~/hadoop-3.3.6#hdfs dfs -ls
hadoop@node01:\~/hadoop-3.3.6#hdfs dfs -ls input
关于hdfs dfs命令可以参考帮助
#hdfs dfs -h
几个常见的命令：（和Linux命令基本相似）
-ls
-put
-get
-mkdir
-copyFromLocal
-copyToLocal
-mv
-chown
-chgrp
-chmod
-appendToFile
-rm
-rmdir

在hdfs里通常使用最多的是读和写，hdfs不支持修改，只支持追加append。这一点有点和对象存储相似。

运行一个例子
#cd #HADOOP_HOME
hadoop@node01:\~/hadoop-3.3.6#bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.6.jar grep input output 'dfs[a-z.]+'
查看输出：
#bin/hdfs dfs -cat output/*
#hdfs dfs -cat output/*

运行pi例子：
#cd #HADOOP_HOME/share/hadoop/mapreduce
#hadoop jar hadoop-mapreduce-examples-3.3.6.jar pi 10 20
......
Job Finished in 23.133 seconds
Estimated valueof Pi is 3.12000000000000000000

最后，停止服务：
#stop-yarn.sh
#stop-dfs.sh 
#jps

JAVA的程序比较吃内存，推荐每个节点至少8G以上内存，否则跑例子程序可能都会造成内存溢出而失败。

### 验证服务

1. Create java sourcecode file

#vi WordCount.java
```java
import java.io.IOException;
import java.util.StringTokenizer;
import org.apache.hadoop.conf.Configuration
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

  public static class TokenizerMapper
       extends Mapper<Object, Text, Text, IntWritable\>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(valuetoString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
      }
    }
  }

  public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable\> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable\> value,
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : value) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }

  public static void main(String[] args) throws Exception {
    Configurationconf = new Configuration);
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValuelass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

2. Compile WordCount.java and create a jar:
#hadoop com.sun.tools.javac.Main WordCount.java
#jar cf wc.jar WordCount*.class

3. Create text file and upload it to hdfs
#vi file1
Hello World  Bye World
Hello Hadoop Bye Hadoop
Bye Hadoop Hello Hadoop
#hdfs dfs -rm -r input
#hdfs dfs -rm -r output
#hdfs dfs -mkdir input
#hdfs dfs -put file1 input
#hdfs dfs -ls input

4. Run the application:
#hadoop jar wc.jar WordCount /user/hadoop/input/file1 /user/hadoop/output/
Output:
#hdfs dfs -ls output
Found 2 items
-rw-r--r--   3 hadoop supergroup          0 2024-05-14 10:31 output/_SUCCESS
-rw-r--r--   3 hadoop supergroup         31 2024-05-14 10:31 output/part-r-00000
#hdfs dfs -cat output/part-r-00000
Bye     3
Hadoop  4
Hello   3
World   2

## Spark-3.4.2 安装配置
安装scala语言支持(所有节点安装，如果不需要scala语言支持则无需安装)

#sudo apt update && sudo apt install scala -y
下载解压spark-3.4.2
#wget /spark-3.4.2-bin-hadoop3.tgz
#tar -zxvf spark-3.4.2-bin-hadoop3.tgz
添加环境变量
#vi .bashrc
#spark env config
export SPARK_HOME=\~/spark-3.4.2-bin-hadoop3
export PATH=#PATH:#SPARK_HOME/bin
export LD_LIBRARY_PATH=#HADOOP_HOME/lib/native/:#LD_LIBRARY_PATH
#source .bashrc
配置
#cd #SPARK_HOME/conf
#cp workers.template workers

添加工作节点（单节点只添加本机）

#vi workers
node01
node02
node03
#cp spark-env.sh.template spark-env.sh
#vi spark-env.sh
export JAVA_HOME=/home/hadoop/jdk1.8.0_401
export HADOOP_HOME=/home/hadoop/hadoop-3.3.6
export HADOOP_CONF_DIR=/home/hadoop/hadoop-3.3.6/etc/hadoop/
export SPARK_MASTER_HOST=node01
export SPARK_PID_DIR=/home/hadoop/spark-3.4.2-bin-hadoop3/data
export SPARK_LOCAL_DIR=/home/hadoop/spark-3.4.2-bin-hadoop3
export SPARK_EXECUTOR_MEMORY=512M
export SPARK_WORKER_MEMORY=2G
export SCALA_HOME=/usr/share/scala


#cp spark-defaults.conf.template spark-defaults.conf
#vi spark-defaults.conf
spark.master                     spark://node01:7077

拷贝到其他节点
#cd
#scp -r spark-3.4.2-bin-hadoop3  node02:/home/hadoop/
#scp -r spark-3.4.2-bin-hadoop3  node03:/home/hadoop/
#scp .bashrc node02:/home/hadoop/
#scp .bashrc node03:/home/hadoop/


启动服务,在node01节点上会看到Master和Worker进程，其他节点会看到Worker进程。

##SPARK_HOME/sbin/start-all.sh
hadoop@node01:\~#jps
7992 Master
8133 Worker
hadoop@node02:\~#jps
4389 Worker
hadoop@node03:\~#jps
3942 Worker
还可访问Web查看
http://node01:8080/

运行一个例子

##SPARK_HOME/bin/run-example SparkPi 10
......
Pi is roughly 3.141795141795142
......


测试spark-shell

#spark-shell
scala\> val textFile=sc.textFile("file:///home/hadoop/spark-3.4.2-bin-hadoop3/README.md")
scala\> textFile.count()
scala\> :quit

测试pyspark

#pyspark
Python 3.10.12 (main, Nov 20 2023, 15:14:05) [GCC 11.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 3.4.3
      /_/

Using Python version 3.10.12 (main, Nov 20 2023 15:14:05)
Spark context Web UI available at http://node01:4040
Spark context available as 'sc' (master = spark://node01:7077, app id = app-20240520172859-0005).
SparkSession available as 'spark'.
\>\>\>
\>\>\> lines=sc.textFile("file:///home/hadoop/spark-3.4.2-bin-hadoop3/README.md")
\>\>\> lines.count()
125
\>\>\> exit()

#wget /spark_examples/try1.txt
#vi try1.py
#imports
from pyspark import SparkConf,SparkContext
conf = SparkConf().setMaster("spark://node01:7077").setAppName"My try1")
sc = SparkContext(conf=conf)
sc.setLogLevel('WARN')
txt = sc.textFile("try1.txt")
print(txt.count())
as_lines = txt.filter(lambda line: 'as' in line.lower())
print(as_lines.count())
将文件上传到hdfs
#hdfs dfs -put try1.txt
提交任务
#spark-submit  try1.py
......
23
5


使用jupyter notebook

#sudo apt update && sudo apt install pip
#sudo pip3 install jupyter
#sudo pip3 install pyspark
#vi \~/.bashrc
export  PYTHONPATH=#PATH:#SPARK_HOME/python
export PYSPARK_PYTHON=python3
#source \~/.bashrc
#jupyter notebook --ip=node01

在浏览器中访问jupyer notebook


停止spark集群服务

##SPARK_HOME/sbin/stop-all.sh


### 运行Spark-shell，解决Unable to load native-hadoop library for your platform
启动spark后，spark-shell或pyspark会出现一个警告
WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable

1、解决方法1
在linux环境变量里设置linux共享库
#vi .bashrc
export LD_LIBRARY_PATH=#HADOOP_HOME/lib/native/:#LD_LIBRARY_PATH

2、解决方法2
环境变量里设置
#vi .bashrc
export JAVA_LIBRARY_PATH=#HADOOP_HOME/lib/native/
并在spark配置文件中设置
#vi spark-env.sh
export LD_LIBRARY_PATH=#JAVA_LIBRARY_PATH

### spark中一些问题的解决
1.ModuleNotFoundError: No module name ‘py4j‘
Ubuntu配置Jupyter Notebook实现和PySpark交互，在运行实例的时候出现ModuleNotFoundError: No module name 'py4j’问题
1.如果不知道自己的py4j版本，可以在命令行中输入cd #SPARK_HOME/python/lib，这表示进入到py4j所在的文件目录下

hadoop@node01:\~#cd #SPARK_HOME/python/lib

2.然后再输入ls

hadoop@node01:\~/spark-3.4.3/python/lib#ls
py4j-0.10.9.7-src.zip  PY4J_LICENSE.txt  pyspark.zip
hadoop@node01:\~/spark-3.4.3/python/lib#ls #SPARK_HOME/python/lib/py4j-0.10.9.7-src.zip
/home/hadoop/spark-3.4.3/python/lib/py4j-0.10.9.7-src.zip

3.这样就能看到自己py4j所对应的版本
然后输入vim \~/.bashrc，将py4j的版本修改为自己虚拟机中对应的版本号，我这里就将其修改为py4j-0.10.7-src.zip，如下图所示：
按下Esc键然后再输入:wq，表示保存并退出这个文件。然后在命令行中输入source \~/.bashrc，使环境配置立马生效

hadoop@node01:\~#vi \~/.bashrc
#python env
export PYTHONPATH=#SPARK_HOME/python:#SPARK_HOME/python/lib/py4j-0.10.9.7-src.zip
export PYSPARK_PYTHON=python3
hadoop@node01:\~#source \~/.bashrc


2.启动SparkContext报错--Cannot run multiple SparkContexts at once; existing SparkContext(...)
创建SparkContext的最基本方法，只需要传递两个参数：
（1）集群URL：告诉Spark如何连接到集群上，使用local可以让spark运行在单机单线程上。
（2）应用名：使用"monter"，当连接到一个集群时，这个值可以在集群管理器的用户界面中找到你的应用。 
报错：出现这个错误是因为之前已经启动了SparkContext，所以需要先关闭spark，然后再启动。
sc.stop()    // 关闭spark
sc = SparkContext(conf = spark)

## ubuntu2204 openstack
### 0-prepare
#### 一、使用云平台创建实验虚拟机或自行创建虚拟机。
最少需要一个虚拟机，可以将所有组件安装在同一台机器。
如果条件允许，可使用多台机器完成不同的功能。例如控制节点、计算节点1、计算节点2，使用多个计算节点可以实现实例(虚拟机)迁移功能。
当前使用node01做为控制节点，node02/node03为计算节点。
自创建虚拟机按以下要求完成基本设置：
0. 最小化安装操作系统，并打开虚拟化嵌套，支持虚拟化功能
1. 配置网络，使用固定IP地址
#ip add
#vi /etc/netp/.yaml
network:
    version: 2
    ethernets:
        ens3:
            dhcp4: true
            set-name ens3
            addresses: [192.168.1.101/24]
            routes:
            - to: default
              via: 192.168.1.1
        ens4:
            dhcp4: false
            addresses: [192.168.2.101/24]
            set-name ens4
        ens8:
            dhcp4: false
            set-name ens8
            match:
当前机器有三块网卡，ens3为openstack集群网络，ens8用来作为vm上联接口，二层接入网段为192.168.2.0/24。
做openstack实验至少需要两块网卡。
手动设置DNS
#rm /etc/resolv.conf
#vi /etc/resolv.conf
nameerver 202.201.0.133
关闭systemd-resolved.service服务
#systemctl stop systemd-resolved.service
#systemctl disable systemd-resolved.service
2. 配置时区和时钟同步
#apt update && apt install chrony -y
#vi /etc/chrony/chrony.conf
pool \<ntpservername>        iburst maxsources 1
#systemctl restart chrony
3. 配置主机名和主机名解析
#hostnametl set-hostnamenode01
#vi /etc/hosts
192.168.1.101  node01
192.168.1.102  node02
192.168.1.103  node03
4. 配置ssh密钥认证和免密认证
5. 关闭不必要的服务，以节省系统资源
#systemctl set-default multi-user.target 
#systemctl disable cloud-config.service cloud-final.service cloud-init-local.service cloud-init.service cloud-init-hotplugd.socket cloud-config.target cloud-init.target  multipathd.service multipathd.socket iscsid.socket  apparmor.service systemd-resolved.service ufw.service fwupd.service  graphical.target cron
6. 基本的文件系统优化,默认ulimit太低，可进行优化。
7. 升级系统到最新
8. 添加所需软件仓库（推荐使用自带版本即可）。ubuntu2204默认使用openstack Y版本。如果要安装更新版本，需要手动添加镜像仓库。

#### 安装rabbitmq-server
Openstack诸多组件需要使用rabbitmq-server实现消息队列交换。

#apt -y install rabbitmq-server
#vi /etc/rabbitmq/rabbitmq-env.conf
NODENAMErabbit@node01
NODE_IP_ADDRESS=0.0.0.0
NODE_PORT=5672
#systemctl restart rabbitmq-server
#systemctl enable rabbitmq-server
创建rabbimq-server用户openstack密码为Openstack!2024并授权
#rabbitmqctl add_user openstack 'Openstack!2024'
#rabbitmqctl set_permissions openstack ".*" ".*" ".*"

#rabbitmqctl list_users
会看到默认使用了guest为administrator，其默认密码为guest，可删除
#rabbitmqctl set_user_tags openstack administrator
设置openstack用户为administrator。用户通过AMQP方式登录时，不会有任何影响；但是如果通过其他方式，例如管理插件方式登录时，就可以去管理用户、vhost 和权限。
#rabbitmqctl delete_user guest
插件查看，默认插件都禁用。也可启用rabbitmq_management用Web方式对其进行管理。
#rabbitmq-plugins list
#rabbitmq-plugins enable rabbitmq_management
web管理端口为15672
通常为了节省资源不会开启。关闭使用如下命令：
#rabbitmq-plugins disable rabbitmq_management
生产环境中rabbitma-server使用集群方式。


#### 安装memcached
#apt -y install memcached python3-memcache
#vi /etc/memcached.conf
-l 0.0.0.0
#systemctl restart memcached
#systemctl enable memcached

#### 安装数据库
#apt -y install mariadb-server python3-pymysql
#vi /etc/mysql/mariadb.conf.d/50-server.cnf

[mysqld]
bind-address            =  0.0.0.0
default-storage-engine = innodb
max_connections = 4096
innodb_file_per_table = on
collation-server = utf8_general_ci
character-set-server = utf8
#systemctl restart mariadb
#systemctl enable mariadb
#mysql_secure_installation

### 1-keystone认证服务

#### 一、创建数据库

#mysql
create database keystone;
grant all privileges on keystone.* to keystone@'localhost' identified by 'Keystone!2024';
grant all privileges on keystone.* to keystone@'%' identified by 'Keystone!2024';
exit

生产环境禁止使用"%"允许所有访问数据库。如果客户端在同一地址段可缩小范围，例如"192.168.1.%"。

#### 二、安装软件

#apt -y install keystone python3-openstackclient apache2 libapache2-mod-wsgi-py3 python3-oauth2client

keystone组件默认配置文件为/etc/keystone/keystone.conf，其中大部分为注释和配置实例，基本为空。将原有配置文件备份，新创建配置文件。

#mv  /etc/keystone/keystone.conf /etc/keystone/keystone.conf.org
#grep -vE "^#|^#" /etc/keystone/keystone.conf.org

#vi  /etc/keystone/keystone.conf

[DEFAULT]
log_dir = /var/log/keystone
[cache]
memcache_servers = 192.168.1.101:11211
[database]
connection = mysql+pymysql://keystone:Keystone!2024@192.168.1.101/keystone
[token]
provider = fernet

查看，恢复文件权限

#ls -l /etc/keystone/
#chown keystone.keystone  /etc/keystone/keystone.conf
#chmod 640 /etc/keystone/keystone.conf

填充数据库，填充Identity服务数据库。

#su -s /bin/sh -c "keystone-manage db_sync" keystone
初始化Fernet密钥存储库
#keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
#keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

如果数据库访问无误，数据库里会填充数据表

#mysql -h 192.168.1.101 -u keystone -p
\> show databases;
\> use keystone;
\> show tables;
\> exit

引导身份服务，并给初始用户管理员admin设置密码,在此设置为了Keystone!2024。

#controller=192.168.1.101
#keystone-manage bootstrap --bootstrap-password 'Keystone!2024' \
  --bootstrap-admin-url http://#controller:5000/v3/ \
  --bootstrap-internal-url http://#controller:5000/v3/ \
  --bootstrap-public-url http://#controller:5000/v3/ \
  --bootstrap-region-id RegionOne

#### 三、配置Apache，启动服务

1. 配置“ServerName选项为该控制节点名称。
#vi /etc/apache2/apache2.conf
ServerNamenode01
#vi /etc/apache2/conf-enabled/security.conf
ServerTokens  Prod
2. 重启服务
#systemctl restart apache2
#systemctl enable apache2
#cat /etc/apache2/sites-enabled/keystone.conf
#netstat -an | grep 5000
#lsof | grep 5000


#### 四、验证keystone

1. 创建和使用openstack客户端环境脚本。

#vi \~/keystonerc

export OS_PROJECT_DOMAIN_NAMEdefault
export OS_USER_DOMAIN_NAMEdefault
export OS_PROJECT_NAMEadmin
export OS_USERNAMEadmin
export OS_PASSWORD='Keystone!2024'
export OS_AUTH_URL=http://192.168.1.101:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='[\u@\h \W(keystone)]\#'

#chmod 600 \~/keystonerc
#source \~/keystonerc
#echo "source \~/keystonerc " \>\> \~/.bash_profile
创建系统所需service项目
#openstack project create --domain default --description "Service Project" service
查看
#openstack project list

| ID                               | Name   |
| ---- | ---- |
| 763635551f3148e181449ef098cc0b28 | service |
| 7dce5080c565411a8b202f499288b130 | admin   |

#openstack service list

| ID                               | Name    | Type     |
| ---- | ---- | ---- |
| b7217ae3aca94748891314cdc889eddd | keystone | identity |

#openstack endpoint list

| ID                               | Region    | Service Name| Service Type | Enabled | Interface | URL                           |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 0a6c0af89e0d4c138cd83821f6bfd9a4 | RegionOne | keystone     | identity     | True    | admin     | http://192.168.1.101:5000/v3/ |
| 14412604f8b9449dbd30f37ce8f0e691 | RegionOne | keystone     | identity     | True    | public    | http://192.168.1.101:5000/v3/ |
| 60821f95b0ab4562b6f22163b1da6b7c | RegionOne | keystone     | identity     | True    | internal  | http://192.168.1.101:5000/v3/ |

#openstack user list

| ID                               | Name |
| ---- | ---- |
| a215e19211f44e70a9691f6500c926d5 | admin |


### 2-glance镜像服务
#### 一、创建glance数据库
#mysql -u root -p
 create database glance;
 grant all privileges on glance.* to glance@'localhost' identified by 'Glance!2024';
 grant all privileges on glance.* to glance@'%' identified by 'Glance!2024';
 flush privileges;
 exit

#### 二、创建用户和角色
创建glance用户。
 openstack user create --domain default --project service --password 'Glance!2024' glance
将glance用户添加到service项目和admin角色。
 openstack role add --project service --user glance admin
创建服务实体。
 openstack service create --nameglance --description "OpenStack Image" image
创建Image服务API端点。
安装在哪台服务器使用那台服务器的IP，生产环境建议单独安装，这样glance即可以提供镜像服务，还可以制作镜像，做控制器数据库备份等。
当前仍然使用控制节点服务器。
glance=192.168.1.101
openstack endpoint create --region RegionOne image public http://#glance:9292
openstack endpoint create --region RegionOne image internal http://#glance:9292
openstack endpoint create --region RegionOne image admin http://#glance:9292

#### 三、安装配置glance
#apt install glance -y
#mv /etc/glance/glance-api.conf /etc/glance/glance-api.conf.org

编辑配置文件内容如下：

#vi /etc/glance/glance-api.conf
```conf
[DEFAULT]
bind_host = 0.0.0.0
show_image_direct_url = True
[database]
connection = mysql+pymysql://glance:Glance!2024@192.168.1.101/glance

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

[keystone_authtoken]
www_authenticate_uri = http://192.168.1.101:5000
auth_url = http://192.168.1.101:5000
memcached_servers = 192.168.1.101:11211
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= glance
password = Glance!2024

[paste_deploy]
flavor = keystone
```

恢复文件权限
#ls -l /etc/glance/
#chmod 640 /etc/glance/glance-api.conf 
#chgrp glance /etc/glance/glance-api.conf 

填充数据库
#su -s /bin/sh -c "glance-manage db_sync" glance

启动服务并将其配置为在系统引导时启动
#systemctl restart glance-api
#systemctl enable glance-api

glance-api服务使用端口9292
#netstat -an | grep 9292


#### 四、验证glance
1、使用admin凭据
#source \~/keystonerc

2、下载镜像
#wget https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img


3、镜像格式转换
绝大部分镜像默认是qcow2格式，包括我们自己制作的镜像。
文件格式qcow2的比较省硬盘空间，计算节点从镜像服务器下载了镜像后还会将其转换为raw格式，如果glance硬盘空间足够，网络带宽足够，可直接在glance上将镜像保存为raw格式。
查看镜像文件格式可使用命令(使用qemu-img命令需要安装qemu-utils)：

#qemu-img info cirros-0.6.2-x86_64-disk.img
image: cirros-0.6.2-x86_64-disk.img
file format: qcow2
virtual size: 112 MiB (117440512 bytes)
disk size: 20.4 MiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false

转换镜像文件格式：
#qemu-img convert -f qcow2 -O raw cirros-0.6.2-x86_64-disk.img cirros-0.6.2-x86_64-disk.raw
#qemu-img info cirros-0.6.2-x86_64-disk.raw
image: cirros-0.6.2-x86_64-disk.raw
file format: raw
virtual size: 112 MiB (117440512 bytes)
disk size: 35 MiB

4、上传镜像到glance
#openstack image create "cirros-0.6.2" --file /root/cirros-0.6.2-x86_64-disk.raw  --disk-format raw --container-format bare --public
#openstack image list

| ID                       | Name     | Status |
| ---- | ---- | ---- |
| f0041c50-702e-4681-842c-f67ed1e27bcc | cirros-0.6.2 | active |

（设置hw_qemu_guest_agent属性，需要镜像支持，可用来修改镜像用户密码）
#glance image-update --propertyhw_qemu_guest_agent=yes f0041c50-702e-4681-842c-f67ed1e27bcc
或
#openstack image set --propertyhw_qemu_guest_agent=yes cirros-0.6.2
查看镜像详细信息
#openstack image show  cirros-0.6.2

镜像可从网络下载或自己制作。
Centos官网镜像下载：http://cloud.centos.org/centos/
ubuntu官网镜像下载：http://cloud-images.ubuntu.com/
debian官网镜像下载：https://cloud.debian.org/images/cloud/
制作Linux镜像需使用ISO安装盘镜像文件即可
制作window镜像除了使用ISO安装盘镜像文件之外，还需下载virtio驱动盘和cloud-init文件，分别为
https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/
https://cloudbase.it/cloudbase-init/

### 3-placement资源管理服务
在控制节点安装配置并验证Placement。
#### 一、创建Placement数据库
#mysql -u root -p
 create database placement;
 grant all privileges on placement.* to placement@'localhost' identified by 'Placement!2024';
 grant all privileges on placement.* to placement@'%' identified by 'Placement!2024';
 exit
#### 二、配置用户和端点
创建用户placement并设置密码。
 openstack user create --domain default --project service --password 'Placement!2024' placement
将用户placement添加为service项目下admin角色，并创建placement服务。
 openstack role add --project service --user placement admin
 openstack service create --nameplacement --description "Placement API" placement
创建Placement API服务端点。
 controller=192.168.1.101
 openstack endpoint create --region RegionOne placement public http://#controller:8778
 openstack endpoint create --region RegionOne placement internal http://#controller:8778
 openstack endpoint create --region RegionOne placement admin http://#controller:8778
 openstack endpoint list

#### 三、安装和配置组件
#apt  install -y placement-api

#mv /etc/placement/placement.conf /etc/placement/placement.conf.org
#vi /etc/placement/placement.conf

```conf
[DEFAULT]
debug = false
[api]
auth_strategy = keystone
[keystone_authtoken]
auth_url = http://192.168.1.101:5000/v3
memcached_servers = 192.168.1.101:11211
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= placement
password = Placement!2024
[placement_database]
connection = mysql+pymysql://placement:Placement!2024@192.168.1.101/placement
```

修改文件权限和组
 chmod 640 /etc/placement/placement.conf
 chgrp placement /etc/placement/placement.conf

填充Placement数据库。
#su -s /bin/sh -c "placement-manage db sync" placement
。
重启apache2服务。
#systemctl restart apache2
#cat /etc/apache2/sites-enabled/placement-api.conf

四、验证Placement
#placement-status upgrade check

| Upgrade Check Results                     |
| ---- |
| Check: Missing Root Provider IDs          |
| Result: Success                           |
| Details: None                             |
| ---- |
| Check: Incomplete Consumers               |
| Result: Success                           |
| Details: None                             |
| ---- |
| Check: Policy File JSON to YAML Migration |
| Result: Success                           |
| Details: None                             |


### 4-nova计算服务
#### 一、创建数据库并授权给用户nova
#mysql 
 create database nova;
 create database nova_api;
 create database nova_cell0;
 grant all privileges on nova.* to nova@'localhost' identified by 'Nova!2024';
 grant all privileges on nova.* to nova@'%' identified by 'Nova!2024';
 grant all privileges on nova_api.* to nova@'localhost' identified by 'Nova!2024';
 grant all privileges on nova_api.* to nova@'%' identified by 'Nova!2024';
 grant all privileges on nova_cell0.* to nova@'localhost' identified by 'Nova!2024';
 grant all privileges on nova_cell0.* to nova@'%' identified by 'Nova!2024';
 flush privileges;
 exit

#### 二、创建用户和角色

创建nova用户。
 openstack user create --domain default --project service --password 'Nova!2024' nova
将admin角色添加到nova用户。
 openstack role add --project service --user nova admin

创建nova实体。
 openstack service create --namenova --description "OpenStack Compute" compute

创建compute API服务端点。
 controller=192.168.1.101
 openstack endpoint create --region RegionOne compute public http://#controller:8774/v2.1/%\(tenant_id\)s
 openstack endpoint create --region RegionOne compute internal http://#controller:8774/v2.1/%\(tenant_id\)s
 openstack endpoint create --region RegionOne compute admin http://#controller:8774/v2.1/%\(tenant_id\)s

#### 三、安装和配置Nova（控制节点）
#apt -y install nova-api nova-conductor nova-scheduler nova-novncproxy python3-novaclient
#mv /etc/nova/nova.conf /etc/nova/nova.conf.org

#vi /etc/nova/nova.conf
```conf
[DEFAULT]
my_ip = 192.168.1.101
state_path = /var/lib/nova
enabled_apis = osapi_compute,metadata
log_dir = /var/log/nova
transport_url = rabbit://openstack:Openstack!2024@192.168.1.101

[api]
auth_strategy = keystone

[api_database]
connection = mysql+pymysql://nova:Nova!2024@192.168.1.101/nova_api

[database]
connection = mysql+pymysql://nova:Nova!2024@192.168.1.101/nova

[glance]
api_servers = http://192.168.1.101:9292

[keystone_authtoken]
www_authenticate_uri = http://192.168.1.101:5000
auth_url = http://192.168.1.101:5000
memcached_servers = 192.168.1.101:11211
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= nova
password = Nova!2024
[oslo_concurrency]
lock_path = #state_path/tmp

[placement]
auth_url = http://192.168.1.101:5000
os_region_name= RegionOne
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= placement
password = Placement!2024

[scheduler]
discover_hosts_in_cells_interval = 300

[wsgi]
api_paste_config = /etc/nova/api-paste.ini
```

恢复文件权限
 chmod 640 /etc/nova/nova.conf
 chgrp nova /etc/nova/nova.conf

填充数据库

#su -s /bin/sh -c "nova-manage api_db sync" nova
#su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
#su -s /bin/sh -c "nova-manage cell_v2 create_cell --namecell1 --verbose" nova
#su -s /bin/sh -c "nova-manage db sync" nova

验证cell0和cell1是否正确注册
#su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova

|  Name|                 UUID                 |             Transport URL             |                Database Connection                 | Disabled |
| ---- | ---- | ---- | ---- | ---- |
| cell0 | 00000000-0000-0000-0000-000000000000 |                 none:/                | mysql+pymysql://nova:****@192.168.1.101/nova_cell0 |  False   |
| cell1 | 3da346db-9fce-41ce-af94-7710ffb155a1 | rabbit://openstack:****@192.168.1.101 |    mysql+pymysql://nova:****@192.168.1.101/nova    |  False   |

启动nova服务
#systemctl restart nova-api nova-conductor nova-scheduler nova-novncproxy
#systemctl enable nova-api nova-conductor nova-scheduler nova-novncproxy

#openstack compute service list

| ID                                   | Binary         | Host   | Zone     | Status  | State | Updated At                 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| d7e1ea8a-7d55-4b04-8b85-062c9f8e3650 | nova-scheduler | node01 | internal | enabled | up    | 2024-04-27T10:25:49.000000 |
| 073db484-ef23-412e-b468-15a52b2f127a | nova-conductor | node01 | internal | enabled | up    | 2024-04-27T10:25:49.000000 |


#### 四、安装和配置Nova（计算节点）
控制节点也可以安装计算服务。（单节点实验直接在控制节点安装计算服务即可）

1、安装虚拟化软件
所需虚拟化组件会被自动安装。
确认当前节点支持KVM虚拟化。
root@node02:\~#lsmod | grep kvm
kvm_intel             372736  0
kvm                  1032192  1 kvm_intel

2、安装计算服务
安装openstack软件
#apt -y install nova-compute nova-compute-kvm
#mv /etc/nova/nova.conf /etc/nova/nova.conf.org
计算节点配置文件相比控制节点少了数据库连接信息多了vnc部分。控制节点要起计算服务只需安装nova-compute以及配置中添加vnc部分。

#vi /etc/nova/nova.conf
```conf
[DEFAULT]
my_ip = 192.168.1.102
state_path = /var/lib/nova
enabled_apis = osapi_compute,metadata
log_dir = /var/log/nova
transport_url = rabbit://openstack:Openstack!2024@192.168.1.101

[api]
auth_strategy = keystone

[glance]
api_servers = http://192.168.1.101:9292

[keystone_authtoken]
www_authenticate_uri = http://192.168.1.101:5000
auth_url = http://192.168.1.101:5000
memcached_servers = 192.168.1.101:11211
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= nova
password = Nova!2024
[oslo_concurrency]
lock_path = #state_path/tmp
[placement]
auth_url = http://192.168.1.101:5000
os_region_name= RegionOne
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= placement
password = Placement!2024

[scheduler]
discover_hosts_in_cells_interval = 300

[wsgi]
api_paste_config = /etc/nova/api-paste.ini

[vnc]                                                     
enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = #my_ip
novncproxy_base_url = http://192.168.1.101:6080/vnc_auto.html 
```
恢复文件权限和属组
 ls -l /etc/nova/
 chmod 640 /etc/nova/nova.conf
 chgrp nova /etc/nova/nova.conf
启动计算服务
 systemctl start libvirtd
 systemctl enable libvirtd
 systemctl restart nova-compute
 systemctl enable nova-compute


在控制节点查看计算节点情况，发现主机
#nova-manage cell_v2 discover_hosts --verbose
#openstack compute service list

| ID                                   | Binary         | Host   | Zone     | Status  | State | Updated At                 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| d7e1ea8a-7d55-4b04-8b85-062c9f8e3650 | nova-scheduler | node01 | internal | enabled | up    | 2024-04-27T10:38:19.000000 |
| 073db484-ef23-412e-b468-15a52b2f127a | nova-conductor | node01 | internal | enabled | up    | 2024-04-27T10:38:20.000000 |
| e476c840-b3c6-4a06-9667-24b89543c03d | nova-compute   | node02 | nova     | enabled | up    | 2024-04-27T10:38:26.000000 |


#openstack hypervisor list

| ID | Hypervisor Hostname| Hypervisor Type | Host IP       | State |
| ---- | ---- | ---- | ---- | ---- |
|  1 | node02              | QEMU            | 192.168.1.102 | up    |


添加新计算节点时，需要在控制器节点执行以下命令，以注册这些新计算节点。
nova-manage cell_v2 discover_hosts --verbose
或者，在以下位置设置适当的间隔 ，让控制节点周期性发现计算节点。
#vi /etc/nova/nova.conf
[scheduler]
discover_hosts_in_cells_interval = 300
所有计算节点配置相同，只有my_ip不同，所以但有多个结算节点时，在安装完软件后可将nova.conf拷贝后修改my_ip即可。
在安装了其他组件后还需将相关信息加入到配置文件中去。


### 5-neutron网络组件(Provider network)
Neutron组件的网络实现由Provider Network和Self-service Network两种类型：
Provider Network依赖于外部网络，即网关在外部三层交换机上。每个节点都需要一个连接外部网络的接口。
常见的有flat网络、vlan网络和vxlan网络三种。
flat网络上行接口连接交换机的access口，vlan网络连接交换机的trunk口，这都属于二层网络接入。
vxlan网络中各个节点使用三层接口接入vxlan leaf设备，使用vxlan可实现跨三层网络的二层通信。

Self-service Network需要网络中一个节点安装neutron-l3组件，用来创建路由器和提供Floating IP，这个节点有外部网络出口。所有其他计算节点的实例网络都使用隧道网络终结到网络节点创建的虚拟路由器，并使用Floating IP和内部IP做一对一映射实现对外部网络暴漏服务。

二层网络组件我们使用openvswitch，在openvswitch+dpdk技术下，实例网络基本能实现线速。

#### 一、创建Neutron数据库
#mysql -u root -p
 create database neutron;
 grant all privileges on neutron.* to neutron@'localhost' identified by 'Neutron!2024';
 grant all privileges on neutron.* to neutron@'%' identified by 'Neutron!2024';
 flush privileges;
 exit

#### 二、创建neutron用户和服务发布点
 openstack user create --domain default --project service --password 'Neutron!2024' neutron
 openstack role add --project service --user neutron admin
 openstack service create --nameneutron --description "OpenStack Networking service" network
 export network=192.168.1.101   
 openstack endpoint create --region RegionOne network public http://#network:9696
 openstack endpoint create --region RegionOne network internal http://#network:9696
 openstack endpoint create --region RegionOne network admin http://#network:9696

#### 三、控制节点安装
我们使用provider网络，只考虑控制节点和计算节点。由于neutron-dhcp-agent和neutron-metadata-agent也安装在控制节点，所以控制节点也需要安装二层组件以保持和所有计算节点上行网络互通。
#apt -y install neutron-server  neutron-metadata-agent neutron-plugin-ml2  neutron-dhcp-agent   python3-neutronclient  neutron-openvswitch-agent


网络准备工作
修改用于网络服务的网卡配置：ens3为管理以及openstack集群服务，ens8网络为provider网络网卡(虚拟机业务网络上联口)
网卡连接到交换机，flat网络时连接access口，vlan模式时连接trunk口。
实验环境均为flat网络。
#ip add
修改ubuntu网络配置文件，对provider网卡进行配置，因为是二层接口，网卡接口UP即可，不配置IP地址。
#vi /etc/netplan/50-cloud-init.yaml
network:
    version: 2
    ethernets:
        ens8:
            dhcp4: false
#netplan apply
启动openvswitch并设置开机启动
#systemctl restart openvswitch-switch
#systemctl enable openvswitch-switch
增加网桥br-int（如果不存在）
#ovs-vsctl add-br br-int
#ovs-vsctl show
创建名为br-eth的网桥，该网桥名称会在配置中使用到，所有虚拟网卡使用该网桥连接到物理网络。
#ovs-vsctl add-br br-eth
给该网桥增加上行接口ens8 （根据实际情况使用正确的网卡）
#ovs-vsctl add-port br-eth ens8


控制节点需要配置的文件：

1、neutron.conf文件
#mv /etc/neutron/neutron.conf /etc/neutron/neutron.conf.org

#vi /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
state_path = /var/lib/neutron
dhcp_agent_notification = True
allow_overlapping_ips = True
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
transport_url = rabbit://openstack:Openstack!2024@192.168.1.101

[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf

[database]
connection = mysql+pymysql://neutron:Neutron!2024@192.168.1.101/neutron


[keystone_authtoken]
www_authenticate_uri = http://192.168.1.101:5000
auth_url = http://192.168.1.101:5000
memcached_servers = 192.168.1.101:11211
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= neutron
password = Neutron!2024

[nova]
auth_url = http://192.168.1.101:5000
auth_type = password
project_domain_name= default
user_domain_name= default
region_name= RegionOne
project_name= service
username= nova
password = Nova!2024

[oslo_concurrency]
lock_path = #state_path/tmp


2、dhcp_agent.ini 文件
#mv /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.org
#vi /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true

3、metadata_agent.ini 文件
#mv /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.org
#vi /etc/neutron/metadata_agent.ini 
[DEFAULT]
nova_metadata_host = 192.168.1.101
metadata_proxy_shared_secret = metadata_secret!2024
[cache]
memcache_servers = 192.168.1.101:11211

4、ml2_conf.ini 文件
#mv /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.org
#vi /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types =
mechanism_drivers = openvswitch
extension_drivers = port_security
[ml2_type_flat]
flat_networks = physnet1
[ml2_type_vlan]
network_vlan_ranges = physnet1:1:1000

该配置文件中使用了openvswitch作为而成二层网络驱动并支持端口安全特性，逻辑网桥为physnet1，在后面的配置文件中physnet1对应虚拟网桥br-eth。配置了flat、vlan两种二层网络的支持。

5、openvswitch_agent.ini 文件
#mv /etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini.org
#vi  /etc/neutron/plugins/ml2/openvswitch_agent.ini
[DEFAULT]
[ovs]
bridge_mappings = physnet1:br-eth
[securitygroup]
firewall_driver = openvswitch
enable_security_group = true
enable_ipset = true



对新增文件修改权限和用户组
#chmod 640 /etc/neutron/neutron.conf  /etc/neutron/dhcp_agent.ini /etc/neutron/metadata_agent.ini  /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini 
#chgrp neutron /etc/neutron/neutron.conf  /etc/neutron/dhcp_agent.ini /etc/neutron/metadata_agent.ini  /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini 

6、在nova配置文件中增加如下配置：
#vi /etc/nova/nova.conf
[DEFAULT] 
use_neutron = True
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
vif_plugging_is_fatal = True
vif_plugging_timeout = 300
[neutron]
auth_url = http://192.168.1.101:5000
auth_type = password
project_domain_name= default
user_domain_name= default
region_name= RegionOne
project_name= service
username= neutron
password = Neutron!2024
service_metadata_proxy = True
metadata_proxy_shared_secret = metadata_secret!2024

创建链接
#ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

填充数据库
#su -s /bin/bash neutron -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugin.ini upgrade head"

启动并使能服务
#systemctl restart neutron-server neutron-metadata-agent neutron-dhcp-agent neutron-openvswitch-agent
#systemctl enable neutron-server neutron-metadata-agent neutron-dhcp-agent neutron-openvswitch-agent
#systemctl restart nova-api
#systemctl restart openstack-nova-api

查看网络服务
#openstack network agent list

| ID                                   | Agent Type         | Host   | Availability Zone | Alive | State | Binary                    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 15c0d7b3-959a-4f0f-a48c-c99be653030d | DHCP agent         | node01 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| abc40428-be84-4703-85c5-e40c0f9379d9 | Open vSwitch agent | node01 | None              | :-)   | UP    | neutron-openvswitch-agent |
| c1f8ec0a-625a-4dc0-89d2-29311098c5fc | Metadata agent     | node01 | None              | :-)   | UP    | neutron-metadata-agent    |


#### 四、计算节点安装
#apt -y install neutron-common neutron-plugin-ml2 neutron-openvswitch-agent

配置外部网络接口和网桥，外部网络使用网络接口为eth2
#vi /etc/netplan/50-cloud-init.yaml
network:
    version: 2
    ethernets:
        ens8:
            dhcp4: false
#netplan apply
启动openvswitch并设置开机启动
#systemctl restart openvswitch-switch
#systemctl enable openvswitch-switch
增加网桥br-int（如果不存在）
#ovs-vsctl add-br br-int
#ovs-vsctl show
创建名为br-eth的网桥，该网桥名称会在配置中使用到，所有虚拟网卡使用该网桥连接到物理网络。
#ovs-vsctl add-br br-eth
给该网桥增加上行接口ens8 （根据实际情况使用正确的网卡）
#ovs-vsctl add-port br-eth ens8


计算节点需要修改的文件

1、neutron.conf文件，和控制节点基本一样，没有数据库连接信息部分，而且所有计算节点一样，可直接scp使用，只需注意文件权限和属主即可。
#mv /etc/neutron/neutron.conf /etc/neutron/neutron.conf.org
#vi /etc/neutron/neutron.conf
[DEFAULT]
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
state_path = /var/lib/neutron
allow_overlapping_ips = True
transport_url = rabbit://openstack:Openstack!2024@192.168.1.101

[agent]         
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf     


[keystone_authtoken]
www_authenticate_uri = http://192.168.1.101:5000
auth_url = http://192.168.1.101:5000
memcached_servers = 192.168.1.101:11211
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= neutron
password = Neutron!2024

[oslo_concurrency]
lock_path = #state_path/lock


在Provider网络中，ml2_conf.org和openvswitch_agent.ini与控制节点一致，也可直接从控制节点拷贝。

2、ml2_conf.ini 文件（在provider网络中和控制节点基本一致，可直接远程拷贝）
#mv /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.org
#vi /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types =
mechanism_drivers = openvswitch
extension_drivers = port_security
[ml2_type_flat]
flat_networks = physnet1
[ml2_type_vlan]
network_vlan_ranges = physnet1:1:1000


3、openvswitch_agent.ini 文件（在provider网络中和控制节点基本一致，可直接远程拷贝）
#mv /etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini.org
#vi  /etc/neutron/plugins/ml2/openvswitch_agent.ini 
[DEFAULT]
[ovs]
bridge_mappings = physnet1:br-eth
[securitygroup]
firewall_driver = openvswitch
enable_security_group = true
enable_ipset = true

修改文件权限和属组
 chmod 640 /etc/neutron/neutron.conf /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini
 chgrp neutron /etc/neutron/neutron.conf /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini
创建链接文件
#ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

4、修改计算节点nova.conf文件，增加neutron相关配置
#vi /etc/nova/nova.conf
[DEFAULT] 
use_neutron = True
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
vif_plugging_is_fatal = True
vif_plugging_timeout = 300

[neutron]
auth_url = http://192.168.1.101:5000
auth_type = password
project_domain_name= default
user_domain_name= default
region_name= RegionOne
project_name= service
username= neutron
password = Neutron!2024
service_metadata_proxy = True
metadata_proxy_shared_secret = metadata_secret!2024



重启计算服务，启动网络相关服务
 systemctl restart neutron-openvswitch-agent
 systemctl enable neutron-openvswitch-agent
 systemctl restart nova-compute


到控制节点查看：
#openstack network agent list

| ID                                   | Agent Type         | Host   | Availability Zone | Alive | State | Binary                    |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 15c0d7b3-959a-4f0f-a48c-c99be653030d | DHCP agent         | node01 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| abc40428-be84-4703-85c5-e40c0f9379d9 | Open vSwitch agent | node01 | None              | :-)   | UP    | neutron-openvswitch-agent |
| ad9ad6db-b0f9-44b5-b787-861303cdce27 | Open vSwitch agent | node02 | None              | :-)   | UP    | neutron-openvswitch-agent |
| c1f8ec0a-625a-4dc0-89d2-29311098c5fc | Metadata agent     | node01 | None              | :-)   | UP    | neutron-metadata-agent    |


#### 五、provider网络(所有计算节点)
虚拟网桥br-eth用于连接所有虚拟机(实际是br-int进行连接的)，上行使用一个物理接口连接到物理交换机。
增加网桥命令
#ovs-vsctl add-br br-eth
虚拟网桥增加接口
#ovs-vsctl add-port br-eth ens8

在openstack中使用逻辑名称为physnet1的网桥，它映射到虚拟网桥br-eth
#vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
[ovs]
bridge_mappings = physnet1:br-eth1
二层网络插件配置中使用该逻辑网桥名physnet1，当前配置了flat和vlan网络支持。
#vi /etc/neutron/plugins/ml2/ml2_conf.ini
[ml2_type_flat]
flat_networks = physnet1
[ml2_type_vlan]
network_vlan_ranges = physnet1:100:1000
所有节点的逻辑网桥名称必须一致，虚拟网桥尽管只在本节点生效，为了方便统一配置名称也尽量一致。逻辑网桥的物理接口每个节点根据实际情况接入相同的网络即可。

#### 六、创建网络、子网、端口和安全组
1、创建网络
#openstack network create
usage: openstack network create [-h] [-f {json,shell,table,valueyaml}]
                                [-c COLUMN] [--max-width <integer\>]
                                [--fit-width] [--print-empty] [--noindent]
                                [--prefix PREFIX] [--share | --no-share]
                                [--enable | --disable] [--project <project\>]
                                [--description <description\>] [--mtu <mtu\>]
                                [--project-domain <project-domain\>]
                                [--availability-zone-hint <availability-zone\>]
                                [--enable-port-security | --disable-port-security]
                                [--external | --internal]
                                [--default | --no-default]
                                [--qos-policy <qos-policy\>]
                                [--transparent-vlan | --no-transparent-vlan]
                                [--provider-network-type <provider-network-type\>]
                                [--provider-physical-network <provider-physical-network\>]
                                [--provider-segment <provider-segment\>]
                                [--dns-domain <dns-domain\>]
                                [--tag <tag\> | --no-tag]
                                <name>
创建一个名为network1的网络，在provider网络中network只能由管理员创建，且要根据上联口ens8对端交换机配置和网络配置来创建。
#openstack network create --project service --share --external  \
  --provider-network-type flat --provider-physical-network physnet1 network1
其中
--project  指明为service项目
--share    表示共享，其他项目(用户)也能使用
--external 表示外部网络
--provider-network-type  网络类型，当前为flat类型，即上联交换机口为untagedd access接口
--provider-physical-network  指明使用的逻辑网桥为physnet1
其他参数
--enable-port-security 默认使能了端口安全
--provider-segment     当--provider-network-type类型为vlan时设置vlanid

#openstack network list

| ID                                   | Name    | Subnets |
| ---- | ---- | ---- |
| e596d11d-4ad3-48c0-a377-3302cea68b74 | network1 |         |

2、创建子网，Provider网络由外部网络决定。创建了名为subnent1的子网，网络地址段为192.168.3.0/24。
#openstack subnet create subnet1 --network network1 --subnet-range 192.168.3.0/24  \
  --allocation-pool start=192.168.3.100,end=192.168.3.120  \
  --gateway 192.168.3.1 --dns-nameerver  202.201.13.22

#openstack subnet list

| ID                                   | Name   | Network                              | Subnet         |
| ---- | ---- | ---- | ---- |
| 8533968b-535d-42c4-a780-3cb26f2bb080 | subnet1 | e596d11d-4ad3-48c0-a377-3302cea68b74 | 192.168.3.0/24 |

#openstack port list

| ID                                   | Name| MAC Address       | Fixed IP Addresses                                                           | Status |
| ---- | ---- | ---- | ---- | ---- |
| 43ffcb0e-612e-4a96-abee-d1bb82cb1e93 |      | fa:16:3e:19:de:a7 | ip_address='192.168.3.100', subnet_id='8533968b-535d-42c4-a780-3cb26f2bb080' | ACTIVE |

dhcp默认占用了一个接口地址，状态为ACTIVE，说明Neutron网络正常。如果上行网络互联没问题，此时可ping通该地址。

3、创建端口
在创建虚拟机时会自动创建端口，虚拟机网卡接口会从dhcp池中获取到一个地址。
dhcp获取地址是随机的，有时我们想用指定地址或连续的几个地址时就需要自己手动创建端口，在创建虚拟机时选择使用该端口即可。
#openstack port create
usage: openstack port create [-h] [-f {json,shell,table,valueyaml}]
                             [-c COLUMN] [--max-width <integer\>] [--fit-width]
                             [--print-empty] [--noindent] [--prefix PREFIX]
                             --network <network\> [--description <description\>]
                             [--device <device-id\>]
                             [--mac-address <mac-address\>]
                             [--device-owner <device-owner\>]
                             [--vnic-type <vnic-type\>] [--host <host-id\>]
                             [--dns-domain dns-domain] [--dns-name<dns-name>]
                             [--fixed-ip subnet=<subnet\>,ip-address=<ip-address\> | --no-fixed-ip]
                             [--binding-profile <binding-profile\>]
                             [--enable | --disable]
                             [--enable-uplink-status-propagation | --disable-uplink-status-propagation]
                             [--project <project\>]
                             [--project-domain <project-domain\>]
                             [--extra-dhcp-option name<name>[,value<value>,ip-version={4,6}]]
                             [--security-group <security-group\> | --no-security-group]
                             [--qos-policy <qos-policy\>]
                             [--enable-port-security | --disable-port-security]
                             [--allowed-address ip-address=<ip-address\>[,mac-address=<mac-address\>]]
                             [--tag <tag\> | --no-tag]
                             <name>
通过dhcp创建接口
#openstack port create --network network1 nic-1
#openstack port list

| ID                                   | Name | MAC Address       | Fixed IP Addresses                                                           | Status |
| ---- | ---- | ---- | ---- | ---- |
| 0ded3d06-4743-40f1-9ca3-e7fe0fceba94 |       | fa:16:3e:96:7a:d2 | ip_address='192.168.3.100', subnet_id='06ef2c2a-a1fe-49ac-841a-4868799ad07d' | ACTIVE |
| 21304e3d-393a-4984-9c35-41756d9b8e74 | nic-1 | fa:16:3e:49:1f:57 | ip_address='192.168.3.102', subnet_id='06ef2c2a-a1fe-49ac-841a-4868799ad07d' | DOWN   |

查看接口详情
#openstack port show nic-1

创建指定IP地址的端口
#openstack port create --network network1 --fixed-ip subnet=subnet1,ip-address=192.168.3.110  nic-2

默认启用了端口安全，可以创建没有启用端口安全的接口
#openstack port create --network network1 --fixed-ip subnet=subnet1,ip-address=192.168.3.111 --disable-port-security  nic-3

删除端口使用delete命令
#openstack port delete nic-1

4、安全组(security group)
安全组相当于每一个接口的防火墙，当启用端口安全时必须选择使用一个安全组。端口安全一方面限定用户只能使用指定的IP地址，另一方面根据安全组对端口的数据包进行过滤。
每一个项目都有一个default安全组，默认只允许向外发包。

#openstack security group list

| ID                                   | Name   | Description            | Project                          | Tags |
| ---- | ---- | ---- | ---- | ---- |
| e7e39d08-bce6-4371-ae49-bb3d4f65a11b | default | Default security group | 0f8257eba3184873be356a25e1b2fbef | []   |
| f79a3a96-d6d0-4b32-9fd5-ffabe2791177 | default | Default security group | dc5fe2241c7344a885f475449516b828 | []   |

#openstack project list


| ID                               | Name   |
| ---- | ---- |
| 0f8257eba3184873be356a25e1b2fbef | service |
| dc5fe2241c7344a885f475449516b828 | admin   |

创建一个名为web-default的安全组，安全规则为允许icmp ping、ssh、http和https
#openstack security group create web-default

设置安全规则
  openstack security group rule create --protocol icmp --icmp-type 8 --icmp-code 0  --ingress web-default
  openstack security group rule create --protocol tcp --dst-port 22:22 --ingress web-default
  openstack security group rule create --protocol tcp --dst-port 80:80 --ingress web-default
  openstack security group rule create --protocol tcp --dst-port 443:443 --ingress web-default

查看
#openstack security group list
#openstack security group show web-default

### 6-创建一个实例(虚拟机)

创建一个实例需要以下信息：
1、image，这需要我们上传到glance
#openstack image list

| ID                                   | Name        | Status |
|  ----  | ----  | ----  |
| f0041c50-702e-4681-842c-f67ed1e27bcc | cirros-0.6.2 | active |

2、实例类型，这需要管理员创建，指明cpu数、内存大小和磁盘大小，默认为public，即公用。还可以为某个用户创建特殊需求的实例类型。
创建一个测试实例类型
#openstack flavor create --id 0 --vcpus 1 --ram 1024 --disk 1 small1
#openstack flavor list
+----+--------+------+------+-----------+-------+-----------+
| ID | Name  |  RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+--------+------+------+-----------+-------+-----------+
| 0  | small1 | 1024 |    1 |         0 |     1 | True      |
+----+--------+------+------+-----------+-------+-----------+

3、网络或端口
#openstack network list

| ID                                   | Name    | Subnets                              |
|  ----  | ----  | ----  |
| e596d11d-4ad3-48c0-a377-3302cea68b74 | network1 | 8533968b-535d-42c4-a780-3cb26f2bb080 |

#openstack port list

| ID                                  | Name| MAC Address       | Fixed IP Addresses                                                           | Status |
|  ----  | ----  | ----  |  ----  | ----  |
| 43ffcb0e-612e-4a96-abee-d1bb82cb1e93 |      | fa:16:3e:19:de:a7 | ip_address='192.168.3.100', subnet_id='8533968b-535d-42c4-a780-3cb26f2bb080' | ACTIVE |

4、安全组

#openstack security group list

| ID                                   | Name       | Description            | Project                          | Tags |
|  ----  | ----  | ----  | ----  | ----  |
| 42b8dcb5-a168-4896-8b88-e0c38878c330 | default     | Default security group | 7dce5080c565411a8b202f499288b130 | []   |
| 4bb48b19-40ae-44d0-8719-277648b43e45 | web-default | web-default            | 7dce5080c565411a8b202f499288b130 | []   |
| f777d1d1-8c95-47be-bc94-720b7c1cf635 | default     | Default security group | 763635551f3148e181449ef098cc0b28 | []   |

5、密钥对keypair
keypair里只包含公钥，该公钥会被注入Linux的ssh认证文件中，客户端使用私钥访问虚拟机。我们使用当前已有密钥对的公钥。
创建密钥对
#openstack keypair create --public-key \~/.ssh/id_rsa.pub key1
#openstack keypair list

| Name| Fingerprint                                     | Type |
|  ----  | ----  | ----  |
| key1 | 91:72:a3:87:71:5b:84:e3:8f:23:5e:92:99:96:fe:30 | ssh  |



创建一个实例
#openstack server create --flavor small1 --image  cirros-0.6.2 \
--security-group web-default --nic net-id=network1 --key-namekey1 cirrostest1
#openstack server list

| ID                                   | Name       | Status | Networks | Image        | Flavor |
|  ----  | ----  | ----  |  ----  | ----  | ----  |
| b40bf9a2-a94c-4106-97b1-3661ef1af552 | cirrostest1 | BUILD  |          | cirros-0.6.2 | small1 |

#openstack server list


| ID                                   | Name       | Status | Networks               | Image        | Flavor |
|  ----  | ----  | ----  | ----  | ----  | ----  |
| b40bf9a2-a94c-4106-97b1-3661ef1af552 | cirrostest1 | ACTIVE | network1=192.168.3.112 | cirros-0.6.2 | small1 |

当前网络192.168.1.0/24和192.168.3.0/24通过路由器router直连可以互通。
#ping 192.168.3.112
PING 192.168.3.112 (192.168.3.112) 56(84) bytes of data.
64 bytes from 192.168.3.112: icmp_seq=1 ttl=63 time=2.58 ms

#ssh cirros@192.168.3.112
#ip add
1: lo: <LOOPBACK,UP,LOWER_UP\> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP\> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether fa:16:3e:7e:75:a1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.3.112/24 brd 192.168.3.255 scope global dynamic noprefixroute eth0
       valid_lft 86334sec preferred_lft 75534sec
    inet6 fe80::f816:3eff:fe7e:75a1/64 scope link
       valid_lft forever preferred_lft forever
#exit
Connection to 192.168.3.117 closed.

三层网络不互通或转发的可到虚拟机所在节点进入虚拟机验证到dhcp的连通性即可。


创建时VM状态为ERROR无法删除处理：先改变状态，再删除。查看日志修改配置重启服务直至解决问题。
#openstack server list

| ID                                   | Name       | Status | Networks | Image        | Flavor |
|  ----  | ----  | ----  |  ----  | ----  | ----  |
| ad16aee7-48bb-4799-b5fe-10c61bee6462 | cirrostest1 | ERROR  |          | cirros-0.6.2 | small1 |

#nova  reset-state --active  ad16aee7-48bb-4799-b5fe-10c61bee6462
#openstack server delete ad16aee7-48bb-4799-b5fe-10c61bee6462

### 7-cinder块存储服务

在没有cinder卷时，没法给vm添加硬盘。vm系统盘默认创建在物理机即使用本地存储，且随着vm的删除系统盘也会被删除。

#### 一、创建Cinder数据库
#mysql -u root -p
 create database cinder;
 grant all privileges on cinder.* to cinder@'localhost' identified by 'Cinder!2024';
 grant all privileges on cinder.* to cinder@'%' identified by 'Cinder!2024';
 flush privileges;
 exit

#### 二、创建用户和服务
openstack user create --domain default --project service --password 'Cinder!2024' cinder
openstack role add --project service --user cinder admin
openstack service create --namecinderv3 --description "OpenStack Block Storage" volumev3
export controller=192.168.1.101
openstack endpoint create --region RegionOne volumev3 public http://#controller:8776/v3/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev3 internal http://#controller:8776/v3/%\(tenant_id\)s
openstack endpoint create --region RegionOne volumev3 admin http://#controller:8776/v3/%\(tenant_id\)s

#### 三、安装配置（控制节点），控制节点安装启用中cinder-api和cinder-schdule，存储节点安装cinder-volume和cinder-backup。通常后端存储为ceph。
#apt -y install cinder-api cinder-scheduler python3-cinderclient
#mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.org
#vi /etc/cinder/cinder.conf
[DEFAULT]
my_ip = 192.168.1.101
rootwrap_config = /etc/cinder/rootwrap.conf     
api_paste_confg = /etc/cinder/api-paste.ini                  
state_path = /var/lib/cinder
auth_strategy = keystone
transport_url = rabbit://openstack:Openstack!2024@192.168.1.101
enable_v3_api = True

[database]
connection = mysql+pymysql://cinder:Cinder!2024@192.168.1.101/cinder

[keystone_authtoken]
www_authenticate_uri = http://192.168.1.101:5000
auth_url = http://192.168.1.101:5000
memcached_servers = 192.168.1.101:11211
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= cinder
password = Cinder!2024

[oslo_concurrency]
lock_path = #state_path/tmp

恢复文件权限
chmod 640 /etc/cinder/cinder.conf
chgrp cinder /etc/cinder/cinder.conf
初始化数据库
#su -s /bin/bash cinder -c "cinder-manage db sync"

启动服务
 systemctl restart cinder-scheduler
 systemctl enable cinder-scheduler



#echo "export OS_VOLUME_API_VERSION=3" \>\> \~/keystonerc
#source \~/keystonerc
#openstack volume service list

| Binary           | Host   | Zone | Status  | State | Updated At                 |
|  ----  | ----  | ----  | ----  | ----  | ----  |
| cinder-scheduler | node01 | nova | enabled | up    | 2024-04-27T13:02:29.000000 |


#### 四、安装配置（存储节点）
存储节点和计算节点使用同一传输协议，如iscsi、rbd、nfs等，即计算节点作为客户端应该也安装相应协议。
#apt -y install cinder-volume python3-mysqldb
#mv /etc/cinder/cinder.conf /etc/cinder/cinder.conf.org

基础配置和控制节点一致。当前仍然使用控制节点。
#vi /etc/cinder/cinder.conf
[DEFAULT]
my_ip = 192.168.1.101
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
state_path = /var/lib/cinder
auth_strategy = keystone
transport_url = rabbit://openstack:Openstack!2024@192.168.1.101
enable_v3_api = True
glance_api_servers = http://192.168.1.101:9292
enabled_backends =

[database]
connection = mysql+pymysql://cinder:Cinder!2024@192.168.1.101/cinder

[keystone_authtoken]
www_authenticate_uri = http://192.168.1.101:5000
auth_url = http://192.168.1.101:5000
memcached_servers = 192.168.1.101:11211
auth_type = password
project_domain_name= default
user_domain_name= default
project_name= service
username= cinder
password = Cinder!2024

[oslo_concurrency]
lock_path = #state_path/tmp

#vi /etc/cinder/cinder.conf 
#chown root.cinder /etc/cinder/cinder.conf
#chmod 640 /etc/cinder/cinder.conf
#systemctl enable cinder-volume
由于没有有效的存储后端，cinder-volume服务并不会正常运行。
cinder可使用的存储方式有多种。包括本地LVM、NFS共享存储、Ceph云存储等。

1、LVM卷，即使用iscsi协议将存储节点的lvm卷分配给计算节点的虚拟机挂载使用。
给node01添加一块硬盘，在系统中显示了/dev/vdc
创建物理卷
#lsblk
#pvcreate /dev/vdc
创建卷组，该vg名称在cinder配置中要使用，客户端申请的块存储，存储节点会自动使用该VG创建LV卷，以iSCSI协议提供服务。
#vgcreate cinder_data /dev/vdc
增加volume服务相关配置
#vi /etc/cinder/cinder.conf
[DEFAULT] 
enabled_backends = lvm
[lvm]
target_helper = lioadm
target_protocol = iscsi
#IP address of Storage Node
target_ip_address = 192.168.1.101
#volume group namejust created
volume_group = cinder_data
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volumes_dir = #state_path/volumes

启动服务
#systemctl restart cinder-volume.service

控制器端查看
#openstack volume service list

| Binary           | Host       | Zone | Status  | State | Updated At                 |
|  ----  | ----  | ----  | ----  | ----  | ----  |
| cinder-scheduler | node01     | nova | enabled | up    | 2024-04-27T13:11:39.000000 |
| cinder-volume    | node01@lvm | nova | enabled | up    | 2024-04-27T13:11:38.000000 |


当前版本还需添加如下配置，
控制节点添加service角色，并将nova和cinder赋予该角色
openstack role list
openstack role create service
openstack role add --user cinder --project service service
openstack role add --user nova --project service service

存储节点cinder.conf添加
[service_user]
send_service_user_token = True
auth_url = http://192.168.1.101:5000
project_domain_name= Default
project_name= service
user_domain_name= Default
auth_type = password
username= cinder
password = Cinder!2024

重启服务
#systemctl restart cinder-volume.service

计算节点nova.conf添加
[keystone_authtoken]
......
service_token_roles = service
service_token_roles_required = true

[service_user]
send_service_user_token = True
auth_url = http://192.168.1.101:5000
project_domain_name= Default
project_name= service
user_domain_name= Default
auth_type = password
username= cinder
password = Cinder!2024
[cinder]
os_region_name= RegionOne



重启服务
#systemctl restart nova-compute


测试：
创建卷
#openstack volume create --size 10 disk01
查看
#openstack volume list

| ID                                   | Name  | Status    | Size | Attached to |
|  ----  | ----  | ----  | ----  | ----  |
| 0c03364e-9de8-40f1-9e9e-1fb044a1576f | disk01 | available |   10 |             |

到存储节点查看
#vgdisplay
#lvdisplay

#### 五、计算节点配置
安装客户端协议，iscsi默认已经安装了。
apt -y install open-iscsi

配置nova
#vi /etc/nova/nova.conf
[cinder]
os_region_name= RegionOne
重启计算服务
#systemctl restart nova-compute.service

六、实例挂载块存储
实例挂载卷
#openstack server list

| ID                                   | Name       | Status | Networks               | Image        | Flavor |
|  ----  | ----  | ----  | ----  | ----  | ----  |
| b40bf9a2-a94c-4106-97b1-3661ef1af552 | cirrostest1 | ACTIVE | network1=192.168.3.112 | cirros-0.6.2 | small1 |

#openstack server add volume  cirrostest1 disk01
挂载后状态为in-use
#openstack volume list

| ID                                   | Name  | Status | Size | Attached to                          |
|  ----  | ----  | ----  | ----  | ----  |
| 0c03364e-9de8-40f1-9e9e-1fb044a1576f | disk01 | in-use |   10 | Attached to cirrostest1 on /dev/vdb  |

分离卷
#openstack server remove volume cirrostest1 disk01
#openstack volume list

| ID                                   | Name  | Status    | Size | Attached to |
|  ----  | ----  | ----  | ----  | ----  |
| 0c03364e-9de8-40f1-9e9e-1fb044a1576f | disk01 | available |   10 |             |

修改卷大小(原则上只能增加)
#openstack volume set --size 20 disk01
#openstack volume list

| ID                                   | Name  | Status    | Size | Attached to |
|  ----  | ----  | ----  | ----  | ----  |
| 12eca95c-5fde-4858-8f87-32e4f7f6885f | disk01 | available |   20 |             |

删除卷
#openstack volume delete disk01

使用cinder卷创建实例
#openstack server create --flavor small1 --image  cirros-0.6.2 --boot-from-volume 2 --security-group web-default --nic net-id=network1 --key-namekey1 cirrostest2

#openstack server list

| ID                                   | Name       | Status | Networks               | Image                    | Flavor |
|  ----  | ----  | ----  | ----  | ----  | ----  |
| 96d6d79e-8a87-412b-93df-06d67d19e3ea | cirrostest2 | BUILD  |                        | N/A (booted from volume) | small1 |
| b40bf9a2-a94c-4106-97b1-3661ef1af552 | cirrostest1 | ACTIVE | network1=192.168.3.112 | cirros-0.6.2             | small1 |

#openstack server list

| ID                                   | Name       | Status | Networks               | Image                    | Flavor |
|  ----  | ----  | ----  | ----  | ----  | ----  |
| 96d6d79e-8a87-412b-93df-06d67d19e3ea | cirrostest2 | ACTIVE | network1=192.168.3.115 | N/A (booted from volume) | small1 |
| b40bf9a2-a94c-4106-97b1-3661ef1af552 | cirrostest1 | ACTIVE | network1=192.168.3.112 | cirros-0.6.2             | small1 |

#openstack volume list

| ID                                   | Name| Status | Size | Attached to                          |
|  ----  | ----  | ----  | ----  | ----  |
| d7d583c1-812e-444d-a048-698399482d55 |      | in-use |    2 | Attached to cirrostest2 on /dev/vda  |

#openstack server delete  cirrostest2
#openstack volume list

| ID                                   | Name| Status    | Size | Attached to |
|  ----  | ----  | ----  | ----  | ----  |
| d7d583c1-812e-444d-a048-698399482d55 |      | available |    2 |             |



### 8-dashboard管理界面
openstack主要所需服务都已安装，基本具备使用条件，为了方便的进行配置，我们安装管理界面。云计算核心计算、网络、存储条件都已具备。

管理界面可以安装在任意一台机器上，如安装在控制节点，或单独拿一台机器来安装，只要和openstack服务器网络可达即可。
基本环境安装“安装前准备工作”部署即可。

#apt -y install openstack-dashboard
#mv /etc/openstack-dashboard/local_settings.py /etc/openstack-dashboard/local_settings.py.org
#grep -Ev '^#|#' /etc/openstack-dashboard/local_settings.py.org \> /etc/openstack-dashboard/local_settings.py
#chown root:horizon /etc/openstack-dashboard/local_settings.py
#chmod 640 /etc/openstack-dashboard/local_settings.py
#vi /etc/openstack-dashboard/local_settings.py
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '127.0.0.1'
    },
}
OPENSTACK_HOST = "127.0.0.1"
OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST #出错%s后加:5000
TIME_ZONE = "Asia/Shanghai "

增加
SESSION_ENGINE = "django.contrib.sessions.backends.cache" #出错改为.file
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = False
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'


#systemctl restart apache2
在浏览器中打开http://ip/horizon/

### 9-集群增加新计算节点和冷热迁移

从控制节点可以看到，当前只有node02一个计算节点。
#openstack compute service list

| ID                                   | Binary         | Host   | Zone     | Status  | State | Updated At                 |
|  ----  | ----  | ----  | ----  | ----  | ----  | ----  |
| d7e1ea8a-7d55-4b04-8b85-062c9f8e3650 | nova-scheduler | node01 | internal | enabled | up    | 2024-04-28T01:05:36.000000 |
| 073db484-ef23-412e-b468-15a52b2f127a | nova-conductor | node01 | internal | enabled | up    | 2024-04-28T01:05:41.000000 |
| e476c840-b3c6-4a06-9667-24b89543c03d | nova-compute   | node02 | nova     | enabled | up    | 2024-04-28T01:05:34.000000 |

#openstack hypervisor list

| ID | Hypervisor Hostname| Hypervisor Type | Host IP       | State |
|  ----  | ----  | ----  | ----  | ----  |
|  1 | node02              | QEMU            | 192.168.1.102 | up    |

#openstack network agent list

| ID                                   | Agent Type         | Host   | Availability Zone | Alive | State | Binary                    |
|  ----  | ----  | ----  | ----  | ----  | ----  | ----  |
| 15c0d7b3-959a-4f0f-a48c-c99be653030d | DHCP agent         | node01 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| abc40428-be84-4703-85c5-e40c0f9379d9 | Open vSwitch agent | node01 | None              | :-)   | UP    | neutron-openvswitch-agent |
| ad9ad6db-b0f9-44b5-b787-861303cdce27 | Open vSwitch agent | node02 | None              | :-)   | UP    | neutron-openvswitch-agent |
| c1f8ec0a-625a-4dc0-89d2-29311098c5fc | Metadata agent     | node01 | None              | :-)   | UP    | neutron-metadata-agent    |


准备一台和node02配置一致的节点node03，以完成基本配置。
node03安装nova-copute和neutron-openvswitch-agent
#apt -y install nova-compute nova-compute-kvm
#apt -y install neutron-common neutron-plugin-ml2 neutron-openvswitch-agent
计算节点只需将其他计算节点配置nova.conf拷贝过来修改myip即可。
#scp node02:/etc/nova/nova.conf  /etc/nova/
#vi /etc/nova/nova.conf 
[DEFAULT]
my_ip = 192.168.1.103

#systemctl restart nova-compute
#systemctl enable nova-compute
#systemctl enable libvirtd

计算节点neutron相关配置所有节点一样(provider网络模式)，直接拷贝过来即可。
#scp node02:/etc/neutron/neutron.conf /etc/neutron/
#scp node02:/etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/
#scp node02:/etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/
创建链接文件
#ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

创建虚拟网桥br-eth，并增加上联口
#ovs-vsctl add-br br-int
#ovs-vsctl show
#ovs-vsctl add-br br-eth
#ovs-vsctl add-port br-eth ens8
确认上联接口配置，接口开机UP没有配置地址当作二层接口。
#vi /etc/netplan/50-cloud-init.yaml
network:
    version: 2
    ethernets:
        ens8:
            dhcp4: false
#netplan apply
#systemctl restart neutron-openvswitch-agent
#systemctl enable neutron-openvswitch-agen

到控制节点查看：
#openstack compute service list

| ID                                   | Binary         | Host   | Zone     | Status  | State | Updated At                 |
|  ----  | ----  | ----  | ----  | ----  | ----  | ----  |
| d7e1ea8a-7d55-4b04-8b85-062c9f8e3650 | nova-scheduler | node01 | internal | enabled | up    | 2024-04-28T01:26:56.000000 |
| 073db484-ef23-412e-b468-15a52b2f127a | nova-conductor | node01 | internal | enabled | up    | 2024-04-28T01:26:51.000000 |
| e476c840-b3c6-4a06-9667-24b89543c03d | nova-compute   | node02 | nova     | enabled | up    | 2024-04-28T01:26:54.000000 |
| a85c6bdc-fae8-4f87-a1e1-048c79d10bbf | nova-compute   | node03 | nova     | enabled | up    | 2024-04-28T01:26:57.000000 |

#openstack hypervisor list

| ID | Hypervisor Hostname| Hypervisor Type | Host IP       | State |
|  ----  | ----  | ----  | ----  | ----  |
|  1 | node02              | QEMU            | 192.168.1.102 | up    |
|  2 | node03              | QEMU            | 192.168.1.103 | up    |

#openstack network agent list

| ID                                   | Agent Type         | Host   | Availability Zone | Alive | State | Binary                    |
|  ----  | ----  | ----  | ----  | ----  | ----  | ----  |
| 15c0d7b3-959a-4f0f-a48c-c99be653030d | DHCP agent         | node01 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 848bf25c-071b-4763-a15c-4b0b26b4c656 | Open vSwitch agent | node03 | None              | :-)   | UP    | neutron-openvswitch-agent |
| abc40428-be84-4703-85c5-e40c0f9379d9 | Open vSwitch agent | node01 | None              | :-)   | UP    | neutron-openvswitch-agent |
| ad9ad6db-b0f9-44b5-b787-861303cdce27 | Open vSwitch agent | node02 | None              | :-)   | UP    | neutron-openvswitch-agent |
| c1f8ec0a-625a-4dc0-89d2-29311098c5fc | Metadata agent     | node01 | None              | :-)   | UP    | neutron-metadata-agent    |


可见openstack集群进入扩展是非常容易的。
再对计算节点做nova冷热迁移配置即可。
冷迁移只需做计算节点间nova用户ssh免密认证即可。

在计算节点完成：
在node02上完成nova用户ssh免密认证
#usermod -s /bin/bash nova
#su - nova
#ssh-keygen -t rsa -P '' -f \~/.ssh/id_rsa
#cat \~/.ssh/id_rsa.pub \>\> \~/.ssh/authorized_keys
#chmod 0600 \~/.ssh/authorized_keys
#vi \~/.ssh/config
Host *
    StrictHostKeyChecking no
#chmod 600 \~/.ssh/config
验证
#ssh node02
#exit
#exit

其他计算节点从完成ssh免密认证的节点远程拷贝配置，并修改文件属主和属组
#ssh node03
#scp -r node02:/var/lib/nova/.ssh /var/lib/nova/
#chown -R nova.nova /var/lib/nova/.ssh/
验证
#su - nova
#ssh node03
#exit
#exit
给所有节点nova.conf配置文件[DEFAULT]下添加冷迁移配置并重启服务
#vi /etc/nova/nova.conf
[DEFAULT]
resume_guests_state_on_host_boot = true
allow_resize_to_same_host=True
enabled_filters = AvailabilityZoneFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter
控制节点重启服务
#systemctl restart nova-api nova-conductor nova-scheduler nova-novncproxy
计算节点重启服务
#systemctl restart nova-compute

完成冷迁移后可对实例关机进行更改flavor和节点迁移。

热迁移需要的对计算节点libvirt进行配置，并在nova.conf中添加配置。并需要共享存储，通常使用ceph集群作为后端存储。实验课还可以使用nfs进行验证。
#vi /etc/libvirt/libvirtd.conf
listen_tls = 0
listen_tcp = 1
tcp_port = "16509"
tls_port = "16514"
listen_addr = "0.0.0.0"
auth_tcp = "none"
#systemctl stop libvirtd
#systemctl disable libvirtd
#systemctl start libvirtd-tcp.socket
#systemctl enable libvirtd-tcp.socket
当前使用了tcp方式，没有认证，这种方式仅供实验验证热迁移。
生产环境使用TLC方式，需要配置很多证书，比较繁琐。

拷贝到其他节点并登录到节点重启服务。
#scp /etc/libvirt/libvirtd.conf node03:/etc/libvirt/
 ssh node03
 systemctl stop libvirtd
 systemctl disable libvirtd
 systemctl start libvirtd-tcp.socket
 systemctl enable libvirtd-tcp.socket

所有计算节点添加支持热迁移配置并重启nova计算服务
#vi /etc/nova/nova.conf
[libvirt]
live_migration_flag="VIR_MIGRATE_UNDEFINE_SOURCE,VIR_MIGRATE_PEER2PEER,VIR_MIGRATE_LIVE,VIR_MIGRATE_PERSIST_DEST,VIR_MIGRATE_TUNNELLED"
#systemctl restart nova-compute



nfs-server安装
#apt -y install nfs-kernel-server
#vi /etc/idmapd.conf
Domain = openstack.\<your_domain>
#mkdir /var/nfsshare
#vi /etc/exports
/var/nfsshare 192.168.1.0/24(rw,insecure,no_subtree_check,no_root_squash,crossmnt)
#systemctl restart nfs-server

计算节点作为nfs客户端，安装客户端工具
#apt -y install nfs-common
#vi /etc/idmapd.conf
Domain = openstack.\<your_domain>
计算节点挂载共享目录到虚拟机目录
#mount -t nfs 192.168.1.101:/var/nfsshare /var/lib/nova/instances/
#chown -R nova.nova /var/lib/nova/instances/

在控制节点生成一个新虚拟机
#openstack server create --flavor small1 --image  cirros-0.6.2  --security-group web-default --nic net-id=network1 --key-namekey1 cirroslive
#openstack server list

| ID                                   | Name      | Status | Networks               | Image        | Flavor |
|  ----  | ----  | ----  | ----  | ----  | ----  |
| 6adc326d-461d-449b-a62a-433d23c73210 | cirroslive | ACTIVE | network1=192.168.3.108 | cirros-0.6.2 | small1 |

#openstack server show  cirroslive
......
| OS-EXT-SRV-ATTR:host                | node03                                                   |
| OS-EXT-SRV-ATTR:hypervisor_hostname| node03                                                   |
......
实施热迁移
#openstack server migrate --live cirroslive
#openstack server list

| ID                                   | Name      | Status | Networks               | Image        | Flavor |
|  ----  | ----  | ----  | ----  | ----  | ----  |
| 6adc326d-461d-449b-a62a-433d23c73210 | cirroslive | ACTIVE | network1=192.168.3.108 | cirros-0.6.2 | small1 |

#openstack server show cirroslive
| OS-EXT-SRV-ATTR:host                | node02                                                   |
| OS-EXT-SRV-ATTR:hypervisor_hostname| node02                                                   |
已成功迁移到了node02节点
当有多个节点还可以指定迁移到一个节点
当前测试命令行报错，Web控制台可以。
#openstack server migrate --live  node03 cirroslive
usage: openstack server migrate [-h] [--live-migration] [--host <hostname>] [--shared-migration | --block-migration]
                                [--disk-overcommit | --no-disk-overcommit] [--wait]
                                <server\>
openstack server migrate: error: unrecognized arguments: cirroslive

#openstack server migrate --live  --host node03 cirroslive
#openstack server migrate --live-migration  --host node03 cirroslive
--os-compute-api-version 2.30 or greater is required when using --host
当前为v2.1

### 10-Self-service Network using Vxlan
自服务网络(Self-servcie Network)中需要一个节点做为Network节点，用户自己创建网络，网关终结到创建的虚拟路由器上，路由器使用外部网络做SNAT访问外网，内部服务使用浮动IP通过一对一映射实现被外网访问。而网络节点到计算节点使用隧道协议实现私有网络，通常为Vxlan。
实验中我们的控制节点也用于网络节点。
Self-servcie Network中计算节点和网络节点之间使用三层网络连接，只有网络节点通过上行网络接口连接到物理网络。
在当前控制节点node01安装neutron-l3-agent
#apt install -y neutron-l3-agent
#mv /etc/neutron/l3_agent.ini /etc/neutron/l3_agent.ini.org
#vi /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = openvswitch
[agent]
extensions = port_forwarding

#chmod 640 /etc/neutron/l3_agent.ini
#chgrp neutron /etc/neutron/l3_agent.ini

#vi /etc/neutron/neutron.conf
[DEFAULT]
service_plugins = router,segments,port_forwarding


#vi /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch
extension_drivers = port_security
[ml2_type_flat]
flat_networks = physnet1
[ml2_type_vlan]
network_vlan_ranges = physnet1:1:4094
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
enable_ipset = true

#vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
[DEFAULT]
[ovs]
local_ip = 192.168.1.101
tunnel_bridge = br-tun
bridge_mappings = physnet1:br-eth
[securitygroup]
firewall_driver = openvswitch
enable_security_group = true
enable_ipset = true
[agent]
tunnel_types = vxlan
prevent_arp_spoofing = True

控制网络节点：
#systemctl restart neutron-l3-agent
#systemctl enable neutron-l3-agent
#systemctl restart  neutron-server neutron-openvswitch-agent
#ip add
会看到一个br-tun的隧道接口
: br-tun: <BROADCAST,MULTICAST\> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 16:aa:47:48:88:49 brd ff:ff:ff:ff:ff:ff

#openstack network agent list

| ID                                   | Agent Type         | Host   | Availability Zone | Alive | State | Binary                    |
|  ----  | ----  | ----  | ----  | ----  | ----  | ----  |
| 15c0d7b3-959a-4f0f-a48c-c99be653030d | DHCP agent         | node01 | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 848bf25c-071b-4763-a15c-4b0b26b4c656 | Open vSwitch agent | node03 | None              | :-)   | UP    | neutron-openvswitch-agent |
| 8b230da7-6225-4b1c-a6f0-2ee8369dc397 | L3 agent           | node01 | nova              | :-)   | UP    | neutron-l3-agent          |
| abc40428-be84-4703-85c5-e40c0f9379d9 | Open vSwitch agent | node01 | None              | :-)   | UP    | neutron-openvswitch-agent |
| ad9ad6db-b0f9-44b5-b787-861303cdce27 | Open vSwitch agent | node02 | None              | :-)   | UP    | neutron-openvswitch-agent |
| c1f8ec0a-625a-4dc0-89d2-29311098c5fc | Metadata agent     | node01 | None              | :-)   | UP    | neutron-metadata-agent    |


发现到了L3 agent。

计算节点也按此配置，openvswitch_agent.ini中修改local_ip为vxlan所用接口地址即可。
#vi /etc/neutron/neutron.conf
[DEFAULT]
service_plugins = router,segments,port_forwarding

#vi /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch
extension_drivers = port_security
[ml2_type_flat]
flat_networks = physnet1
[ml2_type_vlan]
network_vlan_ranges = physnet1:1:4094
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
enable_ipset = true

#vi /etc/neutron/plugins/ml2/openvswitch_agent.ini
[DEFAULT]
[ovs]
local_ip = 192.168.1.102
tunnel_bridge = br-tun
bridge_mappings = physnet1:br-eth
[securitygroup]
firewall_driver = openvswitch
enable_security_group = true
enable_ipset = true
[agent]
tunnel_types = vxlan
prevent_arp_spoofing = True

或从node01远程复制过来，修改openvswitch_agent.ini配置文件中local_ip即可。
#scp node01:/etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/
#scp node01:/etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/
#vi /etc/neutron/plugins/ml2/openvswitch_agent.ini

重启服务
#systemctl restart neutron-openvswitch-agent
所有节点节点均需配置。




到控制节点验证(web界面可能会更直观)
创建路由器
#openstack router create router01
#openstack router list

| ID                                   | Name    | Status | State | Project                          | Distributed | HA    |
|  ----  | ----  | ----  | ----  | ----  | ----  | ----  |
| c5157bbd-49e4-4955-9d46-f93fe9e59a78 | router01 | ACTIVE | UP    | 7dce5080c565411a8b202f499288b130 | False       | False |

添加外部网络上行接口（外部网络由管理员创建）
#openstack network list
#openstack network show  network1
| router:external           | External                             |
| shared                    | True                                 |
| status                    | ACTIVE                               |
路由器上行接口和浮动IP只能使用External网络
#openstack router set router01 --external-gateway  network1
#openstack router show  router01
router接口会从network1获取到一个IP地址，并默认启用了SNAT
路由器router提供网络互联和SNAT以及一对一NAT。

创建一个用户自己的网络（创建的网络类型为vxlan，mls2文件中已经指定），这和公有云的VPC是一样的。
#openstack network create inside-network1
#openstack network show inside-network1

修改MTU值，默认为1450。（当前实验虚拟机MTU为1450，再减50设置为1400）
VXLAN封装的50字节（20字节IP头+8字节UDP头 + 8字节VXLAN头 + 14字节MAC头）
#openstack network set --mtu 1400 inside-network1

创建子网（网关会被自动排除到dhcp池外）
#openstack subnet create inside-subnet1 --network  inside-network1 \
--subnet-range 192.168.100.0/24 --gateway 192.168.100.1 \
--dns-nameerver 202.201.13.22
#openstack subnet show inside-subnet1 

添加内部网络到路由器
#openstack router add subnet router01 inside-subnet1

使用当前创建的网络可以用来创建虚拟机，但是虚拟机只能通过router访问到外部网络，不能被外部网络访问到，这需要创建浮动IP进行一对一NAT映射。

使用外部网络创建浮动IP
#openstack floating ip create network1

查看浮动IP
#openstack floating ip list

创建一个使用自建网络的虚拟机
#openstack server create --flavor small1 --image  cirros-0.6.2 --security-group web-default --nic net-id=inside-network1 --key-namekey1 cirrostest2

给虚拟机添加浮动IP
#openstack server add floating ip cirrostest2 192.168.3.113
#openstack floating ip list
通过floating ip访问虚拟机
#ping 192.168.3.113
PING 192.168.3.113 (192.168.3.113) 56(84) bytes of data.
64 bytes from 192.168.3.113: icmp_seq=1 ttl=62 time=5.86 ms
#ssh cirros@192.168.3.113
Warning: Permanently added '192.168.3.113' (ED25519) to the list of known hosts.
#ip add
1: lo: <LOOPBACK,UP,LOWER_UP\> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP\> mtu 1400 qdisc pfifo_fast qlen 1000
    link/ether fa:16:3e:26:a8:16 brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.140/24 brd 192.168.100.255 scope global dynamic noprefixroute eth0
       valid_lft 86329sec preferred_lft 75529sec
    inet6 fe80::f816:3eff:fe26:a816/64 scope link
       valid_lft forever preferred_lft forever
#exit

大型数据中心云计算平台都使用vxlan网络封装提供自服务网络模式，这样vm可实现跨数据中心的迁移。
如果不跨越公有网络，网络均可自行管理，设置合理的mtu可提高网络效率。
建议在部署前就对MTU进行全局规划，有如下建议，其中推荐第一条。

方式一（推荐）：修改应用层服务器发送报文的长度值，修改后的长度值加上VXLAN封装的50字节后，需保证在整个承载网中，均小于设备的MTU值。使用此方法，修改难度低，需要IT侧配合。
方式二：修改承载网中每一跳网络设备的MTU值，需保证MTU值大于收到的VXLAN报文长度，从而保证不分片（建议承载网中设备设置的MTU值需大于核心交换机的默认JUMBO的大小，即9216字节）。此方式常常受到约束：承载网络中设备众多、分布广泛，且涉及不同厂商，修改难度大；常常没有修改权限（他人资产，不可控）

## install docker-ce on ubuntu2204
1.Add Docker's official GPG key:
#sudo apt-get update
#sudo apt-get install ca-certificates curl
#sudo install -m 0755 -d /etc/apt/keyrings
#sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
#sudo chmod a+r /etc/apt/keyrings/docker.asc

2.Add the repository to Apt sources:

#echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
#sudo apt-get update

3.List the available versions:
#apt-cache madison docker-ce | awk '{ print $3 }'
5:26.0.0-1~ubuntu.22.04~jammy
5:25.0.5-1~ubuntu.22.04~jammy
5:24.0.9-1~ubuntu.22.04~jammy
5:23.0.6-1~ubuntu.22.04~jammy
5:20.10.24~3-0~ubuntu-jammy
...

Install the specific version Docker packages
#VERSION_STRING=5:20.10.24~3-0~ubuntu-jammy
#sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin

Lock the version
#dpkg -l  | grep docker
#apt list --installed | grep docker
#apt-mark hold docker-* 
#apt-mark showhold
#apt-mark docker-*


查看docker-ce版本和信息
#docker version
......
 Version:           20.10.24
......
#docker info

创建docker配置文件，增加部分配置(docker配置各版本略有差异,当前版本为20.10.24)
#vi /etc/docker/daemon.json
{
  "oom-score-adjust": -1000,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 10,
  "bip": "169.254.123.1/24",
  "dns": ["202.201.0.133"],
  "data-root": "/var/lib/docker",
  "insecure-registries": ["https://yourdomain"],
  "registry-mirrors": ["https://jwvm07bp.mirror.aliyuncs.com"],
  "storage-driver": "overlay2",
  "storage-opts": ["overlay2.override_kernel_check=true"]
}

重启服务
#systemctl daemon-reload
#systemctl restart docker

在数据中心和实验环境还可添加代理，以解决访问互联网问题，加快访问速度和镜像下载速度
#mkdir /etc/systemd/system/docker.service.d
#cd /etc/systemd/system/docker.service.d/
#vi http_proxy.conf
[Service]
Environment="HTTP_PROXY=http://noc:8912027@yourdomain:8080"
Environment="HTTPS_PROXY=http://noc:8912027@yourdomain:8080"
Environment="NO_PROXY=localhost,127.0.0.1,*.yourdomain"
#systemctl daemon-reload
#systemctl restart docker

查看
#docker info
......
 HTTP Proxy: http://xxxxx:xxxxx@yourdomain:8080
 HTTPS Proxy: http://xxxxx:xxxxx@yourdomain:8080
......

下载镜像
#docker image pull mysql:8
8: Pulling from library/mysql
........

Docker官方镜像仓库网址为https://hub-stage.docker.com/，从官方下载镜像的tag信息可从页面去查看，尽量使用Docker official image以保证镜像安全。

我们通过搭建一个wordpress网站来掌握使用docker以及制作镜像的基本步骤。
mysql数据库我们使用官方镜像直接生成数据库容器。

创建一个数据库数据持久化目录，用于映射mysql数据库默认存储目录/var/lib/mysql
#mkdir -p /opt/mysql8_data

使用镜像生成容器，
#docker run --name mysql8 -v /opt/mysql8_data:/var/lib/mysql -p 33061:3306 -e MYSQL_ROOT_PASSWORD=MYmysql2024 -d --restart=always mysql:8

使用host主机33061映射mysql数据库默认端口3306，设置root密码为MYmysql2024
-d为后台运行，--restart=always为启动策略随着docker启动而启动，镜像为mysql:8

创建容器后查看
#docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                                                    NAMES
486baf4b8bd5   mysql:8   "docker-entrypoint.s…"   14 seconds ago   Up 13 seconds   33060/tcp, 0.0.0.0:33061->3306/tcp, :::33061->3306/tcp   mysql8

查看宿主机监听端口
#netstat -an | grep 33061
tcp        0      0 0.0.0.0:33061           0.0.0.0:*               LISTEN
tcp6       0      0 :::33061                :::*                    LISTEN

如果宿主机安装了mysql客户端，可直接连接
#mysql -h 127.0.0.1 -P 33061 -u root -p

或使用docker exec运行bash进入到容器
#docker exec -ti mysql8 bash
root@486baf4b8bd5:/#

使用容器内自带的客户端进入数据库操作界面，创建数据库和用户并授权
root@da8286481caa:/# mysql -p
mysql> create database wordpress;
mysql> create user wordpress@'%' identified by 'Wordpress!2024';
mysql> grant all privileges on wordpress.* to wordpress@'%';
mysql> exit
root@da8286481caa:/#
按Ctr+P和Ctr+Q退出容器返回到宿主机。

我们使用一个RockyLinux8.9镜像来安装apache+php环境，用于wordpress运行环境
#docker pull rockylinux/rockylinux:8.9
#docker run -ti --name rockylinux8.9  -p 80:80 --restart=always rockylinux/rockylinux:8.9

在容器里给dnf单独添加代理，以提高访问速度
#vi ~/dockerfile/dnf.conf
proxy=http://noc:8912027@yourdomain:8080

更新软件
#dnf update -y

查看当前php默认版本
#dnf module list php

php当前默认版本为7.2，修改为8.0
#dnf module -y reset php
#dnf module -y enable php:8.0

安装epel扩展软件仓库
#dnf install epel-release -y

安装wordpress所需依赖包
#dnf --enablerepo=epel -y install httpd php php-fpm php-pear php-mbstring php-pdo php-gd php-mysqlnd php-IDNA_Convert php-enchant enchant hunspell

修改配置文件
#mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.org
#vi /etc/httpd/conf/httpd.conf
#89行，修改管理员邮箱
ServerAdmin root@localhost
#167行增加index.php
    DirectoryIndex index.html index.php

#最后一行增加配置隐藏版本信息
ServerTokens Prod

#vi /etc/php-fpm.d/www.conf
#在末尾增加php配置
php_value[max_execution_time] = 600
php_value[memory_limit] = 2G
php_value[post_max_size] = 2G
php_value[upload_max_filesize] = 2G
php_value[max_input_time] = 600
php_value[max_input_vars] = 2000
php_value[date.timezone] = Asia/Shanghai

创建php-fpm运行目录
#mkdir -p /run/php-fpm/

运行php-fpm
#php-fpm -D

运行apache
#httpd -k start

安装ps命令包（容器内的操作缺什么工具就安装什么工具）
#dnf install procps-ng -y

查看进程
#ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 01:48 pts/0    00:00:00 /bin/bash
root         410       1  0 02:09 ?        00:00:00 php-fpm: master process (/etc/php-fpm.conf)
apache       411     410  0 02:09 ?        00:00:00 php-fpm: pool www
apache       412     410  0 02:09 ?        00:00:00 php-fpm: pool www
apache       413     410  0 02:09 ?        00:00:00 php-fpm: pool www
apache       414     410  0 02:09 ?        00:00:00 php-fpm: pool www
apache       415     410  0 02:09 ?        00:00:00 php-fpm: pool www
root         427       1  0 02:10 ?        00:00:00 httpd -k start
apache       428     427  0 02:10 ?        00:00:00 httpd -k start
apache       429     427  0 02:10 ?        00:00:00 httpd -k start
apache       430     427  0 02:10 ?        00:00:00 httpd -k start
apache       432     427  0 02:10 ?        00:00:00 httpd -k start
root         643       1  0 02:10 pts/0    00:00:00 ps -ef

生成phpinfo的验证文件
#echo '<?php phpinfo(); ?>' > /var/www/html/info.php

按Ctr+p和Ctr+q退出容器返回宿主机（直接exit容器内所运行进程会被初始化）
从客户端浏览器访问测试

由于要启动php-fpm和httpd两个进程，进入容器安装supervisor
#yum install -y supervisor
#vi /etc/supervisord.conf
[supervisord]
nodaemon=false

创建要启动服务的配置文件
#vi /etc/supervisord.d/startup.ini
[program:php-fpm]
command= /sbin/php-fpm -D
[program:httpd]
command=/sbin/httpd -k start

运行supervisord启动php-fpm和httpd服务
#/bin/supervisord

此时下载wordpress镜像到/var/www/html根目录即可进行初始化了
#cd /var/www/html/
#wget https://cn.wordpress.org/latest-zh_CN.zip
#dnf install -y unzip
#unzip latest-zh_CN.zip

修改权限
#chown -R apache. wordpress/

删除下载文件
#rm latest-zh_CN.zip

将wordpress文件放置到apache根目录
#mv /var/www/html/wordpress/* /var/www/html/
#rmdir /var/www/html/wordpress/

修改wordpress配置文件，配置数据库信息
#cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php
#vi /var/www/html/wp-config.php
/** WordPress 数据库名称 */
define( 'DB_NAME', 'wordpress' );
/** 数据库用户名 */
define( 'DB_USER', 'wordpress' );
/** 数据库密码 */
define( 'DB_PASSWORD', 'Wordpress!2024' );
/** 数据库主机 */
define( 'DB_HOST', '172.16.249.229:33061' );
数据库信息配置正确就可以进行初始化了。

日志文件目录/var/log/httpd/
php配置文件/etc/httpd/conf.d/php.conf
php-fpm配置文件/etc/php-fpm.d/www.conf
apache配置文件/etc/httpd/conf/httpd.conf
apache的php配置文件 /etc/httpd/conf.d/php.conf
supervisor配置文件/etc/supervisord.conf
supervisor增加的启动服务配置文件/etc/supervisord.d/startup.ini

### 创建rocky8.9+php-fpm8的镜像
在宿主机创建一个目录
#mkdir ~/dockerfile

将修改过的文件从正在运行的容器中复制出来
#docker cp rockylinux8.9:/etc/httpd/conf/httpd.conf  ~/dockerfile
#docker cp rockylinux8.9:/etc/php-fpm.d/www.conf  ~/dockerfile
#docker cp rockylinux8.9:/etc/supervisord.conf  ~/dockerfile
#docker cp rockylinux8.9:/etc/supervisord.d/startup.ini  ~/dockerfile
#docker cp rockylinux8.9:/etc/dnf/dnf.conf ~/dockerfile

创建一个dockerfile
#vi ~/dockerfile/Dockerfile
FROM rockylinux/rockylinux:8.9
MAINTAINER cg
COPY dnf.conf /etc/dnf/
RUN dnf update -y && dnf install epel-release -y && dnf module -y reset php && dnf module -y enable php:8.0 &&  dnf --enablerepo=epel -y install httpd php php-fpm php-pear php-mbstring php-pdo php-gd php-mysqlnd php-IDNA_Convert php-enchant enchant hunspell procps-ng  supervisor && mkdir -p /run/php-fpm/ && mv /etc/httpd/conf.d/welcome.conf /etc/httpd/conf.d/welcome.conf.org && dnf clean all && history -c
COPY httpd.conf /etc/httpd/conf/httpd.conf
COPY www.conf /etc/php-fpm.d/
COPY supervisord.conf /etc/
COPY startup.ini  /etc/supervisord.d/
EXPOSE 80
CMD ["/usr/bin/supervisord"]

#cd ~/dockerfile/
#docker build -t="rockylinux8.9-php-fpm8"  .

查看镜像
#docker image ls
REPOSITORY               TAG       IMAGE ID       CREATED         SIZE
rockylinux8.9-php-fpm8   latest    06a9580ec819   9 seconds ago   467MB

测试新镜像
#docker stop rockylinux8.9
#mkdir /opt/wordpress
#docker run -d --name wordpress -v /opt/wordpress:/var/www/html -p 80:80 --restart=always rockylinux8.9-php-fpm8
#docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS        PORTS                                                    NAMES
0ee49eaeec2f   rockylinux8.9-php-fpm8   "/usr/bin/supervisord"   2 seconds ago   Up 1 second   0.0.0.0:80->80/tcp, :::80->80/tcp                        wordpress

当前只是将数据目录映射到了容器中，有时还会将配置文件拷贝出来以方便修改将配置文件也进行映射，或将日志目录也进行映射以实现数据持久化保存，因为容器一但被删除容器中所有数据将不存在。

进入正在运行的容器
#docker exec -ti wordpress bash
[root@0ee49eaeec2f /]# ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 04:02 ?        00:00:00 /usr/bin/python3.6 /usr/bin/supervisord
root          11       1  0 04:02 ?        00:00:00 /sbin/httpd -k start
apache        13      11  0 04:02 ?        00:00:00 /sbin/httpd -k start
apache        14      11  0 04:02 ?        00:00:00 /sbin/httpd -k start
apache        15      11  0 04:02 ?        00:00:00 /sbin/httpd -k start
root         227       1  0 04:02 ?        00:00:00 php-fpm: master process (/etc/php-fpm.conf)
apache       228     227  0 04:02 ?        00:00:00 php-fpm: pool www
apache       229     227  0 04:02 ?        00:00:00 php-fpm: pool www
apache       230     227  0 04:02 ?        00:00:00 php-fpm: pool www
apache       231     227  0 04:02 ?        00:00:00 php-fpm: pool www
apache       232     227  0 04:02 ?        00:00:00 php-fpm: pool www
root         248       0  0 04:02 pts/0    00:00:00 bash
root         323     248  0 04:03 pts/0    00:00:00 ps -ef
PID1进程对于操作系统而言具有特殊意义。操作系统的PID1进程是init进程，以守护进程方式运行，是所有其他进程的祖先，具有完整的进程生命周期管理能力。在Docker容器中，PID1进程是启动进程，它也会负责容器内部进程管理的工作。而这也将导致进程管理在Docker容器内部和完整操作系统上的不同。

镜像导出
#docker image save rockylinux8.9-php-fpm8:latest -o rockylinux8.9-php-fpm8.tar
删除容器和镜像
#docker rm -f wordpress
#docker image rm rockylinux8.9-php-fpm8:latest
#docker image ls
导入镜像
#docker image load -i rockylinux8.9-php-fpm8.tar


总结：
1.docker容器可以让我们很快的部署一个应用，绝大部分镜像可以直接从各个容器仓库下载；
2.自己制作镜像需要对所部署应用的运行环境非常熟悉；
3.运行环境和数据一般分离，如当前例子，只创建了apache+php的运行环境；当下有很多应用也将应用和数据打在一个容器中，在生成容器时需要考虑数据持久化；
4.熟练掌握镜像操作是学习容器的关键。

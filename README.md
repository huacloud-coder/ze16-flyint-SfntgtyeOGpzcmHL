
**注意：请确保已经安装Redis和keepalived，本文不在介绍如何安装。**


#### 1、使用版本说明


Redis版本：5\.0\.2


Keepalived版本：1\.3\.5


Linux 版本：Centos7\.9


查看Redis版本：



```
/usr/local/redis/bin/redis-cli -v

```

![](https://img2024.cnblogs.com/blog/2661519/202408/2661519-20240829164538975-860102896.png)


查看Keepalived版本信息：



```
rpm -qa|grep keepalived 或者 keepalived -v

```

![](https://img2024.cnblogs.com/blog/2661519/202408/2661519-20240829164557286-2038240079.png)


#### 2、功能实现说明：


* 使用Keepalived提供虚拟IP对外访问Redis
* Redis搭建主从数据同步，主用来读写数据 、从主要进行主数据同步备份。
* 当主出现宕机，Keepalived虚拟IP自动指向从服务器。从服务器临时变为主服务器继续工作。
* 待主服务器重新启动后，Keepalived虚拟IP重新指向主服务器。主服务器同步从服务器数据后继续工作。从服务器由临时主变为从继续进行主数据同步备份。


#### 3、说明图


**Keepalived会生成一个虚拟IP。客户端需要访问虚拟IP进行Redis连接:**


##### 3\.1 、主和备服务器运行中


![](https://img2024.cnblogs.com/blog/2661519/202408/2661519-20240829164735136-1362191460.png)


##### 3\.2、主宕机，备服务器运行中


![](https://img2024.cnblogs.com/blog/2661519/202408/2661519-20240829164804984-408619253.png)


##### 3\.3、 主恢复，备服务器运行中


![](https://img2024.cnblogs.com/blog/2661519/202408/2661519-20240829164833021-447147970.png)


#### 4、搭建Redis主从


首先确保两台服务器都安装了Redis服务，Redis的端口号和密码两台服务器必须保持一致。我这里两台服务器都是使用端口号：6379和密码：1234qwer


服务器IP：


主服务器：192\.168\.42\.130


备服务器：192\.168\.42\.133


##### 4\.1、 修改配置文件


首先需要修改备服务器redis配置文件，把备服务器redis挂载到主服务器redis下面实现主从配置。


进入redis目录



```
cd /usr/local/redis/

```

修改redis.conf文件



```
vim redis.conf

```

找到replicaof和masterauth属性进行配置



```
# replicaof  
replicaof 192.168.42.130 6379
# If the master is password protected (using the "requirepass" configuration
# directive below) it is possible to tell the replica to authenticate before
# starting the replication synchronization process, otherwise the master will
# refuse the replica request.
#
# masterauth 
masterauth "1234qwer"


```

replicaof：主服务器IP和端口号


masterauth：用于在进行主从复制时,保护Redis主节点的数据安全。密码就是设置相同的主从节点的密码。以确保只有经过授权的从节点才能够连接到主节点。如果不配置，会导致节点连接失败：`master_link_status:down`


通过登录redis输入：`info replication` 可以查看Redis集群配置信息。如果搭建成功，会显示节点信息。


**注意：** 在Redis 5\.0及以上版本，SLAVEOF 命令已经被废弃，并且在服务器上使用该命令会导致命令失效。所以在Redis 5\.0及以上版本，设置复制的正确方法是使用 REPLICAOF 命令。为了兼容旧版本，通过配置的方式仍然支持 slaveof，但是通过命令的方式则不行了。


##### 4\.2、验证主从复制功能


验证方式：登录主服务器Redis，插入一条key数据。在备服务器中登录Redis进行通过该key进行查询，查看是否获取到数据。如果数据获取成功，说明Redis主从复制搭建成功。


进入主服务器登录redis



```
/usr/local/redis/bin/redis-cli

```

进行密码认证



```
auth 1234qwer

```

输出OK，表示认证成功。存入数据



```
set verify-key "Test Verify is Success"

```

输出OK，表示插入成功。接下来登录备服务器查看数据。进入备服务器登录redis：



```
/usr/local/redis/bin/redis-cli

```

进行密码认证



```
auth 1234qwer

```

输出OK，表示认证成功。获取key数据



```
get verify-key

```

输出："Test Verify is Success"，说明Redis主从搭建成功。


#### 5、配置Keepalived信息


/etc/keepalived目录中存放keepalived.conf文件。在该目录下创建scripts\_redis文件夹，目录 /etc/keepalived/scripts\_redis，将 redis\_stop.sh 、redis\_master.sh、redis\_fault.sh 、redis\_check.sh 、redis\_backup.sh 放入scripts\_redis文件目录下。


##### 5\.1 主服务器配置


编写keepalived.conf



```
! Configuration File for keepalived

global_defs {
   router_id redis-master #唯一标识 注意主备服务名不可相同
   script_user root
   enable_script_security
}

vrrp_script redis_check { #脚本检测名称,下方调用必须和这个名称一致
    script "/etc/keepalived/scripts_redis/redis_check.sh" #监听redis是否启动脚本路径
    interval 4  #监听心跳
    weight -5
    fall 3
    rise 2
}

vrrp_instance VI_redis {
    state MASTER    #当前keepalived状态  MASTER 或者 BACKUP
    interface eth0  #网卡名称根据实际情况设置可通过命令ifconfig查看
    virtual_router_id 21
    priority 110	#权重 主服务要高于备服务
    garp_master_refresh 10
    garp_master_refresh_repeat 2
    advert_int 1
    nopreempt
    unicast_src_ip 192.168.42.130 #单播模式 当前服务器主服务器IP地址
    unicast_peer {
        192.168.42.133 #备服务器Ip
    }
	
    authentication {  #keepalived之间通信的认证账号、密码
        auth_type PASS
        auth_pass 1111
    }
	
    virtual_ipaddress {
        192.168.42.161	#虚拟IP地址 客户端统一的访问地址
    }
	
    garp_master_delay 1
    garp_master_refresh 5
	track_interface {
        eth0	#网卡
    }
	
    track_script {
        redis_check #脚本检测调用名称
    }
	
    notify_master /etc/keepalived/scripts_redis/redis_master.sh	#master脚本  keepalived设置的状态为master时触发或者master停止后，backup升级为master时触发
    notify_backup /etc/keepalived/scripts_redis/redis_backup.sh #backup脚本  keepalived设置的状态为backup时触发
    notify_fault /etc/keepalived/scripts_redis/redis_fault.sh #fault脚本
    notify_stop /etc/keepalived/scripts_redis/redis_stop.sh  #stop脚本 keepalived停止时触发
	   
}

```

编写 redis\_master.sh，当主脚本启动时，需要先同步备服务器redis数据后，在设置为主节点进行启动：



```
#!/bin/bash

LOGFILE=/var/log/keepalived-redis-status.log
REDISCLI="/usr/local/redis/bin/redis-cli"

echo "Running redis_master.sh..." >>$LOGFILE
echo "[Master]" >> $LOGFILE
date >> $LOGFILE
echo "Being Master..." >> $LOGFILE
echo "Running SLAVEOF cmd..." >> $LOGFILE
$REDISCLI -h 192.168.42.130 -p 6379 -a 1234qwer CONFIG SET masterauth "1234qwer" 2>&1
$REDISCLI -h 192.168.42.130 -p 6379 -a 1234qwer REPLICAOF  192.168.42.133 6379 2>&1

sleep 5s

echo "Run slaveof no one cmd..." >>$LOGFILE

$REDISCLI -h 192.168.42.130 -p 6379 -a 1234qwer REPLICAOF NO ONE >>$LOGFILE 2>&1

echo "Finished running redis_master.sh..." >>$LOGFILE

```

编写redis\_backup.sh，



```
#!/bin/bash

LOGFILE=/var/log/keepalived-redis-status.log
REDISCLI="/usr/local/redis/bin/redis-cli"
echo "Running redis_bakcup.sh..." >>$LOGFILE
echo "[Backup]" >> $LOGFILE
date >> $LOGFILE
echo "Being Slave..." >> $LOGFILE
echo "Run SLAVEOF cmd..." >> $LOGFILE
$REDISCLI -h 192.168.42.130 -p 6379 -a 1234qwer CONFIG SET masterauth "1234qwer"  >>$LOGFILE 2>&1
$REDISCLI -h 192.168.42.130 -p 6379 -a 1234qwer REPLICAOF 192.168.42.133 6379 >>$LOGFILE 2>&1
echo "Finished running redis_backup.sh..." >>$LOGFILE


```

##### 5\.2、备服务器配置


编写keepalived.conf



```
! Configuration File for keepalived

global_defs {
   router_id redis-slave #唯一标识 注意主备服务名不可相同
   script_user root
   enable_script_security
}

vrrp_script redis_check {
    script "/etc/keepalived/scripts_redis/redis_check.sh" #监听redis是否启动脚本路径
    interval 4 #监听心跳
    weight -5
    fall 3  
    rise 2
}

vrrp_instance VI_redis {
    state BACKUP  #当前keepalived状态 设置为BACKUP
    interface eth0
    virtual_router_id 21
    priority 100
    garp_master_refresh 10
    garp_master_refresh_repeat 2
    advert_int 1
    nopreempt
    unicast_src_ip 192.168.42.133 #单播模式 当前服务器IP地址
    unicast_peer {
        192.168.42.130 #主服务器Ip
    }
	
	
    authentication {
        auth_type PASS
        auth_pass 1111
    }
	
    virtual_ipaddress {
        192.168.42.161 #虚拟IP地址 客户端统一的访问地址
    }
	
    garp_master_delay 1
    garp_master_refresh 5

    track_interface {
        eth0
    }

    track_script {
        redis_check
    }
	
    notify_master /etc/keepalived/scripts_redis/redis_master.sh
    notify_backup /etc/keepalived/scripts_redis/redis_backup.sh
    notify_fault /etc/keepalived/scripts_redis/redis_fault.sh 
    notify_stop /etc/keepalived/scripts_redis/redis_stop.sh 
}



```

编写redis\_master.sh



```
#!/bin/bash
# LOGFILE文件需要跟据实际情况更改
LOGFILE=/var/log/keepalived-redis-status.log
REDISCLI="/usr/local/redis/src/redis-cli"

echo "Running redis_master.sh..." >>$LOGFILE
echo "[Master]" >> $LOGFILE
date >> $LOGFILE
echo "Begin Master ..." >> $LOGFILE
echo "Run slaveof no one cmd...">>$LOGFILE
# SLAVEOF 5.0以上已经弃用 REPLICAOF 
$REDISCLI -h 192.168.42.133 -p 6379 -a 1234qwer REPLICAOF  NO ONE >>$LOGFILE 2>&1
echo "Finished running redis_master.sh..." >>$LOGFILE

```

编写redis\_backup.sh



```
#!/bin/bash

LOGFILE=/var/log/keepalived-redis-status.log
REDISCLI="/usr/local/redis/src/redis-cli"

echo "Running redis_bakcup.sh..." >>$LOGFILE
echo "[Backup]" >> $LOGFILE
date >> $LOGFILE
echo "Being Slave..." >> $LOGFILE
sleep 15s #休眠15秒，确保主服务器脚本redis_master.sh执行完毕后在执行主从命令
echo "Run SLAVEOF cmd..." >> $LOGFILE
# SLAVEOF 5.0已经弃用 改为：REPLICAOF
$REDISCLI -h 192.168.42.133 -p 6379 -a 1234qwer CONFIG SET masterauth "1234qwer"  >>$LOGFILE 2>&1
$REDISCLI -h 192.168.42.133 -p 6379 -a 1234qwer REPLICAOF  192.168.42.130 6379 >>$LOGFILE 2>&1
echo "Finished running redis_backup.sh..." >>$LOGFILE

```

##### 5\.3、编写验证Redis是否启动脚本


编写redis\_check.sh脚本,通过监听端口号判断（主备一致）



```
#!/bin/bash
LOGFILE=/var/log/check-redis-status.log
echo "Running redis_check.sh..." >> $LOGFILE
date >> $LOGFILE
CHECK=$(ss -tnlp|grep 6379)
if [ $? -ne 0 ]; then
   echo "redis-server is not running..." >> $LOGFILE
   systemctl stop keepalived.service
   exit 1
else
   echo "redis-server is running..." >> $LOGFILE
   exit 0
fi
echo "Finished running redis_check.sh..." >> $LOGFILE

```

##### 5\.4 其他脚本


编写 redis\_fault.sh （主备一致）



```
#!/bin/bash

LOGFILE=/var/log/keepalived-redis-status.log
echo "Running redis_fault.sh..." >>$LOGFILE
echo "[Fault]" >> $LOGFILE
date >> $LOGFILE
echo "Finished running redis_fault.sh..." >> $LOGFILE


```

编写 redis\_stop.sh （主备一致）



```
#!/bin/bash
LOGFILE=/var/log/keepalived-redis-status.log
echo "Running redis_stop.sh...." >>$LOGFILE
echo "[Stop]" >> $LOGFILE
date >> $LOGFILE
echo "Finished running redis_stop.sh...." >>$LOGFILE

```

##### 5\.5、给脚本授可执行权限


`chmod +x /etc/keepalived/scripts_redis/*.sh`


##### 5\.6、 keepalived相关命令


Keepalived 安装命令


`yum install keepalived -y`


Keepalived配置所在目录


 `/etc/keepalived` 


Keepalived 日志文件


`/var/log/message`


启动Keepalived命令


 `systemctl start keepalived.service`


重启Keepalived命令


`systemctl restart keepalived.service`


查看Keepalived状态命令


 `systemctl status keepalived.service`


查看Keepalived虚拟VIP ip


`ip addr`


关闭Keepalived命令


`systemctl stop keepalived.service`


#### 6、验证主备双活


我们可以通过连接工具RedisDesktopManager进行测试主备双活。首先连接地址填写keepalived生成的虚拟IP地址：192\.168\.42\.161、 输入端口号：6379和密码：1234qwer


![](https://img2024.cnblogs.com/blog/2661519/202408/2661519-20240829165010347-1833152795.png)


连接成功后，插入一条数据进行数据测试。之后在把主服务器redis停止模拟服务器宕机，测试连接继续进行数据插入。然后在把主服务器redis启动，keepalived也需要启动。启动完成后查看数据是否一致。如果一致说明主备双活搭建成功。


![](https://img2024.cnblogs.com/blog/2661519/202408/2661519-20240829165021262-103784479.png)


 本博客参考[西部世界官网](https://tianchuang88.com)。转载请注明出处！

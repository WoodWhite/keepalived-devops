# keepalived-devops
* [官网](http://www.keepalived.org/doc/)
* 原理

```
keepalived 是一个用于做双机热备（HA）的软件，它的主要功能是实现真实机的故障隔离及负载均衡器间的失败切换，提高系统的可用性。
keepalived 常配合使用的负载均衡器有 LVS、HAProxy、Nginx 等 。
keepalived 通过选举（看服务器设置的权重）挑选出一台热备服务器做MASTER机器，MASTER机器会被分配到一个指定的虚拟ip，外部程序可通过该ip访问这台服务器，如果这台服务器出现故障（断网，重启，或者本机器上的keepalived crash等），keepalived会从其他的备份机器上重选（还是看服务器设置的权重）一台机器做MASTER并分配同样的虚拟IP，充当前一台MASTER的角色。
选举策略是根据VRRP协议，完全按照权重大小，权重最大（0～255）的是MASTER机器，下面几种情况会触发选举。
1. keepalived启动的时候；
2. master服务器出现故障（断网，重启，或者本机器上的keepalived crash等）；
3. 有新的备份服务器加入且权重最大。
```
* [VRRP](https://tools.ietf.org/html/rfc5798)

---
Nginx + Keepalived

* 安装

```
yum -y install keepalived
```

* 配置

```
# keepalived 配置

! Configuration File for keepalived

global_defs {
   router_id NGINX_HA
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 3
}

vrrp_instance VI_1 {
    state BACKUP # MASTER
    interface bond0
    virtual_router_id 77
    priority 100
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        xxx.xxx.xxx.xxx
    }
    track_script {
       check_nginx
    }
    notify_master /etc/keepalived/clean_arp.sh
}


# 检查nginx进程存活脚本 check_nginx.sh 

#!/bin/bash
counter=$(ps -C nginx --no-heading|wc -l)
if [ "${counter}" = "0" ]; then
    systemctl start nginx
    sleep 2
    counter=$(ps -C nginx --no-heading|wc -l)
    if [ "${counter}" = "0" ]; then
       systemctl stop nginx
    fi
fi

# ARP 缓存清理脚本 clean_arp.sh

#!/bin/bash
VIP=xxx.xxx.xxx.xxx
GATEWAY=xxx.xxx.xxx.xxx
IN = xxxxxx
/sbin/arping -I $IN -c 5 -s $VIP $GATEWAY &>/dev/null


```

* 启动

```
systemctl start keepalived
```

---
LVS + Nginx + Keepalived

* 情景

```
VIP : 1.1.1.1
LVS IP : 1.1.1.2
LVS IP : 1.1.1.3
Real Server IP : 1.1.1.4
Real Server IP : 1.1.1.5
Real Server IP : 1.1.1.6

```

* 安装

```
# LVS
yum -y install ipvsadm keepalived
```

* 配置

```

! Configuration File for keepalived

global_defs {
    router_id LVS_1
}

vrrp_sync_group VSG_1 {     
    group{
        VI_1
    }
}

vrrp_instance VI_1 {
    state MASTER
    interface bond0
    lvs_sync_daemon_inteface bond0
    virtual_router_id 38
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass 123456
    }
    virtual_ipaddress {
        1.1.1.1
    }
}

virtual_server 1.1.1.1 80 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    protocol TCP
    real_server 1.1.1.4 80 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 1.1.1.5 80 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 1.1.1.6 80 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

virtual_server 1.1.1.1 443 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    protocol TCP
    real_server 1.1.1.4 443 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 1.1.1.5 443 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 1.1.1.6 443 {
        weight 1
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

# 修改keepalived备的配置，基本都和主一样，只需要修改几处
1. state BACKUP
2. priority 99

在两台lvs服务器上开启路由转发功能
1. # vim /etc/sysctl.conf
2. net.ipv4.ip_forward = 1
3. # sysctl -p

```

* 启动

```
# systemctl start keepalived
# systemctl enable keepalived
```

* real server 脚本

```
vi /etc/init.d/ipvsadm


#!/bin/bash
VIP=1.1.1.1
case "$1" in
start)
    echo "start LVS of RealServer DR"
    /sbin/ifconfig lo:0 $VIP broadcast $VIP netmask 255.255.255.255 up
    /sbin/route add -host $VIP dev lo:0
    echo "1" > /proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "2" > /proc/sys/net/ipv4/conf/lo/arp_announce
    echo "1" > /proc/sys/net/ipv4/conf/all/arp_ignore
    echo "2" > /proc/sys/net/ipv4/conf/all/arp_announce
    ;;
stop)
    /sbin/ifconfig lo:0 down
    echo "close LVS of RealServer DR"
    echo "0" > /proc/sys/net/ipv4/conf/lo/arp_ignore
    echo "0" > /proc/sys/net/ipv4/conf/lo/arp_announce
    echo "0" > /proc/sys/net/ipv4/conf/all/arp_ignore
    echo "0" > /proc/sys/net/ipv4/conf/all/arp_announce
    ;;
*)
    echo "Usage: $0 {start|stop}"
    exit 1
esac
exit 0


# chmod +x /etc/init.d/ipvsadm
```

* 添加开机启动

```
# vim /etc/rc.d/rc.local
/etc/init.d/ipvsadm start
# chmod +x /etc/rc.d/rc.local
```

---
keepalived + redis

* 安装

```
# yum -y install keepalived redis
```

* 配置

```
keepalived master/backup

vrrp_script chk_redis {

script "/etc/keepalived/scripts/redis_check.py" ###监控脚本

interval 1 ###监控时间设置为1s

}

vrrp_instance VI_1 {

state MASTER ###设置为MASTER

interface eno16780032 ###监控网卡

virtual_router_id 70

priority 101 ###权重值

authentication {

auth_type PASS ###加密

auth_pass redis ###密码

}

track_script {

chk_redis ###调用上面定义的chk_redis

}

virtual_ipaddress {

192.168.91.112 ###对外的虚拟IP

}

notify_master /etc/keepalived/scripts/redis_master.py

notify_backup /etc/keepalived/scripts/redis_backup.py

notify_fault /etc/keepalived/scripts/redis_fault.py

notify_stop /etc/keepalived/scripts/redis_stop.py

}

master/backup区别
1. state MASTER/BACKUP
2. interface
3. priority

```

```
redis master/slave

注意
master/slave 
1. bind 0.0.0.0

slave
1. save "" 去掉注释
2. 其他save项注释掉
3. slaveof masterip masterport
4. appendonly yes

```

* 脚本

```
redis_check.py 

#!/usr/bin/python

import os
import sys
import time

PING = 'redis-cli ping' #redis ping command, observe network state
os.chdir("/var/log/redis/") #set log file path
fp = open("redis_dump.log",'a') #open log file with append mode
result = os.system(PING) #exec command
if 0 == result: #network state ok
    logtime = time.strftime("%Y-%m-%d %H:%M:%S",time.localtime())
    fp.write("[check]" + logtime + ":" + 'redis running!\n')
    fp.close()
    sys.exit(0)
else:
    logtime = time.strftime("%Y-%m-%d %H:%M:%S",time.localtime())
    fp.write("[check]" + logtime + ":" + 'redis stop service!\n')
    fp.close()
    sys.exit(1)
```

```
redis_master.py

#!/usr/bin/python

import os
import time

SLAVEOF = 'redis-cli slaveof 192.168.91.42 6379' #backup data form slave
SLAVENO = 'redis-cli slaveof no one' #being master
os.chdir("/var/log/redis/") #set log file path
fp = open("redis_dump.log",'a') #open log file with append mode
result = os.system(SLAVEOF) #exec command
logtime = time.strftime("%Y-%m-%d %H:%M:%S",time.localtime()) #get time info
if 0 == result: #backup data from slave
    fp.write("[master]" + logtime + ":" + 'start copy data from slave!\n')
else:
    fp.write("[master]" + logtime + ":" + 'copy data from slave falue!\n')
time.sleep(10) #set backup time
result = os.system(SLAVENO) #being master
logtime = time.strftime("%Y-%m-%d %H:%M:%S",time.localtime())
if 0 == result:
    fp.write("[master]" + logtime + ":" + 'stop copy data, being master!\n')
else:
    fp.write("[master]" + logtime + ":" + 'being master falue!\n')
fp.close()
```

```
redis_backup.py

#!/usr/bin/python

import os
import time

time.sleep(15) #set data backup time
SLAVEOF = 'redis-cli slaveof 192.168.91.42 6379'
os.chdir("/var/log/redis/") #set log file path
fp = open("redis_dump.log",'a') #open log file with append mode
result = os.system(SLAVEOF) #exec command
logtime = time.strftime("%Y-%m-%d %H:%M:%S",time.localtime())
if 0 == result:
    fp.write("[backup]" + logtime + ":" + 'being slave!\n')
else:
    fp.write("[backup]" + logtime + ":" + 'being slave falue!\n')
fp.close()
```

* 启动

```
systemctl start redis
systemctl start keepalived

```

* 测试

```
ip a
redis-cli info
redis-cli

1. redis master 写入
 > set name xxx

2. redis slave 查看
 > keys *

4. tailf /var/log/redis_dump.log

3. redis master stop
 # systemctl stop redis

4. ip a

5. slave 变 master
 # redis-cli info

6. 新 master 测试写入

7. 旧 master 重启

8. 检查 master/slave 切换，以及数据同步情况

```

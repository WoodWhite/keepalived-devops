# keepalived-devops
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

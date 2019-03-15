---
bg: "grace.jpg"
layout: post
title:  "shadowsocks-libev"
crawlertitle: "shadowsocks-libev"
summary: "shadowsocks-libev 透明代理"
date:   2019-03-15
author: snailqh
---
#### shadowsocks-libev
```shell
wget https://copr.fedorainfracloud.org/coprs/librehat/shadowsocks/repo/epel-7/librehat-shadowsocks-epel-7.repo -O /etc/yum.repos.d/librehat-shadowsocks-epel-7.repo

yum update
yum install shadowsocks-libev ipset redis iptables-services nodejs libcurl-devel
```
#### /etc/shadowsocks-libev/config.json
```
{
    "server":"144.34.154.158",
    "server_port":443,
    "local_address":"0.0.0.0",
    "local_port":1080,
    "password":"vXEi71zToSgbFwvo",
    "timeout":60,
    "method":"aes-256-cfb"
}
```
```shell
nohup ss-redir -c /etc/shadowsocks-libev/config.json -l 1080 &>> /var/log/ss-redir.log &
nohup ss-tunnel -c /etc/shadowsocks-libev/config.json -l 15353 -u -L 8.8.8.8:53 &>> /var/log/ss-tunnel.log &
```
#### ipset
```shell
curl 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > cidr_cn
ipset -N cidr_cn hash:net maxelem 100000
for i in `cat cidr_cn`; do echo ipset -A cidr_cn $i >> ipset.sh; done
bash ipset.sh
ipset -S > /etc/ipset.cidr_cn.rules
echo "ipset restore -f /etc/ipset.cidr_cn.rules" >> /etc/rc.local
```
#### iptables
```shell
iptables -t nat -N shadowsocks
iptables -t nat -A shadowsocks -d 0/8 -j RETURN
iptables -t nat -A shadowsocks -d 127/8 -j RETURN
iptables -t nat -A shadowsocks -d 10/8 -j RETURN
iptables -t nat -A shadowsocks -d 169.254/16 -j RETURN
iptables -t nat -A shadowsocks -d 172.16/12 -j RETURN
iptables -t nat -A shadowsocks -d 192.168/16 -j RETURN
iptables -t nat -A shadowsocks -d 224/4 -j RETURN
iptables -t nat -A shadowsocks -d 240/4 -j RETURN
iptables -t nat -A shadowsocks -d  144.34.154.158 -j RETURN
iptables -t nat -A shadowsocks -d  49.51.70.237 -j RETURN


iptables -t nat -A shadowsocks -m set --match-set cidr_cn dst -j RETURN
iptables -t nat -A shadowsocks ! -p icmp -j REDIRECT --to-ports 1080
iptables -t nat -A OUTPUT ! -p icmp -j shadowsocks
iptables -t nat -A PREROUTING ! -p icmp -j shadowsocks
```
#### 开启转发
```shell
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```
#### hev-dns-forwarder  UDP to TCP
```shell
 git clone https://github.com/aa65535/hev-dns-forwarder.git
 autoreconf -ivf
 make
./hev-dns-forwarder -b 127.0.0.1 -p 5300 -s 8.8.8.8:53
```
#### DNSKeeper
```shell
git clone https://github.com/billtt/dnskeeper.git
npm install
npm install forever -g

curl https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt > GFWBasicRules.list
```
#### config/production.json
```shell
{
  "debug": true,
  "host": "0.0.0.0",
  "port": 53,
  "fastResponse": true,
  "foreignServer": {
    "address": "127.0.0.1",
    "port": 5300,
    "type": "udp",
    "memo": "Google"
  },
  "domesticServer": {
    "address": "114.114.114.114",
    "port": 53,
    "type": "udp",
    "memo": "DNSPod"
  },
  "static": {
    "ttl": 120,
    "table": {
      "t.e.s.t": "1.2.3.4"
    }
  },
  "cache": {
    "host": "localhost",
    "port": 6379,
    "options": {}
  },
  "extraGFWRules": "/home/wqh/dnskeeper/GFWExtraRules.list"
}
# ./service-dns-server.sh start
```

#### 或chinadns
```shell
https://github.com/shadowsocks/ChinaDNS
wget https://github.com/shadowsocks/ChinaDNS/releases/download/1.3.2/chinadns-1.3.2.tar.gz
./configure && make

#获取最新chinaip
curl 'http://ftp.apnic.net/apnic/stats/apnic/delegated-apnic-latest' | grep ipv4 | grep CN | awk -F\| '{ printf("%s/%d\n", $4, 32-log($5)/log(2)) }' > chnroute.txt
#配合dnsmasq
no-resolv
server=127.0.0.1#5354
```
#### chinadns启动脚本
```shell
#!/bin/sh
### BEGIN INIT INFO
# Provides:          chinadns
# Required-Start:    $network $local_fs $remote_fs $syslog
# Required-Stop:     $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start ChinaDNS at boot time
### END INIT INFO
### Begin Deploy Path
# Put this file at /etc/init.d/
### End Deploy Path
DAEMON=/usr/local/bin/chinadns
DESC=ChinaDNS
NAME=chinadns
PIDFILE=/var/run/$NAME.pid
test -x $DAEMON || exit 0
case "$1" in
  start)
    echo -n "Starting $DESC: "
    $DAEMON \
        -c /etc/chinadns/chnroute.txt \
 -m \
        -p 5354 \
 -s 114.114.114.114,127.0.0.1:15353 \
        1> /var/log/$NAME.log \
        2> /var/log/$NAME.err.log &
    echo $! > $PIDFILE
    echo "$NAME."
    ;;
  stop)
    echo -n "Stopping $DESC: "
    kill `cat $PIDFILE`
    rm -f $PIDFILE
    echo "$NAME."
    ;;
  restart|force-reload)
    $0 stop
    sleep 1
    $0 start
    ;;
  *)
    N=/etc/init.d/$NAME
    echo "Usage: $N {start|stop|restart|force-reload}" >&2
    exit 1
    ;;
esac
exit 0
```
#### 切换peers
```shell
git clone https://github.com/billtt/catapult.git
```

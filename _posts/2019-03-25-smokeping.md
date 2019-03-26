---
bg: "blacksea.jpg"
layout: post
title:  "smokeping"
crawlertitle: "smokeping"
summary: "smokeping"
date:   2019-03-25
author: snailqh
---
# Master
```shell
yum install rrdtool perl-rrdtool spawn-fcgi fping httpd nginx -y

https://oss.oetiker.ch/smokeping/pub/
```
#### 创建web认证密码
```shell
htpasswd -c /opt/smokeping/htdocs/htpasswd admin
chmod 600 /opt/smokeping/etc/smokeping_secrets
```

#### nginx.conf
```shell
server {
        listen       80;
        server_name  default;
        location / {
	    auth_basic "smokeping";
            auth_basic_user_file /opt/smokeping/htdocs/htpasswd;
            root   /opt/smokeping/htdocs/;
            index  index.html index.htm index.cgi;
        }
        location ~ .*\.fcgi$ {
            root  /opt/smokeping/htdocs/;
            fastcgi_pass   127.0.0.1:9007;
            include /etc/nginx/fastcgi_params;
        }
}
```
#### 目录
```shell
mkdir data
mkdir cache
mkdir var

chown nginx.nginx -R /opt/smokeping/data/
chown nginx.nginx -R /opt/smokeping/htdocs/
chown nginx.nginx -R /opt/smokeping/var/
```
#### smokeping 启动脚本
```shell
# cat /etc/init.d/smokeping


#!/bin/bash

#
# chkconfig: 2345 80 05
# Description: Smokeping init.d script
# Write by : linux-Leon_xiedi
# Get function from functions library
. /etc/init.d/functions
# Start the service Smokeping
function start() {
                echo -n "Starting Smokeping: "
                /opt/smokeping/bin/smokeping --config=/opt/smokeping/etc/config --logfile=/opt/smokeping/var/smokeping.log
                ### Create the lock file ###
                touch /var/lock/subsys/smokeping
                success $"Smokeping startup"
                echo
}

# Restart the service Smokeping
function stop() {
                echo -n "Stopping Smokeping: "
                kill -9 `ps ax |grep "/opt/smokeping/bin/smokeping" | grep -v grep | awk '{ print $1 }'` >/dev/null 2>&1
                ### Now, delete the lock file ###
                rm -f /var/lock/subsys/smokeping
                success $"Smokeping shutdown"
                echo

}

#Show status about Smokeping
function status() {
                NUM="`ps -ef|grep smokeping|grep -v grep|wc -l`"
                if [ "$NUM" == "0" ];then
                   echo "Smokeping is not run"
                else
                   echo "Smokeping is running"
                fi
}

### main logic ###
case "$1" in
start)
        start
        ;;
stop)
        stop
        ;;
status)
        status
        ;;
restart|reload)
        stop
        start
;;
*)
echo $"Usage: $0 {start|stop|restart|reload|status}"
exit 1
esac
exit 0
```
#### fastcgi 启动脚本
```shell
cat /etc/init.d/smokeping-fastcgi
#!/bin/bash
FCGI_SCRIPT=/opt/smokeping/smokeping-fastcgi
FASTCGI_USER=nginx
PIDFILE=/var/run/smokeping-fastcgi.pid
RETVAL=0
case "$1" in
start)
$FCGI_SCRIPT
RETVAL=$?
;;
stop)
PID=`cat $PIDFILE`
kill -9 $PID $(pgrep -P $PID)
RETVAL=$?
;;
restart)
PID=`cat $PIDFILE`
kill -9 $PID $(pgrep -P $PID)
$FCGI_SCRIPT
RETVAL=$?
;;
*)
echo "Usage: smokeping-fastcgi {start|stop|restart}"
exit 1
;;
esac
exit $RETVAL
```
#### smokeping-fastcgi
```shell
cat smokeping-fastcgi

#!/bin/bash
/usr/bin/spawn-fcgi -a 127.0.0.1 -p 9007 -P /var/run/smokeping-fastcgi.pid -u nginx -f /opt/smokeping/htdocs/smokeping.fcgi
```

```shell
# cat restart.sh

/etc/init.d/smokeping restart
sleep 1
/etc/init.d/smokeping-fastcgi restart
```
#### config
#### General
```shell
*** General ***
##########################
owner    = Peter Random
contact  = some@address.nowhere
mailhost = my.mail.host
sendmail = /sbin/sendmail
imgcache = /opt/smokeping/htdocs/cache
imgurl   = cache
datadir  = /opt/smokeping/data
piddir  = /opt/smokeping/var
cgiurl   = http://10.136.47.3/smokeping.fcgi
smokemail = /opt/smokeping/etc/smokemail.dist
tmail = /opt/smokeping/etc/tmail.dist
syslogfacility = local0

*** Probes ***

+ FPing

binary = /usr/sbin/fping

#+ tcping
#
##binary=/bin/tcping
##forks = 5
##offset = 50%
##step = 60
##timeout = 15
#
##+ echoping
##forks = 5
##offset = 50%
##step = 60

@include /opt/smokeping/etc/config.d/alerts
@include /opt/smokeping/etc/config.d/slaves
@include /opt/smokeping/etc/config.d/targets


*** Database ***

step     = 60
pings    = 20


AVERAGE  0.5   1  1008
AVERAGE  0.5  12  4320
    MIN  0.5  12  4320
    MAX  0.5  12  4320
AVERAGE  0.5 144   720
    MAX  0.5 144   720
    MIN  0.5 144   720

*** Presentation ***

template = /opt/smokeping/etc/basepage.html.dist
htmltitle = yes
graphborders = no
charset = utf-8

+ overview

width = 600
height = 50
range = 3h

+ detail

width = 600
height = 150
unison_tolerance = 2

"Last 1 Hours"   1h
"Last 3 Hours"   3h
"Last 12 Hours"  12h
#"Last 1 month"   20d

#+ hierarchies
#++ test
#title = test

#++ location
#title = Location
```
#### Alerts
```shell
*** Alerts ***
to = monitor@noc.mgmt.conference.apricot.net
from = smokealert@noc.mgmt.conference.apricot.net

+bigloss
type = loss
# in percent
pattern = ==0%,==0%,==0%,==0%,>0%,>0%,>0%
comment = suddenly there is packet loss

+someloss
type = loss
# in percent
pattern = >0%,*12*,>0%,*12*,>0%
comment = loss 3 times  in a row

+startloss
type = loss
# in percent
pattern = ==S,>0%,>0%,>0%
comment = loss at startup

+rttdetect
type = rtt
# in milli seconds
pattern = <10,<10,<10,<10,<10,<100,>100,>100,>100
comment = routing messed up again ?

+hostdown
type = loss
# in percent
pattern = ==0%,==0%,==0%, ==U
comment = no reply

+lossdetect
type = loss
# in percent
pattern = ==0%,==0%,==0%,==0%,>20%,>20%,>20%
comment = suddenly there is packet loss
```

```shell
*** Alerts ***
to = |/opt/smokeping/bin/detemine.sh
from = smokealert@yyyy-inc.com

+ hostdown
type = loss
pattern = <100%,<100%,==100%,==100%
comment = hostdown
edgetrigger = yes

+ loss>20
type = loss
pattern = >20%<100%,*20*,>20%<100%
comment = loss>20%
edgetrigger = yes

+ 20>loss>10
type = loss
pattern = >10%<=20%,*20*,>10%<=20%
comment = 20%>loss10%
edgetrigger = yes

+ 10>loss>0
type = loss
pattern = >0%<=10%,*20*,>0%<=10%,*20*,>0%<=10%
comment = 10%>loss>0%
edgetrigger = yes

+ rtt>50ms
type = rtt
pattern = ==S,>50,>50
comment = ttl > 50ms
edgetrigger = yes

+ rtt>100ms
type = rtt
pattern = ==S,>100,>100
comment = ttl > 100ms
edgetrigger = yes
```
#### Slaves
```shell
*** Slaves ***
secrets=/opt/smokeping/etc/smokeping_secrets

+ sh-n05-105-60-5.yyyy.com
display_name=sh-4033-slave1
location=sh-4033
color=f54235

+c2-v02-126-152-200.yyyy.com
display_name=c2-cp-slave1
location=c2-cp
color=71c904

+c1-c10-120-7-200.yyyy.com
display_name=c1-sh-slave1
location=c1-sh
color=6fb0f3
```

```shell
# ll etc/smokeping_secrets
-rw------- 1 nginx nginx 114 Apr  9 11:12 etc/smokeping_secrets
# cat etc/smokeping_secrets
sh-n05-105-60-5.yyyy.com:oiuytrewq
c2-v02-126-152-200.yyyy.com:qwertyuio
c1-c10-120-7-200.yyyy.com:asdfghjk
```
#### Targets
```shell
*** Targets ***
probe = FPing

menu = Top
title = Network Latency Grapher

slaves = sh-n05-105-60-5.yyyy.com c2-v02-126-152-200.yyyy.com c1-c10-120-7-200.yyyy.com

@include /opt/smokeping/etc/config.d/targets_partner
@include /opt/smokeping/etc/config.d/targets_lan4033
@include /opt/smokeping/etc/config.d/targets_c1
@include /opt/smokeping/etc/config.d/targets_c2
@include /opt/smokeping/etc/config.d/targets_c3

+ itself
remark = 自监控
menu = itself
title = itself
alerts = hostdown,20>loss>10,loss>20

++ c3-a07-136-47-3
menu = c3-a07-136-47-3.yyyy.com
title = c3-master
host = c3-a07-136-47-3.yyyy.com
remark = test test
```



#### apache  conf
```shell
DocumentRoot "/var/www/html"
Alias /cache "/opt/smokeping/htdocs/cache/"
Alias /cropper "/opt/smokeping/htdocs/cropper/"
Alias /smokeping "/opt/smokeping/htdocs/smokeping.fcgi"
Alias /css "/opt/smokeping/htdocs/css/"
Alias /js "/opt/smokeping/htdocs/js/"
<Directory "/opt/smokeping">
	AllowOverride None
	Options All
	AddHandler cgi-script .fcgi .cgi
	Order allow,deny
	Allow from all
	AuthName "Smokeping"
	AuthType Basic
	AuthUserFile /opt/smokeping/htdocs/htpasswd
	Require valid-user
	DirectoryIndex smokeping.fcgi
</Directory>
```
## alerts
#### cat bin/detemine.sh
```shell
#!/bin/bash
#########################################################
# Script to determine which smokeping will send a alert #
#########################################################
exectime=`date +'%H_%M_%S'`
exec > >(tee /tmp/detemine_${exectime}.log)
exec 2>&1

alertname=$1
target=$2
losspattern=$3
rtt=$4
hostname=$5

echo "$(date +%F-%T)" >> /tmp/invoke.log
echo $@ >> /tmp/invoke.log


#if [ "$losspattern" = "loss: 0%" ];
#then
#    subject="Clear-Alert:${target},${hostname}"
#else
    subject="${target},${hostname}"
#fi

kill_script() {
	pid=`ps -elf | grep detemine | grep -v grep | awk '{print $4}' | head -n 1`
	sleep 3 && kill -9 $pid
}

# judgement
judgement="from"
result=$(echo $target | grep "${judgement}")
if [[ "$result" != "" ]]
#if [[ ${judgement} == *${target}* ]]
then
	slavename=`echo ${target} | awk -F "[] ]+" '{print $3}'`
	target=`echo ${target} | awk -F "[] ]+" '{print $1}'`
	ssh -n $slavename "/bin/bash -x /opt/smokeping/bin/send_alert.sh '$alertname' '$target' '$losspattern' '$rtt' '$hostname'"
	kill_script
else
	/bin/bash -x /opt/smokeping/bin/send_alert.sh "$alertname" "$target" "$losspattern" "$rtt" "$hostname" | tee /tmp/send_alert.log &
	kill_script
fi
```
#### cat bin/send_alert.sh
```
#!/bin/bash
#########################################################
# Script to  report on alert from Smokeping #
#########################################################

alertname=$1
target=$2
losspattern=$3
rtt=$4
hostname=$5
#raise=$6

email="wangqinghai@yyyy-inc.com"
mobile_list="18518308191"
chat="WangQingHai"

#email="wangqinghai@yyyy-inc.com"
#mobile_list="18518308191"
#chat="WangQingHai"

sendtime=`date +'%H:%M:%S'`
sendtime1=`date +'%H_%M_%S'`
random=${RANDOM}
smokeping_mail_content=/tmp/smokeping_mail_content_${sendtime1}_${random}

#smokeping_im_content=/tmp/smokeping_im_content
#smokeping_sms_content=/tmp/smokeping_sms_content

#echo "$(date +%F-%T)" >> /tmp/invoke.log
#echo $@ >> /tmp/invoke.log

#if [ "$losspattern" = "loss: 0%" ];
#then
#    subject="Clear-Alert:${target},${hostname}"
#else
    subject="${target},${hostname}"
#fi

# send sms im
#judge_alert_type=`echo ${alertname} | egrep "hostdown|bigloss|rttdetect"|wc -l`
#    if [ "${judge_alert_type}" -eq 1 ];then
#curl  http://10.136.47.2:5000/sms -d "tos=${mobile_list}&content=[1][2][${subject}][${alertname},from_C3]"
#curl  http://10.136.47.2:5000/sms -d "tos=${mobile_list=[smokeping-alert]时间:${sendtime};服务:${subject};原因:${alertname},from_C3;服务:${subject};原因:${alertname},from_C3"
curl  http://10.136.47.2:4567/send -d "tos=${chat}&content=[[smokeping-alert]时间:${sendtime},服务:${subject},原因:${alertname},from_C3"
curl  -d "mobile_list=$mobile_list&message=[smokeping-alert]时间:${sendtime};服务:${subject};原因:${alertname},from_C3【yd】" --user api:key-d67f875e6e89011d35de10276b9fee99 "http://sms-api.luosimao.com/v1/send_batch.json"

##fi

# generate mail content
echo "Alert FROM C3: " $alertname > ${smokeping_mail_content}
echo "Target: " $target >> ${smokeping_mail_content}
echo "Hostname: " $hostname >> ${smokeping_mail_content}
echo "Loss Pattern: " $losspattern >> ${smokeping_mail_content}
echo "RTT Pattern: " $rtt >> ${smokeping_mail_content}
echo "" >> ${smokeping_mail_content}
echo "MTR Report:mtr -n -c 30 -r ${hostname} " >> ${smokeping_mail_content}
mtr -n -c 30 -r ${hostname} >> ${smokeping_mail_content}
echo "" >> ${smokeping_mail_content}
echo "Ping Report:" >> ${smokeping_mail_content}
ping ${hostname} -c 4 -i 0.5 >> ${smokeping_mail_content}

# send mail
content=`cat ${smokeping_mail_content}`
echo "${content}"| mail -s "${subject}" ${email}
rm -f /tmp/smokeping_mail_content_${sendtime1}_${random}
```

# Slave
```shell
ll etc/smokeping_secrets
-rw------- 1 root root 9 Apr  9 11:12 etc/smokeping_secrets
# cat etc/smokeping_secrets
asdfghjk
```
```shell
# cat bin/smokeping.sh


#!/bin/bash
SMKEPING=/opt/smokeping/bin/smokeping
MASTERURL=http://admin:yyyy.com@10.136.47.3/smokeping.fcgi
SLAVENAME=c1-c10-120-7-200.yyyy.com
CACHEDIR=/opt/smokeping/cache
SECRET=/opt/smokeping/etc/smokeping_secrets
LOGFILE=/opt/smokeping/var/smokeping.log
PIDFILE=/opt/smokeping/var/smokeping.pid
#if [ -f $PIDFILE ] ; then
#	PID=`cat $PIDFILE`
#if kill -0 $PID 2>/dev/null ; then
#echo "smokeping is running with PID $PID"
#exit 0
#else
#echo "smokeping not running but PID file exists => delete PID file"
#rm -f $PIDFILE
#fi
#else
#echo "smokeping (no pid file) not running"
#fi
#if
 $SMKEPING --master-url=$MASTERURL --slave-name=$SLAVENAME --cache-dir=$CACHEDIR --shared-secret=$SECRET --logfile=$LOGFILE
echo "smokeping started"
#else
#echo "smokeping could not be started"
#fi
```

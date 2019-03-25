---
bg: "sea3.jpg"
layout: post
title:  "linux-Firefox"
crawlertitle: "linux-Firefox"
summary: "linux-Firefox"
date:   2019-03-25
author: snailqh
---
### java 安装
```shell
# 下载 JDK
http://www.oracle.com/technetwork/java/javase/downloads/index.html
# 安装
rpm -ivh jdk-8u111-linux-x64.rpm
# 默认路径 /usr/java/jdk1.8.0_111/
#设置环境变量
export  JAVA_HOME=/usr/java/default
export  CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export  PATH=$PATH:$JAVA_HOME/bin
```
### firefox 插件设置
```shell
ln -s /usr/java/latest/lib/amd64/libnpjp2.so /usr/lib64/mozilla/plugins/

/usr/java/default/jre/lib/amd64/libnpjp2.so /usr/lib64/mozilla/plugins/

#验证 http://java.com/zh_CN/download/installed.jsp

flash bet
http://labs.adobe.com/downloads/flashplayer.html

rpm -e jdk1.8.0_131-1.8.0_131-fcs.x86_64
cd /root
rpm -ivh jdk-9_linux-x64_bin.rpm
rm -rf /usr/lib64/mozilla/plugins/libnpjp2.so
ln -s /usr/java/default/lib/libnpjp2.so  /usr/lib64/mozilla/plugins/libnpjp2.so
```

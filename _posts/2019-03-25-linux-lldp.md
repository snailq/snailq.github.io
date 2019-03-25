---
bg: "sea3.jpg"
layout: post
title:  "lldp"
crawlertitle: "lldp"
summary: "lldp"
date:   2019-03-25
author: snailqh
---
```shell
yum install lldpad -y  
systemctl start lldpad  
lldptool set-lldp -i eth0 adminStatus=rxtx  ## 开启 eth0网卡的lldp
lldptool get-tlv -n -i "eth0"   ##获取邻居关系
```

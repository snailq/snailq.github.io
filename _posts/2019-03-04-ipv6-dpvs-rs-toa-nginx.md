---
bg: "building.jpg"
layout: post
title:  "ipv6 dpvs rs toa nginx"
crawlertitle: "ipv6 dpvs rs toa nginx"
summary: "ipv6 dpvs rs toa nginx"
date:   2019-03-04
categories: posts
author: snailqh
---
```shell
log_format  main  '$toa_remote_addr $toa_remote_port $remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
```

[nginx 1.14.0 patch][patch]

[patch]: https://github.com/iqiyi/dpvs/blob/master/kmod/toa/example_nat64/nginx/nginx-1.14.0-nat64-toa.patch

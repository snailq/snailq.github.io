---
bg: "sea3.jpg"
layout: post
title:  "Glusterfs"
crawlertitle: "Glusterfs"
summary: "Glusterfs"
date:   2019-03-15
author: snailqh
---
### 部署
```shell
# On all server:
yum install centos-release-gluster -y
yum --enablerepo=centos-gluster*-test install glusterfs-server -y
umount /dev/sdb1
mkfs.xfs -i size=512 /dev/sdb1 -f
mkdir -p /data/brick1
echo '/dev/sdb1 /data/brick1 xfs defaults 1 2' >> /etc/fstab
mount -a && mount
mkdir -p /data/brick1/gv0
echo  "alias gv='gluster volume'" >> ~/.bash_profile
# On any single server:
gluster peer probe 10.136.27.200
gluster peer probe 10.136.27.201
gluster peer probe 10.136.27.202
gluster peer probe 10.136.27.203
gluster peer status

#From any single server 添加volume gv0 复制卷模式 :
gluster volume create gv0 replica 2 10.136.27.200:/data/brick1/gv0-1 10.136.27.201:/data/brick1/gv0-2
gluster volume start gv0
gluster volume info

# 创建文件测试
mount -t glusterfs server1:/gv0 /mnt
for i in `seq -w 1 100`; do cp -rp /var/log/messages /mnt/copy-test-$i; done

# 查看状态
gluster peer status
gluster volume list
gluster volume info
gluster volume info gv0
gluster volume get all all
gluster volume status
Status of volume: gv0
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick 10.136.27.200:/data/brick1/gv0-1      49152     0          Y       286436
Brick 10.136.27.201:/data/brick1/gv0-2      49152     0          Y       290463
Self-heal Daemon on localhost               N/A       N/A        Y       286459
Self-heal Daemon on 10.136.27.202           N/A       N/A        Y       293562
Self-heal Daemon on 10.136.27.203           N/A       N/A        Y       42536
Self-heal Daemon on 10.136.27.201           N/A       N/A        Y       290486

Task Status of Volume gv0
------------------------------------------------------------------------------
```
### 扩容
```shell
Note
When expanding distributed replicated and distributed dispersed volumes, you need to add a number of bricks that is a multiple of the replica or disperse count. For example, to expand a distributed replicated volume with a replica count of 2, you need to add bricks in multiples of 2 (such as 4, 6, 8, etc.).

When shrinking distributed replicated and distributed dispersed volumes, you need to remove a number of bricks that is a multiple of the replica or stripe count. For example, to shrink a distributed replicate volume with a replica count of 2, you need to remove bricks in multiples of 2 (such as 4, 6, 8, etc.). In addition, the bricks you are trying to remove must be from the same sub-volume (the same replica or disperse set).
# 扩容添加brick
gluster volume add-brick gv0 replica 2 10.136.27.202:/data/brick1/gv0-1 10.136.27.203:/data/brick1/gv0-2
#添加后数据同步
gluster volume rebalance gv0 start
#查看同步状态
 gluster volume rebalance gv0 status
Status of volume: gv0
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick 10.136.27.200:/data/brick1/gv0-1      49152     0          Y       286436
Brick 10.136.27.201:/data/brick1/gv0-2      49152     0          Y       290463
Brick 10.136.27.202:/data/brick1/gv0-3      49152     0          Y       293765
Brick 10.136.27.203:/data/brick1/gv0-4      49152     0          Y       43075
Self-heal Daemon on localhost               N/A       N/A        Y       286828
Self-heal Daemon on 10.136.27.203           N/A       N/A        Y       43098
Self-heal Daemon on 10.136.27.202           N/A       N/A        Y       293788
Self-heal Daemon on 10.136.27.201           N/A       N/A        Y       290897

Task Status of Volume gv0
------------------------------------------------------------------------------
# 移除brick
 gluster volume remove-brick gv0 10.136.27.203:/data/brick1/gv0 start
 gluster volume remove-brick gv0 10.136.27.202:/data/brick1/gv0  10.136.27.203:/data/brick1/gv0 commit
# 客户端挂载
#lvs
10.136.102.13:/gv0  /mnt/ glusterfs defaults,noatime,direct-io-mode=disable,_netdev 0 0
#gluster
10.136.27.203:/gv0  /mnt/ glusterfs defaults,noatime,direct-io-mode=disable,_netdev,backupvolfile-server=10.136.27.201,backupvolfile-server=10.136.27.202,backupvolfile-server=10.136.27.200 0 0
```

### 故障恢复
- 将设备重新加入集群
Gluster采用UUID来标识每个gluster实例，这个信息存储在/var/lib/glusterd/glusterd.info中，因此只要恢复之前的UUID，好么Gluster集群就认为其和原来是同一设备。当然最好主机名和IP与原来一样，当然不一样也完全没有关系。
在一台正常服务器上查看故障机器的UUID：/var/lib/glusterd/peers/\*，记录下UUID的历史值。
在重装过的故障机器上，修改/var/lib/glusterd/glusterd.info的UUID为其历史值，删除除glusterd.info其他文件，重启glusterd进程。
添加信任池，执行gluster peer probe server1。重启后，正常情况就能同步到集群的peer信息。
通常加入集群后，自动就可以获得卷信息，如果没有获取到，可以执行gluster volume sync gfserver20 all来强制获取。
-  恢复volume
同理，每个卷也有一个UUID，因此只需要修改brick的属性，将卷的UUID为原来值，那么brick就认为自己属于集群和原来的brick是同一个。下面的脚本替换相应的卷名和brick后执行即可恢复brick的卷UUID
- 创建新的数据目录并挂载
mkfs.xfs -i size=512 /dev/sdb1
- 一个相同的目录
mkdir -p /data/brick1/gv0
（不可以与之前目录一样）
mkdir -p /data/brick1/gv1
重启glusterd   systemctl restart glusterd
- 挂载
mount -t glusterfs 10.136.27.203:/gv0 /mnt
- 触发自愈
mkdir /mnt/testDir001
rmdir /mnt/testDir001
setfattr -n trusted.non-existent-key -v abc /mnt
setfattr -x trusted.non-existent-key /mnt
- 查看自愈情况
gluster volume heal gv0 info
- 强制提交替换 birck目录replace brick
gluster volume replace-brick gv0 10.136.27.203:/data/brick1/gv0 10.136.27.203:/data/brick1/gv1 commit force
gluster volume status
- 同步数据
通常情况下，卷信息恢复后，复本卷会自动做修复操作。当然也可以强制开始。
gluster volume heal msdat-vol full

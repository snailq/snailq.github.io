---
bg: "tools.jpg"
layout: post
title:  "vmware vsan"
date:   2019-01-23
categories: posts
author: snailqh
---
```
esxcli vm process list
esxcli vm process kill -w 903475 -t force
esxcli vsan health cluster list
esxcli vsan health cluster get --test="Congestion"
esxcli vsan cluster leave

esxcli system  maintenanceMode  set --enable=true  --vsanmode="evacuateAllData"
esxcli system  maintenanceMode set
--enable | -e
enable maintenance mode (required)
--timeout | -t
Time to perform operation in seconds (default 0 seconds)
--vsanmode | -m
Action the VSAN service must take before the host can enter maintenance mode (default ensureObjectAccessibility). Allowed values are:
ensureObjectAccessibility: Evacuate data from the disk to ensure object accessibility in the Virtual SAN cluster, before entering maintenance mode. evacuateAllData: Evacuate all data from the disk before entering maintenance mode.
noAction: Do not move Virtual SAN data out of the disk before entering maintenance mode.
```


```
esxcli software vib install -v file:/tmp/nhpsa-2.0.14-1OEM.650.0.0.4598673.x86_64.vib --force
esxcli software vib update  -d "/vmfs/volumes/iso_10.120.10.10/ESXi650-201710001.zip"
esxcli software vib install -v file:/vmfs/volumes/iso_10.120.10.10/nhpsa-2.0.14-1OEM.650.0.0.4598673.x86_64.vib
```

```
passwd vcha
chage -m 0 -M 99999 vcha
```

```
JU408-DG24N-088Y1-V3A7M-3GA66
4A6M8-2X11K-H8528-WKCG6-CCH40
5V6ER-DHH0Q-H8541-X9A7M-07206


JU6J2-DL10Q-H89A9-HH954-8V8P0
NZ4N8-APK02-M8038-C32XH-23KK2
MY6JR-2Z011-H81W0-089E0-AZ0JD
```
### vsan中的vmdk迁移
#### Step 1:  
SSH into the host where the AppStack currently resides.
Execute the following command to clone the AppStack to block-level storage. Note that after you execute this command there are two files on the block-level storage. One is the header file, and the other is the “flat” file, which was previously integrated with VSAN as a storage object.
```
vmkfstools -d thin -i <VSAN path to App Stack>/cloudvolumes/apps/<filename>.vmdk <path to block level storage>/AppVolumes_Staging/<filename>.vmdk
```
Example:
```
vmkfstools -d thin -i /vmfs/volumes/vsan:4a65d9cbe47d44af-80f530e9e2b98ac5/76f05055-98b3-07ab-ef94-002590fd9036/apps/<filename>.vmdk /vmfs/volumes/54e5e55d-97561a60-50de-002590fd9036/AppVolumes_Staging/<filename>.vmdk
```
#### Step 2:  
Execute the following commands to copy (SCP) an AppStack from one environment to another.
```
scp <path to vmdk clone on block level storage>/<filename>.vmdk root@<esxi mgt IP>:<path to staging folder>/<filename>.vmdk
scp <path to vmdk “flat” file clone on block level storage>/<filename>-flat.vmdk root@<esxi mgt IP>:<path to staging folder>/<filename>-flat.vmdk
```
Example:  
```
scp /vmfs/volumes/54e5e55d-97561a60-50de-002590fd9036/AppVolumes_Staging/<filename>.vmdk root@10.10.10.10:/vmfs/volumes/vsan:265d91daeb2841db-82d3d8026326af8e/6efbac55-f2f7-f86a-033f-0cc47a59dc1c/Staging/<filename>.vmdk
scp /vmfs/volumes/54e5e55d-97561a60-50de-002590fd9036/AppVolumes_Staging/<filename>-flat.vmdk root@10.10.10.10:/vmfs/volumes/vsan:265d91daeb2841db-82d3d8026326af8e/6efbac55-f2f7-f86a-033f-0cc47a59dc1c/Staging/<filename>-flat.vmdk
```
#### Step 3:  
Run the commands below to delete the AppStack from the staging folder on the source environment.  
```
rm <path to staging folder>/<filename>.vmdk  
rm <path to staging folder>/<filename>-flat.vmdk  
```
Example:  
```
rm /vmfs/volumes/54e5e55d-97561a60-50de-002590fd9036/AppVolumes_Staging/<filename>.vmdk  
rm /vmfs/volumes/54e5e55d-97561a60-50de-002590fd9036/AppVolumes_Staging/<filename>-flat.vmdk  
```
Step 4:  
SSH into the host where the AppStack has been copied to. In this example the host IP address is 10.10.10.10.  
Run the command below to clone the copied AppStack from the staging folder to the App Volumes “apps” folder, and re-integrate the VMDK into VSAN as a storage object.  
```
vmkfstools -d thin -i <path to staging folder>/<filename>.vmdk <path to cloud volumes root folder>/cloudvolumes/apps/<filename>.vmdk  
```
Example:  
```
vmkfstools -d thin -i /vmfs/volumes/vsan:265d91daeb2841db-82d3d8026326af8e/6efbac55-f2f7-f86a-033f-0cc47a59dc1c/Staging/<filename>.vmdk   /vmfs/volumes/vsan:265d91daeb2841db-82d3d8026326af8e/6efbac55-f2f7-f86a-033f-0cc47a59dc1c/apps/<filename>.vmdk  
```
#### Step 5:  
Run the commands below to delete the AppStack from the staging folder on the destination environment.  
```
rm <path to staging folder>/<filename>.vmdk  
rm <path to staging folder>/<filename>-flat.vmdk  
```
Example:  
```
rm /vmfs/volumes/vsan:265d91daeb2841db-82d3d8026326af8e/6efbac55-f2f7-f86a-033f-0cc47a59dc1c/Staging/<filename>.vmdk  
rm /vmfs/volumes/vsan:265d91daeb2841db-82d3d8026326af8e/6efbac55-f2f7-f86a-033f-0cc47a59dc1c/Staging/<filename>-flat.vmdk
```
After completing these steps, you will have successfully copied a VMDK from one VSAN storage platform to another.  
App Volumes also creates a “metadata” file during the creation of an AppStack, as shown in the screenshot below.  

旧集群  
磁盘转换 到nfs上：  
vmkfstools -d thin -i /vmfs/volumes/b12b13-DS-06/c2-v06-126-155-109/c2-v06-126-155-109.vmdk /vmfs/volumes/nfs-scratch-vcsac2-b12b13/c2-v06-126-155-109.vmdk  
拷贝vmx文件到新集群  

新集群  
vsan上建立文件夹  
磁盘转换到VSAN：  
vmkfstools -d thin -i /vmfs/volumes/nfs_10.120.10.10/10.126.155.109/c2-v06-126-155-109.vmdk /vmfs/volumes/v04-ds/c2-v06-126-155-109/c2-v06-126-155-109.vmdk  
注册虚拟机，更改虚拟机配置,修改MAC  
f32f7859-7867-7e43-f63c-48df3706927c

### 重置VCSA 6.5的SSO Administrator密码  
登录到VCSA 6.5的命令行界面，输入“shell”命令激活bash shell，然后来到如下位置确认缺省的额Domain名字是什么：  
/usr/lib/vmware-vmafd/bin/vmafd-cli get-domain-name --server-name localhost  
得到了缺省的SSO Domain讯息后，执行如下命令启动vdcadmintool命令准备恢复密码  

/usr/lib/vmware-vmdir/bin/vdcadmintool  
输入字母“3”重置Account password，然后，输入UPN讯息“administrator@vsphere.local”，然后可以看到系统生成了新的密码，记录下这个新密码，之后登录到Web Client界面里修改即可；

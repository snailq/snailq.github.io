---
bg: "tools.jpg"
layout: post
title:  "服务器性能调优参数"
crawlertitle: "About server performance"
summary: "服务器性能调优参数"
date:   2019-03-14
categories: posts
author: snailq
---
### CPUfreq
所有的 CPUfreq 调控器都被内置于 kernel 工具包中，并且会被自动选定，因此若要设置 CPUfreq，您需要选择一个调控器。

您可以通过下列指令查看特定 CPU 可以使用的调控器：
cpupower frequency-info --governors
之后，您可以通过下列指令在所有 CPU 上启用这些调控器：
cpupower frequency-set --governor [governor]
若要只在特定核上启用某个调控器，请使用 -c 指令和 CPU 数的范围或是逗号分隔的清单。例如，若要为 CPU 1-3 和 5 启用用户空间调控器，指令为：
cpupower -c 1-3,5 frequency-set --governor cpufreq_userspace

微调 CPUfreq 策略和速度

选择适当的 CPUfreq 调控器后，您就可以使用 cpupower frequency-info 指令查看 CPU 速度和策略的信息，并可以使用 cpupower frequency-set 指令的选项进一步微调每一个 CPU 的速度。
cpupower frequency-info 有以下可用选项：
--freq — 根据 CPUfreq 核显示该 CPU 的当前速度，单位为 KHz。
--hwfreq — 根据硬件显示 CPU 的当前速度，单位为 KHz（仅限 root 用户）。
--driver — 显示这个 CPU 中用来设定频率的 CPUfreq 驱动器。
--governors — 显示此 kernel 上可用的 CPUfreq 调控器。若您想要使用此文件中未列出的 CPUfreq 调控器，请查看〈第 3.2.2 节 “CPUfreq 设置”〉。
--affected-cpus — 列出需要频率协调软件的 CPU。
--policy — 显示当前 CPUfreq 策略范围，单位为 KHz，以及当前活跃的调控器。
--hwlimits — 列出该 CPU 的可用频率，单位为 KHz。
cpupower frequency-set 有以下可用选项：
--min <freq> 和 --max <freq> — 设定 CPU 的 “策略限制”，单位为 KHz。

### tuned

常用模式介绍：
- balanced
它的目的是成为性能和功耗之间的折衷。它试图尽量使用自动调节。它有好的结果对于大多数负载。唯一的缺点是增加了延迟。在当前调释放它使CPU、磁盘、音频和视频插件和激活ondemand调控器。radeon_powersave设置为自动。

- latency-performance
低延迟的性能模式。它禁用电能节约机制,使sysctl设置提高延迟。CPU调节器将性能低的CPU锁定C状态(通过PM QoS)。

- throughput-performance
高吞吐量优化模式。它禁用电能节约机制,使sysctl设置提高吞吐量性能的磁盘、网络IO和转向最后期限的调度器。CPU调试器设置为性能模式。

- virtual-guest
基于企业存储配置文件,在其他任务,增加虚拟内存swappiness和减少磁盘预读值。它没有禁用磁盘屏障。

- oracle
基于throughput-performance模式，它另外禁用透明的巨大的页面和修改内核参数相关的一些其他性能。这个配置文件是由tuned-profiles-oracle包。在6.8及以后版本可用

```shell
tuned-adm active  # 显示host的当前性能模式
- Current active profile: latency-performance
tuned-adm profile latency-performance  # 切换至性能模式
tuned-adm off
/etc/tuned/active_profile # 当前的性能模式
```

### CPU: min_perf_pct
Intel 处理器 P-State（Performance States，性能状态） 的最小值，指最大化性能级别的百分比
max_perf_pct：P-State 的最大值，指可用性能的百分比
num_pstates：硬件支持的 P-State 数
查询 min_perf_pct
/sys/devices/system/cpu/intel_pstate

### x86_energy_perf_policy
```shell
x86_energy_perf_policy -r  #查看
cpu0: 0x0000000000000000 #高性能模式
cpu0: 0x0000000000000006 # 代表 normal
x86_energy_perf_policy performance
```
### 内存: transparent_hugepages
```shell
cat /sys/kernel/mm/transparent_hugepage/enabled             # 查看
echo "always" > /sys/kernel/mm/transparent_hugepage/enabled # 设置

always #尝试为任意进程分配巨页
madvise #利用 madvise() 系统调用只为个别进程分配巨页
never #禁用透明巨页
```
### 内存: vm.*
vm.dirty_background_ratio: 设置 dirty pages 开始后台回写时的百分比
vm.dirty_ratio: 设置 dirty pages 开始回写时的百分比
vm.swappiness: 控制从物理内存换出到交换空间的相对权重，取值为 0 到 100。更低的值导致避免交换，而更高的值导致尝试使用交换空间

### 磁盘: readahead
读取文件列表的内容到内存，以便当实际需要时可从缓存读取
/sys/block/sda/queue/read_ahead_kb

### 挂载参数
relatime/noatime: 对于如何更新 inode 访问时间的策略
barrier=<0|1>(barrier/nobarrier): 该选项开启或禁用在 jbd 代码中使用写入 barrier
discard/nodiscard: 控制是否执行 discard/TRIM 命令，对 SSD 设备有用

### 磁盘: scheduler
- I/O 调度器

cfq：Completely Fair Queueing（完全公平队列）调度器，它将进程分为实时、尽其所能和空闲三个独立的类别。实时类别的进程先于尽其所能类别的进程执行，而尽其所能类别的进程总是在空闲类别的进程之前执行。默认情况下分配到尽其所能类别的进程
deadline：尝试为 I/O 请求提供有保障的延迟。适用于大多数情况，尤其是读取操作比写入操作更频繁的请求
noop：执行简单的 FIFO（先进先出）调度算法，并实现请求合并。适合使用快速存储的 CPU 计算密集型系统
blk-mq：即 Multi-Queue Block IO Queuing Mechanism（多队列块 IO 队列机制），它利用具有多核的 CPU 来映射 I/O 队列到多队列。与传统的 I/O 调度器相比，通过多线程及多个 CPU 核心来分发任务，从而能够加速读写操作。该调度器适合高性能的闪存设备（如 PCIe SSD）
```shell
cat /sys/block/sda/queue/scheduler                # 查看当前使用的 I/O 调度器
echo "deadline" > /sys/block/sda/queue/scheduler  # 临时将 I/O 调度器设为 deadline
#追加 elevator=deadline 内核参数                     # 永久设置
scsi_mod.use_blk_mq=y dm_mod.use_blk_mq=y         # 注意启用 blk-mq 后，将禁用所有别的 I/O 调度器
```

### kernel.sched_*
kernel.sched_min_granularity_ns: 针对 CPU 计算密集型任务设置调度器的最小抢占粒度
kernel.sched_wakeup_granularity_ns: 设置调度器的唤醒粒度，这将延迟抢占效应，并减少过度调度
kernel.sched_migration_cost_ns: 调度器认为迁移的进程“cache hot”因而更少可能被重新迁移的总时间

### 网络: net.ipv4.{tcp_rmem,tcp_wmem,udp_mem}
``` shell
tcp_rmem #用于 autotuning 函数，设置 TCP 接收缓冲的最小、默认及最大字节数
tcp_wmen #用于 autotuning 函数，设置 TCP 发送缓冲的最小、默认及最大字节数
udp_mem #设置 UDP 队列的页数
```

### 网络: net.core.busy_{read,poll}
```
net.core.busy_read: 针对 socket 读取设置低延迟 busy poll 超时
net.core.busy_poll: 针对 poll 和 select 设置低延迟 busy poll 超时
net.ipv4.tcp_fastopen: TCP 快速打开（TFO）
```
### common
```
governor=performance
energy_perf_bias=performance
min_perf_pct=100
transparent_hugepages=always *
readahead=>4096
scheduler=deadline *
```

### throughput-performance
```
kernel.sched_min_granularity_ns = 10000000
kernel.sched_wakeup_granularity_ns = 15000000
vm.dirty_ratio = 40
vm.dirty_background_ratio = 10
vm.swappiness = 10
```

### latency-performance
```
kernel.sched_min_granularity_ns = 10000000
kernel.sched_migration_cost_ns = 5000000
vm.dirty_ratio = 10
vm.dirty_background_ratio = 3
vm.swappiness = 10
```
### network-throughput
```
在 throughput-performance 基础上增加网络调优
kernel.sched_min_granularity_ns = 10000000
kernel.sched_wakeup_granularity_ns = 15000000
vm.dirty_ratio = 40
vm.dirty_background_ratio = 10
vm.swappiness = 10
net.ipv4.tcp_rmem = 4096        87380   16777216
net.ipv4.tcp_wmem = 4096        16384   16777216
net.ipv4.udp_mem = 3145728      4194304 16777216
```
### virtual-host
```
kernel.sched_min_granularity_ns = 10000000
kernel.sched_wakeup_granularity_ns = 15000000
kernel.sched_migration_cost_ns = 5000000
vm.dirty_ratio = 40
vm.dirty_background_ratio = 5
vm.swappiness = 10
```
### virtual-guest
```
kernel.sched_min_granularity_ns = 10000000
kernel.sched_wakeup_granularity_ns = 15000000
vm.dirty_ratio = 30
vm.dirty_background_ratio = 10
vm.swappiness = 30
```

scheduler: 相比 cfq 的表现，deadline 无论在读还是在写上都更有优势。对于具有固态存储设备的场景而言，blk-mq 值得一试
kernel.sched_min_granularity_ns: 比默认值调得更大一些，推荐设为 10000000（1 毫秒），从而稍微延迟抢占，具有更好的性能表现。该参数值适合上述所有场景
kernel.sched_wakeup_granularity_ns: 比默认值调大，从而避免过度调度，推荐设为 15000000（1.5 毫秒）。仅在注重吞吐量的情况下设置该参数，低延迟的情况不要设置
kernel.sched_migration_cost_ns: 比默认值调大，从而减少任务的重新迁移，推荐设为 5000000（0.5 毫秒）。仅在注重低延迟的情况下设置该参数，高吞吐量的情况不要设置
vm.dirty_ratio: 高吞吐量的情况一般设置为 40，低延迟的情况通常设置为 10
vm.dirty_background_ratio: 高吞吐量的情况可设为 10，低延迟的情况可设为 3
vm.swappiness: 一般设为 10，从而避免过多 swap 交换。仅在作为虚拟客户机的情况下可设高一些（30）


毫无疑问 noatime 应该作为默认挂载参数，nobarrier 在写上的性能优势十分明显，discard 适合 SSD 的场合
noatime
nobarrier
discard

仅在注重网络吞吐量的情况下调节
```
net.ipv4.tcp_rmem
net.ipv4.tcp_wmem
net.ipv4.udp_mem
```
仅在注重网络低延迟的情况下调节
```
net.core.busy_read
net.core.busy_poll
net.ipv4.tcp_fastopen
```

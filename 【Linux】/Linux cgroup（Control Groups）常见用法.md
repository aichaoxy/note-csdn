
Linux cgroup（Control Groups）是一种强大的资源管理工具，它允许系统管理员对系统资源进行细粒度的控制。cgroup通过将进程分组并对这些组施加限制，从而实现资源的隔离和优先级控制。以下是一些cgroup的常见用法：

## 1. 限制资源使用：

* CPU：通过cpu子系统限制进程组的CPU使用率。
* 内存：使用memory子系统设置进程组的内存使用上限。
* 磁盘I/O：通过blkio子系统限制进程组的磁盘I/O带宽。
* 网络带宽：结合net_cls和net_prio子系统，可以控制网络带宽和优先级。

```shell
export MYGROUP=test_cg1

# 挂在cgroup文件系统
mount -t cgroup -o net_cls,net_prio cgroup /sys/fs/cgroup/net_cls
mount -t cgroup -o cpuset cgroup /sys/fs/cgroup/cpuset

# 查看已挂载的cgroup文件系统
mount -t cgroup

# 创建CGROUP组
cgcreate -g cpuset:${MYGROUP} -g cpu:${MYGROUP} -g cpuacct:${MYGROUP} -g blkio:${MYGROUP} -g memory:${MYGROUP} -g net_cls:${MYGROUP}

# CPU权重：shares=规格核数*1024
cgset -r cpu.shares=1024 ${MYGROUP}

# CPU配额：quota_us=规格核数*100000
cgset -r cpu.cfs_quota_us=100000 ${MYGROUP}

# IO权重 for kernel ver <5.4
cgset -r blkio.weight=20 ${MYGROUP}

# IO权重 for kernel ver >=5.4
cgset -r blkio.bfq.weight=20 ${MYGROUP}

# 内存配额：2GB
cgset -r memory.limit_in_bytes=2147483648 ${MYGROUP}

# 网络带宽：配置tc规则并与cgroup绑定
# 为eth0网络接口创建了一个名为1:20的class，限制其速率为1Mbps
tc qdisc add dev eth0 root handle 1: htb default 20
tc class add dev eth0 parent 1: classid 1:20 htb rate 1mbit
# 在cgroup目录下创建一个名为net_cls.classid的文件，并写入你在tc中设置的classid
echo 1:20 > /sys/fs/cgroup/net_cls/${MYGROUP}/net_cls.classid
echo: write error: invalid argument ??


# 获取PID及其所有子PID
head /path/to/datadir/postmaster.pid
6272
ps -h -o pid --ppid 6272
 6290
 6315

# 将PID加入CGROUP组
cgclassify -g cpuset,cpu,cpuacct,blkio,memory:${MYGROUP} 6272
cgclassify -g cpuset,cpu,cpuacct,blkio,memory:${MYGROUP} 6290
cgclassify -g cpuset,cpu,cpuacct,blkio,memory:${MYGROUP} 6315

# 查看一个CGROUP的资源限制
cgget -nvr cpu.cfs_quota_us ${MYGROUP}
cgget -nvr memory.limit_in_bytes ${MYGROUP}

# 删除CGROUP组
cgdelete -r cpuset:${MYGROUP} cpu:${MYGROUP} cpuacct:${MYGROUP} \
        blkio:${MYGROUP} memory:${MYGROUP} net_cls:${MYGROUP}
```
## 2. 监控资源使用

* 使用cpuacct和memory子系统来监控进程组的资源使用情况。

```shell
export MYGROUP=test_cg1

# 读取cgroup的cpuacct.usage，单位纳秒。一般取两次差值/自然时间差作为CPU百分比
cgget -n -v -r cpuacct.usage ${MYGROUP}

# 读取cgroup的memory.stat里rss字段，作为内存占用，单位B
cgget -r memory.stat ${MYGROUP} | grep -w 'rss' | awk '{print $2}'
```

## 3. 进程隔离

* 使用`cpuset`子系统为进程组分配独立的CPU核心和内存节点，实现物理资源的隔离。

```shell
export MYGROUP=test_cg1

# 假设你的系统有4个CPU核心（0, 1, 2, 3），你想要为 MYGROUP 分配CPU核心1和2。
cgset -r cpuset.cpus=0-1 ${MYGROUP}

# 如果你的系统有多个NUMA节点，你可以为 MYGROUP 分配特定的内存节点。
# 这里假设只有一个NUMA节点（节点0），如果有更多的节点，
# 你可以使用逗号分隔的列表来指定多个节点，例如 0,2
cgset -r cpuset.mems=0 ${MYGROUP}

```

## 4. 使用cgroup事件通知

* 配置cgroup.event_control来接收资源使用达到限制时的通知。

> 在较新的 Linux 内核版本中，`cgroup.event_control` 文件可能不存在，或者其功能已经被集成到了其他机制中。在早期的 cgroup 实现中，`cgroup.event_control` 用于配置事件通知，但在后续的版本中，这些功能可能已经被废弃或替换。

> 在现代的 cgroup 实现中，事件通知通常是通过其他机制来实现的，例如使用 `auditd`（Linux 审计守护进程）或者 `cgroup2`（cgroup 的下一代版本）中的事件通知功能。

```shell
export MYGROUP=test_cg1

# 假设我们想要设置一个名为 MYGROUP 的 cgroup 的 CPU 使用率限制。
# 我们可以使用 cgset 来设置 CPU 配额和周期。
cgset -r cpu.cfs_quota_us=50000 ${MYGROUP}
cgset -r cpu.cfs_period_us=100000 ${MYGROUP}
# 这里，cpu.cfs_quota_us 是 cgroup 的 CPU 使用配额，cpu.cfs_period_us 是周期。
# 设置配额为 50000（即 50% 的 CPU 时间），周期为 100000 微秒（即 100 毫秒）。

# 设置内存限制 2GB
cgset -r memory.limit_in_bytes=2147483648 ${MYGROUP}

# 手动设置 cgroup.event_control
# 对于 cgroup.event_control 的设置，你需要直接编辑相应的文件。
# 这个文件通常位于 /sys/fs/cgroup/cpu/${MYGROUP}/cgroup.event_control。
echo 'cpu.usage > 70%' > /sys/fs/cgroup/cpu/${MYGROUP}/cgroup.event_control
# 这会在 CPU 使用率超过 70% 时触发一个事件。
```
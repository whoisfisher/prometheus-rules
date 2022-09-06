##### 内存告警，当容器占用的内存大于节点内存的85%

```
alert: ContainerMemoryUsage
expr: sum by(kubernetes_io_hostname) (container_memory_working_set_bytes{id="/"})
  / sum by(kubernetes_io_hostname) (machine_memory_bytes) * 100 > 85
for: 2m
labels:
  severity: critical
annotations:
  DESCRIPTION: Container memory total usage over 85% .
  SUMMARY: 'NODE: {<!-- -->{ $labels.kubernetes_io_hostname }} , container memory total
    usage: {<!-- -->{ $value }}%'
```

##### cpu使用量告警，当容器的cpu大于节点cpu的40%

```
alert: containerCpuUsage
expr: sum by(kubernetes_io_hostname) (rate(container_cpu_usage_seconds_total{id="/"}[4m]))
  / sum by(kubernetes_io_hostname) (machine_cpu_cores) * 100 > 40
for: 2m
labels:
  severity: critical
annotations:
  DESCRIPTION: Container cpu total usage over 20% .
  SUMMARY: 'NODE: {<!-- -->{ $labels.kubernetes_io_hostname }} , container cpu total usage:
    {<!-- -->{ $value }}%'
```

##### 容器告警，当k8s启动一个容器

```
alert: KubeStartContainer
expr: (rate(kubelet_docker_operations{operation_type="start_container"}[5m]) >
  0)
for: 2m
labels:
  severity: critical
annotations:
  DESCRIPTION: '{<!-- -->{$labels.instance}}: start container'
  SUMMARY: '{<!-- -->{$labels.instance}}: start container'
```

##### 容器告警，当k8s删除一个容器

```
  - alert: KubeRemoveContainer
    expr: (rate(kubelet_docker_operations{operation_type="remove_container"}[5m]) > 0)
    for: 2m
    labels:
      severity: critical
    annotations:
      DESCRIPTION: '{<!-- -->{$labels.instance}}: remove container'
      SUMMARY: '{<!-- -->{$labels.instance}}: remove container'
```



##### 磁盘告警，当磁盘空间少于10%

```
alert: LowDiskSpace
expr: node_filesystem_avail{fstype=~"ext.|xfs",job="kubernetes-service-endpoints"}
  / node_filesystem_size{fstype=~"ext.|xfs",job="kubernetes-service-endpoints"}
  * 100 <= 10
for: 2m
labels:
  severity: critical
annotations:
  DESCRIPTION: low disk space .
  SUMMARY: 'Really low disk space left on {<!-- -->{ $labels.mountpoint }} on {<!-- -->{ if $labels.fqdn
    }}{<!-- -->{ $labels.fqdn }}{<!-- -->{ else }}{<!-- -->{ $labels.instance }}{<!-- -->{ end }}: {<!-- -->{ $value }}%'

```

##### 节点告警，当节点内存少于75%

```
alert: NodeMemoryUsage
expr: (((node_memory_MemTotal - node_memory_MemFree - node_memory_Cached) / (node_memory_MemTotal)

100)) > 75
for: 2m
labels:
  severity: critical
annotations:
  DESCRIPTION: '{<!-- -->{$labels.instance}}: Memory usage is above 75% (current value
is: {<!-- -->{ $value }})'
  SUMMARY: '{<!-- -->{$labels.instance}}: High memory usage detected'
```

##### 节点负载告警，当5分钟的load大于1

```
alert: NodeLoadAverage
expr: ((node_load5 / count without(cpu, mode) (node_cpu{mode="system"})) > 1)
for: 2m
labels:
  severity: critical
annotations:
  DESCRIPTION: Load is high.
  SUMMARY: '{<!-- -->{$labels.instance}}: High Load : {<!-- -->{ $value }}'
```



## 主机和硬件监控

### 可用内存指标

主机中可用内存容量不足 10%

```bash
  - alert: HostOutOfMemory
    expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host out of memory (instance {{ $labels.instance }})
      description: Node memory is filling up (< 10% left)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 内存

节点内存压力大。主要页面故障率高

```bash
  - alert: HostMemoryUnderMemoryPressure
    expr: rate(node_vmstat_pgmajfault[1m]) > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host memory under memory pressure (instance {{ $labels.instance }})
      description: The node is under heavy memory pressure. High rate of major page faults
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机网络接口流入流量异常

主机网络接口可能接收了太多的数据（> 100 MB/s）。阀值根据自己机器背板网卡决定

```bash
  - alert: HostUnusualNetworkThroughputIn
    expr: sum by (instance) (rate(node_network_receive_bytes_total[2m])) / 1024 / 1024 > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host unusual network throughput in (instance {{ $labels.instance }})
      description: Host network interfaces are probably receiving too much data (> 100 MB/s)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机网络接口流出流量异常

主机网络接口可能发送了太多的数据（> 100 MB/s）。

```bash
  - alert: HostUnusualNetworkThroughputOut
    expr: sum by (instance) (rate(node_network_transmit_bytes_total[2m])) / 1024 / 1024 > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host unusual network throughput out (instance {{ $labels.instance }})
      description: Host network interfaces are probably sending too much data (> 100 MB/s)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机网络接收错误

{{ $labels.instance }}接口{{ $labels.device }}在过去5分钟内遇到{{ printf "%.0f" $value }}接收错误。

```bash
  - alert: HostNetworkReceiveErrors
    expr: increase(node_network_receive_errs_total[5m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host Network Receive Errors (instance {{ $labels.instance }})
      description: {{ $labels.instance }} interface {{ $labels.device }} has encountered {{ printf "%.0f" $value }} receive errors in the last five minutes.
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机网络传输错误

{{ $labels.instance }} 接口 {{ $labels.device }} 在过去五分钟内遇到 {{ printf "%.0f" $value }} 发送错误。

```bash
  - alert: HostNetworkTransmitErrors
    expr: increase(node_network_transmit_errs_total[5m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host Network Transmit Errors (instance {{ $labels.instance }})
      description: {{ $labels.instance }} interface {{ $labels.device }} has encountered {{ printf "%.0f" $value }} transmit errors in the last five minutes.
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机磁盘读速率

磁盘每秒读数据（> 50 MB/s）。

```bash
  - alert: HostUnusualDiskReadRate
    expr: sum by (instance) (rate(node_disk_read_bytes_total[2m])) / 1024 / 1024 > 50
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host unusual disk read rate (instance {{ $labels.instance }})
      description: Disk is probably reading too much data (> 50 MB/s)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机磁盘写速率

磁盘每秒写数据

```bash
  - alert: HostUnusualDiskWriteRate
    expr: sum by (instance) (rate(node_disk_written_bytes_total[2m])) / 1024 / 1024 > 50
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host unusual disk write rate (instance {{ $labels.instance }})
      description: Disk is probably writing too much data (> 50 MB/s)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机磁盘剩余空间

磁盘可用空间（<10% left）

```bash
  # please add ignored mountpoints in node_exporter parameters like
  # "--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|run)($|/)"
  - alert: HostOutOfDiskSpace
    expr: (node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host out of disk space (instance {{ $labels.instance }})
      description: Disk is almost full (< 10% left)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 根据磁盘目前的增长速度，在几个小时内是否会写满

根据当前一小时内磁盘增长量，判断磁盘在 4 个小时内会不会被写满

```bash
  - alert: HostDiskWillFillIn4Hours
    expr: predict_linear(node_filesystem_free_bytes{fstype!~"tmpfs"}[1h], 4 * 3600) < 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host disk will fill in 4 hours (instance {{ $labels.instance }})
      description: Disk will fill in 4 hours at current write rate
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机中inode 文件句柄报警

磁盘可用的inode快用完了（<10%）。

```bash
  - alert: HostOutOfInodes
    expr: node_filesystem_files_free{mountpoint ="/rootfs"} / node_filesystem_files{mountpoint ="/rootfs"} * 100 < 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host out of inodes (instance {{ $labels.instance }})
      description: Disk is almost running out of available inodes (< 10% left)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 磁盘读延迟

磁盘读取延迟大（读取操作>100ms）

```bash
  - alert: HostUnusualDiskReadLatency
    expr: rate(node_disk_read_time_seconds_total[1m]) / rate(node_disk_reads_completed_total[1m]) > 0.1 and rate(node_disk_reads_completed_total[1m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host unusual disk read latency (instance {{ $labels.instance }})
      description: Disk latency is growing (read operations > 100ms)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 磁盘写入延迟大

磁盘写入延迟大（写操作>100ms）

```bash
  - alert: HostUnusualDiskWriteLatency
    expr: rate(node_disk_write_time_seconds_total[1m]) / rate(node_disk_writes_completed_total[1m]) > 0.1 and rate(node_disk_writes_completed_total[1m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host unusual disk write latency (instance {{ $labels.instance }})
      description: Disk latency is growing (write operations > 100ms)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机 cpu 负载高

cpu 负载大于 > 80%

```bash
  - alert: HostHighCpuLoad
    expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host high CPU load (instance {{ $labels.instance }})
      description: CPU load is > 80%
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机上下文切换

上下文切换的节点越来越多(>1000/s)

```bash
  # 1000 context switches is an arbitrary number.
  # Alert threshold depends on nature of application.
  # Please read: https://github.com/samber/awesome-prometheus-alerts/issues/58
  - alert: HostContextSwitching
    expr: (rate(node_context_switches_total[5m])) / (count without(cpu, mode) (node_cpu_seconds_total{mode="idle"})) > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host context switching (instance {{ $labels.instance }})
      description: Context switching is growing on node (> 1000 / s)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机 swap 分区使用

主机 swap 交换分区使用情况 (> 80%)

```bash
  - alert: HostSwapIsFillingUp
    expr: (1 - (node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes)) * 100 > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host swap is filling up (instance {{ $labels.instance }})
      description: Swap is filling up (>80%)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机 systemctl 管理的服务 down 了

主机上systemctl 管理的服务不正常，failed了，根据自己的实际情况来判断哪些服务

```bash
  - alert: HostSystemdServiceCrashed
    expr: node_systemd_unit_state{state="failed"} == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host SystemD service crashed (instance {{ $labels.instance }})
      description: SystemD service crashed
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机物理元设备(有的虚拟机可能没有此指标)

物理机温度过高

```bash
  - alert: HostPhysicalComponentTooHot
    expr: node_hwmon_temp_celsius > 75
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host physical component too hot (instance {{ $labels.instance }})
      description: Physical hardware component too hot
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机节点超温报警(有的虚拟机可能没有此指标)

触发物理节点温度报警

```bash
  - alert: HostNodeOvertemperatureAlarm
    expr: node_hwmon_temp_alarm == 1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Host node overtemperature alarm (instance {{ $labels.instance }})
      description: Physical node temperature alarm triggered
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机RAID 卡阵列失效(虚拟机可能没有此指标)

RAID阵列{{$labels.device }}由于一个或多个磁盘故障而处于退化状态。备用硬盘的数量不足以自动修复问题。

```bash
  - alert: HostRaidArrayGotInactive
    expr: node_md_state{state="inactive"} > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Host RAID array got inactive (instance {{ $labels.instance }})
      description: RAID array {{ $labels.device }} is in degraded state due to one or more disks failures. Number of spare drives is insufficient to fix issue automatically.
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机RAID磁盘故障(虚拟机可能没有此指标)

在{{ $labels.instance }} 的RAID阵列中至少有一个设备失败。阵列{{ $labels.md_device }}需要注意，可能需要进行磁盘更换

```bash
  - alert: HostRaidDiskFailure
    expr: node_md_disks{state="failed"} > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host RAID disk failure (instance {{ $labels.instance }})
      description: At least one device in RAID array on {{ $labels.instance }} failed. Array {{ $labels.md_device }} needs attention and possibly a disk swap
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 主机内核版本偏差

不同的内核版本正在运行

```bash
  - alert: HostKernelVersionDeviations
    expr: count(sum(label_replace(node_uname_info, "kernel", "$1", "release", "([0-9]+.[0-9]+.[0-9]+).*")) by (kernel)) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host kernel version deviations (instance {{ $labels.instance }})
      description: Different kernel versions are running
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 检测主机 OOM 杀进程

```bash
  - alert: HostOomKillDetected
    expr: increase(node_vmstat_oom_kill[5m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host OOM kill detected (instance {{ $labels.instance }})
      description: OOM kill detected
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 检测到主机EDAC可纠正的错误

{{ $labels.instance }}在过去5分钟内，EDAC报告了{{ printf "%.0f" $value }}可纠正的内存错误。

```bash
  - alert: HostEdacCorrectableErrorsDetected
    expr: increase(node_edac_correctable_errors_total[5m]) > 0
    for: 5m
    labels:
      severity: info
    annotations:
      summary: Host EDAC Correctable Errors detected (instance {{ $labels.instance }})
      description: {{ $labels.instance }} has had {{ printf "%.0f" $value }} correctable memory errors reported by EDAC in the last 5 minutes.
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 检测到主机EDAC不正确的错误

{{ $labels.instance }}在过去5分钟内，EDAC报告了{{ printf "%.0f" $value }}不可纠正的内存错误。

```bash
  - alert: HostEdacUncorrectableErrorsDetected
    expr: node_edac_uncorrectable_errors_total > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Host EDAC Uncorrectable Errors detected (instance {{ $labels.instance }})
      description: {{ $labels.instance }} has had {{ printf "%.0f" $value }} uncorrectable memory errors reported by EDAC in the last 5 minutes.
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

## Docker 容器

### 一个容器消失

```bash
  - alert: ContainerKilled
    expr: time() - container_last_seen > 60
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Container killed (instance {{ $labels.instance }})
      description: A container has disappeared
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 容器 cpu 的使用量

容器CPU使用率超过80%。

```bash
  # cAdvisor有时会消耗大量的CPU，所以这个警报会不断地响起。
  # If you want to exclude it from this alert, just use: container_cpu_usage_seconds_total{name!=""}
  - alert: ContainerCpuUsage
    expr: (sum(rate(container_cpu_usage_seconds_total[3m])) BY (instance, name) * 100) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Container CPU usage (instance {{ $labels.instance }})
      description: Container CPU usage is above 80%
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 容器内存的使用量

容器内存使用率超过 80%。

```bash
  # See https://medium.com/faun/how-much-is-too-much-the-linux-oomkiller-and-used-memory-d32186f29c9d
  - alert: ContainerMemoryUsage
    expr: (sum(container_memory_working_set_bytes) BY (instance, name) / sum(container_spec_memory_limit_bytes > 0) BY (instance, name) * 100) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Container Memory usage (instance {{ $labels.instance }})
      description: Container Memory usage is above 80%
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### 容器磁盘的使用量

容器磁盘使用量超过 80%

```bash
  - alert: ContainerVolumeUsage
    expr: (1 - (sum(container_fs_inodes_free) BY (instance) / sum(container_fs_inodes_total) BY (instance)) * 100) > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Container Volume usage (instance {{ $labels.instance }})
      description: Container Volume usage is above 80%
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

## Redis 相关报警信息

### redis down

redis 服务 down 了，报警

```bash
  - alert: RedisDown
    expr: redis_up == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Redis down (instance {{ $labels.instance }})
      description: Redis instance is down
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### redis 缺少主节点(集群，或者sentinel 模式才有)

redis 集群中缺少标记的主节点

```bash
  - alert: RedisMissingMaster
    expr: (count(redis_instance_info{role="master"}) or vector(0)) < 1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Redis missing master (instance {{ $labels.instance }})
      description: Redis cluster has no node marked as master.
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Redis 主节点过多

redis 集群中被标记的主节点过多

```bash
  - alert: RedisTooManyMasters
    expr: count(redis_instance_info{role="master"}) > 1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Redis too many masters (instance {{ $labels.instance }})
      description: Redis cluster has too many nodes marked as master.
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Redis 复制中断

Redis实例丢失了一个slave

```bash
  - alert: RedisReplicationBroken
    expr: delta(redis_connected_slaves[1m]) < 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Redis replication broken (instance {{ $labels.instance }})
      description: Redis instance lost a slave
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Redis 集群 flapping

在Redis副本连接中检测到变化。当复制节点失去与主节点的连接并重新连接（也就是flapping）时，会发生这种情况。

```bash
  - alert: RedisClusterFlapping
    expr: changes(redis_connected_slaves[5m]) > 2
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Redis cluster flapping (instance {{ $labels.instance }})
      description: Changes have been detected in Redis replica connection. This can occur when replica nodes lose connection to the master and reconnect (a.k.a flapping).
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Redis缺少备份

Redis已经有24小时没有备份了。

```bash
  - alert: RedisMissingBackup
    expr: time() - redis_rdb_last_save_timestamp_seconds > 60 * 60 * 24
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Redis missing backup (instance {{ $labels.instance }})
      description: Redis has not been backuped for 24 hours
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Redis内存不足

Redis内存耗尽（>90%）。

```bash
#需要 redis 实例设置 maxmemory maxmemory-policy 最大使用内存参数
- alert: RedisOutOfMemory
    expr: redis_memory_used_bytes / redis_total_system_memory_bytes * 100 > 90
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Redis out of memory (instance {{ $labels.instance }})
      description: Redis is running out of memory (> 90%)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Redis连接数过多

Redis实例有太多的连接

```bash
  - alert: RedisTooManyConnections
    expr: redis_connected_clients > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Redis too many connections (instance {{ $labels.instance }})
      description: Redis instance has too many connections
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Redis连接数不足

Redis实例应该有更多的连接（> 5）。

```bash
  - alert: RedisNotEnoughConnections
    expr: redis_connected_clients < 5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Redis not enough connections (instance {{ $labels.instance }})
      description: Redis instance should have more connections (> 5)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Redis拒绝连接

一些与Redis的连接已被拒绝

```bash
  - alert: RedisRejectedConnections
    expr: increase(redis_rejected_connections_total[1m]) > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Redis rejected connections (instance {{ $labels.instance }})
      description: Some connections to Redis has been rejected
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

## rabbitmq 监控 : [rabbitmq/rabbitmq-prometheus ]

### rabbitmq 节点 down

节点数量少于 1 个

```bash
  - alert: RabbitmqNodeDown
    expr: sum(rabbitmq_build_info) < 3
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Rabbitmq node down (instance {{ $labels.instance }})
      description: Less than 3 nodes running in RabbitMQ cluster
  VALUE = {{ $value }}
  LABELS: {{ $labels }}  - alert: RabbitmqDown
    expr: rabbitmq_up == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Rabbitmq down (instance {{ $labels.instance }})
      description: RabbitMQ node down
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Rabbitmq实例的不同版本

在同一集群中运行不同版本的Rabbitmq，可能会导致失败。

```bash
  - alert: RabbitmqInstancesDifferentVersions
    expr: count(count(rabbitmq_build_info) by (rabbitmq_version)) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq instances different versions (instance {{ $labels.instance }})
      description: Running different version of Rabbitmq in the same cluster, can lead to failure.
  VALUE = {{ $value }}
  LABELS: {{ $labels }}  - alert: RabbitmqClusterPartition
    expr: rabbitmq_partitions > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Rabbitmq cluster partition (instance {{ $labels.instance }})
      description: Cluster partition
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Rabbitmq内存高

一个节点使用了90%以上的内存分配。

```bash
  - alert: RabbitmqMemoryHigh
    expr: rabbitmq_process_resident_memory_bytes / rabbitmq_resident_memory_limit_bytes * 100 > 90
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq memory high (instance {{ $labels.instance }})
      description: A node use more than 90% of allocated RAM
  VALUE = {{ $value }}
  LABELS: {{ $labels }}  - alert: RabbitmqOutOfMemory
    expr: rabbitmq_node_mem_used / rabbitmq_node_mem_limit * 100 > 90
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq out of memory (instance {{ $labels.instance }})
      description: Memory available for RabbmitMQ is low (< 10%)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Rabbitmq文件描述符的用法

一个节点使用90%以上的文件描述符。

```bash
  - alert: RabbitmqFileDescriptorsUsage
    expr: rabbitmq_process_open_fds / rabbitmq_process_max_fds * 100 > 90
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq file descriptors usage (instance {{ $labels.instance }})
      description: A node use more than 90% of file descriptors
  VALUE = {{ $value }}
  LABELS: {{ $labels }}  - alert: RabbitmqTooManyConnections
    expr: rabbitmq_connectionsTotal > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq too many connections (instance {{ $labels.instance }})
      description: RabbitMQ instance has too many connections (> 1000)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Rabbitmq连接数太多

节点的总连接数过高。

```bash
  - alert: RabbitmqTooMuchConnections
    expr: rabbitmq_connections > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq too much connections (instance {{ $labels.instance }})
      description: The total connections of a node is too high
  VALUE = {{ $value }}
  LABELS: {{ $labels }}  - alert: RabbitmqTooManyMessagesInQueue
    expr: rabbitmq_queue_messages_ready{queue="my-queue"} > 1000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq too many messages in queue (instance {{ $labels.instance }})
      description: Queue is filling up (> 1000 msgs)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Rabbitmq无队列消费

一个队列的消费者少于1个

```bash
  - alert: RabbitmqNoQueueConsumer
    expr: rabbitmq_queue_consumers < 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq no queue consumer (instance {{ $labels.instance }})
      description: A queue has less than 1 consumer
  VALUE = {{ $value }}
  LABELS: {{ $labels }}  - alert: RabbitmqSlowQueueConsuming
    expr: time() - rabbitmq_queue_head_message_timestamp{queue="my-queue"} > 60
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq slow queue consuming (instance {{ $labels.instance }})
      description: Queue messages are consumed slowly (> 60s)
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```

### Rabbitmq不可路由的消息

一个队列有不可更改的消息

```bash
  - alert: RabbitmqUnroutableMessages
    expr: increase(rabbitmq_channel_messages_unroutable_returned_total[5m]) > 0 or increase(rabbitmq_channel_messages_unroutable_dropped_total[5m]) > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: Rabbitmq unroutable messages (instance {{ $labels.instance }})
      description: A queue has unroutable messages
  VALUE = {{ $value }}
  LABELS: {{ $labels }}  - alert: RabbitmqNoConsumer
    expr: rabbitmq_queue_consumers == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Rabbitmq no consumer (instance {{ $labels.instance }})
      description: Queue has no consumer
  VALUE = {{ $value }}
  LABELS: {{ $labels }}
```





```
groups:
- name: Host
 rules:
 - alert: HostMemory Usage
   expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / node_memory_MemTotal_bytes * 100 >  90
   for: 1m
   labels:
     name: Memory
     severity: Warning
   annotations:
     summary: " {{ $labels.appname }} "
     description: "宿主机内存使用率超过90%."
     value: "{{ $value }}"
 - alert: HostCPU Usage
   expr: sum(avg without (cpu)(irate(node_cpu_seconds_total{mode!='idle'}[5m]))) by (instance,appname) > 0.8
   for: 1m
   labels:
     name: CPU
     severity: Warning
   annotations:
     summary: " {{ $labels.appname }} "
     description: "宿主机CPU使用率超过80%."
     value: "{{ $value }}"
 - alert: HostLoad
   expr: node_load5 > 20
   for: 1m
   labels:
     name: Load
     severity: Warning
   annotations:
     summary: "{{ $labels.appname }} "
     description: " 主机负载5分钟超过20."
     value: "{{ $value }}"
 - alert: HostFilesystem Usage
   expr: (node_filesystem_size_bytes-node_filesystem_free_bytes)/node_filesystem_size_bytes*100>80
   for: 1m
   labels:
     name: Disk
     severity: Warning
   annotations:
     summary: " {{ $labels.appname }} "
     description: " 宿主机 [ {{ $labels.mountpoint }} ]分区使用超过80%."
     value: "{{ $value }}%"
 - alert: HostDiskio writes
   expr: irate(node_disk_writes_completed_total{job=~"Host"}[1m]) > 10
   for: 1m
   labels:
     name: Diskio
     severity: Warning
   annotations:
     summary: " {{ $labels.appname }} "
     description: " 宿主机 [{{ $labels.device }}]磁盘1分钟平均写入IO负载较高."
     value: "{{ $value }}iops"
 - alert: HostDiskio reads
   expr: irate(node_disk_reads_completed_total{job=~"Host"}[1m]) > 10
   for: 1m
   labels:
     name: Diskio
     severity: Warning
   annotations:
     summary: " {{ $labels.appname }} "
     description: " 宿机 [{{ $labels.device }}]磁盘1分钟平均读取IO负载较高."
     value: "{{ $value }}iops"
 - alert: HostNetwork_receive
   expr: irate(node_network_receive_bytes_total{device!~"lo|bond[0-9]|cbr[0-9]|veth.*|virbr.*|ovs-system"}[5m]) / 1048576  > 10
   for: 1m
   labels:
     name: Network_receive
     severity: Warning
   annotations:
     summary: " {{ $labels.appname }} "
     description: " 宿主机 [{{ $labels.device }}] 网卡5分钟平均接收流量超过10Mbps."
     value: "{{ $value }}3Mbps"
 - alert: hostNetwork_transmit
   expr: irate(node_network_transmit_bytes_total{device!~"lo|bond[0-9]|cbr[0-9]|veth.*|virbr.*|ovs-system"}[5m]) / 1048576  > 10
   for: 1m
   labels:
     name: Network_transmit
     severity: Warning
   annotations:
     summary: " {{ $labels.appname }} "
     description: " 宿主机 [{{ $labels.device }}] 网卡5分钟内平均发送流量超过10Mbps."
     value: "{{ $value }}3Mbps"
```


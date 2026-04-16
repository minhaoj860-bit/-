# 🖥️ Linux SRE 面试实战：Load 高但 CPU% 低

---

**场景**: 有一台 Ubuntu 服务器，执行命令非常慢，执行 `ls` 都要等 3 秒。

**考察维度**: CPU 上下文切换 | 磁盘 I/O 压力 | 内存 Swap 交换 | Docker 容器定位 | /proc 定位真凶

---

| 题目 | 考察点 | 状态 |
|------|--------|------|
| Q1 | ls 慢的原因分析 | ⬜ |
| Q2 | %util 高但 wa% 低的含义 | ⬜ |
| Q3 | si/so 单位是什么 | ⬜ |
| Q4 | Docker 容器卡顿定位 | ⬜ |
| Q5 | /proc 定位真凶进程 | ⬜ |

---

# 📚 核心概念：Load vs CPU%

```
Load Average = Runnable(R) + Uninterruptible Sleep(D) 进程数
CPU% = 只统计 R 状态的计算部分
```

**关键结论**: D 状态的进程不占 CPU，但会拉高 Load！

| 状态 | 名称 | 占用 CPU？ | 拉高 Load？ | 可被 kill 吗？ |
|:----:|------|:----------:|:-----------:|:--------------:|
| R | Running/Runnable | ✅ | ✅ | ✅ |
| D | Uninterruptible Sleep | ❌ | ✅ | ❌ (kill -9 也杀不掉) |
| S | Interruptible Sleep | ❌ | ❌ | ✅ |

---

# 第一部分：vmstat 全面解析

## 1.1 快速入门

```bash
# 基础命令
vmstat 1  # 每秒刷新一次
```

## 1.2 输出字段详解

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si    so    bi    bo    in    cs    us sy id wa st
 2  0      0  12345  67890 234567   0     0    123   456   789  1234    5  2 90  3  0
```

| 列 | 名称 | 含义 | 危险阈值 |
|:--:|------|------|----------|
| **r** | runnable | 等待 CPU 的进程数 | > CPU 核心数 |
| **b** | blocked | 不可中断睡眠的进程数 (D状态) | > 0 持续 |
| **si** | swap in | 磁盘→内存 (KB/s) | > 0 持续 |
| **so** | swap out | 内存→磁盘 (KB/s) | > 0 持续 |
| **cs** | context switch | 每秒上下文切换次数 | > 100000 |
| **in** | interrupt | 每秒中断次数 | > 1000 |
| **wa** | wait | CPU 等待 I/O 的比例 | > 20% |

---

# 第二部分：CPU 上下文切换

## 2.1 什么是上下文切换？

当 CPU 从运行一个进程切换到另一个进程时，会发生上下文切换：

```
进程 A (CPU) → 保存寄存器/状态 → 进程 B (CPU)
            → 恢复寄存器/状态
```

**成本**: 每次切换大约 10-30 微秒，频繁切换会拖慢系统

## 2.2 如何查看？

### 方法 1: vmstat

```bash
vmstat 1
# cs 列 = 每秒上下文切换次数
```

### 方法 2: pidstat (需要安装 sysstat)

```bash
# 看每个进程的上下文切换
pidstat -w 1

# 输出:
# UID       PID    cswch/s nvcswch/s  Command
#   0         1      2.34      0.01  systemd
# cswch/s = 自愿切换 (进程主动让出 CPU)
# nvcswch/s = 非自愿切换 (被调度器抢走 CPU)
```

## 2.3 排查命令速查

```bash
# 查看系统级上下文切换
vmstat 1

# 查看每个进程的上下文切换
pidstat -w 1

# 查看 CPU 核心数 (对比 r 列)
nproc
grep 'processor' /proc/cpuinfo | wc -l
```

**判断标准**:
- `cs` > 100000 次/秒 → 上下文切换过多
- `r` > CPU 核心数 → 进程在排队等 CPU

---

# 第三部分：磁盘 I/O 压力

## 3.1 iostat 快速入门

```bash
# 查看磁盘 I/O 统计
iostat -xz 1
```

## 3.2 核心指标解释

| 指标 | 含义 | 危险阈值 |
|:----:|------|----------|
| **%util** | 磁盘忙碌时间百分比 | > 80% 持续 |
| **await** | 平均 I/O 等待时间 (ms) | > 20ms (SSD) / > 100ms (HDD) |
| **avgqu-sz** | 平均 I/O 队列长度 | > 1 (SSD) / > 4 (HDD) |
| **r/s, w/s** | 每秒读写次数 | 根据磁盘类型判断 |
| **wa%** | CPU 等待 I/O 时间占比 | > 20% |

## 3.3 %util 高但 wa% 低的含义

**四象限判断法**:

```
                    %util 高
                       │
        ┌──────────────┼──────────────┐
        │   【场景 1】  │   【场景 2】  │
        │  磁盘饱和     │  异步 I/O     │
        │  %util 高    │  %util 高    │
  wa%   │  wa% 高      │  wa% 低      │
  高    │  → CPU在等   │  → 内核线程  │
        │              │   处理I/O    │
        ├──────────────┼──────────────┤
        │   【场景 3】  │   【场景 4】  │
        │  磁盘空闲     │  正常空闲     │
        │  %util 低    │  %util 低    │
  wa%   │  wa% 高      │  wa% 低      │
  低    │  → 可能有    │  → 完全正常   │
        │   统计问题    │              │
        └──────────────┼──────────────┘
                       │
                    %util 低
```

### 你的场景: %util 高 + wa% 低

**可能原因**:
1. **Direct I/O**: 绕过 page cache，直接读写磁盘
2. **内核线程处理 I/O**: pdflush/flush 线程在做 I/O，不在 CPU 统计里
3. **RAID 卡缓存**: RAID 卡 BBU 充电期间
4. **SSD 固件活动**: 垃圾回收(Wear Leveling)
5. **NVMe 特性**: NVMe 设备的 I/O 完成不在 wa% 中统计

## 3.4 定位 I/O 真凶

```bash
# 方法1: iotop (需要 root)
iotop -o  # 只显示有 I/O 活动的进程

# 方法2: 查看进程的 I/O 统计
cat /proc/1/io  # 查看 PID 1 的 I/O

# 方法3: pidstat
pidstat -d 1  # 看每个进程的磁盘 I/O
```

---

# 第四部分：内存 Swap

## 4.1 Minor vs Major Page Fault

```
┌─────────────────────────────────────────────────────┐
│                   进程的虚拟地址空间                  │
│  ┌──────────────────────────────────────────────┐  │
│  │              代码/数据段                      │  │
│  │  ┌─────────┐                                │  │
│  │  │ Page A  │ ← 在物理内存中                  │  │
│  │  ├─────────┤     Minor Fault                │  │
│  │  │ Page B  │ ← 不在内存，需要分配            │  │
│  │  └─────────┘                                │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
                      ↓
         Major Fault: 需要从磁盘读取
         (如 swap 文件、mmap 文件)
```

| 类型 | 含义 | 速度影响 |
|:----:|------|----------|
| **Minor Page Fault** | 页在内存中，只是映射到页表 | 微秒级，无感知 |
| **Major Page Fault** | 页不在内存，需要从磁盘读取 | **毫秒~秒级，会卡顿** |

## 4.2 排查 Swap 问题

```bash
# 查看 swap 状态
swapon -s

# vmstat 看 si/so
vmstat 1

# 查看哪些进程在用 swap
for f in /proc/*/status; do awk '/VmSwap/{if($2>0)print FILENAME,$2} END {close(FILENAME)}' $f 2>/dev/null; done | sort -k2 -rn | head
```

## 4.3 调优参数

```bash
# 查看 swappiness (0-100，越高越积极使用 swap)
cat /proc/sys/vm/swappiness

# 临时修改
sysctl vm.swappiness=10

# 永久修改
echo 'vm.swappiness=10' >> /etc/sysctl.conf
```

**建议**:
- SSD: swappiness = 10-30
- HDD: swappiness = 60-80
- 数据库服务器: swappiness = 0-10

---

# 第五部分：Docker 容器环境定位

## 5.1 Docker 容器卡顿定位

**判断矩阵**:

| 容器内 iostat | 宿主机 iostat | 结论 |
|:-------------:|:-------------:|------|
| %util 高 | %util 正常 | 容器 cgroup 限制 |
| %util 高 | %util 也高 | 宿主机瓶颈 |
| %util 正常 | %util 高 | 其他容器在抢资源 |

```bash
# 查看容器资源使用
docker stats --no-stream

# 查看容器 I/O 限制
docker inspect <container_id> --format '{{.HostConfig.BlkioDeviceReadBps}} {{.HostConfig.BlkioDeviceWriteBps}}'
```

## 5.2 cgroup blkio 定位

```bash
# 1. 找到容器的 cgroup 路径
docker inspect <container_id> | grep -i cgroup
# 输出: "CgroupParent": "/docker/abc123..."

# 2. 查看 blkio throttling 限制
cat /sys/fs/cgroup/blkio/docker/<container_id>/blkio.throttle.read_iops
cat /sys/fs/cgroup/blkio/docker/<container_id>/blkio.throttle.write_iops

# 3. 查看实际 I/O 使用
cat /sys/fs/cgroup/blkio/docker/<container_id>/blkio.throttle.io_service_bytes

# 4. 对比方法: 在容器内外同时跑 iostat
# 宿主机:
iostat -xz 1
# 容器内:
docker exec <container_id> iostat -xz 1
```

---

# 第六部分：/proc 定位真凶进程

## 6.1 核心文件清单

| 文件 | 作用 |
|:----:|------|
| `/proc/<pid>/status` | 进程状态 (Stat列: R/D/S) |
| `/proc/<pid>/stack` | 内核堆栈 (看在内核哪里等) |
| `/proc/<pid>/wchan` | 阻塞的内核函数名 |
| `/proc/<pid>/fd/` | 打开的文件描述符 |
| `/proc/<pid>/io` | I/O 统计 (读/写字节数) |
| `/proc/<pid>/maps` | 内存映射 |
| `/proc/<pid>/syscall` | 当前正在执行的系统调用 |

## 6.2 完整排查路径

```bash
# 步骤1: 找到 D 状态进程
ps aux | awk '$8 ~ /D/ {print}'

# 步骤2: 查看内核堆栈
cat /proc/<pid>/stack
# 示例输出:
# [<ffffffff81234567>] ext4_sync_file+0x123/0x456
# → 进程在等 ext4 文件系统同步

# 步骤3: 查看 wchan (内核函数)
cat /proc/<pid>/wchan
# 示例输出: ext4_sync_file

# 步骤4: 查看打开的文件描述符
ls -la /proc/<pid>/fd | grep -E 'sda|sdb'
# 示例输出: lr-x 1 root  123 -> /data/logs/app.log
# → 真凶是疯狂写日志的进程
```

## 6.3 常用命令脚本

```bash
# 脚本1: 找出所有 D 状态进程及其等待原因
for pid in $(find /proc -maxdepth 1 -type d -name '[0-9]*'); do
    state=$(grep 'State' $pid/status 2>/dev/null | awk '{print $2}')
    if [[ $state == *D* ]]; then
        echo "=== PID: $(basename $pid) ==="
        cat $pid/wchan 2>/dev/null
        echo ""
    fi
done

# 脚本2: 查看进程的 I/O 读写量
for pid in $(pgrep -d',' -x java python node); do
    echo "=== PID: $pid ==="
    cat /proc/$pid/io 2>/dev/null | grep -E 'rchar|wchar|syscr|syscw'
done
```

---

# 🎯 综合排查路径总结

## 场景: ls 命令执行要 3 秒

```
ls 执行慢 (3秒)
  │
  ├── vmstat r 列 > CPU核心数？
  │     │
  │     ├── 是 → CPU 调度瓶颈
  │     │     → pidstat -w 找切换多的进程
  │     │
  │     └── 否 → 继续排查
  │
  ├── vmstat wa% > 20%？
  │     │
  │     ├── 是 → 磁盘 I/O 瓶颈
  │     │     → iostat -xz 找热点磁盘
  │     │     → iotop 找 I/O 进程
  │     │
  │     └── 否 → 继续排查
  │
  ├── vmstat si/so > 0？
  │     │
  │     ├── 是 → Swap 瓶颈
  │     │     → 找用 swap 最多的进程
  │     │     → 增加内存或调 swappiness
  │     │
  │     └── 否 → 继续排查
  │
  └── strace ls
        │
        └── 定位卡在哪一行系统调用
```

---

# 📝 练习题答案

## Q1: ls 要 3 秒的可能原因？

**参考答案**:

ls 命令本身很轻量，正常应该毫秒级完成。3 秒说明 **ls 进程在排队或被阻塞**。

可能的根因:
1. **CPU 调度**: 大量进程在等 CPU，`vmstat r` 列 > CPU 核心数
2. **磁盘 I/O**: `ls` 依赖的目录 inode 在磁盘上需要读取，`wa%` 高
3. **网络文件系统**: 如果 `ls` 的路径是 NFS/CIFS，网络卡会导致延迟
4. **D 状态阻塞**: 依赖的系统资源被其他 D 状态进程占用

**快速验证**:
```bash
strace ls  # 看卡在哪一行系统调用
```

## Q2: %util 高但 wa% 低的含义？

**参考答案**:

`wa%` 只统计 CPU 等待 I/O 的时间，如果 I/O 由内核线程异步处理，CPU 不会被阻塞。

常见场景:
1. **Direct I/O**: 应用绕过 page cache 直接读写
2. **内核线程处理**: pdflush/flush 线程在做同步
3. **NVMe 设备**: 部分 I/O 完成不在 wa% 中统计
4. **RAID 卡缓存**: BBU 充放电期间
5. **SSD 固件活动**: 垃圾回收(Wear Leveling)

**排查方法**: 结合 `avgqu-sz` 和 `await` 判断是队列积压还是设备真的慢

## Q3: si/so 单位？

**答案**: `KB/s`（默认，基于 1024 bytes）

```bash
# 默认
vmstat 1
  si    so
   0  1024   ← 表示每秒换出 1MB

# 切换单位
vmstat -S M 1  # 改为 MB/s
vmstat -S k 1  # 改为 1000 bytes/s
```

## Q4: Docker 容器卡顿定位？

**参考答案**:

```bash
# 1. 对比容器内外 iostat
# 宿主机
iostat -xz 1
# 容器内
docker exec <container_id> iostat -xz 1

# 2. 查看容器资源统计
docker stats --no-stream

# 3. 查看容器 cgroup 限制
cat /sys/fs/cgroup/blkio/docker/<container_id>/blkio.throttle.*

# 判断逻辑:
# 容器内 %util 高 + 宿主机 %util 正常 = 容器限制
# 容器内 %util 高 + 宿主机 %util 也高 = 宿主机瓶颈
```

## Q5: /proc 定位真凶进程？

**参考答案**:

```bash
# 1. 找到 D 状态进程
ps aux | awk '$8 ~ /D/ {print}'

# 2. 查看内核堆栈 (关键!)
cat /proc/<pid>/stack
# 示例: ext4_sync_file+0x123 → 在等文件系统同步

# 3. 查看 wchan (内核函数名)
cat /proc/<pid>/wchan

# 4. 查看打开的文件描述符
ls -la /proc/<pid>/fd | grep -E 'sda|sdb'
# 或
lsof -p <pid>

# 5. 查看 I/O 统计
cat /proc/<pid>/io
# rchar: 读的字节数
# wchar: 写的字节数
```

---

# ✅ 学习完成

**下一步**:
1. 在本地 Linux 机器上跑一遍这些命令
2. 尝试用 `stress` 制造压力场景
3. 在 Docker 容器中体验 cgroup 限制

```bash
# 安装工具
apt install sysstat iotop  # Debian/Ubuntu
yum install sysstat iotop   # RHEL/CentOS

# 制造 I/O 压力测试
stress -i 4 --hdd 2 --hdd-bytes 1G

# 制造 CPU 压力测试
stress -c 4
```

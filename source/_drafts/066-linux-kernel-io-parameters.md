---
title: linux内核参数调优 I/O相关
date: 2024-08-05 09:24:15
tags:
  - Linux
categories: Linux
---

### 1. 内核参数查看和修改方法

#### 1.1 通过sysctl查看参数值
```bash
root@Ubuntu:~# sysctl -n kernel.printk
7	4	1	7
```

#### 1.2 直接修改文件（临时生效）
```bash
root@Ubuntu:~# echo 7 > /proc/sys/kernel/printk
```
#### 1.3 通过命令修改（临时生效）
```bash
root@Ubuntu:~# sysctl -w kernel.printk=4
kernel.printk = 4
```
#### 1.4 修改配置文件（永久生效）
修改/etc/sysctl.conf
```bash
root@Ubuntu:~# sudo vim /etc/sysctl.conf
kernel.printk = 4
```
执行下面的命令，从`/etc/sysctl.conf`中读取参数
```bash
root@Ubuntu:~# sudo sysctl -p
```

### 2. linux I/O缓存机制
linux操作系统默认情况下写都是有写缓存的，可以使用direct IO方式绕过操作系统的写缓存。当写一串数据时，系统会开辟一块内存区域缓存这些数据，这块区域就是我们常说的page cache。

在使用fio进行写操作时，通过/proc/meminfo查看详细的内存使用情况
```bash
root@Ubuntu:~#  cat /proc/meminfo 
MemTotal:        8096764 kB
MemFree:         2973444 kB
MemAvailable:    7084680 kB
Buffers:          115640 kB
Cached:          4335640 kB
SwapCached:            0 kB
Active:           376828 kB
Inactive:        4282016 kB
Active(anon):     210336 kB
Inactive(anon):   418632 kB
Active(file):     166492 kB
Inactive(file):  3863384 kB
Unevictable:        1948 kB
Mlocked:            1948 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:           1068904 kB
Writeback:            12 kB
AnonPages:        205844 kB
Mapped:           536940 kB
Shmem:            419636 kB
Slab:             341356 kB
SReclaimable:     285456 kB
SUnreclaim:        55900 kB
KernelStack:        3552 kB
PageTables:         5932 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     4048380 kB
Committed_AS:    2406372 kB
VmallocTotal:   135290290112 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
Percpu:             1264 kB
HardwareCorrupted:     0 kB
AnonHugePages:     67584 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
CmaTotal:          32768 kB
CmaFree:           21624 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
```
其中Dirty表示需要写入磁盘的内存页的大小，将近有1GB的大小，linux系统后台运行的一个数据回写线程，这个进程负责把page cahce中的dirty状态的数据定期的输入磁盘。

page cache提供了较好的吞吐量，缩短了IO响应时间。但它也有缺点：
1. 断电可能丢数据（数据安全性）
2. 如果page cache过大，那么就会缓存太多的数据，当需要统一刷入磁盘的时候就会出现一个IO峰值和瓶颈，在这其间对用户的IO访问出现明显影响

### 3. 磁盘I/O性能指标
1. 使用率： 指磁盘处理 I/O 的时间百分比
2. 饱和度： 是指磁盘处理 I/O 的繁忙程度
3. IOPS（Input/Output Per Second）： 指每秒的 I/O 请求数
4. 吞吐量： 指每秒的 I/O 请求大小
5. 响应时间： 指I/O请求从发出到收到响应的间隔时间

### 3.1 重要的IO相关的内核参数

1. **vm.dirty_background_ratio**： 脏数据可以占总可用内存的百分比，超过这个比例，内核回开始进行数据回写，默认值是10
2. **vm.dirty_ratio**： 脏数据可以占总可用内存的百分比，超过这个比例，内核回开始进行数据回写，并会阻塞所有的写IO，默认值是20
3. **vm.dirty_expire_centisecs**： 脏数据能够存留的时间，超过这个时间，数据会被回写，默认值是3000，对应30秒
4. **read_ahead_kb**：控制内核为每个文件读取操作预读的数据量（以KB为单位）。对于顺序读较多的应用，可以适当增大这个值以提高性能；对于随机读较多的场景，过大的预读可能会浪费资源，可以适当减小该值。


### 4. 测试工具
使用fio（Flexible I/O Tester）来测试磁盘I/O性能，fio的官方文档[https://fio.readthedocs.io/en/latest/fio_doc.html](https://fio.readthedocs.io/en/latest/fio_doc.html)

### 5. 修改vm dirty参数
+ **高性能需求**：如果你的系统需要高性能的磁盘写入，可以降低dirty_ratio的值，以减少脏数据在内存中的积累，从而提高写入的实时性。
+ **稳定性需求**：如果系统稳定性更重要，可以选择较高的dirty_ratio值，以减少频繁的磁盘写入操作，可以带来低延迟和高带宽，但是有丢数据风险

+ 测试命令，随机写入
```bash
fio --name=base_test --ioengine=libaio --rw=randwrite --bs=4K --size=10G --numjobs=1 --runtime=60s --time_based
```

#### 5.1 降低 dirty_ratio 的值
参考tuned中latency-performance配置的参数
```bash
sysctl -w vm.dirty_background_ratio=3
sysctl -w vm.dirty_ratio=10
```

测试结果
```bash
root@Ubuntu:~# fio --name=base_test --ioengine=libaio --rw=randwrite --bs=4K --size=10G --numjobs=1 --runtime=60s --time_based
base_test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [w(1)][100.0%][w=192MiB/s][w=49.1k IOPS][eta 00m:00s]
base_test: (groupid=0, jobs=1): err= 0: pid=5348: Fri Aug  9 15:14:16 2024
  write: IOPS=43.0k, BW=168MiB/s (176MB/s)(9.85GiB/60001msec); 0 zone resets
    slat (usec): min=7, max=26674, avg=18.51, stdev=241.93
    clat (nsec): min=940, max=871000, avg=1394.11, stdev=958.36
     lat (usec): min=8, max=26677, avg=20.32, stdev=242.00
    clat percentiles (nsec):
     |  1.00th=[ 1004],  5.00th=[ 1020], 10.00th=[ 1048], 20.00th=[ 1064],
     | 30.00th=[ 1128], 40.00th=[ 1448], 50.00th=[ 1464], 60.00th=[ 1480],
     | 70.00th=[ 1496], 80.00th=[ 1528], 90.00th=[ 1608], 95.00th=[ 1720],
     | 99.00th=[ 2224], 99.50th=[ 2448], 99.90th=[ 3760], 99.95th=[ 5856],
     | 99.99th=[31104]
   bw (  KiB/s): min=71680, max=290608, per=100.00%, avg=174791.34, stdev=49502.87, samples=117
   iops        : min=17920, max=72652, avg=43697.83, stdev=12375.71, samples=117
  lat (nsec)   : 1000=0.18%
  lat (usec)   : 2=97.91%, 4=1.82%, 10=0.05%, 20=0.01%, 50=0.03%
  lat (usec)   : 100=0.01%, 250=0.01%, 750=0.01%, 1000=0.01%
  cpu          : usr=19.28%, sys=60.48%, ctx=8914, majf=0, minf=28
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,2581033,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=168MiB/s (176MB/s), 168MiB/s-168MiB/s (176MB/s-176MB/s), io=9.85GiB (10.6GB), run=60001-60001msec

Disk stats (read/write):
  nvme0n1: ios=1/2492861, merge=0/51163, ticks=1/176748, in_queue=189884, util=90.09%
```

降低dirty_ratio后，带宽相较默认参数的要低20%左右，且磁盘的利用率相比默认参数要提高20%左右

#### 5.2 提高 dirty_ratio 的值
参考tuned中mssql配置的参数
```bash
sysctl -w vm.dirty_background_ratio=40
sysctl -w vm.dirty_ratio=40
```

测试结果
```bash
root@Ubuntu:~# fio --name=base_test --ioengine=libaio --rw=randwrite --bs=4K --size=10G --numjobs=1 --runtime=60s --time_based
base_test: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [w(1)][100.0%][w=328MiB/s][w=84.1k IOPS][eta 00m:00s]
base_test: (groupid=0, jobs=1): err= 0: pid=5429: Fri Aug  9 15:30:09 2024
  write: IOPS=60.2k, BW=235MiB/s (247MB/s)(13.8GiB/60001msec); 0 zone resets
    slat (usec): min=6, max=3545, avg=10.95, stdev= 4.02
    clat (nsec): min=920, max=721460, avg=1287.98, stdev=649.99
     lat (usec): min=8, max=3548, avg=12.61, stdev= 4.30
    clat percentiles (nsec):
     |  1.00th=[  980],  5.00th=[  980], 10.00th=[ 1004], 20.00th=[ 1020],
     | 30.00th=[ 1048], 40.00th=[ 1080], 50.00th=[ 1224], 60.00th=[ 1480],
     | 70.00th=[ 1496], 80.00th=[ 1528], 90.00th=[ 1544], 95.00th=[ 1576],
     | 99.00th=[ 1864], 99.50th=[ 2064], 99.90th=[ 2576], 99.95th=[ 3024],
     | 99.99th=[29824]
   bw (  KiB/s): min=132048, max=345504, per=100.00%, avg=267057.37, stdev=58509.76, samples=107
   iops        : min=33012, max=86376, avg=66764.37, stdev=14627.41, samples=107
  lat (nsec)   : 1000=5.93%
  lat (usec)   : 2=93.46%, 4=0.58%, 10=0.01%, 20=0.01%, 50=0.02%
  lat (usec)   : 100=0.01%, 250=0.01%, 750=0.01%
  cpu          : usr=24.27%, sys=73.68%, ctx=1000, majf=0, minf=24
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=0,3614048,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=235MiB/s (247MB/s), 235MiB/s-235MiB/s (247MB/s-247MB/s), io=13.8GiB (14.8GB), run=60001-60001msec

Disk stats (read/write):
  nvme0n1: ios=14/2228321, merge=0/106, ticks=12/210637, in_queue=219228, util=54.78%
```
在提高vm dirty的值后，可以看到带宽相较默认参数进一步提高了，磁盘利用率也降低了16%左右

#### 5.3 实际应用中的调整
**高性能和低延迟**：对于延迟敏感的应用，通常需要降低 vm.dirty_background_ratio 和 vm.dirty_ratio 的值，以确保数据更快写入磁盘，减少内存中的脏数据积累。

**高吞吐量和批量处理**：对于需要高吞吐量或批量处理的应用，如大数据处理，可能需要提高这两个参数的值，以减少频繁的磁盘写入，提高系统的总体效率。

### 6. 块设备参数调整
Linux 中块设备参数用于控制块设备的行为和性能。这些参数可以在`/sys/block/<device>/queue/` 目录中查看和调整（例如 `/sys/block/sda/queue/`）。调优这些参数可以根据具体的工作负载、硬件特性和性能需求来实现最佳性能配置。

#### 6.1 常见的参数和其含义
Linux 中块设备参数用于控制块设备的行为和性能。了解这些参数有助于优化存储性能和提高系统效率。以下是一些常见的块设备参数及其含义：

1. `read_ahead_kb`
- **含义**: 预读取数据的大小，以千字节（KB）为单位。它决定了在读取数据时，系统会提前读取多大的数据量到缓存中。
- **用途**: 增大 `read_ahead_kb` 值可以提升顺序读取的性能，但可能会影响随机读取的性能。
- **默认值**: 256（nvme）

2. `max_sectors_kb`
- **含义**: 定义单个I/O操作的最大数据量，以千字节（KB）为单位。该参数限制了单次I/O操作的最大大小。
- **用途**: 调整这个参数可以优化大数据块传输的效率。在高性能设备上，增大这个值可能会提高吞吐量。
- **默认值**: 128 (nvme)

3. `nr_requests`
- **含义**: I/O队列中最多允许的请求数。该参数控制了内核在一个设备上同时处理的I/O请求数量。
- **用途**: 调整此值可以控制I/O操作的并发性和调度。过低的值可能会导致I/O饥饿，而过高的值可能会增加I/O调度的开销。
- **默认值**: 1023 (nvme)

4. `scheduler`
- **含义**: 确定块设备使用的I/O调度算法。常见的调度器有 `noop`、`deadline`、`cfq`（完全公平队列）、`bfq`（预算公平队列）等。
- **用途**: 不同的调度器适用于不同的工作负载。例如，`noop` 适用于 SSD，`deadline` 适用于需要低延迟的应用。
- **默认值**: noop (nvme)

5. `rotational`
- **含义**: 指示设备是否为旋转介质（如传统硬盘驱动器HDD）。
- **用途**: 该参数影响内核的I/O调度策略。如果设备是旋转介质，内核可能会进行优化以减少寻道时间。
- **默认值**: 对于HDD通常为1（表示旋转介质），对于SSD为0（表示非旋转介质）。

6. `rq_affinity`
- **含义**: 控制完成I/O请求的CPU是否优先选择最初提交请求的CPU。
- **用途**: 该参数有助于优化CPU缓存的使用，减少CPU之间的上下文切换。在高性能、多核心环境中，设置为1可以提高I/O性能。
- **默认值**: 1（开启）。

7. `write_cache`
- **含义**: 指示设备是否启用了写缓存。
- **用途**: 如果启用写缓存，写入操作可能会被缓存在设备的内存中，这样可以提升写性能，但在断电或设备故障时有数据丢失的风险。
- **默认值**: 1（开启）

8. `add_random`
- **含义**: 指定I/O操作是否影响系统的熵池。某些设备的I/O操作会影响系统的随机数生成器。
- **用途**: 设置为1时，I/O操作将添加熵到随机数池，这可能在一些需要高熵的环境（如加密）中有用。
- **默认值**: 0 或 1，取决于设备类型。

9. `nomerges`
- **含义**: 控制I/O请求的合并行为。
- **用途**: 通过设置该参数，可以优化I/O请求的合并，减少块设备层面的开销。0 表示允许合并，1 表示禁止合并。
- **默认值**: 0（允许合并）。

10. `iostats`
- **含义**: 控制是否启用块设备的I/O统计信息收集。
- **用途**: 当启用时，系统会收集详细的I/O统计信息，如读取和写入的操作次数和数据量。禁用此功能可以稍微减少系统开销。
- **默认值**: 1（启用）。

#### 6.2 调整read_ahed_kb

##### 6.2.1 测试环境
测试环境：E2000Q demo开发板，希捷1TB机械硬盘，linux 6.6内核，默认的`read_ahead_kb`参数为128

测试用例：

+ 顺序读，注意测试预读时不需要加上`direct`参数，否则`read_ahead_kb`不会生效
```bash
fio --name=read --filename=/dev/sda4 --ioengine=libaio --rw=read --size=10G --numjobs=1 --direct=1 --bs=4k
```

+ 随机读
```bash
fio --name=randread --filename=/dev/sda4 --ioengine=libaio --rw=randread --size=4G --numjobs=1 --bs=4k
```

##### 6.2.2 read_ahead_kb设置为4

+ 顺序读测试结果
```bash
root@Ubuntu:/sys/block/sda/queue# fio --name=read --filename=/dev/sda4 --ioengine=libaio --rw=read --size=10G --numjobs=1 --bs=4k
read: (g=0): rw=read, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [f(1)][100.0%][r=32.3MiB/s][r=8270 IOPS][eta 00m:00s]
read: (groupid=0, jobs=1): err= 0: pid=2563: Thu Aug 29 16:05:15 2024
  read: IOPS=47.6k, BW=186MiB/s (195MB/s)(10.0GiB/55059msec)
    slat (usec): min=3, max=9667, avg=18.29, stdev=40.41
    clat (nsec): min=800, max=1426.2k, avg=1182.46, stdev=961.71
     lat (usec): min=4, max=9668, avg=19.74, stdev=40.56
    clat percentiles (nsec):
     |  1.00th=[  844],  5.00th=[  860], 10.00th=[  860], 20.00th=[  860],
     | 30.00th=[  884], 40.00th=[  900], 50.00th=[ 1288], 60.00th=[ 1400],
     | 70.00th=[ 1416], 80.00th=[ 1464], 90.00th=[ 1544], 95.00th=[ 1608],
     | 99.00th=[ 1864], 99.50th=[ 2024], 99.90th=[ 2320], 99.95th=[ 2512],
     | 99.99th=[14016]
   bw (  KiB/s): min=180176, max=191664, per=99.99%, avg=190417.03, stdev=1446.20, samples=110
   iops        : min=45044, max=47916, avg=47604.15, stdev=361.66, samples=110
  lat (nsec)   : 1000=48.17%
  lat (usec)   : 2=51.24%, 4=0.56%, 10=0.01%, 20=0.03%, 50=0.01%
  lat (usec)   : 100=0.01%
  lat (msec)   : 2=0.01%
  cpu          : usr=13.04%, sys=55.20%, ctx=1172648, majf=0, minf=27
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=2621440,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=186MiB/s (195MB/s), 186MiB/s-186MiB/s (195MB/s-195MB/s), io=10.0GiB (10.7GB), run=55059-55059msec

Disk stats (read/write):
  sda: ios=1306587/0, merge=0/0, ticks=77883/0, in_queue=77883, util=99.88%
```

iostat数据
```bash
Device            r/s     rMB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wMB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dMB/s   drqm/s  %drqm d_await dareq-sz  aqu-sz  %util
sda4          23852.00    186.34     0.00   0.00    0.06     8.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    1.43 100.00
```

+ 随机读测试结果
```bash
root@Ubuntu:/sys/block/sda/queue# echo 4 > read_ahead_kb 
root@Ubuntu:/sys/block/sda/queue# fio --name=randread --filename=/dev/sda4 --ioengine=libaio --rw=randread --size=200M --numjobs=1 --bs=4k
randread: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.16
Starting 1 process
Jobs: 1 (f=0): [f(1)][100.0%][r=3471KiB/s][r=867 IOPS][eta 00m:00s]
randread: (groupid=0, jobs=1): err= 0: pid=2685: Thu Aug 29 16:38:22 2024
  read: IOPS=219, BW=876KiB/s (897kB/s)(200MiB/233789msec)
    slat (usec): min=104, max=107744, avg=4558.74, stdev=3216.91
    clat (nsec): min=1320, max=45320, avg=2446.61, stdev=1117.86
     lat (usec): min=106, max=107753, avg=4561.99, stdev=3217.13
    clat percentiles (nsec):
     |  1.00th=[ 1400],  5.00th=[ 1928], 10.00th=[ 1960], 20.00th=[ 2024],
     | 30.00th=[ 2096], 40.00th=[ 2128], 50.00th=[ 2192], 60.00th=[ 2288],
     | 70.00th=[ 2384], 80.00th=[ 2576], 90.00th=[ 2960], 95.00th=[ 3664],
     | 99.00th=[ 7520], 99.50th=[ 8768], 99.90th=[10944], 99.95th=[22144],
     | 99.99th=[31616]
   bw (  KiB/s): min=  616, max= 2832, per=99.24%, avg=869.38, stdev=152.71, samples=467
   iops        : min=  154, max=  708, avg=217.33, stdev=38.18, samples=467
  lat (usec)   : 2=15.49%, 4=80.46%, 10=3.88%, 20=0.12%, 50=0.06%
  cpu          : usr=0.16%, sys=0.72%, ctx=51202, majf=0, minf=20
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=51200,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=876KiB/s (897kB/s), 876KiB/s-876KiB/s (897kB/s-897kB/s), io=200MiB (210MB), run=233789-233789msec

Disk stats (read/write):
  sda: ios=50851/0, merge=0/0, ticks=231923/0, in_queue=231923, util=100.00%
```

iostat数据
```bash
Device            r/s     rMB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wMB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dMB/s   drqm/s  %drqm d_await dareq-sz  aqu-sz  %util
sda4           230.00      0.90     0.00   0.00    4.31     4.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.99 100.00
```

fio的IOPS数据可能是根据传入的bs参数大小计算的，实际并不准确，所以参考iostat的数据，可以看到最大带宽为186MB/s，iops达到了23852，可以计算出每次io请求的数据大小为8KB，这和我们预读的参数设置可以对上，不过仍然没有达到iops的瓶颈，所以测出来带宽仍然达到最大


##### 6.2.3 read_ahead_kb设置为4096

+ 顺序读测试结果

fio测试数据
```bash
root@Ubuntu:/sys/block/sda/queue# echo 4096 > max_sectors_kb 
root@Ubuntu:/sys/block/sda/queue# fio --name=read --filename=/dev/sda4 --ioengine=libaio --rw=read --size=10G --numjobs=1 --bs=4k
read: (g=0): rw=read, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [f(1)][100.0%][r=32.7MiB/s][r=8372 IOPS][eta 00m:00s]
read: (groupid=0, jobs=1): err= 0: pid=2626: Thu Aug 29 16:13:27 2024
  read: IOPS=47.6k, BW=186MiB/s (195MB/s)(10.0GiB/55070msec)
    slat (usec): min=2, max=41780, avg=18.38, stdev=627.67
    clat (nsec): min=800, max=113500, avg=1143.40, stdev=565.04
     lat (usec): min=3, max=41791, avg=19.79, stdev=627.90
    clat percentiles (nsec):
     |  1.00th=[  844],  5.00th=[  844], 10.00th=[  844], 20.00th=[  860],
     | 30.00th=[  860], 40.00th=[  860], 50.00th=[ 1020], 60.00th=[ 1368],
     | 70.00th=[ 1384], 80.00th=[ 1400], 90.00th=[ 1416], 95.00th=[ 1464],
     | 99.00th=[ 1576], 99.50th=[ 1768], 99.90th=[ 4256], 99.95th=[10560],
     | 99.99th=[29568]
   bw (  KiB/s): min=178848, max=196608, per=100.00%, avg=190464.31, stdev=5637.96, samples=110
   iops        : min=44712, max=49152, avg=47616.05, stdev=1409.51, samples=110
  lat (nsec)   : 1000=48.78%
  lat (usec)   : 2=50.89%, 4=0.21%, 10=0.06%, 20=0.03%, 50=0.02%
  lat (usec)   : 100=0.01%, 250=0.01%
  cpu          : usr=12.60%, sys=31.44%, ctx=1390, majf=0, minf=26
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=2621440,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=186MiB/s (195MB/s), 186MiB/s-186MiB/s (195MB/s-195MB/s), io=10.0GiB (10.7GB), run=55070-55070msec

Disk stats (read/write):
  sda: ios=2582/0, merge=3/0, ticks=144619/0, in_queue=144619, util=88.17%
```

iostat数据
```bash
Device            r/s     rMB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wMB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dMB/s   drqm/s  %drqm d_await dareq-sz  aqu-sz  %util
sda4            46.00    184.00     0.00   0.00   61.26  4096.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    2.82  77.20
```

+ 随机读测试结果

fio测试数据
```bash
root@Ubuntu:/sys/block/sda/queue# fio --name=randread --filename=/dev/sda4 --ioengine=libaio --rw=randread --size=200M --numjobs=1 --bs=4k
randread: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
fio-3.16
Starting 1 process
Jobs: 1 (f=1): [r(1)][99.6%][r=2670KiB/s][r=667 IOPS][eta 00m:01s]
randread: (groupid=0, jobs=1): err= 0: pid=2678: Thu Aug 29 16:34:16 2024
  read: IOPS=219, BW=879KiB/s (900kB/s)(200MiB/233097msec)
    slat (usec): min=50, max=54179, avg=4546.19, stdev=3179.90
    clat (nsec): min=1280, max=43260, avg=2174.57, stdev=937.98
     lat (usec): min=52, max=54187, avg=4549.04, stdev=3180.08
    clat percentiles (nsec):
     |  1.00th=[ 1368],  5.00th=[ 1384], 10.00th=[ 1416], 20.00th=[ 1576],
     | 30.00th=[ 1976], 40.00th=[ 2040], 50.00th=[ 2096], 60.00th=[ 2160],
     | 70.00th=[ 2256], 80.00th=[ 2384], 90.00th=[ 2672], 95.00th=[ 3120],
     | 99.00th=[ 5984], 99.50th=[ 7008], 99.90th=[ 8896], 99.95th=[16768],
     | 99.99th=[30848]
   bw (  KiB/s): min=  704, max= 3856, per=99.65%, avg=874.90, stdev=187.21, samples=466
   iops        : min=  176, max=  964, avg=218.71, stdev=46.80, samples=466
  lat (usec)   : 2=32.17%, 4=65.43%, 10=2.33%, 20=0.04%, 50=0.04%
  cpu          : usr=0.14%, sys=0.63%, ctx=51203, majf=0, minf=22
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=51200,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=879KiB/s (900kB/s), 879KiB/s-879KiB/s (900kB/s-900kB/s), io=200MiB (210MB), run=233097-233097msec

Disk stats (read/write):
  sda: ios=50797/0, merge=0/0, ticks=231362/0, in_queue=231362, util=100.00%
```

iostat数据
```bash
Device            r/s     rMB/s   rrqm/s  %rrqm r_await rareq-sz     w/s     wMB/s   wrqm/s  %wrqm w_await wareq-sz     d/s     dMB/s   drqm/s  %drqm d_await dareq-sz  aqu-sz  %util
sda4           190.00      0.74     0.00   0.00    5.21     4.00    0.00      0.00     0.00   0.00    0.00     0.00    0.00      0.00     0.00   0.00    0.00     0.00    0.99 100.00
```

因为已经到了机械硬盘读取性能极限，可以看到最大带宽同样是186MB/s，iops却降到了46，每次io请求的数据大小为4096KB，这和我们预读的参数设置可以对上，磁盘利用率也显著降低了

#### 6.3 结论
调整`read_ahead_kb`参数，在机械盘和nvme硬盘上，由于控制器和磁盘的性能，使用fio测试顺序读基本上都能达到磁盘的极限性能，不会带来带宽的提升。但是当顺序读的每次io请求大小比较小的情况下（例如fio测试中bs参数设置比较小的情况），增大`read_ahead_kb`可以降低iops，降低磁盘利用率；`read_ahead_kb`对随机读没什么影响，可以看到`read_ahead_kb`设置4096和设置4这两种情况下每次的io请求大小均为4KB，并没有受到预读大小的影响。

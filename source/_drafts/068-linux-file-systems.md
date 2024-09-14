---
title: linux支持的文件系统
tags:
---

## 1. linux下常见的文件系统（不包括集群文件系统）
+ Ext系列
  1. Ext2（Second Extended Filesystem）：
     + 特点：最早的稳定文件系统之一，支持2TB的单个文件系统，最大文件大小为16GB（32位系统）或2TB（64位系统）。
     + 应用场景：过去多用于系统根分区，适合不需要日志的应用场景，比如存储闪存设备的文件系统。
     + 缺点：不支持日志功能，文件恢复和系统崩溃后恢复时间较长。

  2. Ext3（Third Extended Filesystem）：
     + 特点：基于Ext2改进，增加了日志功能，最大文件系统大小支持到32TB。
     + 应用场景：适用于需要日志支持的常规Linux系统，系统崩溃后可以更快恢复。
     + 缺点：在性能和扩展性上不如Ext4。

  3. Ext4（Fourth Extended Filesystem）：
     + 特点：支持更大的文件系统（1EB）和文件大小（16TB），提高了性能（如延迟分配、预分配等）。
     + 应用场景：广泛应用于现代Linux系统，适合大多数通用场景，包括桌面、服务器和嵌入式设备。
     + 优点：向后兼容Ext3，具备更高的性能和可靠性。

+ XFS
  + 特点：一种高性能的64位日志文件系统，尤其在处理大文件时效率更高，支持在线扩展和碎片整理。
  + 应用场景：常用于高性能、大数据、数据密集型服务器，如数据库服务器、大型文件存储系统。
  + 优点：高并发和高吞吐量，适合处理大文件。
  + 缺点：在处理小文件时性能表现一般，不支持文件系统缩减。

+ Btrfs（B-tree Filesystem）
  + 特点：现代化的文件系统，支持写时复制（Copy on Write, CoW）、快照、内置RAID、压缩和子卷管理等功能，具备自我修复能力。
  + 应用场景：适用于对数据安全性和可扩展性要求高的场景，如服务器、虚拟化、容器化环境以及备份系统。
  + 优点：支持数据快照、压缩和动态存储池管理，易于扩展和维护。
  + 缺点：相较于Ext4，性能还在优化中，稳定性不如Ext4和XFS。

+ NTFS
  + 特点：由微软开发，支持大文件、权限管理、压缩、加密等高级功能
  + 应用场景：用于Windows和Linux系统共享存储的场景，尤其是在双系统环境中。
  + 优点：兼容Windows文件系统，支持大文件。
  + 缺点：在Linux上写入性能不如本地文件系统，某些高级功能（如压缩、加密）在Linux下支持有限。

+ JFS
  + 特点：JFS 是一种高性能、低资源消耗的日志文件系统，擅长处理大文件和大容量存储，适合服务器和数据密集型应用。
  + 应用场景：服务器环境，数据密集型应用，嵌入式系统和低资源环境，高稳定性需求的系统
  + 优点：资源占用少，适合低配置硬件，高性能，尤其是处理大文件和大容量存储时，稳定性好，支持长时间运行和数据恢复，支持在线扩展，灵活性强。
  + 缺点：社区支持和开发活跃度较低。在小文件和高并发处理场景下性能一般，不如Ext4和XFS等现代文件系统。

+ F2FS
  + 特点：专门为NAND闪存存储设备（如SSD、eMMC、SD卡等）设计，优化了闪存写入、擦除的特性。
  + 应用场景：适用于嵌入式系统、智能手机、平板电脑以及其他采用闪存存储的设备。
  + 优点：提升闪存设备的寿命和性能。


+ ISO 9660
  + 特点：用于光盘文件系统，ISO 9660是早期光盘格式，UDF（Universal Disk Format）是更新的格式，支持更大容量和文件名长度。
  + 应用场景：用于CD、DVD、Blu-ray等光盘媒体。
  + 优点：广泛应用于光盘存储，支持大部分操作系统。
  + 缺点：不适合常规硬盘存储，功能相对有限。

+ vfat (FAT32)
  + 特点：兼容性极好，支持多种操作系统（Windows、macOS、Linux等），但不支持权限管理，文件大小最大限制为4GB。
  + 应用场景：常用于U盘、存储卡和外部存储设备的跨平台使用场景。
  + 缺点：不支持现代文件系统的功能，如日志、权限管理等，文件系统规模较小。

+ exFAT
  + 特点：微软开发的文件系统，支持更大的文件和分区大小，解决了FAT32的4GB文件大小限制，且跨平台支持较好。
  + 应用场景：用于便携式设备、大容量存储设备，兼容性好。
  + 优点：适合大文件传输，兼容多个平台。

+ UBIFS
  + 特点：UBIFS（UBI File System）是一种为闪存存储优化的文件系统，支持大容量、耐用性强、具备高效的闪存管理和数据恢复功能。
  + 应用场景：UBIFS 适用于嵌入式系统和闪存设备，如智能手机、平板电脑、物联网设备和工业控制系统，特别是需要高效闪存管理和可靠数据存储的环境。
  + 优点：
  	1. 优化闪存存储：专为闪存设备设计，减少写放大和磨损，延长闪存的使用寿命。
  	2. 支持大容量：能够处理大于2TB的存储设备，适合大容量闪存。
  	3. 数据恢复：内建的数据恢复功能，能够有效应对系统崩溃或断电等故障。
  + 缺点：相较于传统的文件系统，如FAT，UBIFS的实现和管理较为复杂。虽然优化了闪存使用，但在一些高性能需求场景下，日志和写时复制机制可能带来性能开销。相较于一些成熟的文件系统，UBIFS的社区支持和文档可能较少，适用性和稳定性在一些特定环境下可能有限。

+ squashFS
  + 特点：SquashFS 是一个高效的只读压缩文件系统，旨在将文件和目录压缩成单一映像，以节省存储空间并加快读取速度。
  + SquashFS 主要用于嵌入式系统、系统恢复和安装程序中，适合需要节省存储空间且只读的场景。
  + 优点：通过压缩技术显著减少存储空间需求，适合存储有限的设备，由于是只读文件系统，不会受到文件系统一致性问题的影响，提高了稳定性
  + 缺点：无法进行文件写入操作，限制了文件系统的灵活性和用途，压缩和解压缩过程可能会带来额外的计算开销，尤其是在处理大型文件时，更新文件或应用程序需要重新创建压缩映像，操作不如动态文件系统方便。


## 2. ext4挂载选项

### 2.1 linux常用的挂载选项
+ async: 对文件系统启用异步输入和输出操作
+ auto: 使用 mount -a 命令使文件系统被自动挂载
+ defaults: 为 `async,auto,dev,exec,nouser,rw,suid` 选项提供别名
+ exec: 允许在特定文件系统中执行二进制文件
+ loop: 将镜像挂载为 loop 设备
+ noauto: 默认行为禁用使用 mount -a 命令对文件系统进行自动挂载
+ noexec: 不允许在特定文件系统中执行二进制文件
+ nouser: 不允许普通用户（即 root 用户）挂载和卸载文件系统
+ ro: 仅挂载文件系统以读取
+ rw: 挂载文件系统以进行读和写操作
+ user: 允许普通用户挂载和卸载该文件系统
+ noatime: 禁止更新文件的访问时间（atime），可以提升性能，适合 SSD 等设备
+ nodiratime: 禁止目录的访问时间更新，但允许文件的访问时间更新
+ relatime: 只有当文件的最后访问时间早于最后修改时间时，才更新访问时间。该选项是 ext4 的默认挂载方式，提供性能和访问时间的平衡。
+ strictatime: 严格更新每次访问的时间

E2000下emmc作为系统盘默认的挂载选项
```bash
root@phytium-Ubuntu:~# mount | grep mmcblk0
/dev/mmcblk0p2 on / type ext4 (rw,relatime)
```

通过修改内核启动参数来修改mmcblk0p2的挂载参数
```bash
root=/dev/mmcblk0p2 defaults,noatime,nodiratime
```

### 2.2 日志文件系统
linux文件属性的三种时间戳：
+ access time: 访问时间，表示最后一次访问文件，但是没有改动文件的时间
+ modify time: 修改时间，表示最后一次对文件进行更改的时间
+ change time: 改变时间，这个改变指的是对文件属性的更改

还可以通过ls查看文件的时间，当我们使用ls -l命令列出当前目录下的文件的时候，通常也会带上文件的时间戳信息，这里显示的时间戳默认是文件的修改时间，通过stat命令可以查看文件的详细时间戳信息，Linux提供了natime这个参数来禁止记录文件的访问时间戳信息

```bash
jack@linux:~/blog (hexo*) $ stat package.json
  File: package.json
  Size: 767             Blocks: 8          IO Block: 4096   regular file
Device: 10302h/66306d   Inode: 10755483    Links: 1
Access: (0664/-rw-rw-r--)  Uid: ( 1000/    jack)   Gid: ( 1000/    jack)
Access: 2024-09-05 08:58:14.991859634 +0800
Modify: 2024-08-30 09:44:39.255138885 +0800
Change: 2024-08-30 09:44:39.255138885 +0800
 Birth: 2022-10-19 09:17:44.276839603 +0800
```

日志文件系统的特点是，在发生变化时，它先把相关的信息写入一个被称为日志的区域，然后再把变化写入主文件系统。这样的好处是当系统发生故障时，日志文件系统更容易保证一致性，并且可以较快恢复。这是优点也是缺点，因为有些信息是我们并不需要的，比如文件的访问时间，如果能禁用这些记录，在系统频繁访问文件时，就可以减少一些不必要的操作，便能显著的提高系统IO的效率。

### 2.3 ext4的三种日志模式
1. journal: 提供了完全的数据块和元数据块的日志，所有的数据都会被先写入到日志里，然后再写入磁盘上。在文件系统崩溃的时候，可以通过日志重放，把数据和元数据恢复到一致性的状态。但同时，journal模式性能是三种模式中最差的，因为所有的数据都需要日志来记录。并且该模式不支持delayed allocation(延迟分配)以及O_DIRECT(直接IO)

2. ordered: 是ext4文件系统默认日志模式，在该模式下，文件系统只提供元数据的日志，但它逻辑上将与数据更改相关的元数据信息与数据块分组到一个称为事务的单元中。当需要把元数据写入到磁盘上的时候，与元数据关联的数据块会首先写入。也就是数据先落盘，再将元数据的日志刷到磁盘。 在机器crash时，未完成的写操作对应的元数据仍然保存在文件系统日志中，因此文件系统可以通过回滚日志将未完成的写操作清除。所以，在ordered模式下，crash可能会导致在crash时操作的文件损坏，但对于文件系统本身以及其他未操作的文件，是可以确保安全的。一般情况下，ordered模式的性能会略逊色于writeback但是比journal模式要快的多

3. writeback: 在writeback模式下，当元数据提交到日志后，data可以直接被提交到磁盘。即会做元数据日志，数据不做日志，并且不保证数据比元数据先落盘。metadata journal是串行操作，因此采用writeback模式就不会出现由于其他进程写journal，阻塞另一个进程的情况，因此IOPS也能得到提升。writeback是ext4提供的性能最好的模式。不过，尽管writeback模式也能保证文件系统自身的安全性，但是在系统crash时文件数据也更容易丢失和损坏。


ext4关闭日志功能
```bash
# 只能在readonly挂载或者未挂载的时候才能修改
tune2fs -O ^has_journal /dev/mmcblk0p2
# 查看是否开启日志功能
dumpe2fs /dev/mmcblk0p2 | grep 'Filesystem features' | grep 'has_journal'
```

### 2.4 ext4不同挂载选项的fio性能测试
+ 测试环境： E2000Q demo板，emmc作为系统盘，linux 6.6
+ 测试用例： 随机读写，70%读 30%写
```bash
fio --name=test --filename=./testfile --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k -rw=randrw -rwmixread=70 --iodepth=32 --numjobs=1
```

1. 挂载选项为：`defaults`，即`async,auto,dev,exec,nouser,rw,suid`
```bash
root@phytium-Ubuntu:~# fio --name=test --filename=./testfile --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k -rw=randrw -rwmixread=70 --iodepth=32 --numjobs=1
test: (g=0): rw=randrw, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.28
Starting 1 process
Jobs: 1 (f=1): [m(1)][100.0%][r=12.9MiB/s,w=5633KiB/s][r=3310,w=1408 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=676: Fri Sep  6 07:55:32 2024
  read: IOPS=3060, BW=12.0MiB/s (12.5MB/s)(717MiB/60006msec)
    slat (usec): min=6, max=51599, avg=103.00, stdev=471.54
    clat (usec): min=94, max=92706, avg=6246.97, stdev=4952.41
     lat (usec): min=376, max=92717, avg=6350.40, stdev=5002.08
    clat percentiles (usec):
     |  1.00th=[  660],  5.00th=[ 1139], 10.00th=[ 1729], 20.00th=[ 2868],
     | 30.00th=[ 4113], 40.00th=[ 5538], 50.00th=[ 6915], 60.00th=[ 7242],
     | 70.00th=[ 7439], 80.00th=[ 7701], 90.00th=[ 8291], 95.00th=[10290],
     | 99.00th=[26608], 99.50th=[44303], 99.90th=[57410], 99.95th=[58983],
     | 99.99th=[65799]
   bw (  KiB/s): min=10368, max=14168, per=100.00%, avg=12244.77, stdev=1157.43, samples=119
   iops        : min= 2592, max= 3542, avg=3061.19, stdev=289.36, samples=119
  write: IOPS=1314, BW=5258KiB/s (5384kB/s)(308MiB/60006msec); 0 zone resets
    slat (usec): min=7, max=50609, avg=104.11, stdev=514.07
    clat (usec): min=283, max=117574, avg=9442.34, stdev=8779.01
     lat (usec): min=324, max=117587, avg=9546.89, stdev=8778.79
    clat percentiles (usec):
     |  1.00th=[   474],  5.00th=[  1139], 10.00th=[  2278], 20.00th=[  4817],
     | 30.00th=[  6849], 40.00th=[  7242], 50.00th=[  7439], 60.00th=[  7701],
     | 70.00th=[  8356], 80.00th=[ 11863], 90.00th=[ 17171], 95.00th=[ 23725],
     | 99.00th=[ 50594], 99.50th=[ 58459], 99.90th=[ 77071], 99.95th=[ 84411],
     | 99.99th=[108528]
   bw (  KiB/s): min= 4296, max= 6272, per=99.97%, avg=5256.54, stdev=548.46, samples=119
   iops        : min= 1074, max= 1568, avg=1314.13, stdev=137.11, samples=119
  lat (usec)   : 100=0.01%, 250=0.01%, 500=0.45%, 750=1.64%, 1000=1.84%
  lat (msec)   : 2=7.37%, 4=14.17%, 10=63.15%, 20=8.25%, 50=2.67%
  lat (msec)   : 100=0.44%, 250=0.01%
  cpu          : usr=2.60%, sys=9.27%, ctx=290623, majf=0, minf=32
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=183641,78871,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: bw=12.0MiB/s (12.5MB/s), 12.0MiB/s-12.0MiB/s (12.5MB/s-12.5MB/s), io=717MiB (752MB), run=60006-60006msec
  WRITE: bw=5258KiB/s (5384kB/s), 5258KiB/s-5258KiB/s (5384kB/s-5384kB/s), io=308MiB (323MB), run=60006-60006msec

Disk stats (read/write):
  mmcblk0: ios=182649/78485, merge=550/181, ticks=642194/523736, in_queue=1165952, util=99.92%
```

2. 挂载选项为：`defaults,noatime,nodiratime`
```bash
root@phytium-Ubuntu:~# fio --name=test --filename=./testfile --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k -rw=randrw -rwmixread=70 --iodepth=32 --numjobs=1
test: (g=0): rw=randrw, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.28
Starting 1 process
Jobs: 1 (f=1): [m(1)][100.0%][r=13.5MiB/s,w=6208KiB/s][r=3456,w=1552 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=515: Fri Sep  6 07:46:06 2024
  read: IOPS=3107, BW=12.1MiB/s (12.7MB/s)(728MiB/60005msec)
    slat (usec): min=5, max=70145, avg=89.63, stdev=483.41
    clat (usec): min=379, max=105761, avg=6073.25, stdev=5218.38
     lat (usec): min=390, max=105779, avg=6163.36, stdev=5264.09
    clat percentiles (usec):
     |  1.00th=[  660],  5.00th=[ 1106], 10.00th=[ 1614], 20.00th=[ 2671],
     | 30.00th=[ 3818], 40.00th=[ 5080], 50.00th=[ 6456], 60.00th=[ 7046],
     | 70.00th=[ 7308], 80.00th=[ 7504], 90.00th=[ 8356], 95.00th=[10421],
     | 99.00th=[27132], 99.50th=[42206], 99.90th=[74974], 99.95th=[77071],
     | 99.99th=[83362]
   bw (  KiB/s): min=10248, max=14928, per=100.00%, avg=12431.73, stdev=1135.27, samples=119
   iops        : min= 2562, max= 3732, avg=3107.93, stdev=283.82, samples=119
  write: IOPS=1334, BW=5340KiB/s (5468kB/s)(313MiB/60005msec); 0 zone resets
    slat (usec): min=7, max=70310, avg=91.05, stdev=500.63
    clat (usec): min=262, max=143064, avg=9520.77, stdev=9213.57
     lat (usec): min=315, max=143083, avg=9612.31, stdev=9213.00
    clat percentiles (usec):
     |  1.00th=[  441],  5.00th=[ 1074], 10.00th=[ 2089], 20.00th=[ 4424],
     | 30.00th=[ 6521], 40.00th=[ 7046], 50.00th=[ 7308], 60.00th=[ 7570],
     | 70.00th=[ 8979], 80.00th=[12387], 90.00th=[17433], 95.00th=[24249],
     | 99.00th=[51119], 99.50th=[60031], 99.90th=[84411], 99.95th=[89654],
     | 99.99th=[98042]
   bw (  KiB/s): min= 4080, max= 6744, per=99.99%, avg=5339.56, stdev=544.30, samples=119
   iops        : min= 1020, max= 1686, avg=1334.89, stdev=136.07, samples=119
  lat (usec)   : 500=0.51%, 750=1.82%, 1000=1.94%
  lat (msec)   : 2=8.11%, 4=15.23%, 10=60.33%, 20=8.82%, 50=2.78%
  lat (msec)   : 100=0.46%, 250=0.01%
  cpu          : usr=2.53%, sys=9.81%, ctx=291060, majf=0, minf=30
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=186445,80100,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: bw=12.1MiB/s (12.7MB/s), 12.1MiB/s-12.1MiB/s (12.7MB/s-12.7MB/s), io=728MiB (764MB), run=60005-60005msec
  WRITE: bw=5340KiB/s (5468kB/s), 5340KiB/s-5340KiB/s (5468kB/s-5468kB/s), io=313MiB (328MB), run=60005-60005msec

Disk stats (read/write):
  mmcblk0: ios=185429/79803, merge=593/194, ticks=696072/574475, in_queue=1270564, util=99.87%
```

3. 挂载选项为`defaults`，关闭ext4日志功能
```bash
root@phytium-Ubuntu:~# fio --name=test --filename=./testfile --size=1G --time_based --runtime=60s --ioengine=libaio --direct=1 --bs=4k -rw=randrw -rwmixread=70 --iodepth=32 --numjobs=1
test: (g=0): rw=randrw, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=32
fio-3.28
Starting 1 process
Jobs: 1 (f=1): [m(1)][100.0%][r=13.2MiB/s,w=5465KiB/s][r=3376,w=1366 IOPS][eta 00m:00s]
test: (groupid=0, jobs=1): err= 0: pid=530: Fri Sep  6 08:21:12 2024
  read: IOPS=3161, BW=12.3MiB/s (12.9MB/s)(741MiB/60005msec)
    slat (usec): min=5, max=50246, avg=62.93, stdev=389.40
    clat (usec): min=359, max=90360, avg=5834.84, stdev=5192.93
     lat (usec): min=370, max=90378, avg=5898.15, stdev=5225.73
    clat percentiles (usec):
     |  1.00th=[  652],  5.00th=[ 1057], 10.00th=[ 1500], 20.00th=[ 2376],
     | 30.00th=[ 3326], 40.00th=[ 4359], 50.00th=[ 5473], 60.00th=[ 6718],
     | 70.00th=[ 7242], 80.00th=[ 7570], 90.00th=[ 8848], 95.00th=[10945],
     | 99.00th=[27395], 99.50th=[44303], 99.90th=[56886], 99.95th=[61080],
     | 99.99th=[77071]
   bw (  KiB/s): min= 9464, max=14408, per=100.00%, avg=12645.71, stdev=1019.47, samples=119
   iops        : min= 2366, max= 3602, avg=3161.41, stdev=254.86, samples=119
  write: IOPS=1356, BW=5426KiB/s (5557kB/s)(318MiB/60005msec); 0 zone resets
    slat (usec): min=7, max=49749, avg=62.14, stdev=304.68
    clat (usec): min=287, max=118046, avg=9768.26, stdev=9407.99
     lat (usec): min=322, max=118058, avg=9830.80, stdev=9403.13
    clat percentiles (usec):
     |  1.00th=[   429],  5.00th=[   947], 10.00th=[  1778], 20.00th=[  3785],
     | 30.00th=[  5735], 40.00th=[  6980], 50.00th=[  7373], 60.00th=[  7898],
     | 70.00th=[ 10290], 80.00th=[ 13304], 90.00th=[ 18744], 95.00th=[ 25822],
     | 99.00th=[ 51119], 99.50th=[ 58459], 99.90th=[ 80217], 99.95th=[ 88605],
     | 99.99th=[104334]
   bw (  KiB/s): min= 4120, max= 6544, per=100.00%, avg=5428.10, stdev=488.43, samples=119
   iops        : min= 1030, max= 1636, avg=1357.03, stdev=122.11, samples=119
  lat (usec)   : 500=0.60%, 750=2.03%, 1000=2.28%
  lat (msec)   : 2=9.38%, 4=17.81%, 10=53.84%, 20=10.44%, 50=3.14%
  lat (msec)   : 100=0.48%, 250=0.01%
  cpu          : usr=2.43%, sys=8.27%, ctx=286962, majf=0, minf=30
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=100.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.1%, 64=0.0%, >=64=0.0%
     issued rwts: total=189697,81403,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=32

Run status group 0 (all jobs):
   READ: bw=12.3MiB/s (12.9MB/s), 12.3MiB/s-12.3MiB/s (12.9MB/s-12.9MB/s), io=741MiB (777MB), run=60005-60005msec
  WRITE: bw=5426KiB/s (5557kB/s), 5426KiB/s-5426KiB/s (5557kB/s-5557kB/s), io=318MiB (333MB), run=60005-60005msec

Disk stats (read/write):
  mmcblk0: ios=188702/81118, merge=591/164, ticks=817839/669273, in_queue=1487112, util=99.95%
```

### 2.5 测试结果分析
![](https://raw.githubusercontent.com/JackHuang021/images/master/20240906164324.png)

从测试结果来看，禁止更新文件访问时间和关闭ext4日志功能都可以提升IO吞吐效率，但是提升的性能有限


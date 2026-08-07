---
title: 088_opencl
tags:
---

## 1. 介绍

OpenCL（Open Computing Language） 是一个开放的、跨平台的 异构计算框架，由 Khronos Group  在 2008 年提出。OpenCL 是一套标准 API + 编程模型。允许开发者使用统一的编程接口，把计算任务分配到不同硬件（CPU、GPU、DSP、FPGA 等）上运行。OpenCL 就是一种跨硬件的并行计算标准，让你能写一次代码，在多种硬件平台运行。

## 2. OpenCL 组成部分

### 2.1 Platform 模型

一个 OpenCL 平台包含：
+ Host（主机）：通常是 CPU，应用程序执行基座
+ Device（设备）：GPU、DSP、FPGA 等加速器，负责计算

### 2.2 Execution 模型

OpenCL 使用 kernel（核函数） 来描述计算任务，Kernel 被分发到 Device 上，执行在多个 Work-item（线程）和 Work-group（线程组）上

### 2.3 Memory 模型

层次化内存：
+ Global memory（全局内存，慢，容量大）
+ Local memory（工作组共享内存）
+ Private memory（线程私有寄存器/缓存）

### 2.4 Programming 模型

OpenCL 程序分为两部分：
+ Host code（C/C++ 编写，运行在 CPU 上，负责调度）
+ Kernel code（OpenCL C 编写，运行在 GPU/加速器上）

## 3. OpenCL 主要作用

### 3.1 跨平台通用计算

+ 支持 Intel/AMD/NVIDIA GPU
+ 支持 ARM Mali、Imagination PowerVR、Qualcomm Adreno 等移动 GPU
+ 支持 FPGA（Intel/Altera、Xilinx）
+ 支持 DSP 和专用 AI 加速芯片

### 3.2 高性能并行计算

特别适合 数据并行任务，如：
+ 图像处理（滤波、边缘检测）
+ 视频编解码
+ 科学计算（分子动力学、气象模拟）
+ 金融计算（蒙特卡洛模拟、风险分析）
+ 机器学习（矩阵乘法、卷积运算）

### 3.3 通用 GPU 计算 (GPGPU)

+ 通过 GPU 的强大并行能力，大幅提升计算速度。
+ 类似于 NVIDIA 的 CUDA，但 OpenCL 不绑定某一厂商。

### 3.4 嵌入式异构计算

在移动设备、车载系统、嵌入式板卡上，利用 GPU/DSP 加速 AI 和图像任务。

## 4. OpenCL 与其它框架比较

| 特性   | OpenCL                | CUDA         | Vulkan Compute | OpenGL Compute |
| ---- | --------------------- | ------------ | -------------- | -------------- |
| 厂商   | Khronos Group（开放标准）   | NVIDIA 专有    | Khronos Group  | Khronos Group  |
| 硬件支持 | CPU、GPU、DSP、FPGA（跨平台） | 仅 NVIDIA GPU | 主要 GPU（跨平台）    | 主要 GPU（跨平台）    |
| 编程难度 | 较高（需要显式管理内存/线程）       | 较高，但生态完善     | 更底层            | 图形 API 扩展      |
| 应用场景 | 通用异构计算                | GPU 高性能计算    | 高性能图形 + 计算     | 图形为主，计算为辅      |

## 5. OpenCL 框架 和 NPU 的对比

| 特性       | OpenCL                            | NPU                 |
| -------- | --------------------------------- | ------------------- |
| **本质**   | 编程框架（软件标准）                        | 专用硬件                |
| **硬件依赖** | CPU / GPU / DSP / FPGA 都可以跑 | 只能跑在 NPU 上          |
| **适用范围** | 通用并行计算                            | 专门为 AI 深度学习优化       |
| **性能**   | 在 AI 上比 NPU 慢（但更灵活）               | 在 AI 上快、能效高（但不灵活）   |
| **调用方式** | 需要写 OpenCL Kernel                 | 用厂商 SDK（如 RKNN）调用模型 |

## 6. 配置OpenCL Platform

一般通过 mesa-opencl-icd 来配置OpenCL platform，mesa-opencl-icd 提供 OpenCL 的 ICD (Installable Client Driver) 库 实现。ICD 的作用是允许系统同时存在多个 OpenCL 实现（比如 NVIDIA 的、AMD 的、Mesa 的），通过中间层 libOpenCL.so 来分发调用。

+ 安装mesa-opencl-icd: `sudo apt-get install mesa-opencl-icd`

### 配置 D3000M GPU OpenCL

+ 添加ICD配置文件，添加 ftg340.icd 配置文件（ftg340/libOpenCL.so由GPU deb包提供），内容如下：

	```bash
	user@phytium-Ubuntu:/etc/OpenCL/vendors$ cat ftg340.icd 
	/usr/lib/aarch64-linux-gnu/ftg340/libOpenCL.so
	```

### 配置 D3000M CPU OpenCL

POCL： POCL（Portable Computing Language）是一个由芬兰坦佩雷理工大学（Tampere University）主导开发的 开源 OpenCL 实现，它的目标是：让 OpenCL 程序可以在任意 CPU 或架构上运行。

安装 POCL 支持库：

  ```bash
  sudo apt install pocl-opencl-icd
  ```

PCOL的局限性如下：
  1. 性能不如 GPU，本质是 CPU 多线程执行 kernel
  2. 不支持所有 OpenCL 3.0 扩展，特定硬件特性无法模拟
  3. 不适合图形类任务，无 GPU 渲染单元支持
  4. 调度依赖线程池，高负载下开销较大

### D3000M clinfo 信息

```bash
root@phytium-Ubuntu:~# clinfo 
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
Number of platforms                               4
  Platform Name                                   Phytium OpenCL Platform
  Platform Vendor                                 Phytium Technology Co., Ltd.
  Platform Version                                OpenCL 3.0 V1.1.5
  Platform Profile                                FULL_PROFILE
  Platform Extensions                             cl_khr_byte_addressable_store cl_khr_fp16 cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_icd cl_khr_command_buffer 
  Platform Extensions with Version                cl_khr_byte_addressable_store                                    0x400000 (1.0.0)
                                                  cl_khr_fp16                                                      0x400000 (1.0.0)
                                                  cl_khr_global_int32_base_atomics                                 0x400000 (1.0.0)
                                                  cl_khr_global_int32_extended_atomics                             0x400000 (1.0.0)
                                                  cl_khr_local_int32_base_atomics                                  0x400000 (1.0.0)
                                                  cl_khr_local_int32_extended_atomics                              0x400000 (1.0.0)
                                                  cl_khr_icd                                                       0x400000 (1.0.0)
                                                  cl_khr_command_buffer                                            0x400000 (1.0.0)
  Platform Numeric Version                        0xc00000 (3.0.0)
  Platform Extensions function suffix             viv
  Platform Host timer resolution                  0ns

  Platform Name                                   Portable Computing Language
  Platform Vendor                                 The pocl project
  Platform Version                                OpenCL 3.0 PoCL 5.0+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 16.0.6, SLEEF, POCL_DEBUG
  Platform Profile                                FULL_PROFILE
  Platform Extensions                             cl_khr_icd cl_pocl_content_size
  Platform Extensions with Version                cl_khr_icd                                                       0x400000 (1.0.0)
                                                  cl_pocl_content_size                                             0x400000 (1.0.0)
  Platform Numeric Version                        0xc00000 (3.0.0)
  Platform Extensions function suffix             POCL
  Platform Host timer resolution                  0ns

  Platform Name                                   Clover
  Platform Vendor                                 Mesa
  Platform Version                                OpenCL 1.1 Mesa 25.0.7-0ubuntu0.24.04.2
  Platform Profile                                FULL_PROFILE
  Platform Extensions                             cl_khr_icd
  Platform Extensions function suffix             MESA

  Platform Name                                   rusticl
  Platform Vendor                                 Mesa/X.org
  Platform Version                                OpenCL 3.0 
  Platform Profile                                FULL_PROFILE
  Platform Extensions                             cl_khr_icd
  Platform Extensions with Version                cl_khr_icd                                                       0x400000 (1.0.0)
  Platform Numeric Version                        0xc00000 (3.0.0)
  Platform Extensions function suffix             MESA
  Platform Host timer resolution                  1ns

  Platform Name                                   Phytium OpenCL Platform
Number of devices                                 1
  Device Name                                     Phytium OpenCL Device FTG340
  Device Vendor                                   Phytium Technology Co., Ltd.
  Device Vendor ID                                0x564956
  Device Version                                  OpenCL 3.0 
  Device Numeric Version                          0xc00000 (3.0.0)
  Driver Version                                  OpenCL 3.0 V1.1.5
  Device OpenCL C Version                         OpenCL C 1.2 
  Device OpenCL C all versions                    OpenCL C                                                         0x400000 (1.0.0)
                                                  OpenCL C                                                         0x401000 (1.1.0)
                                                  OpenCL C                                                         0x402000 (1.2.0)
                                                  OpenCL C                                                         0xc00000 (3.0.0)
  Device OpenCL C features                        __opencl_c_images                                                0x400000 (1.0.0)
                                                  __opencl_c_int64                                                 0x400000 (1.0.0)
  Latest conformance test passed                  v2021-03-25-00
  Device Type                                     GPU
  Device Profile                                  FULL_PROFILE
  Device Available                                Yes
  Compiler Available                              Yes
  Linker Available                                Yes
  Max compute units                               1
  Max clock frequency                             200MHz
  Device Partition                                (core)
    Max number of sub-devices                     0
    Supported partition types                     None
    Supported affinity domains                    (n/a)
  Max work item dimensions                        3
  Max work item sizes                             1024x1024x1024
  Max work group size                             1024
  Preferred work group size multiple (device)     64
  Preferred work group size multiple (kernel)     64
  Max sub-groups per work group                   0
  Preferred / native vector sizes                 
    char                                                 4 / 4       
    short                                                4 / 4       
    int                                                  4 / 4       
    long                                                 4 / 4       
    half                                                 4 / 4        (cl_khr_fp16)
    float                                                4 / 4       
    double                                               0 / 0        (n/a)
  Half-precision Floating-point support           (cl_khr_fp16)
    Denormals                                     No
    Infinity and NANs                             Yes
    Round to nearest                              Yes
    Round to zero                                 Yes
    Round to infinity                             No
    IEEE754-2008 fused multiply-add               No
    Support is emulated in software               No
  Single-precision Floating-point support         (core)
    Denormals                                     No
    Infinity and NANs                             Yes
    Round to nearest                              Yes
    Round to zero                                 Yes
    Round to infinity                             No
    IEEE754-2008 fused multiply-add               No
    Support is emulated in software               No
    Correctly-rounded divide and sqrt operations  No
  Double-precision Floating-point support         (n/a)
  Address bits                                    32, Little-Endian
  Global memory size                              268435456 (256MiB)
  Error Correction support                        Yes
  Max memory allocation                           134217728 (128MiB)
  Unified memory for Host and Device              Yes
  Shared Virtual Memory (SVM) capabilities        (core)
    Coarse-grained buffer sharing                 No
    Fine-grained buffer sharing                   No
    Fine-grained system sharing                   No
    Atomics                                       No
  Minimum alignment for any data type             128 bytes
  Alignment of base address                       2048 bits (256 bytes)
  Preferred alignment for atomics                 
    SVM                                           0 bytes
    Global                                        0 bytes
    Local                                         0 bytes
  Atomic memory capabilities                      relaxed, work-group scope
  Atomic fence capabilities                       relaxed, acquire/release, work-group scope
  Max size for global variable                    0
  Preferred total size of global vars             0
  Global Memory cache type                        Read/Write
  Global Memory cache size                        65536 (64KiB)
  Global Memory cache line size                   64 bytes
  Image support                                   Yes
    Max number of samplers per kernel             16
    Max size for 1D images from buffer            65536 pixels
    Max 1D or 2D image array size                 8192 images
    Base address alignment for 2D image buffers   0 bytes
    Pitch alignment for 2D image buffers          0 pixels
    Max 2D image size                             16384x16384 pixels
    Max 3D image size                             16384x16384x8192 pixels
    Max number of read image args                 128
    Max number of write image args                8
    Max number of read/write image args           0
  Pipe support                                    No
  Max number of pipe args                         0
  Max active pipe reservations                    0
  Max pipe packet size                            0
  Local memory type                               Global
  Local memory size                               32768 (32KiB)
  Max number of constant args                     9
  Max constant buffer size                        65536 (64KiB)
  Generic address space support                   No
  Max size of kernel argument                     1024
  Queue properties (on host)                      
    Out-of-order execution                        Yes
    Profiling                                     Yes
  Device enqueue capabilities                     (n/a)
  Queue properties (on device)                    
    Out-of-order execution                        No
    Profiling                                     No
    Preferred size                                0
    Max size                                      0
  Max queues on device                            0
  Max events on device                            0
  Command buffer capabilities                     (n/a)
    Required queue properties for command buffer  
    Out-of-order execution                        No
    Profiling                                     No
  Prefer user sync for interop                    Yes
  Profiling timer resolution                      1000ns
  Execution capabilities                          
    Run OpenCL kernels                            Yes
    Run native kernels                            No
    Non-uniform work-groups                       No
    Work-group collective functions               No
    Sub-group independent forward progress        No
    IL version                                    (n/a)
    ILs with version                              (n/a)
  printf() buffer size                            1048576 (1024KiB)
  Built-in kernels                                (n/a)
  Built-in kernels with version                   (n/a)
  Device Extensions                               cl_khr_byte_addressable_store cl_khr_fp16 cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_icd cl_khr_command_buffer 
  Device Extensions with Version                  cl_khr_byte_addressable_store                                    0x400000 (1.0.0)
                                                  cl_khr_fp16                                                      0x400000 (1.0.0)
                                                  cl_khr_global_int32_base_atomics                                 0x400000 (1.0.0)
                                                  cl_khr_global_int32_extended_atomics                             0x400000 (1.0.0)
                                                  cl_khr_local_int32_base_atomics                                  0x400000 (1.0.0)
                                                  cl_khr_local_int32_extended_atomics                              0x400000 (1.0.0)
                                                  cl_khr_icd                                                       0x400000 (1.0.0)
                                                  cl_khr_command_buffer                                            0x400000 (1.0.0)

  Platform Name                                   Portable Computing Language
Number of devices                                 1
  Device Name                                     cpu--0x862
  Device Vendor                                   0x70
  Device Vendor ID                                0x13b5
  Device Version                                  OpenCL 3.0 PoCL HSTR: cpu-aarch64-unknown-linux-gnu-(null)
  Device Numeric Version                          0xc00000 (3.0.0)
  Driver Version                                  5.0+debian
  Device OpenCL C Version                         OpenCL C 1.2 PoCL
  Device OpenCL C all versions                    OpenCL C                                                         0x400000 (1.0.0)
                                                  OpenCL C                                                         0x401000 (1.1.0)
                                                  OpenCL C                                                         0x402000 (1.2.0)
                                                  OpenCL C                                                         0xc00000 (3.0.0)
  Device OpenCL C features                        __opencl_c_3d_image_writes                                       0xc00000 (3.0.0)
                                                  __opencl_c_images                                                0xc00000 (3.0.0)
                                                  __opencl_c_atomic_order_acq_rel                                  0xc00000 (3.0.0)
                                                  __opencl_c_atomic_order_seq_cst                                  0xc00000 (3.0.0)
                                                  __opencl_c_atomic_scope_device                                   0xc00000 (3.0.0)
                                                  __opencl_c_program_scope_global_variables                        0xc00000 (3.0.0)
                                                  __opencl_c_generic_address_space                                 0xc00000 (3.0.0)
                                                  __opencl_c_subgroups                                             0xc00000 (3.0.0)
                                                  __opencl_c_atomic_scope_all_devices                              0xc00000 (3.0.0)
                                                  __opencl_c_read_write_images                                     0xc00000 (3.0.0)
                                                  __opencl_c_fp64                                                  0xc00000 (3.0.0)
                                                  __opencl_c_ext_fp32_global_atomic_add                            0xc00000 (3.0.0)
                                                  __opencl_c_ext_fp32_local_atomic_add                             0xc00000 (3.0.0)
                                                  __opencl_c_ext_fp32_global_atomic_min_max                        0xc00000 (3.0.0)
                                                  __opencl_c_ext_fp32_local_atomic_min_max                         0xc00000 (3.0.0)
                                                  __opencl_c_ext_fp64_global_atomic_add                            0xc00000 (3.0.0)
                                                  __opencl_c_ext_fp64_local_atomic_add                             0xc00000 (3.0.0)
                                                  __opencl_c_ext_fp64_global_atomic_min_max                        0xc00000 (3.0.0)
                                                  __opencl_c_ext_fp64_local_atomic_min_max                         0xc00000 (3.0.0)
                                                  __opencl_c_int64                                                 0xc00000 (3.0.0)
  Latest conformance test passed                  v2022-04-19-01
  Device Type                                     CPU
  Device Profile                                  FULL_PROFILE
  Device Available                                Yes
  Compiler Available                              Yes
  Linker Available                                Yes
  Max compute units                               8
  Max clock frequency                             1900MHz
  Device Partition                                (core)
    Max number of sub-devices                     8
    Supported partition types                     equally, by counts
    Supported affinity domains                    (n/a)
  Max work item dimensions                        3
  Max work item sizes                             4096x4096x4096
  Max work group size                             4096
  Preferred work group size multiple (device)     8
  Preferred work group size multiple (kernel)     8
  Max sub-groups per work group                   128
  Sub-group sizes (Intel)                         1, 2, 4, 8, 16, 32, 64, 128, 256, 512
  Preferred / native vector sizes                 
    char                                                16 / 16      
    short                                                8 / 8       
    int                                                  4 / 4       
    long                                                 2 / 2       
    half                                                 0 / 0        (n/a)
    float                                                4 / 4       
    double                                               2 / 2        (cl_khr_fp64)
  Half-precision Floating-point support           (n/a)
  Single-precision Floating-point support         (core)
    Denormals                                     No
    Infinity and NANs                             Yes
    Round to nearest                              Yes
    Round to zero                                 No
    Round to infinity                             No
    IEEE754-2008 fused multiply-add               No
    Support is emulated in software               No
    Correctly-rounded divide and sqrt operations  No
  Double-precision Floating-point support         (cl_khr_fp64)
    Denormals                                     Yes
    Infinity and NANs                             Yes
    Round to nearest                              Yes
    Round to zero                                 Yes
    Round to infinity                             Yes
    IEEE754-2008 fused multiply-add               Yes
    Support is emulated in software               No
  Address bits                                    64, Little-Endian
  Global memory size                              14965276672 (13.94GiB)
  Error Correction support                        No
  Max memory allocation                           4294967296 (4GiB)
  Unified memory for Host and Device              Yes
  Shared Virtual Memory (SVM) capabilities        (core)
    Coarse-grained buffer sharing                 Yes
    Fine-grained buffer sharing                   Yes
    Fine-grained system sharing                   No
    Atomics                                       Yes
  Unified Shared Memory (USM)                     (cl_intel_unified_shared_memory)
  Host USM capabilities (Intel)                   USM access, USM atomic access
  Device USM capabilities (Intel)                 USM access, USM atomic access
  Single-Device USM caps (Intel)                  USM access, USM atomic access
  Cross-Device USM caps (Intel)                   (n/a)
  Shared System USM caps (Intel)                  (n/a)
  Minimum alignment for any data type             128 bytes
  Alignment of base address                       1024 bits (128 bytes)
  Preferred alignment for atomics                 
    SVM                                           64 bytes
    Global                                        64 bytes
    Local                                         64 bytes
  Atomic memory capabilities                      relaxed, acquire/release, sequentially-consistent, work-group scope, device scope, all-devices scope
  Atomic fence capabilities                       relaxed, acquire/release, sequentially-consistent, work-item scope, work-group scope, device scope
  Max size for global variable                    64000 (62.5KiB)
  Preferred total size of global vars             524288 (512KiB)
  Global Memory cache type                        Read/Write
  Global Memory cache size                        2097152 (2MiB)
  Global Memory cache line size                   64 bytes
  Image support                                   Yes
    Max number of samplers per kernel             16
    Max size for 1D images from buffer            268435456 pixels
    Max 1D or 2D image array size                 2048 images
    Base address alignment for 2D image buffers   0 bytes
    Pitch alignment for 2D image buffers          0 pixels
    Max 2D image size                             16384x16384 pixels
    Max 3D image size                             2048x2048x2048 pixels
    Max number of read image args                 128
    Max number of write image args                128
    Max number of read/write image args           128
  Pipe support                                    No
  Max number of pipe args                         0
  Max active pipe reservations                    0
  Max pipe packet size                            0
  Local memory type                               Global
  Local memory size                               524288 (512KiB)
  Max number of constant args                     8
  Max constant buffer size                        524288 (512KiB)
  Generic address space support                   Yes
  Max size of kernel argument                     1024
  Queue properties (on host)                      
    Out-of-order execution                        Yes
    Profiling                                     Yes
  Device enqueue capabilities                     (n/a)
  Queue properties (on device)                    
    Out-of-order execution                        No
    Profiling                                     No
    Preferred size                                0
    Max size                                      0
  Max queues on device                            0
  Max events on device                            0
  Command buffer capabilities                     kernel printf, simultaneous use, out of order
    Required queue properties for command buffer  
    Out-of-order execution                        No
    Profiling                                     No
  Prefer user sync for interop                    Yes
  Profiling timer resolution                      1ns
  Execution capabilities                          
    Run OpenCL kernels                            Yes
    Run native kernels                            Yes
    Non-uniform work-groups                       No
    Work-group collective functions               No
    Sub-group independent forward progress        Yes
    IL version                                    (n/a)
    ILs with version                              (n/a)
    SPIR versions                                 (n/a)
  printf() buffer size                            16777216 (16MiB)
  Built-in kernels                                (n/a)
  Built-in kernels with version                   (n/a)
  Device Extensions                               cl_khr_byte_addressable_store cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atomics cl_khr_3d_image_writes cl_khr_command_buffer cl_pocl_pinned_buffers cl_khr_subgroups cl_intel_unified_shared_memory cl_khr_subgroup_ballot cl_khr_subgroup_shuffle cl_intel_subgroups cl_intel_required_subgroup_size cl_ext_float_atomics cl_khr_spir cl_khr_fp64 cl_khr_int64_base_atomics cl_khr_int64_extended_atomics
  Device Extensions with Version                  cl_khr_byte_addressable_store                                    0x400000 (1.0.0)
                                                  cl_khr_global_int32_base_atomics                                 0x400000 (1.0.0)
                                                  cl_khr_global_int32_extended_atomics                             0x400000 (1.0.0)
                                                  cl_khr_local_int32_base_atomics                                  0x400000 (1.0.0)
                                                  cl_khr_local_int32_extended_atomics                              0x400000 (1.0.0)
                                                  cl_khr_3d_image_writes                                           0x400000 (1.0.0)
                                                  cl_khr_command_buffer                                              0x9004 (0.9.4)
                                                  cl_pocl_pinned_buffers                                             0x1000 (0.1.0)
                                                  cl_khr_subgroups                                                 0x400000 (1.0.0)
                                                  cl_intel_unified_shared_memory                                   0x400000 (1.0.0)
                                                  cl_khr_subgroup_ballot                                           0x400000 (1.0.0)
                                                  cl_khr_subgroup_shuffle                                          0x400000 (1.0.0)
                                                  cl_intel_subgroups                                               0x400000 (1.0.0)
                                                  cl_intel_required_subgroup_size                                  0x400000 (1.0.0)
                                                  cl_ext_float_atomics                                             0x400000 (1.0.0)
                                                  cl_khr_spir                                                      0x801000 (2.1.0)
                                                  cl_khr_fp64                                                      0x400000 (1.0.0)
                                                  cl_khr_int64_base_atomics                                        0x400000 (1.0.0)
                                                  cl_khr_int64_extended_atomics                                    0x400000 (1.0.0)


  Platform Name                                   Clover
Number of devices                                 0

  Platform Name                                   rusticl
Number of devices                                 0

NULL platform behavior
  clGetPlatformInfo(NULL, CL_PLATFORM_NAME, ...)  Phytium OpenCL Platform
  clGetDeviceIDs(NULL, CL_DEVICE_TYPE_ALL, ...)   Success [viv]
  clCreateContext(NULL, ...) [default]            Success [viv]
  clCreateContext(NULL, ...) [other]              Success [POCL]
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_DEFAULT)  Success (1)
    Platform Name                                 Phytium OpenCL Platform
    Device Name                                   Phytium OpenCL Device FTG340
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_CPU)  No devices found in platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_GPU)  Success (1)
    Platform Name                                 Phytium OpenCL Platform
    Device Name                                   Phytium OpenCL Device FTG340
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_ACCELERATOR)  No devices found in platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_CUSTOM)  No devices found in platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_ALL)  Success (1)
    Platform Name                                 Phytium OpenCL Platform
    Device Name                                   Phytium OpenCL Device FTG340

ICD loader properties
  ICD loader Name                                 OpenCL ICD Loader
  ICD loader Vendor                               OCL Icd free software
  ICD loader Version                              2.3.2
  ICD loader Profile                              OpenCL 3.0
```

## 7. clpeak 测试工具

clpeak 是一个开源的 OpenCL 基准测试工具，主要用于测试 GPU / CPU / NPU 等设备在 OpenCL 下的计算性能。它不会跑复杂的应用，而是通过一些简单的 OpenCL kernel，来测量设备的极限性能指标。

项目地址（GitHub）：[https://github.com/krrishnarraj/clpeak](https://github.com/krrishnarraj/clpeak)

### 7.1 测试内容

clpeak 会在系统里的所有 OpenCL Platform 和 Device 上运行测试，输出以下指标：

+ Global memory bandwidth: 测试设备的全局显存读写带宽（GB/s）。
+ Compute FLOPS (SP / DP / HP): 单精度（FP32）、半精度（FP16）的浮点运算性能（GFLOPS / TFLOPS）。
+ Integer compute performance: 整数运算性能（GIOPS）。
+ Transfer bandwidth (Host ↔ Device): 主机内存和设备显存之间的数据传输带宽（GB/s）
+ Kernel launch latency: OpenCL 内核启动的延迟（µs）。

### 7.2 clpeak 测试结果

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250930074220.png)

rk3588 OpenCL clpeak 跑分结果，使用闭源 mali 显卡驱动

```bash
jack@imb3588:~$ clpeak 
arm_release_ver: g24p0-00eac0, rk_so_ver: 6

Platform: ARM Platform
  Device: Mali-G610 r0p0
    Driver version  : 3.0 (Linux ARM64)
    Compute units   : 4
    Clock frequency : 1000 MHz

    Global memory bandwidth (GBPS)
      float   : 27.69
      float2  : 27.15
      float4  : 27.32
      float8  : 25.83
      float16 : 13.39

    Single-precision compute (GFLOPS)
      float   : 434.07
      float2  : 469.70
      float4  : 482.01
      float8  : 487.88
      float16 : 487.54

    Half-precision compute (GFLOPS)
      half   : 441.37
      half2  : 871.65
      half4  : 918.11
      half8  : 945.20
      half16 : 949.92

    No double precision support! Skipped

    Integer compute (GIOPS)
      int   : 125.09
      int2  : 125.41
      int4  : 125.89
      int8  : 125.75
      int16 : 126.26

    Integer compute Fast 24bit (GIOPS)
      int   : 124.99
      int2  : 125.42
      int4  : 125.78
      int8  : 125.84
      int16 : 126.45

    Transfer bandwidth (GBPS)
		enqueueWriteBuffer              : 8.57
		enqueueReadBuffer               : 10.09
		enqueueWriteBuffer non-blocking : 8.56
		enqueueReadBuffer non-blocking  : 10.09
		enqueueMapBuffer(for read)      : 61.25
		memcpy from mapped ptr        : 11.67
		enqueueUnmap(after write)       : 62.19
		memcpy to mapped ptr          : 10.62

    Kernel launch latency : 32.18 us
```

D3000M OpenCL clpeak 测试结果，显卡驱动版本 phytium-d3000m-gpu-driver_1.1.14

将显卡频率设置为800MHz

```bash
root@phytium-Ubuntu:/sys/devices/platform/PHYT0048:00/devfreq/PHYT0048:00# echo userspace > governor
root@phytium-Ubuntu:/sys/devices/platform/PHYT0048:00/devfreq/PHYT0048:00# echo 800000 > userspace/set_freq 
```

```bash
root@phytium-Ubuntu:~# clpeak 
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information
MESA-LOADER: failed to retrieve device information

Platform: Phytium OpenCL Platform
  Device: Phytium OpenCL Device FTG340
    Driver version  : OpenCL 3.0 V1.1.2 (Linux ARM64)
    Compute units   : 4
    Clock frequency : 800 MHz

    Global memory bandwidth (GBPS)
      float   : 28.03
      float2  : 35.36
      float4  : 35.41
      float8  : 29.67
      float16 : 21.33

    Single-precision compute (GFLOPS)
      float   : 102.24
      float2  : 203.59
      float4  : 403.99
      float8  : 400.14
      float16 : 397.09

    Half-precision compute (GFLOPS)
      half   : 203.87
      half2  : 405.17
      half4  : 800.13
      half8  : 792.41
      half16 : 786.47

    No double precision support! Skipped

    Integer compute (GIOPS)
      int   : 102.18
      int2  : 68.08
      int4  : 81.53
      int8  : 81.21
      int16 : 81.02

    Integer compute Fast 24bit (GIOPS)
      int   : 102.19
      int2  : 68.08
      int4  : 81.53
      int8  : 81.22
      int16 : 81.02

    Transfer bandwidth (GBPS)
      enqueueWriteBuffer              : 9.73
      enqueueReadBuffer               : 2.41
      enqueueWriteBuffer non-blocking : 12.71
      enqueueReadBuffer non-blocking  : 2.41
      enqueueMapBuffer(for read)      : 30504.03
        memcpy from mapped ptr        : 2.41
      enqueueUnmap(after write)       : 32736.03
        memcpy to mapped ptr          : 12.71

    Kernel launch latency : 8.19 us
```

### 7.3 测试结果分析

D3000M 测试成绩要略差于 RK3588，D3000M 在带宽测试中有的测试项数据异常，可能和GPU驱动有关系。

### 7.4 带宽测试项异常分析

D3000M 相对 RK3588，clpeak 测试数据异常的项有 enqueueMapBuffer(for read) 和 enqueueMapBuffer(after write)，其中 D3000M 测出来的数据为 4000 多GBPS，RK3588 测出来的数据为 60 GBPS 左右。

D3000M 带宽测试成绩

```bash
    Transfer bandwidth (GBPS)
      enqueueWriteBuffer              : 8.47
      enqueueReadBuffer               : 2.02
      enqueueWriteBuffer non-blocking : 11.30
      enqueueReadBuffer non-blocking  : 2.03
      enqueueMapBuffer(for read)      : 4329.60
        memcpy from mapped ptr        : 2.00
      enqueueUnmap(after write)       : 4676.58
        memcpy to mapped ptr          : 11.43
```

RK3588 带宽测试成绩

```bash
    Transfer bandwidth (GBPS)
		enqueueWriteBuffer              : 8.57
		enqueueReadBuffer               : 10.09
		enqueueWriteBuffer non-blocking : 8.56
		enqueueReadBuffer non-blocking  : 10.09
		enqueueMapBuffer(for read)      : 61.25
		memcpy from mapped ptr        : 11.67
		enqueueUnmap(after write)       : 62.19
		memcpy to mapped ptr          : 10.62
```

enqueueMapBuffer(for read) 测试过程：将一个OpenCL Buffer 映射到 Host 地址空间，返回一个可以被 CPU 访问的指针，它让你可以直接在 CPU 端访问 GPU 处理的 buffer 内存，而不必手动调用 clEnqueueReadBuffer() / clEnqueueWriteBuffer() 去复制数据。clpeak 会循环测试 20 次调用 clEnqueueMapBuffer() 进行映射，然后统计使用时间，计算带宽。

clpeak enqueueMapBuffer(for read)测试源码如下：

```c
// transfer_bandwidth.cpp
// 将 clBuffer 映射到 Host 地址空间，返回 CPU 可访问的指针地址

cl::Buffer clBuffer = cl::Buffer(ctx, (CL_MEM_READ_WRITE | CL_MEM_ALLOC_HOST_PTR), (numItems * sizeof(float)));

for (uint i = 0; i < iters; i++)
{
  Timer timer;
  void *mapPtr;

  timer.start();
  mapPtr = queue.enqueueMapBuffer(clBuffer, CL_TRUE, CL_MAP_READ, 0, (numItems * sizeof(float)));
  queue.finish();
  timed += timer.stopAndTime();

  queue.enqueueUnmapMemObject(clBuffer, mapPtr);
  queue.finish();
}
timed /= static_cast<float>(iters);

gbps = ((float)numItems * sizeof(float)) / timed / 1e3f;
log->print(gbps);
log->print(NEWLINE);
log->xmlRecord("enqueuemapbuffer", gbps);
```

clEnqueueMapBuffer() 的调用流程如下，可拆分为四个阶段

```bash
用户态应用 clpeak 调用 clEnqueueMapBuffer()
  ↓
OpenCL Runtime（libOpenCL.so）
  ↓
Vendor 用户态驱动（pipe_kmsro.so）
  ↓
内核 GPU 驱动（ftg340.ko） hal/os/linux/kernel/gc_hal_kernel_driver.c
drv_ioctl() -> gckDEVICE_Dispatch() -> gckKERNEL_Dispatch() -> gcvHAL_MAP_MEMORY
```

通过 strace 分析 D3000M 和 RK3588 clpeak 测试 enqueueMapBuffer(for read) 的系统调用，D3000M 只有在第一次进行 clEnqueueMapBuffer() 时调用了 ioctl() 进入到了内核GPU驱动进行内存映射，后面均直接从 runtime 中返回了，而 RK3588 在 clEnqueueMapBuffer() 映射过程中则每次都调到了内核驱动层，导致两者测试数据差异比较大

D3000M strace 信息

```bash
5600  16:47:01.233657 write(1, "\n    Transfer bandwidth (GBPS)\n", 31) = 31
5600  16:47:01.233704 write(1, "      enqueueMapBuffer(for read)"..., 40) = 40
5600  16:47:01.233741 write(1, "\n", 1) = 1
5600  16:47:01.233766 write(1, "enqueueMapBuffer", 16) = 16
5600  16:47:01.233819 ioctl(4, _IOC(_IOC_NONE, 0x75, 0x30, 0), 0xfffff86ecf88) = 0
5600  16:47:01.233921 ioctl(4, _IOC(_IOC_NONE, 0x75, 0x30, 0), 0xfffff86ecf78) = 0
5600  16:47:01.234407 write(1, "\n", 1) = 1
5600  16:47:01.234679 write(1, "enqueueMapBuffer", 16) = 16
5600  16:47:01.234852 write(1, "\n", 1) = 1
5600  16:47:01.235056 write(1, "enqueueMapBuffer", 16) = 16
5600  16:47:01.235228 write(1, "\n", 1) = 1
5600  16:47:01.235455 write(1, "enqueueMapBuffer", 16) = 16
```

RK3588 strace 信息

```bash
5665  write(1, "\n    Transfer bandwidth (GBPS)\n", 31) = 31
5665  write(1, "      enqueueMapBuffer(for read)"..., 40) = 40
5665  write(6, "\1\0\0\0\0\0\0\0", 8)   = 8
5665  write(1, "\n", 1 <unfinished ...>
5669  <... ppoll resumed>)              = 1 ([{fd=6, revents=POLLIN}], left {tv_sec=58, tv_nsec=658699566})
5665  <... write resumed>)              = 1
5669  read(6,  <unfinished ...>
5665  write(1, "enqueueMapBuffer", 16 <unfinished ...>
5669  <... read resumed>"\1\0\0\0\0\0\0\0", 8) = 8
5665  <... write resumed>)              = 16
5665  write(6, "\1\0\0\0\0\0\0\0", 8 <unfinished ...>
5669  ppoll([{fd=5, events=POLLIN}, {fd=6, events=POLLIN}, {fd=7, events=POLLIN}], 3, {tv_sec=60, tv_nsec=0}, NULL, 0 <unfinished ...>
5665  <... write resumed>)              = 8
5669  <... ppoll resumed>)              = 1 ([{fd=6, revents=POLLIN}], left {tv_sec=59, tv_nsec=999994750})
5669  read(6, "\1\0\0\0\0\0\0\0", 8)    = 8
5669  ppoll([{fd=5, events=POLLIN}, {fd=6, events=POLLIN}, {fd=7, events=POLLIN}], 3, {tv_sec=60, tv_nsec=0}, NULL, 0 <unfinished ...>
5665  ioctl(5, _IOC(_IOC_NONE, 0x80, 0x2c, 0), 0) = 0
5669  <... ppoll resumed>)              = 1 ([{fd=5, revents=POLLIN}], left {tv_sec=59, tv_nsec=999374979})
5665  ioctl(5, _IOC(_IOC_NONE, 0x80, 0x2c, 0), 0) = 0
5669  read(5,  <unfinished ...>
5665  write(6, "\1\0\0\0\0\0\0\0", 8)   = 8
5669  <... read resumed>"\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 64) = 64
5665  write(1, "\n", 1)                 = 1
5669  futex(0xaaaade917708, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
5665  write(1, "140599976001536.00", 18) = 18
5665  write(6, "\1\0\0\0\0\0\0\0", 8 <unfinished ...>
5671  <... futex resumed>)              = 0
5669  <... futex resumed>)              = 1
5665  <... write resumed>)              = 8
5671  futex(0xaaaade95e0a8, FUTEX_WAIT_PRIVATE, 2, NULL <unfinished ...>
5665  write(6, "\1\0\0\0\0\0\0\0", 8)   = 8
5669  futex(0xaaaade95e0a8, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
5665  ioctl(5, _IOC(_IOC_NONE, 0x80, 0x2c, 0), 0) = 0
5665  ioctl(5, _IOC(_IOC_NONE, 0x80, 0x2c, 0) <unfinished ...>
5669  <... futex resumed>)              = 1
5671  <... futex resumed>)              = 0
5665  <... ioctl resumed>, 0)           = 0
5671  futex(0xaaaade95e0a8, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
5665  write(1, "\n", 1 <unfinished ...>
5671  <... futex resumed>)              = 0
5665  <... write resumed>)              = 1
5671  futex(0xaaaade917708, FUTEX_WAIT_BITSET_PRIVATE|FUTEX_CLOCK_REALTIME, 0, NULL, FUTEX_BITSET_MATCH_ANY <unfinished ...>
5669  ppoll([{fd=5, events=POLLIN}, {fd=6, events=POLLIN}, {fd=7, events=POLLIN}], 3, {tv_sec=60, tv_nsec=0}, NULL, 0 <unfinished ...>
5665  write(1, "enqueueMapBuffer", 16)  = 16
5665  write(6, "\1\0\0\0\0\0\0\0", 8 <unfinished ...>
5669  <... ppoll resumed>)              = 2 ([{fd=5, revents=POLLIN}, {fd=6, revents=POLLIN}], left {tv_sec=59, tv_nsec=999991542})
5665  <... write resumed>)              = 8
5669  read(5, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 64) = 64
5669  futex(0xaaaade917708, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
5671  <... futex resumed>)              = 0
5669  <... futex resumed>)              = 1
5671  futex(0xaaaade95e0a8, FUTEX_WAIT_PRIVATE, 2, NULL <unfinished ...>
5669  futex(0xaaaade95e0a8, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
5671  <... futex resumed>)              = -1 EAGAIN (Resource temporarily unavailable)
5669  <... futex resumed>)              = 0
5671  futex(0xaaaade95e0a8, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
5665  ioctl(5, _IOC(_IOC_NONE, 0x80, 0x2c, 0) <unfinished ...>
```

推测是 vendor runtime 层的实现有区别，RK3588 在 clEnqueueMapBuffer() 调用过程中每次都调到了内核驱动层，内核态和用户态的切换导致了 RK3588 的测试数据偏低，实际 enqueueMapBuffer(for read) 并不能表示真实内存带宽，D3000M 和 RK3588 都是统一内存平台，clEnqueueMapBuffer()的过程应该都是没有内存拷贝开销的，map 后的memcpy()速度，两者的测试成绩在一个数量级。

## 8. OpenCL 应用

### 8.1 OpenCL 应用流程

+ 发现平台和设备：调用 clGetPlatformIDs、clGetDeviceIDs
+ 创建上下文和命令队列：clCreateContext、clCreateCommandQueue
+ 分配内存对象：clCreateBuffer
+ 编译核函数（kernel）：clCreateProgramWithSource、clBuildProgram
+ 设置参数并执行 kernel：clSetKernelArg、clEnqueueNDRangeKernel
+ 读取结果：clEnqueueReadBuffer

### 8.2 demo 程序

枚举平台和设备，编译一个简单的 OpenCL kernel (C[i] = A[i] + B[i])，在设备上运行并返回结果

```c
#include <CL/cl.h>
#include <stdio.h>
#include <stdlib.h>

#define N 10

// OpenCL kernel
const char *programSource =
"__kernel void vec_add(__global float *A, __global float *B, __global float *C) { \n"
"    int id = get_global_id(0);                                                  \n"
"    C[id] = A[id] + B[id];                                                     \n"
"}                                                                               \n";

int main() {
    // 1. 获取平台
    cl_platform_id platform;
    clGetPlatformIDs(1, &platform, NULL);

    // 2. 获取设备
    cl_device_id device;
    clGetDeviceIDs(platform, CL_DEVICE_TYPE_GPU, 1, &device, NULL);

    // 3. 创建上下文
    cl_int status;
    cl_context context = clCreateContext(NULL, 1, &device, NULL, NULL, &status);

    // 4. 创建命令队列
    cl_command_queue queue = clCreateCommandQueue(context, device, 0, &status);

    // 5. 创建程序对象
    cl_program program = clCreateProgramWithSource(context, 1, &programSource, NULL, &status);
    status = clBuildProgram(program, 1, &device, NULL, NULL, NULL);

    // 6. 创建内核
    cl_kernel kernel = clCreateKernel(program, "vec_add", &status);

    // 7. 创建输入输出数据
    float A[N], B[N], C[N];
    for (int i = 0; i < N; i++) {
        A[i] = i;
        B[i] = i * 2;
    }

    cl_mem bufA = clCreateBuffer(context, CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR, N * sizeof(float), A, &status);
    cl_mem bufB = clCreateBuffer(context, CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR, N * sizeof(float), B, &status);
    cl_mem bufC = clCreateBuffer(context, CL_MEM_WRITE_ONLY, N * sizeof(float), NULL, &status);

    // 8. 设置 kernel 参数
    clSetKernelArg(kernel, 0, sizeof(cl_mem), &bufA);
    clSetKernelArg(kernel, 1, sizeof(cl_mem), &bufB);
    clSetKernelArg(kernel, 2, sizeof(cl_mem), &bufC);

    // 9. 执行 kernel
    size_t globalSize = N;
    clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &globalSize, NULL, 0, NULL, NULL);

    // 10. 读取结果
    clEnqueueReadBuffer(queue, bufC, CL_TRUE, 0, N * sizeof(float), C, 0, NULL, NULL);

    // 11. 打印结果
    printf("Result: ");
    for (int i = 0; i < N; i++) {
        printf("%f ", C[i]);
    }
    printf("\n");

    // 12. 清理资源
    clReleaseMemObject(bufA);
    clReleaseMemObject(bufB);
    clReleaseMemObject(bufC);
    clReleaseKernel(kernel);
    clReleaseProgram(program);
    clReleaseCommandQueue(queue);
    clReleaseContext(context);

    return 0;
}
```

#### 测试情况

D3000M 和 RK3588 都能够正常调用OpenCL device进行计算。

### 8.3 OpenCV 使用 OpenCL进行运算

使用 cv::UMat，OpenCV 会尝试用 OpenCL 加速。如果 OpenCL 不可用，会自动 fallback 到 CPU，不需要额外修改代码。

```c++
#include <opencv2/opencv.hpp>
#include <opencv2/core/ocl.hpp>
#include <iostream>

int main()
{
    // 打印 OpenCL 信息
    if (cv::ocl::haveOpenCL())
    {
        std::cout << "OpenCL is available" << std::endl;
        cv::ocl::Context context;
        if (!context.create(cv::ocl::Device::TYPE_GPU))
        {
            std::cout << "Failed to create OpenCL GPU context" << std::endl;
            return -1;
        }
        std::cout << "Using OpenCL device: " 
                  << context.device(0).name() 
                  << " (" << context.device(0).extensions() << ")" 
                  << std::endl;
    }
    else
    {
        std::cout << "OpenCL is NOT available" << std::endl;
    }

    // 读取图片
    cv::Mat img = cv::imread("./lena/lena.png");
    if (img.empty())
    {
        std::cout << "Could not load image!" << std::endl;
        return -1;
    }

    // 转换为 UMat (启用 OpenCL 加速)
    cv::UMat uimg, gray, blur, edges;
    img.copyTo(uimg);

    // 转为灰度
    cv::cvtColor(uimg, gray, cv::COLOR_BGR2GRAY);

    // 高斯模糊
    cv::GaussianBlur(gray, blur, cv::Size(7, 7), 1.5);

    // Canny 边缘检测
    cv::Canny(blur, edges, 50, 150);

    // 显示结果
    cv::imshow("Original", img);
    cv::imshow("Edges with OpenCL", edges);
    cv::waitKey(0);

    return 0;
}
```

#### 测试情况

D3000M 运行时可以识别到OpenCL platform，但是调用OpenCL 计算时桌面偶尔会卡死（Ubuntu 24.04 + gnome），RK3588 能正常运行。

D3000M 测试截图：
![](https://raw.githubusercontent.com/JackHuang021/images/master/opencv.png)

RK3588 测试截图：
![](https://raw.githubusercontent.com/JackHuang021/images/master/Screenshot%20from%202025-09-19%2014-51-47.png)

## 9. 编译 Tensorflow lite

编译 tensorflow lite 主要参考官方文档，不过官方文档可能更新也不及时，需要修改才能编译：
+ [https://ai.google.dev/edge/litert/build/arm](https://ai.google.dev/edge/litert/build/arm)
+ [https://ai.google.dev/edge/litert/build/cmake](https://ai.google.dev/edge/litert/build/cmake)

按照官方文档的描述，cmake 的编译方式可以支持 OpenCL 作为计算代理，于是选择cmake 的方式进行编译。

1. 克隆 tensorflow 源码仓库

    ```bash
    git clone https://github.com/tensorflow/tensorflow.git 
    ```

2. 创建编译目录

    ```bash
    mkdir tflite_build
    cd tflite_build
    ```

3. 运行 cmake 进行编译配置，配置 OpenCL GPU 代理支持，这里注意要指定 tensorflow 源码路径。

    ```bash
    cmake ../tensorflow/tensorflow/lite -DTFLITE_ENABLE_GPU=ON -DTENSORFLOW_SOURCE_DIR=/home/jack/ssd/opencl2/tensorflow/
    ```

4. 编译 tensorflow lite、benchmark_model、label_image例程，在 tflite_build 目录中运行（这里要对label_image 的 CMakeLists.txt 进行修改，要链接上 absl::log 库）:

    ```diff
    diff --git a/tensorflow/lite/examples/label_image/CMakeLists.txt b/tensorflow/lite/examples/label_image/CMakeLists.txt
    index 07ab2343..b1680015 100644
    --- a/tensorflow/lite/examples/label_image/CMakeLists.txt
    +++ b/tensorflow/lite/examples/label_image/CMakeLists.txt
    @@ -85,4 +85,5 @@ target_link_libraries(label_image
    tensorflow-lite
    profiling_info_proto
    libprotobuf
    +  absl::log
    )
    ```

    ```bash
    cmake --build . -j
    cmake --build . -j -t benchmark_model
    cmake --build . -j -t label_image
    ```

5. 编译完成后会生成 libtensorflow-lite.a 静态库

### 测试 tensorflow lite 使用OpenCL

#### benchmark_model 测试

benchmark_model 用于对 TFLite 模型及其各个运算符进行基准测试。该测接收一个 TFLite 模型，生成随机输入，然后重复运行该模型。基准测试运行结束后，会报告汇总的延迟统计信息。

GPU，CPU 调频策略均设置为 performance 进行测试。

1. 使用 GPU 作为运算代理，`./benchmark_model --graph=./models/mobilenet_v1_1.0_224.tflite --use_gpu=true`，GPU delegate 正常工作，可以看到使用到了OpenCL API

    ![](https://raw.githubusercontent.com/JackHuang021/images/master/1cdc288e077bab2c5953abaa64aee785.png)

2. 使用 CPU 作为运算代理（使用XNNPACK），`./benchmark_model --graph=./models/mobilenet_v1_1.0_224.tflite`，GPU 比 CPU 快约 1.85 倍。

    ![](https://raw.githubusercontent.com/JackHuang021/images/master/20260424104627195.png)

| 指标              | CPU（XNNPACK）    | GPU（OpenCL）   |
| --------------- | --------------- | ------------- |
| Init            | **11 ms ✅**     | 488 ms       |
| 稳态性能            | **30.9 ms**     | **16.6 ms ✅** |
| FPS             | 32 FPS          | 60 FPS        |
| 稳定性             | **极稳（std≈14）✅** | 有抖动（std≈5785） |
| 内存              | 43 MB           | 82 MB         |

#### 运行 label_image 例程

label_image 示例展示了如何加载预训练并转换后的 TensorFlow Lite 模型，并使用它来识别图像中的物体。下面是测试图片：

![](https://raw.githubusercontent.com/JackHuang021/images/master/grace_hopper.bmp) 

1. 使用 GPU 作为运算代理，`./label_image --use_gpu=true --tflite_model=./models/mobilenet_v1_1.0_224.tflite --image=./grace_hopper.bmp --labels=./labels.txt`，运行没报错，但是输出的结果不正确，后来又在RK3588上使用GPU OpenCL验证了一下，输出的结果是正确的，这里还需要再看一下是哪里的问题。

    ![D3000M 运行结果](https://raw.githubusercontent.com/JackHuang021/images/master/20260424110058183.png)

    ![RK3588 运行结果](https://raw.githubusercontent.com/JackHuang021/images/master/20260424110503595.png)

2. 使用 CPU 作为运算代理`./label_image --tflite_model=./models/mobilenet_v1_1.0_224.tflite --image=./grace_hopper.bmp --labels=./labels.txt`，运行结果正确

    ![](https://raw.githubusercontent.com/JackHuang021/images/master/20260424110259495.png) 

---
title: 089_vulkan
tags:
---

## 1. 介绍

Vulkan 的两大核心用途
1. 图形渲染：传统用途，绘制图像、3D 场景。
2. 通用计算 (GPGPU)：使用 Compute Shader（计算着色器）运行通用并行程序，不输出像素，而是对任意数据（buffer、image）进行计算。

Vulkan可以理解为一个底层 GPU 驱动 API，既能画图，也能做计算。在 Vulkan 里，Compute Shader 就是 GPU 上跑的“计算核函数”（类似 OpenCL kernel）：
+ 输入：来自 存储缓冲区 (SSBO)、图像 (Image) 或者 推送常量 (push constant)。
+ 运行：由 GPU 的 工作组 (workgroup) 和 线程 (invocation) 执行。
+ 输出：写回缓冲区或图像。

## 2. 使用 vulkaninfo 查看 GPU Vulkan 支持情况

1. 安装 vulkan-tools 软件包

	```bash
	sudo apt install vulkan-tools
	```

2. 在D3000M笔记本上使用 vulkaninfo 查看 Vulkan device信息

	```bash
	user@phytium-Ubuntu:~$ vulkaninfo 
	'DISPLAY' environment variable not set... skipping surface info
	==========
	VULKANINFO
	==========

	Vulkan Instance Version: 1.3.275


	Instance Extensions: count = 11
	===============================
		VK_EXT_debug_report                    : extension revision 10
		VK_KHR_device_group_creation           : extension revision 1
		VK_KHR_display                         : extension revision 23
		VK_KHR_external_fence_capabilities     : extension revision 1
		VK_KHR_external_memory_capabilities    : extension revision 1
		VK_KHR_external_semaphore_capabilities : extension revision 1
		VK_KHR_get_physical_device_properties2 : extension revision 2
		VK_KHR_surface                         : extension revision 25
		VK_KHR_wayland_surface                 : extension revision 6
		VK_KHR_xcb_surface                     : extension revision 6
		VK_KHR_xlib_surface                    : extension revision 6

	Layers:
	=======
	Device Properties and Extensions:
	=================================
	GPU0:
	VkPhysicalDeviceProperties:
	---------------------------
		apiVersion        = 1.3.0 (4206592)
		driverVersion     = 1.1.2 (4198402)
		vendorID          = 0x10002
		deviceID          = 0x84007305
		deviceType        = PHYSICAL_DEVICE_TYPE_INTEGRATED_GPU
		deviceName        = Phytium Technology Co., Ltd.
		pipelineCacheUUID = 05730084-7d09-47bb-b5d1-0ac5c5b13c85
	```

## 3. 安装 Vulkan SDK

Vulkan SDK 是官方提供的一整套工具、库和文档，用于 开发 Vulkan 应用程序。

Vulkan SDK 的主要组成：

| 组件                         | 作用                                                          |
| -------------------------- | ----------------------------------------------------------- |
| **Vulkan Headers**         | 定义 Vulkan API 的头文件（`vulkan.h` 等）                            |
| **Vulkan Loader**          | `vulkan-1.dll` / `libvulkan.so` 等，实现 Vulkan API 调用到显卡驱动的中间层 |
| **Validation Layers（验证层）** | 开发时检查 API 调用是否正确，捕获错误或不合理操作                                 |
| **SPIR-V Tools**           | 包含 `glslc` 编译器，编译 GLSL/HLSL 到 SPIR-V 字节码                    |
| **Vulkan Tools**           | 包含 `vulkaninfo`、调试工具、示例程序等                                  |
| **Samples & Examples**     | Khronos 官方示例，帮助理解 Vulkan 使用方式                               |
| **Documentation**          | Vulkan API 规范、指南、教程                                         |

Vulkan 官网只提供了 X86 平台的 Vulkan SDK，需要在 X86 机器上安装 Vulkan SDK，后续需要用到 glslc 工具来编译 GLSL/HLSL 到 SPIR-V 字节码。Ubuntu 24.04 的安装过程如下：

```bash
wget -qO- https://packages.lunarg.com/lunarg-signing-key-pub.asc | sudo tee /etc/apt/trusted.gpg.d/lunarg.asc
sudo wget -qO /etc/apt/sources.list.d/lunarg-vulkan-noble.list http://packages.lunarg.com/vulkan/lunarg-vulkan-noble.list
sudo apt update
sudo apt install vulkan-sdk
```

## 4. Vulkan 计算模型

1. Compute Shader: Vulkan 里的 GPU 计算函数（用 GLSL/HLSL 编译成 SPIR-V）,等价于 OpenCL Kernel。
2. Workgroup / Invocation: Compute Shader 的调度单元, Workgroup (工作组)表示一批线程，Invocation (调用/线程)表示单个执行单元，对应 OpenCL 的 Work-group / Work-item。
3. Dispatch: 向 GPU 提交任务，指定 启动多少工作组（Workgroups） 执行 Compute Shader。
4. 存储模型：使用 VkBuffer / VkImage 存储数据，在 Shader 中通过 Descriptor Set 暴露给 GPU。

## 5. Vulkan 执行流程

1. 创建实例 + 物理设备 + 队列（支持 compute 的 queue family）。
2. 创建 Buffer（VkBuffer）并分配显存。
3. 写 Compute Shader（GLSL/HLSL → SPIR-V）。
4. 创建 Compute Pipeline（加载 Shader，绑定资源）。
5. 录制 Command Buffer：
	+ 绑定 pipeline
	+ 绑定 descriptor set（资源）
	+ 调用 vkCmdDispatch()
6. 提交 Command Buffer 到 Queue。
7. Pipeline Barrier / Fence 同步后读取结果。

## 6. Vulkan Demo 例程

### 6.1 Compute Shader 计算函数

编写Compute Shader 计算函数，保存为 shader.comp

```c
#version 450
layout(local_size_x = 64) in;

layout(set = 0, binding = 0) buffer BufferA { float A[]; };
layout(set = 0, binding = 1) buffer BufferB { float B[]; };
layout(set = 0, binding = 2) buffer BufferC { float C[]; };

void main() {
    uint idx = gl_GlobalInvocationID.x;
    C[idx] = A[idx] + B[idx];
}
```

将 Compute Shader 编译成 SPIR-V，需要在 X86 机器上进行，将编译后的机器码上传到开发板。

```bash
glslc shader.comp -o shader.comp.spv
```

### 6.2 Demo 例程

使用 C++ 写一个 Vulkan 计算 Demo，其实现的功能是 向量加法 (A+B=C)。其主要运行流程是：
1. 初始化 Vulkan
2. 创建设备和计算队列
3. 创建 buffer 并写入数据
4. 加载一个简单的 Compute Shader (SPIR-V)
5. 创建 Pipeline + Descriptor
6. Dispatch 计算任务
7. 读回结果并打印

下面是 C++ Demo 代码，保存为 main.cpp，编译时要链接 -lvulkan。

```c++
#include <vulkan/vulkan.h>
#include <iostream>
#include <vector>
#include <fstream>
#include <cassert>
#include <cstring>

#define VK_CHECK(x) do { VkResult err = x; if (err) { \
    std::cerr << "Detected Vulkan error: " << err << std::endl; exit(-1);} } while(0)

// 读取SPIR-V文件
std::vector<char> readFile(const std::string& filename) {
    std::ifstream file(filename, std::ios::ate | std::ios::binary);
    assert(file.is_open());
    size_t size = (size_t)file.tellg();
    std::vector<char> buffer(size);
    file.seekg(0);
    file.read(buffer.data(), size);
    file.close();
    return buffer;
}

int main() {
    // ---------------- 1. 初始化 Vulkan ----------------
    VkInstance instance;
    {
        VkApplicationInfo appInfo{VK_STRUCTURE_TYPE_APPLICATION_INFO};
        appInfo.pApplicationName = "ComputeDemo";
        appInfo.apiVersion = VK_API_VERSION_1_2;

        VkInstanceCreateInfo createInfo{VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO};
        createInfo.pApplicationInfo = &appInfo;
        VK_CHECK(vkCreateInstance(&createInfo, nullptr, &instance));
    }

    // 选择物理设备
    uint32_t gpuCount = 0;
    vkEnumeratePhysicalDevices(instance, &gpuCount, nullptr);
    std::vector<VkPhysicalDevice> gpus(gpuCount);
    vkEnumeratePhysicalDevices(instance, &gpuCount, gpus.data());
    VkPhysicalDevice physicalDevice = gpus[0];

    // 查找支持compute的队列
    uint32_t queueFamilyIndex = -1;
    uint32_t queueCount = 0;
    vkGetPhysicalDeviceQueueFamilyProperties(physicalDevice, &queueCount, nullptr);
    std::vector<VkQueueFamilyProperties> props(queueCount);
    vkGetPhysicalDeviceQueueFamilyProperties(physicalDevice, &queueCount, props.data());
    for (uint32_t i = 0; i < queueCount; i++) {
        if (props[i].queueFlags & VK_QUEUE_COMPUTE_BIT) {
            queueFamilyIndex = i;
            break;
        }
    }
    assert(queueFamilyIndex != (uint32_t)-1);

    // 创建设备和队列
    VkDevice device;
    VkQueue queue;
    {
        float priority = 1.0f;
        VkDeviceQueueCreateInfo qinfo{VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO};
        qinfo.queueFamilyIndex = queueFamilyIndex;
        qinfo.queueCount = 1;
        qinfo.pQueuePriorities = &priority;

        VkDeviceCreateInfo dinfo{VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO};
        dinfo.queueCreateInfoCount = 1;
        dinfo.pQueueCreateInfos = &qinfo;
        VK_CHECK(vkCreateDevice(physicalDevice, &dinfo, nullptr, &device));
        vkGetDeviceQueue(device, queueFamilyIndex, 0, &queue);
    }

    // ---------------- 2. 创建Buffer ----------------
    const int N = 256;
    size_t bufferSize = N * sizeof(float);
    auto createBuffer = [&](VkBuffer &buf, VkDeviceMemory &mem, std::vector<float> *initData) {
        VkBufferCreateInfo binfo{VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO};
        binfo.size = bufferSize;
        binfo.usage = VK_BUFFER_USAGE_STORAGE_BUFFER_BIT;
        VK_CHECK(vkCreateBuffer(device, &binfo, nullptr, &buf));

        VkMemoryRequirements memReq;
        vkGetBufferMemoryRequirements(device, buf, &memReq);

        VkPhysicalDeviceMemoryProperties memProp;
        vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProp);

        uint32_t typeIndex = 0;
        for (uint32_t i = 0; i < memProp.memoryTypeCount; i++) {
            if ((memReq.memoryTypeBits & (1 << i)) &&
                (memProp.memoryTypes[i].propertyFlags & (VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT))) {
                typeIndex = i;
                break;
            }
        }

        VkMemoryAllocateInfo ainfo{VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO};
        ainfo.allocationSize = memReq.size;
        ainfo.memoryTypeIndex = typeIndex;
        VK_CHECK(vkAllocateMemory(device, &ainfo, nullptr, &mem));
        vkBindBufferMemory(device, buf, mem, 0);

        if (initData) {
            void* ptr;
            vkMapMemory(device, mem, 0, bufferSize, 0, &ptr);
            memcpy(ptr, initData->data(), bufferSize);
            vkUnmapMemory(device, mem);
        }
    };

    std::vector<float> dataA(N), dataB(N);
    for (int i = 0; i < N; i++) { dataA[i] = i; dataB[i] = i*2; }

    VkBuffer bufA, bufB, bufC;
    VkDeviceMemory memA, memB, memC;
    createBuffer(bufA, memA, &dataA);
    createBuffer(bufB, memB, &dataB);
    createBuffer(bufC, memC, nullptr);

    // ---------------- 3. 创建Descriptor ----------------
    VkDescriptorSetLayoutBinding bindings[3];
    for (int i = 0; i < 3; i++) {
        bindings[i].binding = i;
        bindings[i].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
        bindings[i].descriptorCount = 1;
        bindings[i].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;
        bindings[i].pImmutableSamplers = nullptr;
    }
    VkDescriptorSetLayoutCreateInfo linfo{VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO};
    linfo.bindingCount = 3;
    linfo.pBindings = bindings;
    VkDescriptorSetLayout layout;
    VK_CHECK(vkCreateDescriptorSetLayout(device, &linfo, nullptr, &layout));

    VkPipelineLayout pipelineLayout;
    {
        VkPipelineLayoutCreateInfo pinfo{VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO};
        pinfo.setLayoutCount = 1;
        pinfo.pSetLayouts = &layout;
        VK_CHECK(vkCreatePipelineLayout(device, &pinfo, nullptr, &pipelineLayout));
    }

    VkDescriptorPoolSize poolSize{VK_DESCRIPTOR_TYPE_STORAGE_BUFFER, 3};
    VkDescriptorPoolCreateInfo poolInfo{VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO};
    poolInfo.maxSets = 1;
    poolInfo.poolSizeCount = 1;
    poolInfo.pPoolSizes = &poolSize;
    VkDescriptorPool pool;
    VK_CHECK(vkCreateDescriptorPool(device, &poolInfo, nullptr, &pool));

    VkDescriptorSet descSet;
    VkDescriptorSetAllocateInfo allocInfo{VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO};
    allocInfo.descriptorPool = pool;
    allocInfo.descriptorSetCount = 1;
    allocInfo.pSetLayouts = &layout;
    VK_CHECK(vkAllocateDescriptorSets(device, &allocInfo, &descSet));

    VkDescriptorBufferInfo bufInfos[3] = {
        {bufA, 0, bufferSize},
        {bufB, 0, bufferSize},
        {bufC, 0, bufferSize},
    };
    std::vector<VkWriteDescriptorSet> writes(3);
    for (int i = 0; i < 3; i++) {
        writes[i] = {VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET};
        writes[i].dstSet = descSet;
        writes[i].dstBinding = i;
        writes[i].descriptorCount = 1;
        writes[i].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
        writes[i].pBufferInfo = &bufInfos[i];
    }
    vkUpdateDescriptorSets(device, writes.size(), writes.data(), 0, nullptr);

    // ---------------- 4. 创建Compute Pipeline ----------------
    auto code = readFile("shader.comp.spv");
    VkShaderModule shaderModule;
    {
        VkShaderModuleCreateInfo sinfo{VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO};
        sinfo.codeSize = code.size();
        sinfo.pCode = reinterpret_cast<const uint32_t*>(code.data());
        VK_CHECK(vkCreateShaderModule(device, &sinfo, nullptr, &shaderModule));
    }

    VkPipelineShaderStageCreateInfo stage{VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO};
    stage.stage = VK_SHADER_STAGE_COMPUTE_BIT;
    stage.module = shaderModule;
    stage.pName = "main";

    VkComputePipelineCreateInfo cinfo{VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO};
    cinfo.stage = stage;
    cinfo.layout = pipelineLayout;
    VkPipeline pipeline;
    VK_CHECK(vkCreateComputePipelines(device, VK_NULL_HANDLE, 1, &cinfo, nullptr, &pipeline));

    // ---------------- 5. Command Buffer ----------------
    VkCommandPool cmdPool;
    {
        VkCommandPoolCreateInfo info{VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO};
        info.queueFamilyIndex = queueFamilyIndex;
        VK_CHECK(vkCreateCommandPool(device, &info, nullptr, &cmdPool));
    }

    VkCommandBuffer cmdBuf;
    {
        VkCommandBufferAllocateInfo ainfo{VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO};
        ainfo.commandPool = cmdPool;
        ainfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
        ainfo.commandBufferCount = 1;
        VK_CHECK(vkAllocateCommandBuffers(device, &ainfo, &cmdBuf));
    }

    VkCommandBufferBeginInfo begin{VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO};
    vkBeginCommandBuffer(cmdBuf, &begin);
    vkCmdBindPipeline(cmdBuf, VK_PIPELINE_BIND_POINT_COMPUTE, pipeline);
    vkCmdBindDescriptorSets(cmdBuf, VK_PIPELINE_BIND_POINT_COMPUTE, pipelineLayout, 0, 1, &descSet, 0, nullptr);
    vkCmdDispatch(cmdBuf, (N+63)/64, 1, 1);
    vkEndCommandBuffer(cmdBuf);

    // ---------------- 6. 执行并等待 ----------------
    VkSubmitInfo submit{VK_STRUCTURE_TYPE_SUBMIT_INFO};
    submit.commandBufferCount = 1;
    submit.pCommandBuffers = &cmdBuf;
    vkQueueSubmit(queue, 1, &submit, VK_NULL_HANDLE);
    vkQueueWaitIdle(queue);

    // ---------------- 7. 读回结果 ----------------
    void* ptr;
    vkMapMemory(device, memC, 0, bufferSize, 0, &ptr);
    float* results = (float*)ptr;
    for (int i = 0; i < 10; i++) {
        std::cout << dataA[i] << " + " << dataB[i] << " = " << results[i] << std::endl;
    }
    vkUnmapMemory(device, memC);

    std::cout << "Done!" << std::endl;
}

```

编译：

```bash
# 安装vulkan库
sudo apt install libvulkan-dev

# 编译C++
g++ main.cpp -o vulkan_compute -lvulkan
```

### 6.3 在D3000M上的运行结果

![](https://raw.githubusercontent.com/JackHuang021/images/master/20250926170516.png)

## 7. Vulkan 性能测试 vkpeak

vkpeak 是一个 Vulkan 性能测试工具，旨在评估 Vulkan 设备在不同数据类型和操作模式下的计算性能。

项目地址: [https://github.com/nihui/vkpeak](https://github.com/nihui/vkpeak)

编译：

```bash
git clone https://github.com/nihui/vkpeak.git
cd vkpeak
git submodule update --init --recursive

mkdir build
cd build
cmake ..
cmake --build . -j 8
```

运行 vkpeak

```bash
./vkpeak 0
```

D3000M 运行结果
![](https://raw.githubusercontent.com/JackHuang021/images/master/20260715171557725.png)

总体来看，当前更适合作为图形渲染和通用 GPGPU（FP32/INT32）计算平台，能够满足图像处理、计算着色器等基础应用需求；但在 AI 推理、高性能科学计算以及低精度计算方面能力明显不足。后续如果驱动能够完善 FP16、INT8 Dot Product 及 Cooperative Matrix 等 Vulkan 特性的支持，其计算能力和应用场景将得到进一步扩展。

RK3588 运行结果
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250926175336.png)

## 8. Vulkan 通用计算应用 - Tencent/ncnn

ncnn 是由腾讯优图实验室开源的高性能神经网络前向推理框架（只做推理，不训练），专注于移动端和嵌入式设备的轻量化部署，支持通过 Vulkan 后端加速推理。

参考官方文档进行编译[https://github.com/Tencent/ncnn/wiki/how-to-build#build-for-linux](https://github.com/Tencent/ncnn/wiki/how-to-build#build-for-linux)。

```bash
git clone https://github.com/Tencent/ncnn.git
cd ncnn
git submodule update --init

sudo apt install build-essential git cmake libprotobuf-dev protobuf-compiler libomp-dev libopencv-dev

cd ncnn
mkdir -p build
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DNCNN_VULKAN=ON -DNCNN_BUILD_EXAMPLES=ON ..
make -j$(nproc)
```

ncnn 项目中提供了 examples 例程和 benchmark 测试例程

D3000M 运行结果
![](https://raw.githubusercontent.com/JackHuang021/images/master/20260715171650165.png)

D3000M Vulkan Driver 已具备运行 ncnn Vulkan 后端的基础能力，能够完成常见 CNN 模型的 GPU 推理，在轻量级网络（如 SqueezeNet、ShuffleNet、YOLO-Fastest 等）上表现较好，推理延迟基本处于 10～20 ms 水平，中大型 CNN 模型也能够稳定运行。

不过，从特性支持来看，目前驱动仍存在较明显的短板：

- 支持 Vulkan Compute，但仅具备完整的 FP32/INT32 计算能力；
- FP16、INT8、BF16 仅支持部分数据类型能力，不支持对应算术运算，因此无法获得低精度推理加速；
- Subgroup Size 固定为 1，无法利用 Subgroup 优化；
- 不支持 Cooperative Matrix/Tensor Core 等矩阵计算扩展，Transformer 等模型性能较差；
- 测试过程中出现 vkResetCommandBuffer failed 5，驱动在命令缓冲管理方面仍需进一步验证和完善。

RK3588 运行结果
![](https://raw.githubusercontent.com/JackHuang021/images/master/20250926174716.png)

## 9 clvk 适配

clvk 是基于 Vulkan 的 OpenCL 3.0 的符合规范的实现，使用 clspv 作为编译器。clspv 是 Google 开源的一个编译器，它的作用是：将 OpenCL C Kernel 编译成 Vulkan 可执行的 SPIR-V。

### 9.1 编译clvk

clvk 项目地址：[https://github.com/kpet/clvk.git](https://github.com/kpet/clvk.git)

```bash
git clone https://github.com/kpet/clvk.git
cd clvk
git submodule update --init --recursive
./external/clspv/utils/fetch_sources.py --deps llvm

mkdir -p build
cd build
cmake ../
make -j$(nproc)
```

编译完成后，在build目录下会生成 clvk Runtime 库（libOpenCL.so），它实现了 OpenCL API，OpenCL应用程序最终链接到这个库执行。

### 9.2 搭建clvk OpenCL环境

使用OpenCL ICD Loader，来加载clvk OpenCL  Runtime 库，需要在`/etc/OpenCL/vendors/`目录下创建对应的`icd`配置文件，指向clvk 运行库路径。

```bash
root@Ubuntu:/etc/OpenCL/vendors# cat clvk.icd
/home/jack/Documents/source/clvk/build/libOpenCL.so
```

最终系统下的 OpenCL platform 信息如下，识别到了D3000M显卡原生的OpenCL实现，以及clvk的OpenCL实现：

![](https://raw.githubusercontent.com/JackHuang021/images/master/2026-07-15_16-20.png)

### 9.3 clvk测试

#### 9.3.1 clvk simple_test测试

simple_test 是 clvk 仓库自带的最基础 OpenCL 功能验证程序，它的目的不是测试性能，而是验证 从 OpenCL API 到 Vulkan Compute 的整个执行链路是否正常。

simple_test 运行正常。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20260715165337757.png)

#### 9.3.2 OpenCL demo例程测试

opencl_demo 是一个基于 OpenCL C API 编写的基础测试程序，用于验证 OpenCL Runtime 是否能够正常工作。程序通过执行一个简单的向量加法（Vector Addition）计算，覆盖 OpenCL 应用开发的完整流程，包括平台枚举、设备选择、Context 创建、Kernel 编译、内存管理、Kernel 执行以及结果校验。

```c
// opencl_demo.c
// 编译 gcc opencl_demo.c -o opencl_demo -lOpenCL
#define CL_TARGET_OPENCL_VERSION 120

#include <CL/cl.h>

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define NUM_ELEMENTS 1024
#define MAX_PLATFORMS 16
#define MAX_DEVICES   16

#define CHECK_CL(ret)                                      \
do {                                                       \
    if ((ret) != CL_SUCCESS) {                             \
        printf("OpenCL error %d at line %d\n",             \
                (ret), __LINE__);                          \
        exit(-1);                                          \
    }                                                      \
} while(0)

const char *kernel_source =
"__kernel void vec_add("
"__global int *a,"
"__global int *b,"
"__global int *c)"
"{"
"    int id = get_global_id(0);"
"    c[id] = a[id] + b[id];"
"}";

void print_platform_info(cl_platform_id platform)
{
    char name[256];
    char vendor[256];
    char version[256];


    clGetPlatformInfo(platform,
                      CL_PLATFORM_NAME,
                      sizeof(name),
                      name,
                      NULL);


    clGetPlatformInfo(platform,
                      CL_PLATFORM_VENDOR,
                      sizeof(vendor),
                      vendor,
                      NULL);


    clGetPlatformInfo(platform,
                      CL_PLATFORM_VERSION,
                      sizeof(version),
                      version,
                      NULL);


    printf("Platform:\n");
    printf("  Name   : %s\n", name);
    printf("  Vendor : %s\n", vendor);
    printf("  Version: %s\n", version);
}

void print_device_info(cl_device_id device)
{
    char name[256];
    char version[256];

    cl_uint cu;
    size_t wg;


    clGetDeviceInfo(device,
                    CL_DEVICE_NAME,
                    sizeof(name),
                    name,
                    NULL);


    clGetDeviceInfo(device,
                    CL_DEVICE_VERSION,
                    sizeof(version),
                    version,
                    NULL);


    clGetDeviceInfo(device,
                    CL_DEVICE_MAX_COMPUTE_UNITS,
                    sizeof(cu),
                    &cu,
                    NULL);


    clGetDeviceInfo(device,
                    CL_DEVICE_MAX_WORK_GROUP_SIZE,
                    sizeof(wg),
                    &wg,
                    NULL);



    printf("Device:\n");
    printf("  Name          : %s\n", name);
    printf("  Version       : %s\n", version);
    printf("  Compute Units : %u\n", cu);
    printf("  Max WorkGroup : %zu\n", wg);
}

void list_platforms()
{
    cl_platform_id platforms[MAX_PLATFORMS];

    cl_uint num;

    cl_int ret;


    ret = clGetPlatformIDs(
            MAX_PLATFORMS,
            platforms,
            &num);

    CHECK_CL(ret);


    printf("Found %u platforms\n\n", num);


    for(unsigned int i=0;i<num;i++)
    {
        printf("Platform [%u]\n", i);

        print_platform_info(platforms[i]);


        cl_device_id devices[MAX_DEVICES];

        cl_uint dev_num;


        ret = clGetDeviceIDs(
                platforms[i],
                CL_DEVICE_TYPE_ALL,
                MAX_DEVICES,
                devices,
                &dev_num);


        if(ret != CL_SUCCESS)
        {
            printf("  No device\n\n");
            continue;
        }


        for(unsigned int j=0;j<dev_num;j++)
        {
            printf("  Device [%u]\n", j);

            print_device_info(devices[j]);

        }

        printf("\n");
    }
}

int main(int argc,char **argv)
{
    int platform_id = -1;
    int device_id   = 0;


    int opt;


    while((opt=getopt(argc,argv,"lp:d:"))!=-1)
    {
        switch(opt)
        {
        case 'l':
            list_platforms();
            return 0;


        case 'p':
            platform_id = atoi(optarg);
            break;


        case 'd':
            device_id = atoi(optarg);
            break;


        default:
            printf(
            "Usage:\n"
            "  %s -l\n"
            "  %s -p platform_id -d device_id\n",
            argv[0],
            argv[0]);

            return -1;
        }
    }

    if(platform_id < 0)
    {
        printf("Please specify platform\n");
        return -1;
    }

    cl_int ret;
    cl_platform_id platforms[MAX_PLATFORMS];
    cl_uint num_platforms;

    ret = clGetPlatformIDs(
            MAX_PLATFORMS,
            platforms,
            &num_platforms);

    CHECK_CL(ret);

    if(platform_id >= num_platforms)
    {
        printf("Invalid platform id\n");
        return -1;
    }

    cl_platform_id platform =
        platforms[platform_id];

    cl_device_id devices[MAX_DEVICES];

    cl_uint num_devices;

    ret = clGetDeviceIDs(
            platform,
            CL_DEVICE_TYPE_ALL,
            MAX_DEVICES,
            devices,
            &num_devices);

    CHECK_CL(ret);

    if(device_id >= num_devices)
    {
        printf("Invalid device id\n");
        return -1;
    }

    cl_device_id device =
        devices[device_id];

    print_platform_info(platform);

    print_device_info(device);

    /*
     * context
     */

    cl_context context =
        clCreateContext(
            NULL,
            1,
            &device,
            NULL,
            NULL,
            &ret);

    CHECK_CL(ret);

    /*
     * command queue
     */

    cl_command_queue queue =
        clCreateCommandQueue(
            context,
            device,
            0,
            &ret);

    CHECK_CL(ret);

    /*
     * program
     */

    cl_program program =
        clCreateProgramWithSource(
            context,
            1,
            &kernel_source,
            NULL,
            &ret);

    CHECK_CL(ret);

    ret = clBuildProgram(
            program,
            1,
            &device,
            NULL,
            NULL,
            NULL);


    if(ret != CL_SUCCESS)
    {
        char log[4096];

        clGetProgramBuildInfo(
                program,
                device,
                CL_PROGRAM_BUILD_LOG,
                sizeof(log),
                log,
                NULL);

        printf("Build error:\n%s\n",log);

        return -1;
    }

    cl_kernel kernel =
        clCreateKernel(
            program,
            "vec_add",
            &ret);

    CHECK_CL(ret);

    int *a = malloc(NUM_ELEMENTS*sizeof(int));
    int *b = malloc(NUM_ELEMENTS*sizeof(int));
    int *c = malloc(NUM_ELEMENTS*sizeof(int));

    for(int i=0;i<NUM_ELEMENTS;i++)
    {
        a[i]=i;
        b[i]=i;
    }

    cl_mem buf_a =
        clCreateBuffer(
            context,
            CL_MEM_READ_ONLY |
            CL_MEM_COPY_HOST_PTR,
            sizeof(int)*NUM_ELEMENTS,
            a,
            &ret);

    CHECK_CL(ret);

    cl_mem buf_b =
        clCreateBuffer(
            context,
            CL_MEM_READ_ONLY |
            CL_MEM_COPY_HOST_PTR,
            sizeof(int)*NUM_ELEMENTS,
            b,
            &ret);

    CHECK_CL(ret);

    cl_mem buf_c =
        clCreateBuffer(
            context,
            CL_MEM_WRITE_ONLY,
            sizeof(int)*NUM_ELEMENTS,
            NULL,
            &ret);

    CHECK_CL(ret);

    clSetKernelArg(kernel,0,sizeof(buf_a),&buf_a);
    clSetKernelArg(kernel,1,sizeof(buf_b),&buf_b);
    clSetKernelArg(kernel,2,sizeof(buf_c),&buf_c);

    size_t global =
        NUM_ELEMENTS;

    ret = clEnqueueNDRangeKernel(
            queue,
            kernel,
            1,
            NULL,
            &global,
            NULL,
            0,
            NULL,
            NULL);

    CHECK_CL(ret);

    clFinish(queue);

    clEnqueueReadBuffer(
            queue,
            buf_c,
            CL_TRUE,
            0,
            sizeof(int)*NUM_ELEMENTS,
            c,
            0,
            NULL,
            NULL);

    for(int i=0;i<NUM_ELEMENTS;i++)
    {
        if(c[i] != a[i]+b[i])
        {
            printf("FAILED index=%d\n",i);
            return -1;
        }
    }

    printf("\nOpenCL test PASSED\n");

    clReleaseMemObject(buf_a);
    clReleaseMemObject(buf_b);
    clReleaseMemObject(buf_c);

    clReleaseKernel(kernel);
    clReleaseProgram(program);

    clReleaseCommandQueue(queue);
    clReleaseContext(context);

    free(a);
    free(b);
    free(c);

    return 0;
}
```

列出所有 platform，选择clvk platform进行测试：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20260715170433117.png)

#### 9.3.3 clpeak 测试

clvk 官方显示支持的应用有clpeak，可以借助 clpeak 来测试 clvk 的 OpenCL实现性能，对比显卡原生OpenCL实现的性能。

![](https://raw.githubusercontent.com/JackHuang021/images/master/20260715172007677.png)

OpenCL Kernel 中使用了 8 位数据类型作为 Buffer（SSBO）存储，但当前 Vulkan Device 不支持 VK_KHR_8bit_storage，clpeak运行会报下面的错误，无法测试出结果：

![](https://raw.githubusercontent.com/JackHuang021/images/master/20260715173217653.png)

#### 9.3.4 opencv 测试调用 clvk OpenCL 实现

使用之前的opencv例程，运行后也会出现类似的报错

![](https://raw.githubusercontent.com/JackHuang021/images/master/20260715173710327.png)

### 9.4 clvk 总结

当前 clvk 已完成基础功能适配，能够支持简单 OpenCL 应用运行。测试结果表明，OpenCL API 层正常，clvk 到 Vulkan Compute 基础调用链正常，基础 Kernel 执行正常，但距离完整 OpenCL 应用生态支持仍存在差距。主要限制来自：Vulkan Driver Feature 支持不足。

下一步优化方向：
1. 完善 Vulkan Feature 支持
2. 提升 OpenCL 应用兼容性。

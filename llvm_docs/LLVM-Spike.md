# LLVM 和 Spike 的仿真器接口对接详解
<em style="font-size: 14px; color: gray;">本文档详述了如何使用 Spike 仿真器对 Kernel 程序进行模拟测试，涵盖了设备初始化、内存分配、数据传输、内核加载、运行、验证和清理的全流程。通过严格的内存管理和接口调用链设计，确保了仿真器运行的高效准确。`test.cpp` 是	应用层程序，调用驱动接口，完成任务; `ventus.h` 定义驱动接口，屏蔽底层实现细节。`	spike_device.cc`	底层实现，直接操作设备内存、启动仿真器。</em>

## 模拟测试文件([test.cpp](${VENTUS_INSTALL_PREFIX}/../../ventus-gpgpu-isa-simulator/gpgpu-testcase/driver/test.cpp))

此文件用于测试运行 `vecadd.cl` 的 `kernel` 程序, 包括内存分配, 数据传输, 元数据配置, 内核加载, 运行设备, 从设备拷贝运算结果。  


***1. ventus 模拟器的设备初始化***

设备初始化时需要完成以下任务：
- 分配局部存储 (LDS) 和程序计数器存储 (Program Counter, PC)
- 初始化设备句柄，确保主机和设备的通信。

```cpp
// test.cpp
vt_device_h p = nullptr; 
vt_dev_open(&p);
//spike_device.cc
spike_device::spike_device() : sim(nullptr), buffer(), buffer_data() {
    srcfilename = new char[128]; //分配src文件储存空间
    logfilename = new char[128]; //分配log文件储存空间
    uint64_t lds_vaddr;
    uint64_t pc_src_vaddr;
    alloc_const_mem(0x10000000, &lds_vaddr);  // 分配局部存储
    alloc_const_mem(0x10000000, &pc_src_vaddr);  // 分配程序计数器存储
};

```
- `spike_device` 初始化并且绑定到指针 `&p`
- 调用 `alloc_const_mem` 分配两块内存：
  - `local Data Store` 始于0x70000000, 大小0x10000000
  - `Program Counter`  始于0x80000000, 大小0x10000000
- 基址和分配的大小的最小单位是 `4KB` , `0x1000`
- 更新 `const_buffer` 记录存储设备内存配置, `const_buffer_data` 具体内存数据


***2. 内存分配***
根据 OpenCL 的编程模型将不同类型的内存（全局内存、私有内存等）分配到设备中。

```cpp
// test.cpp
vt_buf_alloc(p, size_0, &vaddr_0, 0, 0, 0); // 分配程序内存
vt_buf_alloc(p, pdssize * num_thread * num_warp * num_workgroup, &pdsbase, 0, 0, 0); // 分配私有内存
...

// ventus.cpp
extern int vt_buf_alloc(vt_device_h hdevice, uint64_t size, uint64_t *vaddr, int BUF_TYPE, uint64_t taskID, uint64_t kernelID) {
    ...
    auto device = ((spike_device*) hdevice);
    return device->alloc_local_mem(size, vaddr);

}
```
- 如果输入的内存大小合法, 就可以确定分配的基址：没有分配过默认 `0x90000000` , 之后从最后一个分配块的结束地址为基址。
- 基址和分配的大小的最小单位是 `4KB` , `0x1000`
- 更新 `buffer` 记录虚拟内存地址和大小, `buffer_data` 保存内存对象  

|分配区块|基址|大小|
|-|-|-|
|Program Buffer|0x90000000|0x10000000|
|Private Memory Buffer|0xa0000000|0x20000|
|Argument 1 Buffer|0xa0020000|0x40 (0x1000 minimum)|
|Argument 2 Buffer|0xa0021000|0x40 (0x1000 minimum)|
|Metadata Buffer|0xa0022000|0x40 (0x1000 minimum)|
|Buffer Base|0xa0023000|0x8 (0x1000 minimum)|
|Print Buffer|0xa0024000|0x10000000|
 <em style="font-size: 14px; color: gray;">实际分配区块</em> 

***3. 数据传输***
在运行 Kernel 前，主机需要将参数和数据传输到设备。

```cpp
// test.cpp
vt_copy_to_dev(p, vaddr_1, data_0, 16 * 4, 0, 0); // 复制A数组到设备内存
vt_copy_to_dev(p, vaddr_2, data_1, 16 * 4, 0, 0); // 复制B数组到设备内存
vt_copy_to_dev(p,vaddr_3,data_2,14*4,0,0); // 复制元数据
vt_copy_to_dev(p,vaddr_4,data_3,2*4,0,0); // 复制 buffer base

// spike_device.cc
buffer_data[i].second->store(vaddr - buffer_data[i].first, size, (const uint8_t*)data);
```
- 检查数据大小是否合法 `(size > 0)`, 调用其 `copy_to_dev` 完成数据到设备的复制
- 遍历 `buffer`, 找到包含目标地址 `vaddr` 的内存块
- 如果 `vaddr + size` 合法, 使用 `buffer_data[i].second->store` 将数据存储到设备内存

|分配区块|基址|大小|
|-|-|-|
|Argument 1 Buffer|0xa0020000|0x40|
|Argument 2 Buffer|0xa0021000|0x40|
 <em style="font-size: 14px; color: gray;">数组分配区块</em>   

配置 `meta_data` 的内容, 包括：
- Kernel 的入口地址 (`KERNEL_ADDRESS`)
- 私有内存基址 (`pdsBaseAddr`)
- 输出缓冲区地址 (`vaddr_print`)

|分配区块|基址|大小|
|-|-|-|
|Metadata Buffer|0xa0022000|0x38|
|Buffer Base|0xa0023000|0x8|
 <em style="font-size: 14px; color: gray;">meta/两数组基址分配区块</em>   

***4. Kernel的加载执行***

```cpp
// test.cpp
vt_upload_kernel_file(p, filename, 0); // 上传 Kernel 文件
vt_start(p, &meta, 0); // 启动 Kernel

// ventus.cpp
extern int vt_start(vt_device_h hdevice, void* metaData, uint64_t taskID) {
    ...
    device->run(knl_data,0x80000000);
    ...
}

// spike_device.cc
int spike_device::run(meta_data* knl_data,uint64_t knl_start_pc)
```

- `vt_upload_kernel_file` 上传 `Kernel` 文件名称到设备, `srcfilename` 内核文件路径, `logfilename` 为同名 + `.log`
- `vt_start` 通过 `spike_device::run` 启动 `Kernel`, 开始模拟器中的执行
  - 调用 `device->run(knl_data, 0x80000000)`
  - `knl_data` 是元数据 `meta_data`, 定义了内核参数
  - `0x80000000` 是内核起始地址
- `device->run` 是内核代码在 `spike` 模拟器上运行的入口, 完成了设备配置、资源初始化、内核调度和运行
- 初始化
  - 解析元数据 `meta_data`, 获取内核的 `warp` 数量每个 `warp` 的线程数、工作组三维、`Local Data Share (LDS)` 的大小、和`Private Data Share (PDS)` 的基址
  - 检查 LDS 的大小是否超出设备内存限制 `0x10000000`
  - 创建 `cfg_t` 配置对象, 定义内存布局、指令集 `RV32GV`、默认特权级 `MSU` (Spike的`isa_parser`参数)、默认架构  (向量寄存器128-bit,元素最大位宽64-bit)、内存分布设置为 `2048MB` or `0x80000000`
  ```ccp
  cfg_t cfg(/*内核 RAM 的起止范围*/                std::make_pair((reg_t)0, (reg_t)0),
              /*内核启动参数*/                     nullptr,
              /*支持的指令集架构*/                 "RV32GV",
              /*默认特权级(用户/内核模式)= MSU*/  DEFAULT_PRIV,
              /*矢量架构配置 = vlen:128,elen:64*/  DEFAULT_VARCH,
              /*模拟器的内存布局*/                 parse_mem_layout("2048"),
              /*核心 (hart) 的 ID 列表*/           std::vector<int>(),
              /*是否支持实时核心本地中断*/          false
    );
  ```
- 设备和调试模块的配置
  - 定义调试模块参数 `debug_module_config_t`  
  - 配置插件设备, 使用 `device_parser` 函数, 将输入字符串拆分为 name、base 和 args, 创建对应的 `mmio_plugin_device_t` 实例并加入 `plugin_devices`
- 解析命令行参数
  - 选项解析器提供调试、配置、插件拓展以及各种命令行选项, 包括不限于: `-d` 设置 `debug`, `-g` 设置 `histogram`, `-m` 解析并配置内存布局, `--isa` 用于定义指令集架构, etc
  - 详见 `spike_device.cc/int spike_device::run/option_parser_t parser`
- 模拟器实例化和运行
  - `sim_t` 是 `SPIKE` 仿真器的核心类, 用于管理所有核心、内存、设备和调试信息
  - 设置缓存仿真器, 初始化并配置 I 缓存、D 缓存和 L2 缓存仿真器, 将 L2 缓存设置为 I 缓存和 D 缓存的 miss handler:
    - `icache_sim_t` 用于指令缓存仿真。
    - `dcache_sim_t` 用于数据缓存仿真。
    - `cache_sim_t`  用于 L2 缓存仿真。
  - 为每个核心注册调试器, 使用 `register_memtracer` 注册 I 缓存和 D 缓存的追踪功能
  - 启动调试, 日志, 和性能统计模式
  - 调用仿真器的核心运行函数 `sim->run()`, 该函数执行以下主要任务：加载 RISC-V 二进制文件到仿真器内存, 启动仿真器核心, 逐条执行指令, 捕获仿真执行过程中的错误或完成状态
  ```cpp
    auto return_code = 0;
    for (uint64_t i = 0; i < num_workgroup / SPIKE_RUN_WG_NUM; i++) {
      sim = new sim_t(...);   // 每次循环创建一个新的仿真器实例
      ...
      return_code = sim->run();  // 执行仿真任务, 更新 return_code
      ...
      delete sim;  // 每次循环结束释放仿真器实例
    }
  ```
- 资源释放
  - 释放 `plugin_devices` 中存储的动态分配的设备对象, argv 动态分配的字符指针数组
  - `return_code` 作为 `int spike_device::run()` 的返回值, 如果是 `0` 的话模拟器运行正常 

***5. 验证***

```cpp
// test.cpp
vt_copy_from_dev(p, vaddr_2, data_1, 16*4, 0, 0); // 从设备复制 data_1
vt_copy_from_dev(p, vaddr_1, data_0, 16*4, 0, 0); // 从设备复制 data_0
...
uint32_t *print_data = new uint32_t[64];
vt_copy_from_dev(p, vaddr_print, print_data, 64*4, 0, 0); // 从设备内存复制到 print_data
```
- 调用 `vt_copy_from_dev`, 从设备内存地址 `vaddr_1` 和 `vaddr_2` 复制数据到主机内存数组 `data_0` 和 `data_1`
- 动态分配缓冲区 `print_data`, 并从设备地址 `vaddr_print` 读取 64 * 4-Byte 的数组

***6. 清理***

```cpp
vt_buf_free(p, 0, nullptr, 0, 0); // 释放内存
delete[] print_data; 
```

- 遍历 `buffer_data` 缓冲区数据, 释放内存对象, 防止悬挂指针, 清空内存数据记录
- 释放动态分配的 `print_data`

## 头文件 ([ventus.h](ventus-gpgpu-isa-simulator/gpgpu-testcase/driver/include/ventus.h))

提供了从主机到设备的全面接口，涵盖设备初始化、内存分配、数据传输、Kernel 执行、结果验证和清理的全流程。

## 模拟头文件 ([spike_main.h](/ventus-gpgpu-isa-simulator/spike_main/spike_main.h))

`spike_main` 是 Spike 仿真器在 `ventus GPGPU` 模拟平台中的核心模块，用于管理设备端的内存、数据传输和 Kernel 执行。其功能通过 `spike_device` 类的接口实现，配合主程序 (test.cpp) 和驱动 (ventus.h) 实现软硬件接口的完整调用链。
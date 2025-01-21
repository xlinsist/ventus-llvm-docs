# 乘影CSR寄存器概述
## 背景概述

在乘影GPGPU的设计中，CSR（Control and Status Register，控制与状态寄存器） 是一种特殊用途的寄存器，用于kernel启动、内存管理和线程控制等。自定义的CSR寄存器使得乘影GPGPU能够高效地管理多个线程、工作组和内存资源，并与主机进行有效的信息交换，为高效的并行计算提供了灵活的控制手段。

|  名称  |  描述  |
|--------|--------|
|  CSR_TID   |   该 warp 中 id 最小的 thread id，其值为 CSR_WID*CSR_NUMT，配合 vid.v 可计算其它thread id   |
|  CSR_NUMW  |   该 workgroup 中的 warp 总数   |
|  CSR_NUMT  |   一个 warp 中的 thread 总数   |
|  CSR_KNL   |   该 workgroup 的 metadata buffer 的 baseaddr   |
|  CSR_WGID  |   该 SM 中本 warp 对应的 workgroup id   |
|  CSR_WID   |   该 workgroup 中本 warp 对应的 warp id   |
|  CSR_LDS   |   该 workgroup 分配的 local memory 的 baseaddr，同时也是该 warp 的 xgpr spill stack 基址   |
|  CSR_PDS   |   该 workgroup 分配的 private memory 的 baseaddr，同时是该 thread 的 vgpr spill stack 基址   |
|  CSR_GDX   |   该 workgroup 在 NDRange 中的 x id   |
|  CSR_GDY   |   该 workgroup 在 NDRange 中的 y id   |
|  CSR_GDZ   |   该 workgroup 在 NDRange 中的 z id   |
|  CSR_PRINT |   向 print buffer 打印时用于与 host 交互的 CSR   |
|  CSR_RPC   |   重汇聚 pc 寄存器   |

来自：[乘影架构文档手册：指令集架构及软硬件接口v202.pdf](https://opengpgpu.org.cn/upload/1/editor/1706683586837.pdf "乘影架构文档手册：指令集架构及软硬件接口v202.pdf")。

## 核心实现
### CSR寄存器的地址

在Ventus-Spike的riscv\encoding.h中定义了Ventus-Spike内部的CSR寄存器地址，这些地址用于在硬件中唯一标识每个CSR寄存器，以便对其进行读取和写入操作。
```
/* rvv-gpgpu extension csr */
#define CSR_TID 0x800
#define CSR_NUMW 0x801
#define CSR_NUMT 0x802
#define CSR_KNL 0x803
#define CSR_WGID 0x804
#define CSR_WID 0x805
#define CSR_LDS 0x806
#define CSR_PDS 0x807
#define CSR_GIDX 0x808
#define CSR_GIDY 0x809
#define CSR_GIDZ 0x80a
#define CSR_PRINT 0x80b
#define CSR_RPC 0x80c
```

在Ventus-LLVM的libclc\riscv32\lib\ventus.h中可以看到相同的寄存器地址，这段代码是编译器层面对硬件特定CSR地址的实现，与Ventus-Spike中的CSR地址定义一脉相承，共同为乘影GPGPU扩展提供支持。
```
// CSRs
#define CSR_TID   0x800    // smallest thread id in a warp
#define CSR_NUMW  0x801    // warp numbers in a workgroup
#define CSR_NUMT  0x802    // thread numbers in a warp
#define CSR_KNL   0x803    // Kernel metadata buffer base address
#define CSR_WGID  0x804    // workgroup id in a warp
#define CSR_WID   0x805    // warp id in a workgroup
#define CSR_LDS   0x806    // baseaddr for local memory allocated by workgroup
#define CSR_PDS   0x807    // baseaddr for private memory allocated by workgroup
#define CSR_GID_X 0x808    // group_id_x
#define CSR_GID_Y 0x809    // group_id_y
#define CSR_GID_Z 0x80a    // group_id_z
#define CSR_PRINT 0x80b    // for print buffer
```

### Ventus-Spike中CSR寄存器

在Ventus-Spike的riscv\processor.h中定义了乘影GPGPU的相关硬件模拟单元，涉及SIMT模型和相关的控制状态寄存器CSR。
```
// CSRs
  class gpgpu_unit_t{
    public:
      warp_schedule_t *w;
      csr_t_p rpc;
    private:
      
      processor_t *p;
      // custom csr
      csr_t_p numw;
      csr_t_p numt;
      csr_t_p tid;
      csr_t_p wgid;
      csr_t_p wid;//warp id
      csr_t_p pds;
      csr_t_p lds;
      csr_t_p knl;
      csr_t_p gidx;
      csr_t_p gidy;
      csr_t_p gidz;
      ...
  }
```

在Ventus-Spike的riscv\processor.cc中定义了 gpgpu_unit_t::reset 函数的实现，它是Ventus-Spike的一个初始化函数。该函数主要作用是对 GPGPU 的控制状态寄存器（CSR）进行初始化，并配置 GPGPU 单元的状态。每个 basic_csr_t 对象都是针对特定 CSR 地址的寄存器对象，并将其初始化为 0。这些寄存器在 GPGPU 执行过程中用于存储和管理 GPGPU 任务的状态信息。

```
// defination of gpgpu_unit_t
void processor_t::gpgpu_unit_t::reset(processor_t *const proc) 
{
  p = proc;
  
  state_t *state = p->get_state();
  auto &csrmap = p->get_state()->csrmap;
  csrmap[CSR_TID] = tid = std::make_shared<basic_csr_t>(proc, CSR_TID, 0);
  csrmap[CSR_NUMW] = numw = std::make_shared<basic_csr_t>(proc, CSR_NUMW, 0);
  csrmap[CSR_NUMT] = numt = std::make_shared<basic_csr_t>(proc, CSR_NUMT, 0);
  csrmap[CSR_KNL] = knl = std::make_shared<basic_csr_t>(proc, CSR_KNL, 0);
  csrmap[CSR_WGID] = wgid = std::make_shared<basic_csr_t>(proc, CSR_WGID, 0);
  csrmap[CSR_WID] = wid = std::make_shared<basic_csr_t>(proc, CSR_WID, 0);
  csrmap[CSR_PDS] = pds = std::make_shared<basic_csr_t>(proc, CSR_PDS, 0);
  csrmap[CSR_LDS] = lds = std::make_shared<basic_csr_t>(proc, CSR_LDS, 0);
  csrmap[CSR_GIDX] = gidx = std::make_shared<basic_csr_t>(proc, CSR_GIDX, 0);
  csrmap[CSR_GIDY] = gidy = std::make_shared<basic_csr_t>(proc, CSR_GIDY, 0);
  csrmap[CSR_GIDZ] = gidz = std::make_shared<basic_csr_t>(proc, CSR_GIDZ, 0);    
  csrmap[CSR_RPC] = rpc = std::make_shared<basic_csr_t>(proc, CSR_RPC, 0);

  
  // initialize csrs to enable vecter extension
  reg_t mstatus_val = state->mstatus->read();
  mstatus_val = set_field(mstatus_val, MSTATUS_VS, 1);
  state->mstatus->write(mstatus_val);

  simt_stack.reset();
}
```

### Ventus-LLVM的kernel启动与CSR寄存器读取

crt0.S：Ventus-LLVM中的crt0.S为 OpenCL C 内核程序设置了入口点_start，并准备好启动程序执行所需的基本资源，乘影GPGPU会从该入口点开始执行。

```
  .text
  .global _start
  .type   _start, @function
_start:
  # set global pointer register
  .option push
  .option norelax
  la gp, __global_pointer$
  .option pop
```

配置 Warp 和线程级栈指针：
csrr t1, CSR_WID：通过 CSR_WID 寄存器读取当前 Warp 的 ID。
csrr t2, CSR_LDS：通过 CSR_LDS 寄存器获取本地内存（Local Memory）的基地址。每个工作组（Workgroup）分配一块本地内存，每个线程（或 Warp）使用这块内存进行数据存储。

```
li t4, 0
csrr t1, CSR_WID         # 获取当前warp id
csrr t2, CSR_LDS         # 获取local memory base地址
```

获取内核元数据和参数：
csrr t0, CSR_KNL：通过 CSR_KNL 获取内核元数据缓冲区的基地址，CSR_KNL 存储着内核执行所需的所有元数据（如内核入口地址、参数地址、打印缓冲区地址等）。
lw t1, KNL_ENTRY(t0)：从内核元数据中加载内核程序的入口地址，准备跳转到内核执行。
lw a0, KNL_ARG_BASE(t0)：获取内核参数缓冲区的地址，这些参数会在执行内核程序时传递给内核。

```
csrr t0, CSR_KNL         # 获取内核元数据的地址
lw t1, KNL_ENTRY(t0)     # 获取内核程序入口地址
lw a0, KNL_ARG_BASE(t0)  # 获取内核参数缓冲区的基地址
```

在kernel启动中，通过访问特定的 CSR 寄存器来获取内核执行所需的上下文信息，并根据这些信息设置程序的栈指针、内存地址等，从而为内核程序的执行提供支持。
# Ventus LLVM 工具链的的编译链接
<em style="font-size: 14px; color: gray;">OpenCL kernel的模拟：编译器生成符合硬件约定的RISC-V ELF, 链接器和启动文件设置程序运行环境, Spike 加载并执行生成的二进制文件。</em>

- [编译](#编译)
  - [编译脚本](#编译脚本buildsh)
  - [LLVM中间层的kernel调用](#llvm中间层的kernel调用vecaddll)
- [链接](#链接)
  - [链接脚本](#链接脚本buildsh)
  - [ventus.h](#ventush)
  - [crt0.s](#crt0s)
  - [elf32lriscv.ld](#elf32lriscvld)


## 编译  

### 编译脚本([build.sh](build.sh)) 

从 `.cl` 到 `.ll`:
```bash
${VENTUS_INSTALL_PREFIX}/bin/clang -S -cl-std=CL2.0 -target riscv32 -mcpu=ventus-gpgpu ${TESTCASE}/${TARGET_CL}.cl -emit-llvm -o $OUTPUT_DIR/${TARGET_CL}.ll
```
使用 `clang` 将 OpenCL kernel 源代码编译为 LLVM IR `(.ll)`,
在 `.ll` 文件中可以看到 kernel 的函数定义和调用模型。

从 `.ll` 到 `.s`:
```bash
${VENTUS_INSTALL_PREFIX}/bin/llc -mtriple=riscv32 -mcpu=ventus-gpgpu $OUTPUT_DIR/${TARGET_CL}.ll -o $OUTPUT_DIR/${TARGET_CL}.s
```
使用 `llc` 将 LLVM IR 转化为`RISC-V asm`, 
`.s` 文件中会包含 kernel 的指令实现, 包括参数加载、寄存器使用等。

---

### LLVM中间层的kernel调用([vecadd.ll])

```opencl
__kernel void vecadd(__global float* A, __global float* B) {
  unsigned tid = get_global_id(0);
  A[tid] += B[tid];
}
```
<p style="font-size: 12px; color: gray;">opencl kernel 源码</p> 

```ll
; ModuleID = 'testcase0/vecadd.cl'
source_filename = "testcase0/vecadd.cl"
target datalayout = "e-m:e-p:32:32-i64:64-n32-S128-A5-G1"
target triple = "riscv32"
```
目标文件的 layout 定义:   
小端储存 `little-endian`, 指针大小与对齐为 `32-Bit/4-Byte`, 整数大小与对齐为  `64-Bit/8-Byte` (long long/double), 整数和计算操作默认 `32-Bit/4-Byte`, 栈对齐默认 `128-Bit/16-Byte`.

```ll
; Function Attrs: convergent mustprogress nofree norecurse nounwind willreturn memory(argmem: readwrite) vscale_range(1,2048)
define dso_local ventus_kernel void @vecadd(
    ptr addrspace(1) nocapture noundef align 4 %0, 
    ptr addrspace(1) nocapture noundef readonly align 4 %1
    ) 
    local_unnamed_addr #0 !kernel_arg_addr_space !6 !kernel_arg_access_qual !7 !kernel_arg_type !8 !kernel_arg_base_type !8 !kernel_arg_type_qual !9 
    {
    ...
    }
```
<p style="font-size: 12px; color: gray;">vecadd kernel函数的声明中间层表达</p> 

函数名称 `vecadd`, 返回 `void`。`%0` 作为数组A的基址, `%1` 作为数组B的只读基址。
- `nocapture`: 确保参数不会被捕获（不会存储到其他位置）。
- `noundef`: 参数始终是已定义的（不会传入未初始化的指针）。
- `align 4`: 参数的对齐方式为 4 字节。
- `local_unnamed_addr`: 指明函数地址在整个模块中是唯一的, 不依赖具体地址值。
- `!kernel_arg_addr_space`: 参数的地址空间信息。
- `!kernel_arg_access_qual`: 参数的访问限定符。
- `!kernel_arg_type 和 !kernel_arg_base_type`: 参数的类型描述。

```ll
... 
    {
    %3 = call i32 @_Z13get_global_idj(i32 noundef 0) #2
    %4 = getelementptr inbounds float, ptr addrspace(1) %1, i32 %3
    %5 = load float, ptr addrspace(1) %4, align 4, !tbaa !10
    %6 = getelementptr inbounds float, ptr addrspace(1) %0, i32 %3
    %7 = load float, ptr addrspace(1) %6, align 4, !tbaa !10
    %8 = fadd float %5, %7
    store float %8, ptr addrspace(1) %6, align 4, !tbaa !10
    ret void
    }
```
<p style="font-size: 12px; color: gray;">vecadd kernel函数的本体中间层表达</p> 
  
`%3` 调用 OpenCL 内置函数 `get_global_id(0)`, 获取全局ID。  
`%4` `%1`偏移`%3`个元素后的地址, %B  
`%5` `%4`地址上的浮点数, B  
`%6` `%0`偏移`%3`个元素后的地址, %A  
`%7` `%6`地址上的浮点数, A  
`%8` 临时寄存器的浮点相加结果  
浮点加法结果存入`%6`, 就是A数组的地址, 实现 `A[tid] += B[tid];`。

---

## 链接
<em style="font-size: 14px; color: gray;">在 OpenCL kernel程序中, crt0.S 作为启动文件, 负责初始化运行环境, 包括寄存器、内存布局以及硬件寄存器设置。而 elf32lriscv.ld 则是链接脚本, 定义了程序的内存布局及段信息。两者共同作用确保生成的 ELF 文件能够被硬件（Spike）正确加载和执行。</em>

### 链接脚本([build.sh](build.sh)) 

从 `.s` 到 `.o` 和 `.riscv`:
```bash
${VENTUS_INSTALL_PREFIX}/bin/llc -mtriple=riscv32 -mcpu=ventus-gpgpu --filetype=obj $OUTPUT_DIR/${TARGET_CL}.ll -o $OUTPUT_DIR/${TARGET_CL}.o  

${VENTUS_INSTALL_PREFIX}/bin/ld.lld -o $OUTPUT_DIR/${TARGET_CL}.riscv -T $VENTUS_INSTALL_PREFIX/../utils/ldscripts/ventus/elf32lriscv.ld $OUTPUT_DIR/${TARGET_CL}.o ${VENTUS_INSTALL_PREFIX}/lib/crt0.o ${VENTUS_INSTALL_PREFIX}/lib/riscv32clc.o -L${VENTUS_INSTALL_PREFIX}/lib -lworkitem --gc-sections --init $OUTPUT_DIR/${TARGET_CL}
```
链接器将多个输入文件组合为一个 `RISC-V ELF` 文件，`crt0.o` 提供初始化逻辑，`kernel.o` `包含内核实现，libworkitem` 提供支持函数，而链接脚本 `elf32lriscv.ld` 定义了内存布局和入口点。最终生成的 ELF 文件可以在 Spike 上执行。

从 `.riscv` 到 `.txt`:
```bash
${VENTUS_INSTALL_PREFIX}/bin/llvm-objdump -d --mattr=+v,+zfinx $OUTPUT_DIR/${TARGET_CL}.riscv >& $OUTPUT_DIR/${TARGET_CL}.txt
```
使用 `llvm-objdump` 反汇编可执行文件, 查看其完整的指令级实现。

---

### [ventus.h](${VENTUS_INSTALL_PREFIX}/../../llvm-project/libclc/riscv32/lib/ventus.h)
作为重要的硬件寄存器头文件, 定义了寄存器地址, 例如当前 warp 的最小线程 `CSR_TID` 在 0x800。
其次定义了内核元数据缓冲区的参数偏移量, 例如内核入口 `KNL_ENTRY` 偏移0。

### [crt0.s](${VENTUS_INSTALL_PREFIX}/../../llvm-project/libclc/riscv32/lib/crt0.S)

crt0.S 是程序的入口文件, 利用 ventus.h 的定义，完成 OpenCL 内核的启动、参数传递和打印缓冲区初始化：

  - 初始化全局指针 `gp`：用于访问全局和静态变量
  - 设置堆栈指针 `sp` 和线程指针 `tp`：为每个线程和工作组分配私有和本地内存
```s
csrr t1, CSR_WID         # 获取工作组 ID
csrr t2, CSR_LDS         # 获取本地数据段起始地址
li t3, 1024              # 每个工作组分配 1 KB 内存
mul t1, t1, t3
add sp, t1, t2           # `sp` 指向本地内存基地址
li tp, 0                 # `tp` 为每个线程分配私有内存
```
  - 清零 .bss 段：为未初始化的全局变量分配并初始化为零
  - 读取内核元数据：从硬件寄存器 `CSR_KNL` 获取内核入口地址、参数地址等
```s
csrr t0, CSR_KNL                 # 获取内核元数据地址
lw t1, KNL_ENTRY(t0)             # 获取内核入口地址
lw a0, KNL_ARG_BASE(t0)          # 获取内核参数地址
lw t2, KNL_PRINT_ADDR(t0)        # 获取打印缓冲区地址
lw t3, KNL_PRINT_SIZE(t0)        # 获取打印缓冲区大小
la t4, BUFFER_ADDR               # 保存打印缓冲区地址
la t5, BUFFER_SIZE               # 保存打印缓冲区大小
```

  - 跳转到内核入口点：通过 `jalr` 指令跳转到内核函数
  - 如果发生异常 (非法指令/内存访问错误/···)或者 warp 执行结束, 程序跳转 `spike_end`, 向 `tohost` 写入结束信号 `1`, 模拟结束

```s
  la t6, spike_end                   # exception to stop spike
  csrw mtvec, t6
  jalr t1                            # call kernel program
  ···
  endprg x0, x0, x0
  j spike_end
```
---

### [elf32lriscv.ld](/llvm-project/utils/ldscripts/ventus/elf32lriscv.ld)  
这是一个RISC-V架构的链接脚本, 它定义了elf的内存布局、段分配及程序入口点。

---


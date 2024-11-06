# Developer Guide for Ventus LLVM

ventus-llvm的背景知识：
- [乘影编译器概述](./乘影编译器概述)

ventus-llvm的特有实现：
- [分支发散](./分支发散.md)
- [位宽拓展](./位宽拓展.md)
- [调用约定](./调用约定.md)，包括ventus-kernel的特殊规定
- 软硬件接口（llvm和spike在源码中如何对接（可能涉及elf32lriscv.ld、crt0.S、ventus.h等文件）？llvm和spike如何保持一致的参数传递规范？
- 从.cl到.ll的过程中，如何支持half数据类型？指令和寄存器描述的td文件如何增量修改？

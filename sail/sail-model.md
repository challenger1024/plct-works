# sail-model学习笔记

## 总体理解
sail-model 由许多模块组成，其中核心模块是core,sys,postlude。
## core 模块
此模块提供其他模块所依赖的基本的类型定义和函数功能。
换句话说，它相当于一个“底层基础库”，里面定义了所有其他模块都会用到的通用类型、变量宽度、辅助函数等。
### 宽度定义模块
- xlen.sail ：定义整数寄存器宽度，如 32 位或 64 位（ xlen ），以及物理地址位宽 ( physaddr_bits )。
- flen.sail ：定义浮点寄存器宽度（如 32/64 位）。
- vlen.sail ：定义向量寄存器宽度（如 128/256 位）。
### 工具箱基础库
prelude.sail包含有用的sail库函数。
这是 Sail 语言的一些基础工具函数，比如数学运算、位操作、打印调试等。
### 一些类型定义
#### regs.sail
regs.sail  文件定义了  通用寄存器组（register file） ；
每个寄存器的类型是  regtype ，这个类型在  reg_type.sail  里定义；
寄存器编号（比如 x0, x1, …, x31）由  types.sail  文件中定义的枚举或索引类型提供。
#### csr_begin.sail
csr_begin.sail  定义了  控制状态寄存器（CSR）系统的基础架构 ；
后续不同模块会“分散地”定义自己的 CSR（例如  mstatus ,  mtvec ,  satp ,  pmpcfg  等）；
它提供统一的机制来支持这些分散定义（例如注册、读写接口等）。
## sys 模块
sys 模块 是整个model的“执行核心”，它负责管理：
- hart（硬件线程）的保留状态（reservation state） ，
用于原子指令（如 LR/SC）的互斥操作；
- 物理内存与虚拟内存管理 ；
- 平台内存映射（memory map） ；
- 中断与异常处理 。
也就是说，它是 CPU 行为仿真的中心模块，连接 ISA 逻辑和外部模拟平台。
##   postlude  module
这个模块完成整个 RISC-V Sail 模型的定义，实现了**取指令（fetch） 和 取-译-执循环（fetch-decode-execute cycle）**的驱动逻辑。
也就是说，这部分文件让模拟器真正能“跑起来”，从内存取指令、译码、执行并循环。
##   异常处理模块（exceptions module）
在  异常处理  过程中，模型如何处理与异常相关的地址（例如触发异常的指令地址、访问异常的内存地址等），由  sys_exceptions.sail  中的函数定义。
sync_exception.sail  定义了一个  结构体（structure） ，用于保存一个异常的体系结构信息。
这两个文件（ sys_exceptions.sail  +  sync_exception.sail ）共同组成了  “异常处理模块” 。
这个模块负责模型中所有同步异常（如指令访问错误、页错误等）的捕获与传递。
## 物理内存保护模块（PMP module）
pmp  模块实现  物理内存保护  功能。
在 RISC-V 中，PMP 是用来控制不同特权级对内存的访问权限的安全机制。
pmp_regs.sail  定义了 PMP 寄存器（如  pmpcfg0 ,  pmpaddr0  等），以及它们的读写函数。
这些寄存器保存了每个 PMP 区域的配置（地址边界、权限标志等）。
pmp_control.sail  负责执行真正的  访问权限检查逻辑  和  优先级匹配 。
## 问题分析
issue#1367需要修改
- pnp_regs.sail:148:pmpWriteAddr()。
- config.json.in，添加2个配置项
- core/sys_regs.sail:889:function legalize_satp()

pa_bits:作用于PPN/PMP，限制了屋里地址宽度。
asid_bits: 作用于satp.asid，限制地址空间标志服宽度，模拟硬件支持的最大地址空间数量。

在 RISC-V 架构中，ASID（Address Space Identifier）字段存在于  satp  寄存器中，用于区分不同进程的地址空间。
RISC-V 有两种主要的  satp  格式：
satp32 ：用于 RV32（32 位）
satp64 ：用于 RV64（64 位）
rv32和rv64这里表示cpu的通用寄存器宽度
对于rv32,Mode  在  satp  的高位（bits [31:31]），即 1 位
ASID  在 bits [30:22]，共  9 位
PPN  在 bits [21:0]
对于rv64,Mode: bits [63:60] → 4 bits
ASID: bits [59:44] →  16 bits
PPN: bits [43:0]
sail语言中，切片操作不能用一个运行时才确定的变量，这样的话编译会出错。


function legalize_satp(
  arch : Architecture,
  prev_value : xlenbits,
  written_value : xlenbits,
) -> xlenbits = {
  
  // Number of implemented physical address bits (PA bits).
  // Determines the width of the physical address space. 
  // Higher bits beyond this value are treated as read-only zeros.
  // Typically 34 for RV32, and up to 56 for RV64 (as defined in the RISC-V Privileged Spec).
  let pa_bits   : range(0, xlen + 2) = if xlen == 32 then config memory.physical_addr_bits_32 else config memory.physical_addr_bits;
  // Number of implemented ASID bits (Address Space Identifier bits).
  // Determines how many bits in the SATP.ASID field are valid.
  // Bits above this number are read-only zeros.
  // Typically 16, but some implementations may use fewer (e.g., 8).
  let asid_bits:nat       = if xlen == 32 then config memory.asid_bits_32 else config memory.asid_bits;

function legalize_satp(
  arch : Architecture,
  prev_value : xlenbits,
  written_value : xlenbits
) -> xlenbits = {
  if xlen == 32 then {
    let s = Mk_Satp32(written_value);
    let s = [
      s with
      // If full 16-bit ASID is not supported then the high bits will be read only zero.
      Asid = zero_extend(s[Asid][asid_bits - 1 .. 0]),
      // Bits above the physically addressable memory are read only zero.
      PPN = zero_extend(s[PPN][pa_bits - pagesize_bits-1 .. 0]),
    ];
    match satpMode_of_bits(arch, 0b000 @ s[Mode]) {
      None()  => prev_value,
      Some(Sv_mode) => match Sv_mode {
        Bare if currentlyEnabled(Ext_Svbare) => s.bits,
        Sv32 if currentlyEnabled(Ext_Sv32) => s.bits,
        _ => prev_value,
      }
    }
  } else if xlen == 64 then {
    let s = Mk_Satp64(written_value);
    let s = [
      s with
      Asid = zero_extend(s0[Asid][asid_bits - 1 .. 0]),
      //Asid = zero_extend(s[Asid] & asid_mask),
      PPN = zero_extend(s[PPN][pa_bits - pagesize_bits-1 .. 0]),
      //PPN  = zero_extend(s[PPN]  & ppn_mask),
    ];
    match satpMode_of_bits(arch, s[Mode]) {
      None()  => prev_value,
      Some(Sv_mode) => match Sv_mode {
        Bare if currentlyEnabled(Ext_Svbare) => s.bits,
        Sv39 if currentlyEnabled(Ext_Sv39) => s.bits,
        Sv48 if currentlyEnabled(Ext_Sv48) => s.bits,
        Sv57 if currentlyEnabled(Ext_Sv57) => s.bits,
        _ => prev_value,
      }
    }
  } else {
    internal_error(__FILE__, __LINE__, "Unsupported xlen" ^ dec_str(xlen))
  }
}

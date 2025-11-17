# sail-model学习笔记

## 如何使用
-**启动**: build/c_emulator/sail_riscv_sim


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
## 问题分析-issue#1367需要修改
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

## 问题分析 - issue#1389
https://github.com/riscv/sail-riscv/issues/1389
修改：./model/sys/mem.sail:line242
- ./model/sys/platform.sail:line343,function within_mmio_writable forall 'n, 0 < 'n <= max_mem_access . (addr : physaddr, width : int('n)) -> bool =
当启用  RVFI 模式 （即  RISC-V Formal Interface ）时，Sail 会输出形式化验证用的信号。
在这种模式下， 不会访问真实设备或 MMIO 区域 。
所以直接返回  false ，表示当前模式下  禁用 MMIO 写操作 。
- ./model/sys/platform.sail:line359,function mmio_write forall 'n, 0 < 'n <= max_mem_access . (paddr : physaddr, width : int('n), data: bits(8 * 'n)) -> MemoryOpResult(bool) =
- ./model/sys/platform.sail:line23,function within_clint forall 'n, 0 < 'n <= max_mem_access . (Physaddr(addr) : physaddr, width : int('n)) -> bool =  
判断某次内存访问是否落在 CLINT 区域内，这里就是是否在MMIO设备里  
这里逻辑就是a的左端点在b的左端点的右侧
且a的右端点在b的右端点的左侧
line40,function within_htif_readable forall 'n, 0 < 'n <= max_mem_access . (addr : physaddr, width : int('n)) -> bool =
表示HTIF 的读写区域一致，无需重复边界检测代码，复用了within_htif_writable函数
HTIF（Host-Target Interface）  是 RISC-V 模拟器（例如 Spike）中常见的一个“虚拟外设”。
line34,function within_htif_writable forall 'n, 0 < 'n <= max_mem_access . (Physaddr(addr) : physaddr, width : int('n)) -> bool =
当前访问的物理地址  addr （宽度为  width  字节）是否 与 HTIF 可写区域（tohost buffer）重叠 。
判断逻辑是:a和b是两个线段
a的左端点在b的右端点的左侧
且a的右端点在b的左端点的右侧
所以他们是有交集的
原函数：
function within_htif_writable forall 'n, 0 < 'n <= max_mem_access . (Physaddr(addr) : physaddr, width : int('n)) -> bool =
  match htif_tohost_base {
    None() => false,
//    Some(base) => (addr <_u base + htif_tohost_size) & (addr + width >_u base)
    Some(base) => (addr+width <=_u base + htif_tohost_size) & (addr  >=_u base)
  }



## 调用逻辑
- 原本只有一个within_mmio_writable函数，改成intersects_mmio_writable函数  
- intersects_mmio_writable:判断padd与MMIO是否有交集  
- 判断过程：如果paddr与MMIO有交集，则交给MMIO处理，及mmio_write函数。  
- 在mmio_write处理的过程中，如果发现paddr不完全包含于MMIO，抛出错误。

在platform.sail添加函数，intersectclin,intersecthtif,
后两个函数调用range_utils.sail中的intersect函数进行范围检查
withinclin和withinheft也可以调用range_utils.sail中的within函数
range_utils.sail中需要写两个函数：intersect和whithin
用于判断两个范围有交集，和两个范围的完全包含情况

./model/core/platform_config.sail:37:
let plat_clint_base : physaddrbits = to_bits_checked(config platform.clint.base : int)


physaddrbits_zero_extend
./model/core/prelude_mem_addrtype.sail:15:newtype physaddr = Physaddr : physaddrbits
./model/core/prelude_mem_addrtype.sail:11:type physaddrbits = bits(physaddrbits_len)
./model/core/xlen.sail:22:type physaddrbits_len : Int = if xlen == 32 then 34 else 64

function intersect_clint forall 'n, 0 < 'n <= max_mem_access . (Physaddr(addr) : physaddr, width : int('n)) -> bool = {
  // To avoid overflow issues when physical memory extends to the end
  // of the addressable range, we need to perform address bound checks
  // on unsigned unbounded integers.
  let addr_int       = unsigned(addr);
  let clint_base_int = unsigned(plat_clint_base);
  let clint_size_int = unsigned(plat_clint_size);
    clint_base_int <= addr_int+width
  & (addr_int  <= (clint_base_int + clint_size_int)
}

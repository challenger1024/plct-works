# 为预留行为AMOCAS_odd_register 添加配置

## 问题介绍
问题来源于[issue#775](https://github.com/riscv/sail-riscv/issues/775)  
问题摘录：
Currently the Sail model is mostly written so that if code triggers some reserved behaviour the model will do something sensible, e.g. an illegal instruction exception, ignoring CSR writes, etc.
An alternative would be to immediately abort execution. However I think there are use cases where we don't want to do that. Alasdair mentioned that some code relies on reserved behaviour (e.g. bootloaders), and I would be really surprised if you could e.g. boot Linux without hitting any.
I think there are use cases for both modes, so I think we want a global flag with these two options:
1. Reserved behaviour will immediately abort the model.  
2. The model will take some realistic action that a sensible real CPU would do.  

Odd-numbered registers in  amocas   rs2  and  rd就是其中的一个保留行为，本文中称之为amocas_odd_register.  

---

## 问题分析
要想解决amocas_odd_register问题，首先需要对其进行分析，氛围以下步骤：  
### 了解amocas
参见[The RISC-V Instruction Set Manual, Volume I: Unprivileged Architecture 15.1](https://riscv.github.io/riscv-isa-manual/snapshot/unprivileged/#_worddoublewordquadword_cas_amocas_wdq_instructions).  
#### zacas扩展概述
Zacas（Atomic Compare-And-Swap）是 RISC-V 架构中用于支持硬件级原子比较交换（Compare-and-Swap, CAS）操作的标准扩展。该扩展旨在为多线程和多核系统提供高效的同步机制，使软件层能够在不使用锁的情况下实现线程安全的并发访问。
CAS 指令的主要功能是在一个原子操作中执行：
1. 从内存加载数据；
2. 将其与寄存器中保存的比较值进行比对；
3. 若匹配，则写入新的替换值；
4. 最后将原始值返回给调用方。

Zacas 扩展定义了三种数据宽度下的 CAS 操作：
- AMOCAS.W：操作 32 位字；
- AMOCAS.D：操作 64 位双字；
- AMOCAS.Q：操作 128 位四字（仅 RV64 支持）。

---

#### 寄存器对（Register Pair）机制与对齐约束
对于宽度超过当前 XLEN 的操作（如在 RV32 中执行 64 位 CAS，或在 RV64 中执行 128 位 CAS），Zacas 扩展要求指令使用寄存器对（register pair） 表示操作数与结果：
- 比较值来源寄存器：(rd, rd+1)
- 交换值来源寄存器：(rs2, rs2+1)

其中，rd 与 rs2 必须是偶数编号寄存器，例如 x10 或 x12。  
RV32的AMOCAS.d代码示例:
```sail
    temp0 = mem[X(rs1)+0]
    temp1 = mem[X(rs1)+4]
    comp0 = (rd == x0)  ? 0 : X(rd)
    comp1 = (rd == x0)  ? 0 : X(rd+1)
    swap0 = (rs2 == x0) ? 0 : X(rs2)
    swap1 = (rs2 == x0) ? 0 : X(rs2+1)
    if ( temp0 == comp0 ) && ( temp1 == comp1 )
        mem[X(rs1)+0] = swap0
        mem[X(rs1)+4] = swap1
    endif
    if ( rd != x0 )
        X(rd)   = temp0
        X(rd+1) = temp1
    endif
```

---

#### 奇数寄存器（Odd-numbered Register）保留的技术原因
根据 Zacas 规范，凡在 AMOCAS.D（RV32）和 AMOCAS.Q（RV64）中出现奇数编号的 rd 或 rs2 均属于保留编码（reserved encoding）。这一设计并非任意约束，而是基于以下三方面的体系结构与实现考量：
1. 数据对齐要求（Alignment Constraint）
宽操作需要寄存器对在物理上以 2×XLEN 的边界对齐。若允许奇数寄存器作为寄存器对的起始位置，则会破坏硬件访问对齐规则，增加寄存器文件设计复杂度。
2. 寄存器重叠风险（Overlap Hazard）
奇数寄存器可能与前一寄存器对的高半部重叠。例如若 x11 被单独指定为 rd，而 x10/x11 又同时作为另一寄存器对参与运算，则会导致结果覆盖冲突，破坏原子性。
3. 硬件管线简化（Hardware Simplicity）
强制要求寄存器对以偶数编号起始，可令硬件在执行阶段仅需一次并行访问寄存器文件的连续区域，简化读写通路与旁路（bypass）逻辑，降低设计复杂度与功耗。

因此，Zacas 扩展中所有奇数寄存器起始的 AMOCAS 编码被定义为“保留行为”（Reserved Behaviour），在形式化模型（如 Sail 模型）中通常直接触发异常或中止执行 (abort the model)，以避免不可预测的硬件状态。

---

### 指令执行流程
- sail-riscv的指令调度主要有model/postlude/step.sail:184 的try_step()函数控制。  
  try_step 函数是 Sail 模型的主要内部驱动程序，他的主要任务：
  - 执行前置钩子；
  - 根据当前 CPU 状态（active 或 waiting）选择执行路径；
  - 处理执行结果（异常、中断、WFI、非法指令等）；
  - 如果 CPU 仍在等待 → 返回 true，否则 → 正常执行完返回 false。

- try_step()会调用step.sail中第89行的run_hart_active()函数。
该函数完成了获取指令，解码指令，执行指令的工作。
- 解码指令使用ext_decode(),其默认实现定义在model/postlude/decode_ext.sail:15，但AMOCAS的实现位于model/extensions/A/zaamo_insts.sail 
- model/sys/insts_begin.sail 定义了一些指令执行结果的枚举。  
- amocas的解码指令和执行指令的实现位于model/extensions/A/zaamo_insts.sail

---

## 问题解决方案
- 在配置文件中添加reserve_behaviour配置项，amocas_odd_register是其中一个，方便后续添加更多保留行为的配置项。
- amocas_odd_behaviour 可以是"exit"(立即退出)或"illegal"(非法指令).
- 添加一个专用的amocas-odd-register处理函数，让模型能够立即退出.
- 在解码过程中添加判断，如果rs2或rd是奇数寄存器且配置项设置为"exit"，则调用上面的amocas-odd-register函数
- 如果配置项是"illegal"，则amocas_encoding_valid()仍然返回false,模型将触发illegal instruction exception. 

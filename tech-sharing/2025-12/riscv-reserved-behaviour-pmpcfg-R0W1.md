# 技术分享：PMPcfg

## PMP介绍
PMP（Physical Memory Protection，物理内存保护）是一种 硬件级访问控制机制 ，它让我们可以精确地规定： 什么特权级的软件才能访问某段物理内存， 以及能不能读(R)、写(W)、执行(X)。  
PMP可以有效防止：用户程序误操作内存，错误代码破坏关键数据等。  
PMP条目由一个8位配置寄存器和一个MXLEN位地址寄存器描述:
- pmpcfg： 配置寄存器，设置访问权限（R/W/X）和匹配模式；
- pmpaddr： 地址寄存器，定义保护区域的边界。

---

PMPcfg各位含义表
| 位编号（bit） | 字段名称  | 位宽 | 含义说明                          | 备注                                                                                                 |
| -------- | ----- | -- | ----------------------------- | -------------------------------------------------------------------------------------------------- |
| 7        | **L** | 1  | 锁定位（Lock bit）                 | 当置位为 1 时，该 PMP 条目被锁定；即使在 M 模式下也不能修改，直到复位。                                                          |
| 6–5      | —     | 2  | 保留位（Reserved）                 | 目前未定义，应保持为 0。                                                                                      |
| 4–3      | **A** | 2  | 地址匹配类型（Address-matching mode） | 控制该条目如何解释 `pmpaddr`：<br>00=OFF（关闭）<br>01=TOR（Top of Range）<br>10=NA4（4字节）<br>11=NAPOT（自然对齐的2ⁿ字节区域） |
| 2        | **X** | 1  | 执行权限（Execute）                 | 1=允许执行指令；0=禁止执行。                                                                                   |
| 1        | **W** | 1  | 写权限（Write）                    | 1=允许写入；0=禁止写入。                                                                                     |
| 0        | **R** | 1  | 读权限（Read）                     | 1=允许读取；0=禁止读取。                                                                                     |

---

## pr 介绍
为riscv的保留行为PMPcfg 中R=0,W=1添加配置选项：
- Fatal,退出模型;  
- SetToR0W0, 将其设置为R=0,W=0,X=0;  
### 修改步骤：
- 在config.json.in中添加配置选项 reserved_behavior.pmpcfg_with_R0W1  
- 在types.sail中添加 ReservedBehaviorPolicy枚举，与配置选项对应，并为ReservedBehaviorPolicy添加string映射,从而重载该枚举与string之间的转换函数  
- 在errors.sail中，添加一个ReservedBehavior错误类型和一个专用于处理ReservedBehavior的错误函数  
- 在PMP_regs.sail中修改R=0,W=1时的逻辑，添加一个match，根据配置选项不同，执行不同分支。  

下面结合代码讲解，位于[https://github.com/riscv/sail-riscv/pull/1422/files](https://github.com/riscv/sail-riscv/pull/1422/files)



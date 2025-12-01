# linux下，如何开始sail编程

## 下载和配置sail的二进制文件
### 下载和配置
下载sail的二进制文件，访问[https://github.com/rems-project/sail/releases](https://github.com/rems-project/sail/releases)。  
查看自己的linux架构:
```bash
uname -m 
```
下载与系统架构对应版本的二进制sail.  
例如我的linux系统是`x86_64`架构，则下载`sail-Linux-x86_64.tar.gz`。  
执行命令:
```bash
wget  https://github.com/rems-project/sail/releases/download/0.20-linux-binary/sail-Linux-x86_64.tar.gz
tar -xvzf sail-Linux-x86_64.tar.gz
cd sail
```
我们使用`ls`命令，可以看到一个名为bin的目录，其中包含了sail的二进制可执行文件，我们需要将它放到环境变量中，这样就可以在系统的任意位置用sail命令执行它了  
```bash
export PATH="$PWD/bin:$PATH"
```
如果想永久添加环境变量：
```bash
nano ~/.bashrc
# 在文件末尾添加一行，路径改为你的bin所在目录
export PATH="/home/moses/sail/bin:$PATH"

```
保存并退出，并使其立即生效:
```bash
source ~/.bashrc
```
验证：
```bash
sail --version
```
### 注意事项
有几点需要注意：
- Sail 自带一些  库（libraries）  和  插件（plugins） ，放在解压目录的子目录中。默认情况下，Sail 会自动根据  bin  目录的位置去找到这些库和插件。也就是说，一般不用手动设置路径，只要不乱         动目录结构，Sail 就能正常运行。  
- Sail 自带了  Z3 SMT 求解器  的预编译版本。Z3 用于  类型检查、符号执行  等功能。因此你不需要额外安装 Z3 或其他依赖，直接解压就能用。  
- 如果你  移动了整个解压目录 （比如把解压后的文件夹从下载目录搬到  /opt/sail ），Sail 默认寻找库和插件的路径就可能找不到了。这时候需要手动设置两个环境变量：`$SAIL_DIR`  → 指向  `share/sail`  文件夹。`$SAIL_PLUGIN_DIR`  → 指向  `share/libsail/plugins`  文件夹。  

## 第一个sail程序
新建一个文件夹，在其中新建一个文件
```bash
mkdir sail-learn
code hello.sail
```
由于我使用的是vscode，所以命令是`code hello.sail`，你也可以用vim或nano  
### sail代码
在hello.sail中，写入下面的代码：
```sail
default Order dec

$include <prelude.sail>

function main () : unit -> unit = {
    print("Hello Sail!\n")
}
```
### 代码说明
- **`default Order dec`**: 定义vector的默认顺序  
- **`$include <prelude.sail>`**: 引入sail标准库头文件  
### 编译sail代码
#### 编辑makefile
在刚才的`learn-sail`目录下,新建一个`makefile`文件，在其中粘贴下面的脚本
```makefile
# ================== SAIL 编译配置 ==================
SAIL := sail
SAIL_FLAGS := --require-version 0.20 \
              --all-modules \
              -O -Oconstant_fold  \
              -c

# ================== 路径配置 ==================
SAIL_DIR := $(shell $(SAIL) --dir)
SAIL_LIB_DIR := $(SAIL_DIR)/lib

# ================== 外部库配置 ==================
# GMP用于支持任意精度整数与浮点数运算
# pkg-config --cflags gmp ：返回编译时需要的头文件路径；
GMP_FLAGS := $(shell pkg-config --cflags gmp 2>/dev/null || echo "") 
# pkg-config --libs gmp ：返回链接时需要的库；
GMP_LIBS := $(shell pkg-config --libs gmp 2>/dev/null || echo "-lgmp")
# ZLIB用于处理压缩的ELF文件
ZLIB_FLAGS := $(shell pkg-config --cflags zlib 2>/dev/null || echo "")
ZLIB_LIBS := $(shell pkg-config --libs zlib 2>/dev/null || echo "-lz")

# ================== 编译器配置 ==================
CC := gcc
C_FLAGS := -I$(SAIL_LIB_DIR) $(GMP_FLAGS) $(ZLIB_FLAGS)

# ================== 构建流程 ==================
all: hello-sail

# 第一步：用 SAIL 生成 C 文件
hello-sail.c: hello.sail
	$(SAIL) $(SAIL_FLAGS) $< -o hello-sail

# 第二步：用 GCC 编译生成的 C 文件
hello-sail: hello-sail.c
	$(CC) $(C_FLAGS) -o $@ $(SAIL_LIB_DIR)/*.c $(GMP_LIBS) $(ZLIB_LIBS) $<

clean:
	rm -f hello-sail *.c *.h *.o

```
#### 文件中定义的命令:  
- **`make hello-sail.c`**:  将sail源代码文件构建为c源代码文件
- **`make hello-sail`** 构建全部 ,生成可执行的二进制文件  
- **`make clean`**: 清理构建产物  
#### 配置项说明
- **`-Oconstant_fold`**: 性能优化  
- **GMP** 用于支持任意精度整数与浮点数运算
- **ZLIB**: 用于处理压缩的ELF文件
### 运行第一个sail程序
执行命令:
```bash
make hello-sail
./hello-sail
```
可以看到`hello sail!`的输出。
## sail调用c代码
需要在sail编译成c代码的阶段，加入`-c_include <x.h>`选项，这里`x.h`是被调用的c语言代码的头文件  
需要在c代码编译为二进制文件的阶段加入`x.c`与sail生成的c源文件一起编译。这里`x.c中`是对应`x.h`中定义的函数的时限  
## sail语法介绍
### 一些基本类型
| 类型 | sail中的含义 | 编译为c代码后的含义 | 代码示例 |
| --- | --- | --- | --- |
| unit | 在函数声明中表示无返回值 | int |  |
| bool | true/false | true/false | let a:bool=true |
| int | 表示任意精度整数 | int64 | let a:int=1 |
| int('n) | 'n是sail中的类型约束 | int64 | let number:int(1)=1 |
| nat | 任意精度自然数 | int64 | |
| range('m,'n) | 闭区间'm和'n之间的数字 | int64 |  |
| {n1,n2,...} | 一个数字集合，变量值只能是集合中的某个值 | int64 |  |
| bits('n) | 位向量，'n限制了位向量的长度 | uint|  |
| vector(n,type) | 向量：n为元素个数，type为元素类型，例如int | zz5vecz8z5iz9 |  |
| list | 列表 |  | 两种写法：//    let l: list(int)=[|1,2,3|];let l: list(int)=1::2::3::[||]; |
| tuple | tuple,必须有两个或更多元素，每个元素的类型可以不同 |  | let tuple1 : (string, int) = ("Hello, World!", 3) |
| string | string,包裹在两个双引号之间的字符序列 |  |  |


### 一些关键字
| 关键字 | 含义 | 示例 |
| --- | --- | --- |
| let | 定义不可变变量 | let a: int =1; |
| var | 定义可变变量 | var b: int=2; |
| bitzero,bitone | bitvector中的0和1 | let v : bits(2) = [bitzero, bitone]; |
| inc / dec | 决定bitvector的索引按递增或者递减排序 | default Order dec, inc为递增，此时左侧第一位索引为0，是最高位，索引从最高位递增到最低位；dec为递减，此时右侧第一位索引为0，是最高位，索引从最高位递减到最低位。 前者是MSB0,后者是LSB0。一般情况都是使用dec |
| type | 定义类型别名 | type regbits = bits(5)，给bits(5)类型起一个别名，叫做regbits;type square('n, 'a) = vector('n, vector('n, 'a)) |
| match | 模式匹配 | let n : int = f();<br>match n {<br>    1 => print_endline("1"),<br>    2 => print_endline("2"),<br>    3 => print_endline("3"),<br>    _ => print_endline("wildcard"),<br>} |
| overload | 重载关键字 | overload operator / = {div1, div2}，重载除法运算符，div1和div2是两个除法函数的函数名 |


### sail 特性
- **type kind**: type的type,称为kind,sail中有三种kind:Int ,Bool,Type。  
- **mapping**: 是双向映射，其声明类似函数，也是使用val关键字，下面是例子  
  val size_bits : word_width <-> bits(2)
  mapping size_bits = {
    BYTE   <-> 0b00,
    HALF   <-> 0b01,
    WORD   <-> 0b10,
    DOUBLE <-> 0b11
  }
  word_width是一个枚举类型  
  调用时，与函数调用相同：  
  let w1 : word_width = size_bits(0b00);  
  let w2 : bits(2) = size_bits(BYTE);  
  第一句：你传入  bits(2) ，希望得到  word_width ，所以它执行“反向映射”。  
  第二句：你传入  word_width ，希望得到  bits(2) ，所以它执行“正向映射”。  
  也可以只做单向映射，只要把<->改成=.即可  
- **scattered**: 允许你在多个地方为同一个定义(包括function,mapping,union,enum等)逐步添加 “clauses（分支）”。  
  例子：
  声明阶段：
  scattered union U  

  scattered enum E  

  // For scattered functions and mappings, we have to provide a type signature with val  
  val foo : U -> bits(64)  

  scattered function foo  

  val bar : E <-> string  

  scattered mapping bar  
  这实际上是给 4 个“分散定义”分别加上了新的分支:  
  enum clause E = E_one  
  union clause U = U_one : bits(64)  
  function clause foo(U_one(x)) = x  
  mapping clause bar = E_one <-> "first member of E"  
  结束需要写end:  
  end E  
  end U  
  end foo  
  end bar  
  这会告诉sail，不会再添加新的分支。


### 一些运算符
| 运算符 | 含义 | 示例 |
| --- | --- | --- |
| : | 表示类型归属 | x : y,表示x是y类型的 |
| _ | 在bitvector中作为16位分隔符 | 例如一个48位bitvector: 0xFFFF_FFFF_FFFF |
| @ | 连接两个bitvector | 例如：x=0xFFFF,y=0x0000,x@y=0xFFFF0000 |
| [] | 下标访问运算符 | let v : vector(4, int) = [1, 2, 3, 4],let a = v[0],假设Order=dec,那么a=4 |
| [n..m] | 切片运算符 | let v: bits(32)=0xFFFF0001;,let z=v[15..0]; |

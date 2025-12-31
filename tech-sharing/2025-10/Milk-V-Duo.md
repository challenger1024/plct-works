# RuyiSDK下GCC和LLVM对Milk-V Duo 256M的支持
## 目录
- [一、简介](#简介)
- [2、安装官方 buildroot SDK v2 镜像](#安装官方-buildroot-SDK-v2-镜像)
- [3、Ruyi下GCC与LLVM对Milk-v duo 256m的支持测试](#Ruyi SDK下GCC和LLVM对Milk-v duo 256m的支持测试)
- [4、 在Milk-v duo 上调试多线程程序](#在Milk-v上调试多线程程序)
## 简介
本次分享主要包括幾個部分：
- Milk-v duo 256m的buildroot SDK v2 镜像安裝
- Ruyi  SDK下gcc和llvm對milk-v duo 256m的支持測試
- milk-v duo上使用gdb對程序進行調試  
[观看视频可访问：https://www.bilibili.com/video/BV1SxWqzYEbA/?spm_id_from=333.999.0.0&vd_source=7c11325dc15b63618c7af305547018c4](https://www.bilibili.com/video/BV1SxWqzYEbA/?spm_id_from=333.999.0.0&vd_source=7c11325dc15b63618c7af305547018c4)
## 安装官方 buildroot SDK v2 镜像   
### 下载镜像
[点击这里](https://github.com/milkv-duo/duo-buildroot-sdk-v2/releases)下载镜像  
我下载的是`milkv-duo256m-musl-riscv64-sd_v2.0.1.img.zip`
下载之后解压
### 下载系统烧录工具
- [点击这里](https://github.com/balena-io/etcher/releases/tag/v2.1.4)下载`balenaEtcher-2.1.4.Setup.exe`  
由于我是Windows系统，所以下载这个版本，其他系统需要下载对应的安装包  
- 下载后安装  
### 系统烧录
- 将sd卡插入读卡器，连接到电脑
- 将sd卡格式化
- 打开刚才安装的烧录工具后，点击从文件烧录  
选择之前解压好的镜像
-  点击选择目标磁盘  
选择sd卡  
- 点击现在烧录  
会有进度条，等待片刻后提示烧录成功
### 安装驱动程序
使用usb-type-c连接milk-v和windows时，windows系统的CDC NCM 设备可能需要安装驱动程序后才能正常使用  
参考[这个链接](https://milkv.io/zh/docs/duo/getting-started/setup)安装驱动程序
### 使用windows登录到milk-v的系统
打开Windows PowerShell使用 `ping` 命令测试。`ping 192.168.42.1`  
输出  
```bash

正在 Ping 192.168.42.1 具有 32 字节的数据:
来自 192.168.42.1 的回复: 字节=32 时间<1ms TTL=64
来自 192.168.42.1 的回复: 字节=32 时间<1ms TTL=64
来自 192.168.42.1 的回复: 字节=32 时间<1ms TTL=64
来自 192.168.42.1 的回复: 字节=32 时间<1ms TTL=64

192.168.42.1 的 Ping 统计信息:
    数据包: 已发送 = 4，已接收 = 4，丢失 = 0 (0% 丢失)，
往返行程的估计时间(以毫秒为单位):
    最短 = 0ms，最长 = 0ms，平均 = 0ms
```
现在可以使用ssh进行连接
- 用户名:`root`
- 密码:`milkv`
- 地址: `192.168.42.1`
---
## GCC和LLVM对Milk-V Duo 256M的支持测试
测试说明：验证GCC和LLVM对Milk-V Duo 256M的支持
|序号|输入及操作说明|期望测试结果|
|---|---|---|
|1|安装依赖包<br>`sudo apt update`<br>`sudo apt install -y wget tar zstd xz-utils git build-essential`|成功安装|
|2|安装ruyi包管理器<br>`wget https://mirror.iscas.ac.cn/ruyisdk/ruyi/tags/0.40.0/ruyi-0.40.0.amd64`<br>`chmod +x ruyi-0.40.0.amd64`<br>`sudo cp -v ruyi-0.40.0.amd64 /usr/local/bin/ruyi`|成功安B装|
|3|安装GCC和LLVM工具链<br>`ruyi config set repo.remote https://mirror.iscas.ac.cn/git/ruyisdk/packages-index.git` <br>`ruyi update`<br>`ruyi install gnu-plct llvm-plct`|成功安装|
|4|创建并激活ruyi虚拟环境（GCC）<br>`ruyi venv -t toolchain/gnu-plct milkv-duo venv-gnu-plct-duo`<br>`. ~/venv-gnu-plct-duo/bin/ruyi-activate`<br>或者进入`bin`文件夹后执行`source ./ruyi-activate`|成功创建虚拟环境|
|5|验证GCC版本<br>`riscv64-plct-linux-gnu-gcc -v`|输出版本号|
|6|编译Hello World（GCC）<br>cat << EOF > hello.c<br>#include <stdio.h><br>int main() {<br>  printf("Hello, World!\\n");<br>  return 0;<br>}<br>EOF<br>`riscv64-plct-linux-gnu-gcc hello.c -static -o hello-gcc`<br>使用qemu模拟RISC-v架构运行<br>`qemu-riscv64 ./hello-gcc`|成功编译运行|
|7|编译coremark（GCC）<br>`git clone https://github.com/eembc/coremark`<br>`cd coremark`<br>`make CC=riscv64-plct-linux-gnu-gcc XCFLAGS="-mcpu=thead-c906 -static" compile`<br>`mv coremark.exe coremark-gcc`<br>使用qemu模拟RISC-v架构运行<br>`qemu-riscv64 ./coremark-gcc`|成功编译运行|
|8|将GCC构建的二进制传输至开发板<br>`scp ../hello-gcc coremark-gcc root@192.168.42.1:~`|成功传输|
|9|返回上级目录并退出ruyi GCC虚拟环境<br>`cd ..`<br>`ruyi-deactivate`|成功退出虚拟环境|
|10|创建并激活ruyi虚拟环境（LLVM）<br>`ruyi venv -t toolchain/llvm-plct manual --sysroot-from gnu-plct venv-llvm-plct-duo`<br>`. ~/venv-llvm-plct-duo/bin/ruyi-activate`<br>或者进入bin文件夹后执行<br>`source ./ruyi-activate`|成功创建并激活虚拟环境|
|11|验证LLVM版本<br>`clang -v`|输出版本号|
|12|编译Hello World（LLVM）<br>`clang hello.c -static -o hello-llvm`|成功编译|
|13|编译coremark（LLVM）<br>`cd coremark`<br>`make clean`<br>`make CC=clang XCFLAGS="-march=rv64imafdc_xtheadba_xtheadbb_xtheadbs_xtheadcmo_\`<br>`xtheadcondmov_xtheadfmemidx_xtheadmac_xtheadmemidx_xtheadmempair_xtheadsync -static" compile`<br>`mv coremark.exe coremark-llvm`|成功编译|
|14|将LLVM构建的二进制传输到开发板<br>`scp ../hello-llvm coremark-llvm root@192.168.42.1:~`|成功传输|
|15|返回上级目录并退出ruyi llvm虚拟环境<br>`cd ..`<br>`ruyi-deactivate`|成功退出|
|16|SSH连接到开发板并执行编译好的二进制<br>`ssh root@192.168.42.1`<br>如提示Host key verification failed：<br>打开当前用户目录下的 .ssh/known_hosts目录，删除192.168.42.1对应行<br>登录密码为milkv，提示Are you sure you want to continue connecting时输入yes回车即可<br>`./hello-gcc`<br>`./hello-llvm`<br>`./coremark-gcc`<br>`./coremark-llvm|两次运行Hello World 均正确输出Hello, World!<br>两次运行coremark均正常输出coremark结果|
---

## 在milk-v上调试多线程程序
使用llvm编译带调试符号版本
```bash
llvm -Wall -O0 -g -fno-omit-frame-pointer -static  -o bank_account_debug bank_account.c
```
发送到milk-v duo 256m
```bash
scp bank_account_debug bank_account.c root@192.168.42.1:~
```
密码`milkv``

在milk-v duo上启动gdb
```bash
gdb ./bank_account_debug
````
常见gdb设置断点和调试命令
```gdb
break deposit_no_lock if i % 200000 == 0
break deposit_no_lock # 给函数deposit_no_lock打断点
break deposit_with_lock  #给函数deposit_with_lock打断点
break bank_account.c:42  #指定行断点
print balance #查看变量信息
thread apply all print balance  # 查看所有线程共享的变量balance
set scheduler-locking on  # 确保只有当前线程单步执行
thread 1 #切换到编号为1的线程
set print thread-events  # 打印线程事件

info threads
break deposit_no_lock
run
info threads
thread 2
print balance
```


# 附录A:常见问题
## 使用ruyi下载包时卡柱
解决方案，更换镜像源
```bash
ruyi config set repo.remote https://mirror.iscas.ac.cn/git/ruyisdk/packages-index.git 
```
## 使用ruyi更新包时出现报错`_pygit2.GitError: 577 conflicts prevent checkout`
这是由于git冲突引起的，需要先清理之前下载的文件后重新`update`
清理旧缓存后更新
```bash
rm -rf ~/.cache/ruyi
ruyi update
```
## 启动gdbserver后无法退出
原因：在开发板上，`ctrl+c`的信号被屏蔽。
解决方案：执行命令查看pid,随后直接杀死进程
```bash
ps aux | grep gdbserver
kill <pid>
```


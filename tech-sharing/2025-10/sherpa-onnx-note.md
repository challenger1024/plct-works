# 编译构建

## 下载二进制预编译文件
截止到2025.10.28,sherpa-onnx的最新版本是1.10.15，执行命令进行下载
```bash
wget https://huggingface.co/csukuangfj/sherpa-onnx-libs/resolve/main/riscv64/1.12.15/sherpa-onnx-v1.12.15-linux-riscv64-shared.tar.bz2
```
## 下载构建工具链
```bash
mkdir -p $HOME/toolchain
wget -q https://occ-oss-prod.oss-cn-hangzhou.aliyuncs.com/resource//1663142514282/Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.1-20220906.tar.gz
tar xf ./Xuantie-900-gcc-linux-5.10.4-glibc-x86_64-V2.6.1-20220906.tar.gz --strip-components 1 -C $HOME/toolchain
```
将构建工具链中的工具设置到环境变量
```bash
export PATH=$HOME/toolchain/bin:$PATH
```
验证是否配置成功:
```bash
riscv64-unknown-linux-gnu-gcc --version
```
配置正确情况下输出：
```bash

  riscv64-unknown-linux-gnu-gcc (Xuantie-900 linux-5.10.4 glibc gcc Toolchain V2.6.1 B-20220906) 10.2.0
  Copyright (C) 2020 Free Software Foundation, Inc.
  This is free software; see the source for copying conditions.  There is NO
  warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
验证g++
```bash
riscv64-unknown-linux-gnu-g++ --version
```
配置正确情况下输出：
```bash
  riscv64-unknown-linux-gnu-g++ (Xuantie-900 linux-5.10.4 glibc gcc Toolchain V2.6.1 B-20220906) 10.2.0
  Copyright (C) 2020 Free Software Foundation, Inc.
  This is free software; see the source for copying conditions.  There is NO
  warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
## Build sherpa-onnx  
目前，sherpa-onnx只支持动态链接库（shared libraries），将在未来支持静态链接（static linking）。  
克隆源代码，并构建：
```bash
git clone https://github.com/k2-fsa/sherpa-onnx
cd sherpa-onnx
./build-riscv64-linux-gnu.sh
```
构建完成后，使用下面的命令查看构建结果：
```bash
ls -lh build-riscv64-linux-gnu/install/bin
echo "---"
ls -lh build-riscv64-linux-gnu/install/lib
```
输出如下：(moses是用户名)
```bash
total 20M
-rwxr-xr-x 1 moses moses 1.4M Oct 28 12:37 sherpa-onnx
-rwxr-xr-x 1 moses moses 1.4M Oct 28 12:37 sherpa-onnx-alsa
-rwxr-xr-x 1 moses moses 1.4M Oct 28 12:37 sherpa-onnx-alsa-offline
-rwxr-xr-x 1 moses moses 144K Oct 28 12:37 sherpa-onnx-alsa-offline-audio-tagging
-rwxr-xr-x 1 moses moses 176K Oct 28 12:37 sherpa-onnx-alsa-offline-speaker-identification
-rwxr-xr-x 1 moses moses 377K Oct 28 12:37 sherpa-onnx-keyword-spotter
-rwxr-xr-x 1 moses moses 373K Oct 28 12:37 sherpa-onnx-keyword-spotter-alsa
-rwxr-xr-x 1 moses moses 1.4M Oct 28 12:37 sherpa-onnx-offline
-rwxr-xr-x 1 moses moses 136K Oct 28 12:37 sherpa-onnx-offline-audio-tagging
-rwxr-xr-x 1 moses moses 104K Oct 28 12:37 sherpa-onnx-offline-denoiser
-rwxr-xr-x 1 moses moses 136K Oct 28 12:37 sherpa-onnx-offline-language-identification
-rwxr-xr-x 1 moses moses 1.4M Oct 28 12:37 sherpa-onnx-offline-parallel
-rwxr-xr-x 1 moses moses  88K Oct 28 12:37 sherpa-onnx-offline-punctuation
-rwxr-xr-x 1 moses moses 132K Oct 28 12:37 sherpa-onnx-offline-source-separation
-rwxr-xr-x 1 moses moses 204K Oct 28 12:37 sherpa-onnx-offline-speaker-diarization
-rwxr-xr-x 1 moses moses 1.8M Oct 28 12:37 sherpa-onnx-offline-tts
-rwxr-xr-x 1 moses moses 1.8M Oct 28 12:37 sherpa-onnx-offline-tts-play-alsa
-rwxr-xr-x 1 moses moses 1.6M Oct 28 12:37 sherpa-onnx-offline-websocket-server
-rwxr-xr-x 1 moses moses 1.8M Oct 28 12:37 sherpa-onnx-offline-zeroshot-tts
-rwxr-xr-x 1 moses moses 116K Oct 28 12:37 sherpa-onnx-online-punctuation
-rwxr-xr-x 1 moses moses 373K Oct 28 12:37 sherpa-onnx-online-websocket-client
-rwxr-xr-x 1 moses moses 1.7M Oct 28 12:37 sherpa-onnx-online-websocket-server
-rwxr-xr-x 1 moses moses 128K Oct 28 12:37 sherpa-onnx-vad
-rwxr-xr-x 1 moses moses 136K Oct 28 12:37 sherpa-onnx-vad-alsa
-rwxr-xr-x 1 moses moses 1.4M Oct 28 12:37 sherpa-onnx-vad-alsa-offline-asr
-rwxr-xr-x 1 moses moses 6.0K Oct 28 12:37 sherpa-onnx-version
---
total 26M
-rw-r--r-- 1 moses moses  13M Mar  6  2024 libonnxruntime.so
-rw-r--r-- 1 moses moses  13M Mar  6  2024 libonnxruntime.so.1.14.1
drwxr-xr-x 2 moses moses 4.0K Oct 28 12:37 pkgconfig
```
## 使用sherpa-onnx
由于笔者并没有使用开发板进行编译，这里有两种方式使用sherpa-onnx，使用qemu运行，或者发送到开发板上运行。
### 发送到开发板
使用scp发送
```bash
tar czf sherpa-riscv64.tgz build-riscv64-linux-gnu/install
scp sherpa-riscv64.tgz root@<板子IP>:/root/
```
开发板上解压：
```bash
tar xzf sherpa-riscv64.tgz
cd build-riscv64-linux-gnu/install/bin
```
设置动态库路径  
因为这些二进制是  动态链接的（shared libraries） ，所以要让系统知道  .so  在哪。  
在开发板上运行：  
```bash
export LD_LIBRARY_PATH=/root/build-riscv64-linux-gnu/install/lib:$LD_LIBRARY_PATH
```
或者：
```bash
echo "/root/build-riscv64-linux-gnu/install/lib" | sudo tee /etc/ld.so.conf.d/sherpa.conf
sudo ldconfig
```
### 借用qemu运行
注意，只能使用以下安装方法安装的qemu才能正常运行
```bash
mkdir -p $HOME/qemu

mkdir -p /tmp
cd /tmp
wget -q https://files.pythonhosted.org/packages/21/f4/733f29c435987e8bb264a6504c7a4ea4c04d0d431b38a818ab63eef082b9/xuantie_qemu-20230825-py3-none-manylinux1_x86_64.whl
unzip xuantie_qemu-20230825-py3-none-manylinux1_x86_64.whl
cp -v ./qemu/qemu-riscv64 $HOME/qemu
export PATH=$HOME/qemu:$PATH
```
检查是否已安装qemu
```bash
qemu-riscv64 -h
```
### 下载文本转语音(TTS)模型
模型下载地址[https://github.com/k2-fsa/sherpa-onnx/releases/tag/tts-models](https://github.com/k2-fsa/sherpa-onnx/releases/tag/tts-models)  
这里我们使用中文TTS模型`vits-zh-aishell3.tar.bz2`  
执行命令：
```bash
cd /path/to/sherpa-onnx
wget https://github.com/k2-fsa/sherpa-onnx/releases/download/tts-models/vits-zh-aishell3.tar.bz2
tar xf vits-zh-aishell3.tar.bz2
rm  vits-zh-aishell3.tar.bz2

```
下载完成后，用以下命令运行模型
```bash
cd /path/to/sherpa-onnx
export PATH=$HOME/qemu:$PATH
export QEMU_LD_PREFIX=$HOME/toolchain/sysroot
export LD_LIBRARY_PATH=$HOME/toolchain/sysroot/lib:$LD_LIBRARY_PATH
LD_LIBRARY_PATH=./lib 
OMP_NUM_THREADS=4 ./bin/sherpa-onnx-offline-tts \
  --vits-model=./vits-zh-aishell3/vits-aishell3.int8.onnx \
  --vits-tokens=./vits-zh-aishell3/tokens.txt \
  --vits-lexicon=./vits-zh-aishell3/lexicon.txt  \
  --output-filename=./test-zh.wav \
  --print-args \
  "你好，欢迎使用中文语音合成系统，这是一个在 RISC-V 上运行的语音模型。"

```

## 附录A:常见问题
### 构建sharpa-onnx时缺少libtool
#### 问题背景
执行命令：
```bash
git clone https://github.com/k2-fsa/sherpa-onnx
cd sherpa-onnx
./build-riscv64-linu
```
输出：
```bash
+ command -v riscv64-unknown-linux-gnu-g++
+ '[' x = x ']'
+ dir=build-riscv64-linux-gnu
+ mkdir -p build-riscv64-linux-gnu
+ cd build-riscv64-linux-gnu
+ '[' '!' -f alsa-lib/src/.libs/libasound.so ']'
+ echo 'Start to cross-compile alsa-lib'
Start to cross-compile alsa-lib
+ '[' '!' -d alsa-lib ']'
+ git clone --depth 1 --branch v1.2.12 https://github.com/alsa-project/alsa-lib
Cloning into 'alsa-lib'...
remote: Enumerating objects: 441, done.
remote: Counting objects: 100% (441/441), done.
remote: Compressing objects: 100% (394/394), done.
remote: Total 441 (delta 92), reused 199 (delta 32), pack-reused 0 (from 0)
Receiving objects: 100% (441/441), 981.54 KiB | 1.91 MiB/s, done.
Resolving deltas: 100% (92/92), done.
Note: switching to '34422861f5549aee3e9df9fd8240d10b530d9abd'.
You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.
If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:
  git switch -c <new-branch-name>
Or undo this operation with:
  git switch -
Turn off this advice by setting config variable advice.detachedHead to false
+ pushd alsa-lib
~/work/sherpa-onnx/build-riscv64-linux-gnu/alsa-lib ~/work/sherpa-onnx/build-riscv64-linux-gnu
+ CC=riscv64-unknown-linux-gnu-gcc
+ ./gitcompile --host=riscv64-unknown-linux-gnu
./gitcompile: line 89: libtoolize: command not found
```
#### 解决方案
执行命令安装libtool:
```bash
sudo apt update
sudo apt install libtool
```


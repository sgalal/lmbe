# 开发环境配置

## 在 Windows 10 上配置开发环境

### 安装 QEMU

从 [QEMU 官网](https://www.qemu.org/) 下载最新的 QEMU 安装文件，此处下载的是 `qemu-w64-setup-20181128.exe`。

安装后打开 CMD，输入以下命令：

```text
> set PATH=%PATH%;C:\Program Files\NASM
> nasm --version
NASM version 2.14.02 compiled on Dec 26 2018
```

这样的输出说明安装成功。

### 安装 NASM

从 [NASM 官网](https://www.nasm.us/) 下载最新的 NASM 安装文件，此处下载的是 `nasm-2.14.02-installer-x64.exe`。

安装后打开 CMD，输入以下命令：

```text
> set PATH=%PATH%;C:\Program Files\qemu
> qemu-system-x86_64 --version
QEMU emulator version 3.0.93 (v3.1.0-rc3-11733-gdb066b4879-dirty)
Copyright (c) 2003-2018 Fabrice Bellard and the QEMU Project developers
```

这样的输出说明安装成功。

### 测试无限重启

`main.asm`：

```text
mov al,1
out 0x92,al
times 510-($-$$) db 0
db 0x55,0xaa
```

打开 CMD：

```text
> set PATH=%PATH%;C:\Program Files\NASM;C:\Program Files\qemu
> nasm -fbin main.asm -o main.bin -l main.lst
> qemu-system-x86_64 -soundhw all -rtc base=localtime -drive file=main.bin,format=raw,index=0,media=disk
```

如果无限重启，证明安装成功。

## 在 Linux 上配置开发环境

### 安装 QEMU

```bash
$ sudo pacman -S qemu
```

### 安装 NASM

```text
$ sudo pacman -S nasm
```


# 操作系统(H)实验一
### &emsp;&emsp;&emsp;&emsp;———追踪linux内核启动过程中的事件


-------------------


>本次操作系统实验的平台为Ubuntu 16.04 (LTS) ，实验内核为linux-4.15.14


### 主要系统工具：
需要的操作大致有编译内核、创建磁盘镜像及根目录、使用 gdb 远程调试等，使用如下工具：
 
- **QEMU emulator**  2.5.0
- **busybox**              1.28.2
- **GUN gdb**              7.11.1
- **GCC**                      5.4.0

----------------------
### 实验环境搭建：

**编译内核代码**
从https://www.kernel.org/ 下载linux内核源码到本地,并解压编译
```bash
xz -d ***.tar.xz                    #解压
tar -xvf ***.tar
make x86_64_defconfig               #使用当前x86_64平台默认的配置
make -j8 bzImage                    #编译源码，此处使用“-j8"参数以加快编译速度
make modules                        #编译在配置阶段选择的内核模块
```
**制作磁盘镜像**
>linux内核启动时必须已有根文件系统来供内核加载，同时也用于存放编译好的内核模块

**使用qemu-img 创建一个2G的磁盘镜像文件**
```bash
qemu-img create -f raw disk.raw 2G                 #disk.raw文件就相当于一块磁盘
mkfs -t ext4 ./disk.raw                            #使用ext4文件系统格式化虚拟盘
mkdir rootfs 
sudo mount -o loop ./disk.raw ./rootfs             #将磁盘镜像文件挂载到rootfs目录上
sudo make modules_install INSTALL_MODPATH=./rootfs #安装内核模块到磁盘镜像中
```


**准备init程序**

为了完整启动内核到用户态，需要准备一个1号进程(init进程)供内核启动
这里选用busybox作为init程序以及其他命令工具的提供方
下载busybox源码准备编译，这里使用busybox 1.28.2的最新版本
```bash
make defconfig                      #默认配置生效
make menuconfig                     #定制配置，因为 busybox 将被用作 init 程序，而且磁盘镜像中没有任何其它库，所以busybox 需要被静态编译成一个独立、无依赖的可执行文件，以免运行时发生链接错误。
make CONFIG_PREFIX=./rootfs install #安装busybox到镜像文件挂载目录
```
**启动qemu虚拟机**
```bash
qemu-system-x86_64 \
    -m 1024M       \                                 # 指定内存大小
    -smp 4         \                                 # 指定虚拟的 CPU 数量
    -kernel ./bzImage \                              #内核映像路径
    -drive format=raw,file=./disk.raw \              #挂载格式
    -append "init=/linuxrc root=/dev/sda nokaslr" \  #指定init程序为根目录下linuxrc，这是一个软链接
    -gdb tcp::1234 \                                 #启动gdb服务
    -S             \                                 #开始时冻结CPU
```
**gdb远程调试**
>另开一个终端启动gdb服务
```bash
file ~/OSH/linux-4.15.14/vmlinux   #加载符号表
target remote :1234                #链接虚拟机gdbserver
c                                  #运行系统以测试
```



----------
![source](https://github.com/OSH-2018/1-uniqueufo/blob/master/picture/1.png)




**观察到系统已经正常启动，可以进行后续调试分析工作**
----------
## 内核启动流程

>&emsp;&emsp;系统是从BIOS加电自检，载入MBR中的引导程序(LILO/GRUB),再加载linux内核开始运行的，一直到指定shell开始运行告一段落，这时用户开始操作Linux。而大致是在vmlinux的入口startup_32(head.S)中为pid号为0的原始进程设置了执行环境，然后原始进程开始执行start_kernel()完成Linux内核的初始化工作。

**阅读[linux启动协议](https://www.kernel.org/doc/Documentation/x86/boot.txt)可知x86 体系结构大内核内存使用如下:**


```bash
       ~                        ~   
        |  Protected-mode kernel |
100000  +------------------------+
        |  I/O memory hole       |   
0A0000  +------------------------+
        |  Reserved for BIOS     |      Leave as much as possible unused
        ~                        ~   
        |  Command line          |      (Can also be below the X+10000 mark)
X+10000 +------------------------+
        |  Stack/heap            |      For use by the kernel real-mode code.
X+08000 +------------------------+    
        |  Kernel setup          |      The kernel real-mode code.
        |  Kernel boot sector    |      The kernel legacy boot sector.
X       +------------------------+
        |  Boot loader           |      <- Boot sector entry point 0000:7C00
001000  +------------------------+
        |  Reserved for MBR/BIOS |
000800  +------------------------+
        |  Typically used by MBR |
000600  +------------------------+ 
        |  BIOS use only         |   
000000  +------------------------+
```

##### 在/arch/x86/boot/header.S汇编文件中(start_of_kernel)：
- 在对一段汇编代码（QWQ）进行一段编译链接后，会生成 512 字节的 bootsector
- GRUB 等 boot loader 将 setup.elf 读到 0x90000 处，将 vmlinux 读到 0x100000 处，然后跳转到 0x90200 开始执行x90200（_start）了，目的就是跳到 start_of_setup
- 复位硬盘控制器
- 如果 %ss 无效，重新计算栈指针
-    初始化栈，可使用中断
-    将 cs 设置为 ds，与 setup.elf 的入口地址一致
-    检查主引导扇区末尾标志，如果不正确则跳到 setup_bad
-   清空 bss 段
-   跳转到 main函数（定义在 boot/main.c）


>calll   main


##### 在 arch/x86/boot/main.c 中：


- 程序首先将 header 拷贝到内核参数块,并初始化了堆：static void copy_boot_params(void)
然后对硬件进行了各种检测和设置,最后调用了 go_to_protected_mode()，代码在 boot/pm.c。
#####在 arch/x86/boot/pm.c 中：

```cpp
    realmode_switch_hook()                      //在realmode_switch_hook() 中禁用了中断
    reset_coprecessor()                         //重启协处理器
    make_all_interrupts()                       //关闭所有旧 PIC 上的中断。其中的 io_delay 等待 I/O 操作完成。
    setup_idt()                                 //初始化中断描述符表 (空的)
    setup_gdt()                                 //初始化 GDT:
        GDT_ENTRY_BOOT_CS
        GDT_ENTRY_BOOT_DS
        GDT_ENTRY_BOOT_TSS
    protected_mode_jump(boot_params.hdr.code32_start,(u32)&boot_params + (ds() << 4)); //位于/arch/x86/boot/pmjump.S 
```
- 在/pmjump.S文件最后程序跳转至自解压内核过程
- 真正的内核入口是 arch/x86/kernel/head_32.S
汇编函数 startup_32 依次完成以下动作：
  - 初始化参数
        - 初始化 GDT。
        - 清空 BSS 段
        - 复制实模式中的 boot_params 结构体
        - 复制命令行参数到 boot_command_line (供 init/main.c 使用)
        - 有关虚拟环境的一些配置
   - 开启分页机制 
   - 初始化 Eflags

   - 初始化中断向量表
   - 检查处理器类型
   - 载入GDT、IDT
   - 跳转到 i386_start_kernel



- 在pmjump.S中一系列汇编码的操作后,内核跳转到 start_kernel.
>jmp i386_start_kernel


其中 i386_start_kernel 定义在 arch/x86/kernel/head32.c :
```cpp
asmlinkage __visible void __init i386_start_kernel(void)
{
      ...

      start_kernel();
}

```
![source](https://github.com/OSH-2018/1-uniqueufo/blob/master/picture/3.png)







**此时系统状态**



##### 在/linux-4.15.14/init/main.c 中 

start_kernel()中调用了一系列初始化函数，以完成kernel本身的设置。
- 设置与体系结构相关的环境
- 页表结构初始化 
- 核心进程调度器初始化
- 时间、定时器初始化
- 提取并分析核心启动参数
- 控制台初始化
- 剖析器数据结构初始化
- 核心Cache初始化
- 延迟校准
- 内存初始化
- 创建和设置内部及通用cache
- 创建文件cache
- 创建目录cache
- 创建页cache
- 创建信号队列cache
- 初始化内存inode表
- 创建内存文件描述符表

**在start_kernel函数执行的最后一个函数调用rest_init（）时，Linux系统开始有了一个进程，此进程pid为0**
![source](https://github.com/OSH-2018/1-uniqueufo/blob/master/picture/2.png)


##### 启动init过程

- 总线初始化
- 网络初始化
- 创建bdflush核心线程
- 创建kupdate核心线程
- 设置并启动核心调页线程kswapd
- 创建事件管理核心线程
- 设备初始化
- 执行文件格式设置
- 启动任何使用__initcall标识的函数
- 文件系统初始化
- 安装root文件系统

##### 在调试时发现了
>kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);

　　这个pid为1的init进程，它继续完成剩下的初始化工作，然后execve(/sbin/init), 成为系统中的其他所有进程的祖先。总而言之，系统启动后首先执行一系列的初始化工作，直到start_kernel处，它是代码的入口点，相当于main.c函数。然后启动系统的第一个进程init，init是所有进程的父进程，由init再启动子进程，从而使得系统成功运行起来。

--------------------------------------
## 总结

**x86架构的Linux内核启动过程可以分为一下几步**
-（1）实模式的入口函数_start()：在header.S中，这里会进入main函数，它拷贝bootloader的各个参数，执行基本硬件设置，解析命令行参数。
-    （2）保护模式的入口函数startup_32()：在compressed/header_32.S中，这里会解压bzImage内核映像，加载vmlinux内核文件。
-    （3）内核入口函数startup_32()：在kernel/header_32.S中，这就是所谓的进程0，它会进入start_kernel()函数，即Linux内核启动函数。start_kernel()会做大量的内核初始化操作，解析内核启动的命令行参数，并启动一个内核线程来完成内核模块初始化的过程，然后进入空闲循环。
-    （4）内核模块初始化的入口函数kernel_init()：在init/main.c中，这里会启动内核模块、创建基于内存的rootfs、加载initramfs文件或cpio-initrd，并启动一个内核线程来运行其中的/init脚本，完成真正根文件系统的挂载。
-   （5）根文件系统挂载脚本/init：这里会挂载根文件系统、运行/sbin/init，从而启动众所周知的进程1。本次实验使用busybox制作根文件系统供init挂载
-    （6）init进程的系统初始化过程：执行相关脚本，以完成系统初始化，如设置键盘、字体，装载模块，设置网络等，最后运行登录程序，出现登录界面。

   



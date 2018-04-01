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
![source](https://github.com/uniqueufo/OSH_test/blob/master/1.png)
----------
### 内核启动流程


>&emsp;&emsp;系统是从BIOS加电自检，载入MBR中的引导程序(LILO/GRUB),再加载linux内核开始运行的，一直到指定shell开始运行告一段落，这时用户开始操作Linux。而大致是在vmlinux的入口startup_32(head.S)中为pid号为0的原始进程设置了执行环境，然后原始进程开始执行start_kernel()完成Linux内核的初始化工作。


##### 在/arch/x86/boot/header.S汇编文件中(start_of_kernel)：


- 复位硬盘控制器
- 如果 %ss 无效，重新计算栈指针
-    初始化栈，开中断
-    将 cs 设置为 ds，与 setup.elf 的入口地址一致
-    检查主引导扇区末尾标志，如果不正确则跳到 setup_bad
-   清空 bss 段
-  跳到 main（定义在 boot/main.c）


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



##### 启动init过程

- 总线初始化
- 网络初始化
-    创建bdflush核心线程
-    创建kupdate核心线程
-    设置并启动核心调页线程kswapd
-    创建事件管理核心线程
-    设备初始化
-    执行文件格式设置
-    启动任何使用__initcall标识的函数
-    文件系统初始化
-   安装root文件系统


--------------------------------------
#####在调试时发现了
>kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);

　　这个pid为1的init进程，它继续完成剩下的初始化工作，然后execve(/sbin/init), 成为系统中的其他所有进程的祖先。总而言之，系统启动后首先执行一系列的初始化工作，直到start_kernel处，它是代码的入口点，相当于main.c函数。然后启动系统的第一个进程init，init是所有进程的父进程，由init再启动子进程，从而使得系统成功运行起来。



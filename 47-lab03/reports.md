# 实验三 操作系统的引导

## 1. 有时，继承传统意味着蹩手蹩脚。`x86`计算机为了向下兼容， 导致启动过程比较复杂，请找出`x86`计算机启动过程中，被硬件强制，软件必须遵守的两个“多此一举”的步骤（多找几个也无妨），说说它们为什么多此一举，并设计更简洁的替代方案。

### 1） **BIOS 启动过程中的 POST（Power-On Self-Test）**

#### 步骤概述：

当计算机开机时，`BIOS` 会执行一个自检程序（POST），检查硬件是否正常（如内存、硬盘、显卡等）。这个过程通常会花费几秒钟，且会有一些冗余操作，例如检查不再使用的硬件（如过时的串口、并口等），或者重复执行某些检查。

#### 为什么多此一举：

- 现代的硬件通常都在开机前就已经过了制造商的测试，POST 的许多检查其实没有什么实际意义。
- 一些较旧的硬件（如串口、并口接口）已经被淘汰，但 `BIOS` 仍然会进行检查，即便这些接口从未被启用。

#### 设计替代方案：

1. **硬件自检可以只针对必要的硬件进行**：现代系统可以通过内置的硬件管理控制器（如 BMC）来完成硬件自检，而不是通过传统的 `BIOS` 来完成。这样可以减少启动时间。
2. **精简的 UEFI 启动过程**：使用 `UEFI`（统一可扩展固件接口）替代传统的 `BIOS`，可以在启动过程中减少不必要的硬件检查。`UEFI` 允许用户定义具体的硬件检测方案，只检查系统当前所需的硬件。

### 2） **引导加载程序（Bootloader）加载操作系统内核**

#### 步骤概述：

在启动过程中，`BIOS` 或 `UEFI` 会将控制权交给引导加载程序（如 `GRUB`），然后引导加载程序再加载操作系统内核。这意味着操作系统的内核必须经过引导加载程序来加载和配置，而不是直接由 `BIOS` 或 `UEFI` 直接加载。

#### 为什么多此一举：

- 现代操作系统的引导过程通常需要 `Bootloader`，这个额外的中介步骤增加了启动的复杂性。比如，在一个非常简化的系统中，操作系统内核本可以直接由 `UEFI` 或 `BIOS` 加载，跳过引导加载程序的中间步骤。
- `Bootloader` 还会做很多额外的工作，比如允许用户选择不同的启动项（例如，选择不同的操作系统或内核）。然而，现代操作系统往往只需要加载一个固定的内核，因此这个中介过程显得不必要。

#### 设计替代方案：

1. **UEFI 直接引导操作系统**：现代的 `UEFI` 已经可以直接加载操作系统内核，而无需引导加载程序。只需要配置一个简单的启动项，`UEFI` 就可以直接加载并运行操作系统内核。这种方法去掉了多余的引导程序，使启动过程更加简洁。

   例如，使用 `EFI stub`，将内核编译为一个可以直接由 `UEFI` 启动的可执行文件。这样，`UEFI` 就可以直接加载操作系统内核，而不需要额外的 `GRUB` 等引导加载程序。

2. **内核自引导**：类似于 `UEFI` 的方式，现代操作系统可以设计为自包含式内核，内核本身能够管理引导过程中的所有步骤，而无需额外的引导加载程序。例如，Linux 内核可以在启动时直接读取内存中的配置文件，并通过 `UEFI` 或其他机制直接启动。

## 2. 改写`bootsect.s`完成在屏幕上打印`xxx is booting...`

修改`bootsect.s`如下：

修改msg1中的字符串为想要的打印结果，然后计算一共几个字符，例如`zigo is booting...`是18个字符，再加上前后3对回车换行符，一共是24个字符，修改`mov cx #24`为对应的字符长度即可。

```
entry _start
_start:
    mov ah,#0x03
    xor bh,bh
    int 0x10
    mov cx,#24
    mov bx,#0x0007
    mov bp,#msg1
    mov ax,#0x07c0
    mov es,ax
    mov ax,#0x1301
    int 0x10
inf_loop:
    jmp inf_loop
msg1:
    .byte   13,10
    .ascii "zigo is booting..."
    .byte   13,10,13,10
.org 510
boot_flag:
    .word   0xAA55
```

然后使用以下几行命令行进行编译和运行，前两行是编译和链接`bootsect.s`，第三行是删除`bootsect.s`文件首部的32字节，接下来完成将生成的Image文件拷贝出来再运行。

```
as86 -0 -a -o bootsect.o bootsect.s
ld86 -0 -s -o bootsect bootsect.o
dd bs=1 if=bootsect of=Image skip=32
cp Image ../
../../run
```

运行结果如下：

![](images\1.png)

## 3.  完成`bootsect.s`读入`setup.s`，并且`setup.s`向屏幕输出一行`Now we are in SETUP`

修改`bootsect.s`，增加如下代码

```
load_setup:
    mov    dx,#0x0000        ! 设置驱动器和磁头(drive 0, head 0): 软盘0磁头
    mov    cx,#0x0002        ! 设置扇区号和磁道(sector 2, track 0):0磁头、0磁道、2扇区
    mov    bx,#0x0200        ! 设置读入的内存地址：BOOTSEG+address = 512，偏移512字节
    mov    ax,#0x0200+SETUPLEN    ! 设置读入的扇区个数(service 2, nr of sectors)，
                    ! SETUPLEN是读入的扇区个数，Linux 0.11设置的是4，
                    ! 我们不需要那么多，我们设置为1
    int    0x13            ! 应用0x13号BIOS中断读入1个setup.s扇区
    jnc    ok_load_setup        ! 读入成功，跳转到ok_load_setup: ok - continue
    mov    dx,#0x0000        ! 软驱、软盘有问题才会执行到这里
    mov    ax,#0x0000        ! 否则复位软驱
    int    0x13
    j    load_setup        ! 重新循环，再次尝试读取

ok_load_setup:
    jmpi    0,SETUPSEG
```

修改`setup.s`,使其输出`NOW we are in SETUP`

```
entry _start            
_start:
    mov ah,#0x03        
    xor bh,bh           
    int 0x10
    mov cx,#25          
    mov bx,#0x0007      
    mov bp,#msg2        
    mov ax,cs           
    mov es,ax           
    mov ax,#0x1301      
    int 0x10                
inf_loop:               
    jmp inf_loop
msg2:                                           
    .byte   13,10                               
    .ascii  "Now we are in setup"               
    .byte   13,10,13,10
.org 510                                        
boot_flag:
    .word   0xAA55
```

重新编译连接后，命令行输入`make BootImage，`发现报错

![](images\2.png)

这是因为 `make` 根据 `Makefile` 的指引执行了 `tools/build.c` ， 它是为生成整个内核的镜像文件而设计的，没考虑我们只需要 `bootsect.s` 和 `setup.s` 的情况。 它在向我们要“系统”的核心代码。为完成实验，接下来给它打个小补丁。修改`build.c`，将其中对于system处理的部分直接注释掉：

```
//	if ((id=open(argv[3],O_RDONLY,0))<0)
//		die("Unable to open 'system'");
//	if (read(id,buf,GCC_HEADER) != GCC_HEADER)
//		die("Unable to read header of 'system'");
//	if (((long *) buf)[5] != 0)
//		die("Non-GCC header of 'system'");
//	for (i=0 ; (c=read(id,buf,sizeof buf))>0 ; i+=c )
//		if (write(1,buf,c)!=c)
//			die("Write call failed");
//	close(id);
//	fprintf(stderr,"System is %d bytes.\n",i);
//	if (i > SYS_SIZE*16)
//		die("System is too big");
```

再次`make BootImage`成功得到运行结果

![](images\3.png)

然后运行得到输出：

![](images\4.png)

## 4. 修改`setup.s`获取基本硬件参数并输出

以下是包含详细注释的`setup.s`

```
INITSEG = 0x9000          ; 初始化段地址

entry _start              ; 定义入口点，程序开始执行的位置
_start:
    ; 获取当前光标位置，以便将字符串打印到当前光标处
    mov    ah, #0x03      ; 功能号 0x03：获取当前光标位置
    xor    bh, bh         ; bh = 0，获取当前光标位置的页号
    int    0x10           ; 调用 BIOS 中断 0x10 进行操作

    ; 使用 BIOS 中断 0x10 的 0x13 功能打印字符串 "Now we are in SETUP."
    mov    cx, #26        ; 字符串长度为 26
    mov    bx, #0x0007    ; 设置显示属性，字体颜色为白色，背景为黑色
    mov    bp, #msg1      ; 指向要显示的字符串的地址
    mov    ax, cs         ; 将代码段地址加载到 ax
    mov    es, ax         ; 设置 ES 为当前代码段
    mov    ax, #0x1301    ; BIOS 中断 0x10，功能号 0x13 用于打印字符串
    int    0x10           ; 调用 BIOS 中断，打印字符串

    ; 读取一些硬件参数

    ; 读取光标位置并保存到内存地址 0x90000 处
    mov    ax, #INITSEG   ; 初始化数据段为 0x9000
    mov    ds, ax         ; 设置数据段寄存器为 0x9000
    mov    ah, #0x03      ; 功能号 0x03 获取光标位置
    xor    bh, bh         ; bh = 0，表示页号
    int    0x10           ; 调用 BIOS 中断获取光标位置
    mov    [0], ds        ; 将光标位置存储到 0x90000 处

    ; 读取扩展内存的大小，并保存到 0x90002 处
    mov    ah, #0x88      ; 功能号 0x88 用于读取扩展内存大小
    int    0x15           ; 调用 BIOS 中断 0x15 获取内存大小
    mov    [2], ax        ; 将内存大小存储到 0x90002 处

    ; 从 0x41 处拷贝 16 个字节（磁盘参数表）到 0x90004 处
    mov    ax, #0x0000    ; 设置数据段为 0x0000
    mov    ds, ax         ; 设置数据段寄存器为 0x0000
    lds    si, [4*0x41]   ; 加载磁盘参数表地址到 SI
    mov    ax, #INITSEG   ; 将初始化段 0x9000 载入 AX
    mov    es, ax         ; 设置 ES 为 0x9000
    mov    di, #0x0004    ; 设置目的地址为 0x90004
    mov    cx, #0x10      ; 要复制 16 字节
    rep                   ; 重复执行
    movsb                 ; 将字节从 DS:SI 复制到 ES:DI

    ; 开始打印硬件参数

    ; 获取当前光标位置
    mov    ah, #0x03      ; 获取光标位置
    xor    bh, bh         ; bh = 0，页号
    int    0x10           ; 调用 BIOS 中断 0x10 获取光标位置

    ; 打印 "Cursor POS:"
    mov    cx, #11        ; 字符串长度
    mov    bx, #0x0007    ; 显示属性
    mov    ax, cs         ; 设置当前代码段
    mov    es, ax         ; 设置 ES 为当前代码段
    mov    bp, #msg2      ; 指向要显示的字符串
    mov    ax, #0x1301    ; 打印字符串功能
    int    0x10           ; 调用 BIOS 中断打印字符串

    ; 打印光标位置
    mov    ax, #0x9000    ; 数据段设置为 0x9000
    mov    ds, ax         ; 设置数据段寄存器
    mov    dx, 0x0        ; 光标位置存储在 0x90000 处，读取并打印
    call    print_hex     ; 调用打印十六进制数的函数
    call    print_nl      ; 打印换行

    ; 打印内存大小
    mov    ah, #0x03      ; 获取光标位置
    xor    bh, bh         ; bh = 0，页号
    int    0x10           ; 调用 BIOS 中断 0x10 获取光标位置

    ; 打印 "Memory SIZE:"
    mov    cx, #12        ; 字符串长度
    mov    bx, #0x0007    ; 设置显示属性
    mov    ax, cs         ; 设置代码段
    mov    es, ax         ; 设置 ES 为代码段
    mov    bp, #msg3      ; 指向字符串 "Memory SIZE:"
    mov    ax, #0x1301    ; 打印字符串功能
    int    0x10           ; 调用 BIOS 中断打印字符串

    ; 打印内存大小
    mov    ax, #0x9000    ; 数据段设置为 0x9000
    mov    ds, ax         ; 设置数据段寄存器
    mov    dx, 0x2        ; 内存大小存储在 0x90002 处
    call    print_hex     ; 打印内存大小（十六进制）
    call    print_nl      ; 打印换行

    ; 打印 "KB"
    mov    ah, #0x03      ; 获取光标位置
    xor    bh, bh         ; bh = 0，页号
    int    0x10           ; 调用 BIOS 中断 0x10 获取光标位置

    mov    cx, #2         ; "KB" 字符串长度为 2
    mov    bx, #0x0007    ; 设置显示属性
    mov    ax, cs         ; 设置代码段
    mov    es, ax         ; 设置 ES 为代码段
    mov    bp, #msg4      ; 指向字符串 "KB"
    mov    ax, #0x1301    ; 打印字符串功能
    int    0x10           ; 调用 BIOS 中断打印字符串
    call    print_nl      ; 打印换行

    ; 打印 "Cyls"（柱面数）
    mov    ah, #0x03      ; 获取光标位置
    xor    bh, bh         ; bh = 0，页号
    int    0x10           ; 调用 BIOS 中断 0x10 获取光标位置

    mov    cx, #5         ; "Cyls" 字符串长度为 5
    mov    bx, #0x0007    ; 设置显示属性
    mov    ax, cs         ; 设置代码段
    mov    es, ax         ; 设置 ES 为代码段
    mov    bp, #msg5      ; 指向字符串 "Cyls"
    mov    ax, #0x1301    ; 打印字符串功能
    int    0x10           ; 调用 BIOS 中断打印字符串

    ; 打印柱面数
    mov    ax, #0x9000    ; 数据段设置为 0x9000
    mov    ds, ax         ; 设置数据段寄存器
    mov    dx, 0x4        ; 柱面数存储在 0x90004 处
    call    print_hex     ; 打印柱面数（十六进制）
    call    print_nl      ; 打印换行

    ; 打印 "Heads"（磁头数）
    mov    ah, #0x03      ; 获取光标位置
    xor    bh, bh         ; bh = 0，页号
    int    0x10           ; 调用 BIOS 中断 0x10 获取光标位置

    mov    cx, #6         ; "Heads" 字符串长度为 6
    mov    bx, #0x0007    ; 设置显示属性
    mov    ax, cs         ; 设置代码段
    mov    es, ax         ; 设置 ES 为代码段
    mov    bp, #msg6      ; 指向字符串 "Heads"
    mov    ax, #0x1301    ; 打印字符串功能
    int    0x10           ; 调用 BIOS 中断打印字符串

    ; 打印磁头数
    mov    ax, #0x9000    ; 数据段设置为 0x9000
    mov    ds, ax         ; 设置数据段寄存器
    mov    dx, 0x6        ; 磁头数存储在 0x90006 处
    call    print_hex     ; 打印磁头数（十六进制）
    call    print_nl      ; 打印换行

    ; 打印 "sectors"（每磁道扇区数）
    mov    ah, #0x03      ; 获取光标位置
    xor    bh, bh         ; bh = 0，页号
    int    0x10           ; 调用 BIOS 中断 0x10 获取光标位置

    mov    cx, #8         ; "sectors" 字符串长度为 8
    mov    bx, #0x0007    ; 设置显示属性
    mov    ax, cs         ; 设置代码段
    mov    es, ax         ; 设置 ES 为代码段
    mov    bp, #msg7      ; 指向字符串 "Sectors"
    mov    ax, #0x1301    ; 打印字符串功能
    int    0x10           ; 调用 BIOS 中断打印字符串

    ; 打印每磁道的扇区数
    mov    ax, #0x9000    ; 数据段设置为 0x9000
    mov    ds, ax         ; 设置数据段寄存器
    mov    dx, 0x12       ; 每磁道的扇区数存储在 0x90012 处
    call    print_hex     ; 打印扇区数（十六进制）
    call    print_nl      ; 打印换行

Inf_loop:
    jmp Inf_loop          ; 无限循环，程序结束

; print_hex函数：将一个数字转换为 ASCII 码字符，并打印到屏幕上
; 参数值：dx
; 返回值：无
print_hex:
    mov    cx, #4         ; 要打印4个十六进制数字，循环4次
print_digit:
    rol    dx, #4         ; 循环移位，取 dx 的高 4 位到低 4 位
    mov    ax, #0xe0f     ; 掩码，将低 4 位提取出来
    and    al, dl         ; 获取低 4 位
    add    al, #0x30      ; 转换为 ASCII 数字
    cmp    al, #0x3a      ; 判断是否为大于 '9' 的字符
    jl     outp           ; 如果是数字0-9，跳过
    add    al, #0x07      ; 对于字母 a-f，需要加 7
outp:
    int    0x10           ; 调用 BIOS 中断打印字符
    loop    print_digit   ; 循环4次，打印所有数字
    ret                   ; 返回

; 打印回车换行
print_nl:
    mov    ax, #0xe0d     ; 发送回车字符
    int    0x10           ; 调用 BIOS 中断
    mov    al, #0xa       ; 发送换行字符
    int    0x10           ; 调用 BIOS 中断
    ret                   ; 返回

msg1:
    .byte 13,10           ; 回车换行
    .ascii "Now we are in SETUP."   ; 输出字符串
    .byte 13,10,13,10     ; 回车换行

msg2:
    .ascii "Cursor POS:"  ; "Cursor POS:" 字符串

msg3:    
    .ascii "Memory SIZE:" ; "Memory SIZE:" 字符串

msg4:
    .ascii "KB"           ; "KB" 字符串

msg5:    
    .ascii "Cyls:"        ; "Cyls" 字符串

msg6:
    .ascii "Heads:"       ; "Heads" 字符串

msg7:
    .ascii "Sectors:"     ; "Sectors" 字符串

.org 510
boot_flag:
    .word 0xAA55          ; 启动标志，0xAA55 是标准的 MBR 引导扇区标识

```

重新`make BootImage`后运行得到结果

![](images\5.png)
# 操作系统实验报告: ucore Lab 1

[toc]

## 实验要求

* 阅读 uCore 实验项目开始文档 (uCore Lab 0)，准备实验平台，熟悉实验工具。

* uCore Lab 1：系统软件启动过程
    * (1) 编译运行 uCore Lab 1 的工程代码；
    * (2) 完成 uCore Lab 1 练习 1-4 的实验报告；
    * (3) 尝试实现 uCore Lab 1 练习 5-6 的编程作业；
    * (4) 思考如何实现 uCore Lab 1 扩展练习 1-2。

## 实验环境

* 系统：Ubuntu 20.04（虚拟机）
* 汇编器：gas (GNU Assembler) in AT&T mode
* 编译器：gcc

## 实验内容

### 编译运行uCore Lab 1 工程代码

在lab1所在目录下，打开终端并执行如下指令`$ make`

如果程序执行结果如下图所示
![](noTarget.jpg)

产生上述情况说明**文件已经进行过编译且编译后没有文件更新**，只需在终端执行指令`$ make clean`后，再执行`$ make`即可。正确结果如下图所示
![](make_1.jpg)

### 练习 1: 理解通过 make 生成执行文件的过程

#### 前置知识：makefile基本知识

makefile文件描述了整个工程的编译规则，只需执行make指令即可自动编译所有代码

##### makefile基本语法规则：

```makefile
target...:prerequisites...
commend...
```

target表示目标文件，即编译、链接产生的文件，perrequisite表示依赖文件，commend表示make将要执行的指令，可以是任意终端指令。
上述格式第一行，实际上描述了一种依赖关系，这种依赖关系实际上就是描述了目标文件是由哪些文件生成的，换言之，目标文件是哪些文件更新的。定义号依赖关系后，后续行描述了make实际执行的指令，注意，**所有指令都必须以TAB(制表符)开头**。make并不关心这些commend是怎么实现的，它只负责执行，执行时，make会**比较targets文件和prerequisites文件的修改日期**，如果prerequisites文件的日期要比targets文件的日期要新，或者target不存在，那么make就会执行后续的commend。

##### make执行过程

假设我们以默认方式，也就是直接在终端中输入`$ make`，make的执行流程如下：
* 首先找到当前目录下的"makefile"或"Makefile"文件
* 在上一步正确执行的前提下，找到文件中的第一个目标文件，并把这个文件作为最终的目标文件。
* 如果该目标文件不存在，或该目标文件所依赖的文件的修改时间要比目标文件晚，那么，他就会执行后面所定义的commend来试图生成该目标文件。
* 在上一步中，目标文件所依赖的文件可能不存在或需要更新，在这种情况下，make会向下寻找该依赖文件的依赖关系，并执行后面定义的commend来试图生成该文件
* 上述过程反复迭代，直到第一个目标文件生成，从而完成所有编译

> 在上文"编译运行lab1工程代码"中，运行`$ make`如果终端出现语句"make: Nothing to be done for 'TARGETS'."，是因为make在执行第一步时就找到了目标文件，并且其依赖的文件并没有发生更新，因此打印语句提示用户不需要执行make，工程已经编译完毕了。

##### makefile中的变量

1.变量的使用可以更便捷的更新makefile文件。举个例子，如果不使用变量，当工程中需要新加入几个[.c]文件时，需要在所有对应的依赖中添加该文件；而如果有如下语句`cfile = main.c command.c display.c`来整合所有[.c]文件，更新时只需要更改变量的定义即可。makefile中的变量定义有点类似于C/C++中的宏定义。变量在使用时需要在前面跟一个$符号，并用"()"括住。实际举例如下
```makefile
cfile = main.c commend.c display.c
main.o:$(cfile)
    commend...
```
2.makefile中有几个常用的变量：`$@`表示目标文件，`$^`表示所有的依赖文件，`$<`表示第一个依赖文件。这些变量命名是规定好的

3.makefile允许用变量来为变量赋值，并且在使用"="进行赋值时，可以使用下方定义的变量
```makefile
cfile = $(later)
later = main.c commend.c
```
但这样的赋值方式可能引发错误的递归，导致报错
```
cfile = main.c
cfile = $(cfile) -o
```
上述语句会导致cfile在下文无限次展开，make会检测这一情况并报错。如果想对变量进行如上递归定义，可以使用"**:=**"来为变量进行赋值。使用":="赋值将不会使用下文定义的变量的值来为当前变量赋值，这样也就不会发生上文的递归错误。如果有如下语句
```makefile
cfile := main.c
cfilex := $(cfile) commend.c
cfile := displasy.c
```
最终cfilex的值为main.c commend.c，而非display.c commend.c。":="还具有类似于python中的切片操作，比如有如下语句
```makefile
cfile := main.c commend.c
ofile := $(cfile:.c=.o)
```
上述语句在为ofile赋值时，将cfile定义的所有以[.c]结尾的文件的后缀更改为[.o]，因此ofile最终值为main.o commend.o，上述使用方法比较高级

在掌握上述前置基本知识后，可以初步读懂makefile文件中的代码。

#### 练习1-1 ucore.img的生成

##### ucore.img（操作系统镜像文件）

操作系统镜像文件是一种包含操作系统文件系统的完整副本，它可以被复制到存储介质（如硬盘、光盘、U盘等）上，然后用于安装或启动操作系统。操作系统镜像文件包含了操作系统内核、驱动程序、库文件、配置文件等所有必需的文件和数据。它的作用是提供一个完整的操作系统环境，使用户可以在新的计算机或已有计算机的不同存储介质上安装或运行操作系统。

找到makefile文件中写有注释"*# create ucore.img*"的部分，该部分代码如下
```makefile
UCOREIMG := $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

首先是一条变量赋值语句，`UCOREIMG`被赋值为调用`call`对"totarget"与"ucore,img"进行操作，其中的totarget定义在该目录下的`tools/function.mk`中，其赋值语句为`totarget = $(addprefix $(BINDIR)$(SLASH),$(1))`，addprefix代表在前面加上，`$(BINDIR)`代表"bin"，`$(SLASH)`代表"/"，因此totarget,ucore.img的结果就是"bin/ucore.img"。

随后描述UCOREIMG的依赖关系：UCOREIMG的生成依赖于kernel与bootblock。

随后描述进行的指令。`$(V)dd if=/dev/zero of=$@ count=10000`表示创建一个包含10000个大小为512字节的块并用空字符填充的文件，分配给UCOREIMG；`$(V)dd if=$(bootblock) of=$@ conv=notrunc`表示读取bootblock并将数据输出到目标文件，即UCOREIMG中，且不截断输出文件；`$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc`表示读取kernel文件并将数据据输出到目标文件，即UCOREIMG中，且不截断输出文件，输出时跳过1个块再继续复制

`dd`指令用法如下

> **`dd`指令**（*参考[链接](https://www.runoob.com/linux/linux-comm-dd.html)*）：`dd`可从标准输入或文件中读取数据，根据指定的格式来转换数据，再输出到文件、设备或标准输出。它提供的参数，或者说选项有：
    1.if=文件名：输入文件名，默认为标准输入。即指定源文件。
    2.of=文件名：输出文件名，默认为标准输出。即指定目的文件。
    3.ibs=bytes：一次读入bytes个字节，即指定一个块大小为bytes个字节。
    4.obs=bytes：一次输出bytes个字节，即指定一个块大小为bytes个字节。
    5.bs=bytes：同时设置读入/输出的块大小为bytes个字节。
    6.cbs=bytes：一次转换bytes个字节，即指定转换缓冲区大小。
    7.skip=blocks：从输入文件开头跳过blocks个块后再开始复制。
    8.seek=blocks：从输出文件开头跳过blocks个块后再开始复制。
    9.count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数。 
    10.conv=<关键字>，关键字可以有以下11种：
        conversion：用指定的参数转换文件。
        ascii：转换ebcdic为ascii
        ebcdic：转换ascii为ebcdic
        ibm：转换ascii为alternate ebcdic
        block：把每一行转换为长度为cbs，不足部分用空格填充
        unblock：使每一行的长度都为cbs，不足部分用空格填充
        lcase：把大写字符转换为小写字符
        ucase：把小写字符转换为大写字符
        swap：交换输入的每对字节
        noerror：出错时不停止
        notrunc：不截断输出文件
        sync：将每个输入块填充到ibs个字节，不足部分用空（NUL）字符补齐。

根据上文，可以得到命令各个部分的具体含义

##### kernel（内核）

内核是操作系统的核心组成部分，它是操作系统的基本管理者，负责处理计算机硬件与软件之间的交互和通信，以及管理计算机资源的分配和调度。内核通常是操作系统中最底层、最核心的部分，直接运行在计算机的硬件之上。内核的作用是为操作系统提供基本的服务和管理，包括处理器管理、内存管理、文件系统管理、进程管理、设备驱动程序管理等。具体来说，内核需要完成以下主要任务：

* 内存管理：分配和释放内存、虚拟内存管理、页面置换等
* 文件系统管理：对文件和目录的访问、文件读写、文件保护、文件系统安全等
* 进程管理：进程调度、进程通信、进程同步等
* 设备驱动程序管理：安装和卸载驱动程序、设备访问控制等。

根据依赖关系，下面查看kernel文件的生成过程。找到注释为"*# create kernel target*"的部分，该部分代码如下所示
```makefile
KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))

KOBJS	= $(call read_packet,kernel libs)

kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```
首先是三条变量赋值语句，使用"+="为变量KINCLUDE、KSRCDIR与KCFLAGS赋值，"+="表示在该变量已有的定义之上再增加定义。反斜杠表示换行，能够使代码更加规范。KINCLUDE与KSRCDIR语句的执行效果是为将kern目录的前缀定义为kinclude和ksrcdir，KFLAGS语句为kinclude前缀添加"-I"选项，提供交互模式

调用call函数过程，该语句的执行时包含许多嵌套，效果为生成kernel目录下的所有obj文件

之后为两条变量赋值语句，调用call函数链接read_packet和kernal libs给KOBJS；调用call函数生成kernel文件，totarget定义在function.mk中，作用为添加前缀"bin/"

而后是依赖关系描述，kernel的生成依赖于tools与kernel.ld链接配置文件，还依赖与KOBJS的生成。

而后给出一系列指令，首先链接目标文件并在屏幕打印，然后链接指定文件夹下所有obj文件，生成kernel。`-T`表明指定链接器脚本，`LD`变量代表ld指令，`LDFLAGS`应为检查连接器版本是否符合预期。随后使用`objdump`指令对kernel进行反汇编，`-S`参数表示C代码与汇编代码交替显示，并解析kernel以获得符号表

最后生成kernel并返回

注意，指令前的`@`符号表示该行命令不在终端显示，举个例子，如果在makefile文件中有如下语句
```makefile
    @echo Compiling...
```
则在编译过程中，终端只会打印"Compiling..."但不会打印"echo Compiling"

在终端执行`make "V="`来查看实际编译指令，并找到生成kernel的部分，如下图所示
![](ldKernel.jpg)

根据上图可以看到kernel生成过程中依赖的所有obj文件

##### bootblock（引导扇区）

引导扇区是操作系统启动时的重要模块，当计算机启动时，BIOS会读取硬盘的第一个扇区（即主引导扇区），并将该扇区的数据加载到内存中。主引导扇区中的引导程序（bootloader）就是这个阶段执行的代码，它会进一步读取并加载操作系统的核心部分。

在一些操作系统（如FreeBSD和NetBSD）中，引导程序（bootloader）通常也被称为bootblock。在操作系统的启动过程中，bootblock通常负责初始化硬件设备，加载核心部分，并转交控制权给核心部分，让它继续执行操作系统的初始化过程。

> BIOS：BIOS是计算机系统的基本输入输出系统（Basic Input/Output System）的缩写，它是一组存储在计算机主板上的固件程序。BIOS作为计算机系统的基础，负责控制和管理计算机系统的硬件设备，例如启动过程、键盘、硬盘、显示器、USB等。

找到注释为"*# create bootblock*"的部分，该部分代码如下所示

```makefile
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```

首先时一条变量赋值语句，调用call函数，用listf_cc过滤目录下的.c与.S文件，并用boot替换listf_cc中的变量，将返回值赋值给bootfiles

随后调用函数foreach编译bootfiles中的所有obj文件

随后的变量赋值语句，为bootblock添加前缀"bin/"

然后描述文件依赖关系，调用call函数，使用toobj为bootfiles中的所有文件添加前缀"obj/"并更改后缀为".o"；使用totarget为sign添加前缀"bin/"，最后整条语句表示bootblock的生成依赖"obj/.o"与"bin/sign"的生成

随后为一连串指令，基本过程与上文介绍的部分很相似，能够阅读，这里不作详细解释

可以在终端执行指令`make "V="`来查看bootblock具体的生成过程，如下图所示
![](ldBootblock.jpg)

可以看到目标生成过程中依赖的所有文件。

##### sign

找到注释为"*# create 'sign' tools*"的部分，代码如下所示

```makefile
# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

上述代码中包含许多嵌套定义，需要逐层拆解进行阅读。

首先找到`add_files_host`的定义，定义位置在Makefile文件中如下所示，该变量赋值语句需压调用call函数并将后文变量应用到`add_files`变量，再寻找该变量的定义，定义位置在function.mk中，该变量调用eval函数，解析和执行后文call函数的返回值，call函数将后文变量应用到`do_add_files_to_packet`变量中，寻找该变量的定义，定义位置在function.mk文件中，其作用为将一组源文件编译成目标文件，并将目标文件加入一个打包文件中，该定义还用到了`cc_template`，因此寻找器定义，位于function.mk文件中，其作用为自动化C/C++源文件的编译过程，生成依赖文件[.d]并编译生成目标文件[.o]。上述代码定义如下

```makefile
add_files_cc = $(call add_files,$(1),$(CC),$(CFLAGS) $(3),$(2),$(4))

add_files = $(eval $(call do_add_files_to_packet,$(1),$(2),$(3),$(4),$(5)))

# add files to packet: (#files, cc[, flags, packet, dir])
define do_add_files_to_packet
__temp_packet__ := $(call packetname,$(4))
ifeq ($$(origin $$(__temp_packet__)),undefined)
$$(__temp_packet__) :=
endif
__temp_objs__ := $(call toobj,$(1),$(5))
$$(foreach f,$(1),$$(eval $$(call cc_template,$$(f),$(2),$(3),$(5))))
$$(__temp_packet__) += $$(__temp_objs__)
endef

# cc compile template, generate rule for dep, obj: (file, cc[, flags, dir])
define cc_template
$$(call todep,$(1),$(4)): $(1) | $$$$(dir $$$$@)
	@$(2) -I$$(dir $(1)) $(3) -MM $$< -MT "$$(patsubst %.d,%.o,$$@) $$@"> $$@
$$(call toobj,$(1),$(4)): $(1) | $$$$(dir $$$$@)
	@echo + cc $$<
	$(V)$(2) -I$$(dir $(1)) $(3) -c $$< -o $$@
ALLOBJS += $$(call toobj,$(1),$(4))
endef

```

在终端执行指令`make "V="`查看实际执行指令
![](gccSign.jpg)

##### gcc编译参数总结

在上文查看实际执行指令中，可以看到gcc指令的诸多参数，下面进行简单的总结

* `-I <dir>`：添加头文件搜索路径
* `-march`：指定CPU架构，例如`-march=i686`
* `-fno-builtin`：不优化builtin函数，除非使用__builtin__前缀
* `-fno-PIC`：生成位置无关代码
* `-Wall`：开启所有警告
* `-ggdb`：生成gdb调试信息
* `-m32`：生成32位环境下适用的代码
* `-gstabs`：生成stabs格式的调试信息
* `-stdinc`：不使用标准库
* `-fno-stack-protector`：不生成检测缓冲区溢出的代码
* `-Os`：进行优化以减少代码大小
* `-O?`：进行编译优化，'?'处表示0、1、2、3，代表优化程度不同，数字越大优化程度越大

#### 练习1-2 问题解析

* **一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？**

根据上文，硬盘的主引导扇区实际上就是硬盘的第一个扇区，在操作系统启动时，BIOS会将该扇区的内容加载到内存中。在生成bootblock时，依赖sign.c文件，因此想要得知该扇区的信息，需要阅读该文件内容，该文件代码如下所示
```C
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/stat.h>

int
main(int argc, char *argv[]) {
    struct stat st;
    if (argc != 3) {
        fprintf(stderr, "Usage: <input filename> <output filename>\n");
        return -1;
    }
    if (stat(argv[1], &st) != 0) {
        fprintf(stderr, "Error opening file '%s': %s\n", argv[1], strerror(errno));
        return -1;
    }
    printf("'%s' size: %lld bytes\n", argv[1], (long long)st.st_size);
    if (st.st_size > 510) {
        fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
        return -1;
    }
    char buf[512];
    memset(buf, 0, sizeof(buf));
    FILE *ifp = fopen(argv[1], "rb");
    int size = fread(buf, 1, st.st_size, ifp);
    if (size != st.st_size) {
        fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
        return -1;
    }
    fclose(ifp);
    buf[510] = 0x55;
    buf[511] = 0xAA;
    FILE *ofp = fopen(argv[2], "wb+");
    size = fwrite(buf, 1, 512, ofp);
    if (size != 512) {
        fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
        return -1;
    }
    fclose(ofp);
    printf("build 512 bytes boot sector: '%s' success!\n", argv[2]);
    return 0;
}
```

该程序的作用是将输入的文件转化为大小为512字节的引导扇区文件，主要过程如下

* 判断输入参数是否合法，若不合法则输出提示信息并返回错误代码
* 获取输入文件的属性信息，若无法打开则输出错误信息并返回错误代码
* 判断输入文件大小是否小于等于510字节，若大于则输出提示信息并返回错误代码
* 读取输入文件的数据，将其填充到大小为512字节的缓冲区中
* 在缓冲区的511、512位置分别填充0xAA和0x55两个字节
* 打开输出文件，将缓冲区中的数据写入到输出文件中
* 判断写入是否成功，若失败则输出错误信息并返回错误代码
* 输出操作成功信息，并返回0

根据上述分析，可以得知，合法的主引导扇区具有如下特征

* 位于硬盘的第一个扇区
* 大小为512字节
* 最后两个字节为主引导扇区的标志（0x55和0xAA），用于标识该扇区为主引导扇区。

事实上，主引导扇区的前510字节的内容规划也是有规定的：446个字节为主引导程序（bootloader），接下来64个字节为分区表，用于描述硬盘的分区信息

### 练习 2：使用qemu执行并调试lab1中的软件

* 前置知识：gdb与qemu的配合使用，参考ucore实验指导书与其他资料

#### 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行

阅读ucore附录"第一条指令"，根据指引在终端执行`make debug`指令，打开gdb调试与qemu。首先注意到，BIOS的执行停止在第一步，执行`si`指令，可以单步跟踪BIOS，执行`x/i $pc`指令，可以查看当前pc寄存器中的指令，即下一步要执行的指令，i前可以加数字，表示查看接下来执行的指令条数。对上述指令进行尝试，执行结果如下图所示
![](gdbBIOS.jpg)

#### 在初始化位置0x7c00设置实地址断点,测试断点正常

在打开gdb调试界面后，执行指令`break *0x7c00`在实地址0x7c00处设置断点，并执行`c`指令(continue)，使程序运行至断点，查看是否正确设置断点。执行结果如下图所示
![](breakTest.jpg)

可以看到，执行`x/5i $pc`后，打印的指令地址是从0x7c00开始，证明断点设置正确

#### 将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较

阅读实验指导文档的qemu使用部分，可以设置参数`-d`来输出日志，也可以执行指令`log in_asm`，两种方式均可以将反汇编的道德汇编指令保存在tmp/qemu.log文件中，方便查看和对比。找到Makefile文件中debug的依赖关系描述和指令部分，代码如下所示

```makefile
debug: $(UCOREIMG)
	$(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```

上述代码定义debug为依赖于UCOREIMG的目标，执行时，先使用QEMU模拟器启动指定的镜像文件\$(UCOREIMG)，并开启-gdb支持，并将标准输出重定向到终端。然后让程序暂停2秒。最后使用变量\$(TERMINAL)打开一个新的终端，执行gdb调试工具，并指定gdb初始化脚本tools/gdbinit，用于调试目标程序。在第一行指令处添加参数，修改为
```makefile
$(V)$(QEMU) -S -s in_asm -D $(BINDIR)/qemu.log -parallel stdio -hda $< -serial null &
```
该指令表示使用QEMU模拟器启动指定镜像文件，打开gdb支持，并将反汇编结果日志输出到位于同目录"bin/"文件夹下的"qemu.log"中。再次打开终端执行`make debug`指令，并在0x7c00处设置断点，查看汇编代码，执行结果如下
![](asmTest.jpg)

此时查看"bin/qemu.log"的末尾，可以看到从0x7c00到0x7c10的汇编代码，如下所示
```
----------------
IN: 
0x00007c00:  cli    

----------------
IN: 
0x00007c01:  cld    
0x00007c02:  xor    %ax,%ax
0x00007c04:  mov    %ax,%ds
0x00007c06:  mov    %ax,%es
0x00007c08:  mov    %ax,%ss

----------------
IN: 
0x00007c0a:  in     $0x64,%al

----------------
IN: 
0x00007c0c:  test   $0x2,%al
0x00007c0e:  jne    0x7c0a

----------------
IN: 
0x00007c10:  mov    $0xd1,%al
0x00007c12:  out    %al,$0x64
0x00007c14:  in     $0x64,%al
0x00007c16:  test   $0x2,%al
0x00007c18:  jne    0x7c14
# 代码过长，不作展示
```
将之与bootasm.S和 bootblock.asm中的代码进行对比，可以发现完全一致

* 对于练习2中自定义断点进行测试不作详细展示

### 练习 3：分析bootloader进入保护模式的过程

#### 基本知识概述

* 对所需知识进行简要总结

##### 保护模式与实模式

在引导程序接手BIOS的工作时，PC系统处于**实模式**，在这种状态下软件可访问的物理内存空间不能超过1MB，并且操作系统和用户程序并没有区别对待，每一个指针都是指向实际的物理地址。因此用户可能无意中修改了其他用户的数据甚至是操作系统的重要数据，这是及其危险的。在**保护模式**下，80386的全部32根地址线有效，可寻址高达4G字节的线性地址空间和物理地址空间，更重要的，保护模式下，系统可以采取**分段管理机制**。

分段管理机制涉及到几个重要概念：逻辑地址、段描述符（描述段的属性）、段描述符表（管理多个段描述符）、段选择子（段寄存器，用于定位描述符表中某一项索引）。首先，**段描述符**用于描述段的属性，包括段的基址，段的界限以及段的其他属性；**段描述符表**用于存储段描述符，实际上是一个包含段描述符的数组；**段选择子**则可以定位段描述符在段描述符表中的索引，从而访问对应的段；最后，**逻辑地址**包括段选择子与段偏移地址。

可以通过逻辑地址访问实际物理地址，首先通过段选择子找到对应段的基址，然后将基址处理后加上偏移量，即可得到实际地址

需要注意，分段管理机制只有在保护模式中启用，另一方面来说，在切换为保护模式时，必须建立好段描述符表，才能保证分段管理机制正确运作

> 注：实际上，逻辑地址通过上述变换得到的是**线性地址**，也称为虚拟地址，在没有启用分页管理机制时，线性地址与实际地址的值相同，但在启用分页管理机制后，线性地址要通过分页转换，才能得到相应的实际地址。

##### A20 Gate

A20门是IBM PC兼容机上一个控制地址总线的电路。在早期的PC中，由于历史原因，CPU只使用了20根地址线，导致最大只能寻址1MB的内存空间。后来的PC机为了向下兼容，中增加了A20门电路，用于控制CPU能够访问的物理内存范围。当A20门关闭时，CPU只能寻址1MB的内存空间，而当A20门打开时，CPU可以访问1MB以上的内存空间。在保护模式下，操作系统需要控制A20门的状态，以确保程序只能访问被授权的内存空间，而不会越界访问其他内存区域。

#### bootasm.S代码解析

在代码开头的注释内容中，我们得知：BIOS将该代码从硬盘的第一个扇区（主引导扇区）加载到物理地址为0x7c00的内存中，并开始以实模式在`cs=0 ip=7c00`执行。

下面结合注释，对代码进行分析

* 首先设置内核代码段选择子、内核数据段选择子以及保护模式使能标志

```assembly
.set PROT_MODE_CSEG,        0x8                     # kernel code segment selector
.set PROT_MODE_DSEG,        0x10                    # kernel data segment selector
.set CR0_PE_ON,             0x1                     # protected mode enable flag
```

* 进入程序后，清理环境，将段寄存器与flag均置为0

```assembly
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment
    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment
```

* 打开**A20 Gate**，禁止寻址方向回卷，允许访问超过1MB的内存空间。由于A20的地址位是由芯片8042管理，这个芯片与键盘控制器有关，因此通过给8042芯片发命令来激活A20的地址位，8042的两个I/O端口是0x64和0x60，通过发送0xd1命令到0x64端口、发送0xdf到0x60端口就可以激活。下方代码分为两个部分，需要注意的是，这两部分代码都要通过读0x64端口的第2位，确保8042的输入缓冲区为空后再进行操作

```assembly
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al                                 # 如果 %al 第低2位为1，则ZF = 0, 则跳转
    jnz seta20.1                                    # 如果 %al 第低2位为0，则ZF = 1, 则不跳转

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

* 初始化**GDT**(全局段描述符表)。在代码末尾部分，为GDT分配了内存，通过执行`lgdt`将一个已经静态储存在引导区中的简单的GDT表及其描述符载入到内存

```assembly
	lgdt gdtdesc
	
	# Bootstrap GDT
.p2align 2                                          # force 4 byte alignment
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
```

* 通过将**cr0寄存器**的PE位置为1,开启保护模式
```assembly
	movl %cr0, %eax
	orl $CR0_PE_ON, %eax
	movl %eax, %cr0
```

* 通过长跳转更新cs的基地址，跳转位置在下一段代码
```assembly
ljmp $PROT_MODE_CSEG, $protcseg
```

* 进入32位代码模式，设置段寄存器，并建立堆栈（分段管理机制）
```assembly
.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment
```

* 成功切换至保护模式，进入boot主方法
```assembly
	call bootmain
```

* 另外，GDT表的初始化在"kern/mm/pmm.c"中，以C语言的形式也有体现，用一个全局变量数组gdt[]来管理GDT，并使用`gdt_init`函数来初始化

```C
static struct segdesc gdt[] = {
    SEG_NULL,
    [SEG_KTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_KERNEL),
    [SEG_KDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_KERNEL),
    [SEG_UTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_USER),
    [SEG_UDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_USER),
    [SEG_TSS]    = SEG_NULL,
};

static void
gdt_init(void) {
    
    ts.ts_esp0 = (uint32_t)&stack0 + sizeof(stack0);
    ts.ts_ss0 = KERNEL_DS;

    // initialize the TSS filed of the gdt
    gdt[SEG_TSS] = SEG16(STS_T32A, (uint32_t)&ts, sizeof(ts), DPL_KERNEL);
    gdt[SEG_TSS].sd_s = 0;

    // reload all segment registers
    lgdt(&gdt_pd);

    // load the TSS
    ltr(GD_TSS);
}
```

### 练习4：分析Bootloader加载ELF格式O.S.的过程

* 阅读"bootmain.c"代码，分析bootloader如何加载ELF文件

#### 对bootmain.c的整体分析

代码首先宏定义了`SECTSIZE`为512作为扇区大小、`ELFHDR`为虚拟地址的起始地址，语句中的`struct elfhdr`在文件"elf.h"中声明，描述了整个文件的组织。随后程序声明如下函数

* `waitdisk()`，作用是**等待磁盘做好准备**
* `readsect(void *dst, uint32_t sctno)`，作用是读取扇区secno的数据到dst位置
* `readseg(uintptr_t va, uint32_t count, uint32_t offset)`，对readsect进行包装，可以从设备读取任意长度的内容
* `bootmain(void)`，作为bootloader程序的入口函数

#### 对具体函数的分析

查看`readsect(void *dst, uint32_t secno)`函数，该函数作用为**从设备的第secno扇区读取数据到dst位置**，该函数表明了从一个扇区读取数据的流程：**1.等待磁盘做好准备；2.发出读取磁盘的命令；3.等待磁盘做好准备；4.读取磁盘数据至指定位置**。函数代码与注释说明如下
```C
static void
readsect(void *dst, uint32_t secno) {
    waitdisk();

    outb(0x1F2, 1);                         // 设置读取扇区数目为1
    outb(0x1F3, secno & 0xFF);				// 设定0-7位
    outb(0x1F4, (secno >> 8) & 0xFF);		// 设定8-15位
    outb(0x1F5, (secno >> 16) & 0xFF);		// 设定16-23位
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);	// 设定24-31位
    outb(0x1F7, 0x20);                      // cmd 0x20 - 读取扇区
			/* 上面四条指令联合制定了扇区号
	           在联合构成的32位参数中
	           29-31位设为1
	           28位(=0)表示访问"Disk 0"
	           0-27位是偏移量 */
	           
    waitdisk();
    
    // 读取扇区数据到dst位置
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

`readseg(uintptr_t va, uint32_t count, uint32_t offset)`函数对readsect函数进行简单包装，实现了读取任意长度的数据，函数代码与注释如下
```C
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;	//计算读取块的终止地址

    // 计算块的首地址（起始地址减去偏移量）
    va -= offset % SECTSIZE;

    // 从扇区1开始（扇区0已经被占用）
    uint32_t secno = (offset / SECTSIZE) + 1;

    /*读取速度过慢时，我们会同时读取多个扇区，
      但是这会导致向内存写入的数据超出预先请求
      此时，我们会按照升序载入数据*/
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```

`bootmain`函数作为bootloader程序的入口，该函数表明了其加载ELF格式的O.S.的大致过程：**先等待磁盘准备就绪，然后读取ELF头部的`e_magic`判断文件是否合法，然后读取ELF内存位置的描述表，然后按照描述表内容，将ELF文件中的数据载入内存，根据ELF头部的`e_entry`代表的入口信息找到内核入口，执行内核代码。**该部分函数代码及注释如下

```C
bootmain(void) {
    // 从磁盘的第一个扇区读取"kernel"文件
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // 判断是否是一个有效的ELF文件（检查e_magic）
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // 读取ELF头部的e_phoff变量得到描述表的头地址，指示ELF应当加载到内存的什么位置
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    //按照描述表将ELF文件中数据按照偏移、虚拟地址、长度等信息载入内存
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }
    /* ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	   ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000 */

    // 通过ELF头部的e_entry变量储存的入口信息，找到内核的入口地址，并开始执行内核代码
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

### 练习5：实现函数调用堆栈跟踪函数

* 完成位于"kern/debug/kdebug.c"中函数`print_stackframe`的实现

根据"kdebug.c"中的注释指引，实现`print_stackframe`函数。首先使用`read_ebp()`和`read_eip()`获取32位的寄存器`ebp`和`eip`中的值并分别赋给32位变量`ebp_val`和`eip_val`。然后遍历栈，打印每个栈帧的信息：使用变量`arg`指向存放参数的`ss:[ebp+8]`的位置，依次打印调用函数的四个参数，输出换行符后，调用函数`print_debuginfo()`展示当前调用函数层的`eip`和`ebp`相关的信息，最后`eip`指向返回地址，`ebp`指向原`ebp`的地址。代码与注释如下

```C
void print_stackframe(void) {
    uint32_t ebp = read_ebp();	//读取寄存器ebp数据
	uint32_t eip = read_eip();	//读取寄存器eip数据
	int i, j;
    //遍历整个栈，每个栈帧信息
	for(i = 0; i < STACKFRAME_DEPTH && ebp != 0; i++) {
		cprintf("ebp:0x%08x eip:0x%08x", ebp, eip);
		uint32_t *arg = (uint32_t *)ebp + 2;	//读取存放参数位置的数据
		cprintf(" arg:");
		for(j = 0; j < 4; j++) {
			cprintf("0x%08x ", arg[j]);		//依次打印调用函数的四个参数
		}
		cprintf("\n");
		print_debuginfo(eip - 1);
		eip = ((uint32_t *)ebp)[1];
		ebp = ((uint32_t*)ebp)[0];
	}
}
```

在Lab1目录下，在终端执行指令`make qemu`，得到相关执行结果如下图所示。与实验指导书上的参考截图对比，两者基本一致。
![](stackFrame.jpg)

打印内容的最后一行为：`ebp:0x00007bf8 eip:0x00007d6e arg:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8  <unknow>: -- 0x00007d6d --`，下面对各参数进行分析。该行信息为调用栈中的最深一层，代表第一个使用堆栈的函数，即"bootmain"。根据函数调用关系分析，可以得知此时`ebp`的值是`kern_init`函数的栈顶地址，从"obj/bootblock.asm""文件中知可以得知，整个栈的栈顶地址为0x00007c00，`ebp`指向的栈位置存放调用者的`ebp`寄存器的值，`ebp+4`指向的栈位置存放返回地址的值，这意味着`kern_init`函数的调用者（也就是bootmain函数）没有传递任何输入参数给它。（因为存放旧的`ebp`与返回地址就需要8字节）。`eip`的值是`kern_init`函数的返回地址，也就是`bootmain`函数调用`kern_init`对应的指令的下一条指令的地址。这与"obj/bootblock.asm"是相符合的。

### 练习6：完善中断初始化和处理

#### Q1：中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

在x86架构的计算机中，中断向量表中的每个表项占据8个字节（64位）。其中，前4个字节（32位）用于存储中断处理代码的入口地址，后4个字节（32位）用于存储中断处理代码的段选择器和控制信息。

具体的，中断处理代码的入口地址存储在中断向量表的前4个字节中，也就是该中断向量表表项的低32位。在这32位中，从第0位到第15位存储着中断处理代码所在的代码段的段选择器（也称为段描述符），从第16位到第31位存储着中断处理代码的偏移量（即中断处理代码在代码段中的地址）。

> 需要注意，由于中断处理程序的入口地址必须是4字节对齐的，因此中断向量表中存储的地址的最低2个比特位（第0位和第1位）总是0。

#### Q2：编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init

依据"kern/trap/trap.c"中注释指引，对函数`idt_init()`进行编写。首先声明`extern uintptr_t`类型的变量`__vectors[]`，该变量用来存放256个在vectors.S定义的中断处理例程的入口地址（"vector.S"在执行`make`指令后，在"kern/trap"目录下找到）；随后使用SETGATE宏，设置中断描述符表中的每一个表项（该宏定义在"mmu.h"中找到）；最后使用lidt函数加载中断描述符表。该部分代码及注释如下
```C
void idt_init(void) {
    extern uintptr_t __vectors[];
	int i;
	for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i ++) {
        /* 使用宏SETGATE，设置中断描述符表中的每一个表项
           参数中，宏定义GD_KTEXT代表是.text段；宏定义DPL_KERNEL代表内核级；
           宏定义DPL_USER代表用户级；
           宏定义T_SWITCH_TOK是用于用户态切换到内核态的中断号 */
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);		
    }
	SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
	lidt(&idt_pd);	//通过lidt函数加载中断描述符表
}
```

上述函数中的SETGATE宏的声明可以在"mmu.h"中找到，如下所示。其中各个参数的含义为

* 宏的参数gate代表选择的idt数组的项，是处理函数的入口地址
* 参数istrap为1时代表系统段，为0时代表中断门

* 参数sel是中断处理函数的段选择子
* 参数off表示中断处理例程的入口地址
* 参数dpl表示优先级。

```C
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```

#### Q3：编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数

依据"kern/trap/trap.c"中注释指引，对函数`trap_dispatch()`进行完善。首先让用于记录时钟中断次数自增1，该变量声明位于"kern/driver/clock.c"中，且有相应函数初始化该变量为0；随后，在每轮进行`TICK_NUM`次的循环完成时，都调用一次`print_ticks()`函数。为了让`ticks`能够一直对时钟终端次数进行记录，此处通过令`ticks`与`TICK_NUM`做取模运算来判断该轮循环是否结束。`print_ticks()`函数执行打印"100 ticks"的操作。该部分代码如下
```C
case IRQ_OFFSET + IRQ_TIMER:
    ticks++;
    if (ticks % TICK_NUM == 0) {
    	print_ticks();
   	}
    break;
```
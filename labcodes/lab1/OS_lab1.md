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

### Lab 1 练习 1: 理解通过 make 生成执行文件的过程

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

### Lab 1 练习 2：使用qemu执行并调试lab1中的软件

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

### Lab1-练习 3：分析bootloader进入保护模式的过程

#### 保护模式与实模式

* 阅读实验指导书的相关章节，对保护模式与实模式进行总结

在引导程序接手BIOS的工作时，PC系统处于**实模式**，在这种状态下软件可访问的物理内存空间不能超过1MB，并且操作系统和用户程序并没有区别对待，每一个指针都是指向实际的物理地址。因此用户可能无意中修改了其他用户的数据甚至是操作系统的重要数据，这是及其危险的。而在保护模式下，80386的全部32根地址线有效，可寻址高达4G字节的线性地址空间和物理地址空间，可访问64TB（有2^14^个段，每个段最大空间为2^32^字节）的逻辑地址空间


# Lab1 Report

## Ex 1

1.1 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中
每一条相关命令和命令参数的含义,以及说明命令导致的结果)

```
  bin/ucore.img

  生成ucore.img的依赖关系如下：
  $(UCOREIMG): $(kernel) $(bootblock)
 
  由此可见，为了生成ucore.img，首先需要生成bootblock、kernel
 
 >	对于 bootblock
 	| 生成bootblock的相关代码为
 	| $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
		他需要boot/下的文件编译出来的.o文件以及sign
		

 	|>	对于boot/下编译得到的.o文件
 	|	| 生成bootasm.o,bootmain.o的相关makefile代码为
 	|	| bootfiles = $(call listf_cc,boot) 
 	|	| $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),\
 	|	|	$(CFLAGS) -Os -nostdinc))
		实际执行的代码为:
		i386-elf-gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
		i386-elf-gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
		
		再具体而言
 	|	| 生成bootasm.o需要bootasm.S
 	|	| 实际命令为
 	|	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs \
 	|	| 	-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc \
 	|	| 	-c boot/bootasm.S -o obj/boot/bootasm.o
 	|	| 其中关键的参数为
 	|	| 	-ggdb  生成可供gdb使用的调试信息。这样才能用qemu+gdb来调试bootloader or ucore。
 	|	|	-m32  生成适用于32位环境的代码。我们用的模拟硬件是32bit的80386，所以ucore也要是32位的软件。
 	|	| 	-gstabs  生成stabs格式的调试信息。这样要ucore的monitor可以显示出便于开发者阅读的函数调用栈信息
 	|	| 	-nostdinc  不使用标准库。标准库是给应用程序用的，我们是编译ucore内核，OS内核是提供服务的，所以所有的服务要自给自足。
 	|	|	-fno-stack-protector  不生成用于检测缓冲区溢出的代码。这是for 应用程序的，我们是编译内核，ucore内核好像还用不到此功能。
 	|	| 	-Os  为减小代码大小而进行优化。根据硬件spec，主引导扇区只有512字节，我们写的简单bootloader的最终大小不能大于510字节。
 	|	| 	-I<dir>  添加搜索头文件的路径


 	|	| 生成bootmain.o需要bootmain.c
 	|	| 实际命令为
 	|	| gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc \
 	|	| 	-fno-stack-protector -Ilibs/ -Os -nostdinc \
 	|	| 	-c boot/bootmain.c -o obj/boot/bootmain.o
 	|	| 新出现的关键参数有
 	|	| 	-fno-builtin  除非用__builtin_前缀，
 	|	|	              否则不进行builtin函数的优化
 	|
	对于sign文件

 	|	| 生成sign工具的makefile代码为
 	|	| $(call add_files_host,tools/sign.c,sign,sign)
 	|	| $(call create_target_host,sign,sign)
 	|	| 
 	|	| 实际命令为
 	|	| gcc -Itools/ -g -Wall -O2 -c tools/sign.c \
 	|	| 	-o obj/sign/tools/sign.o
 	|	| gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign

 	| 首先生成bootblock.o
 	| ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 \
 	|	obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o

 	| 拷贝二进制代码bootblock.o到bootblock.out
 	| objcopy -S -O binary obj/bootblock.o obj/bootblock.out
 	| 使用sign工具处理bootblock.out，生成bootblock
 	| bin/sign obj/bootblock.out bin/bootblock
 
	对于kernel:

 	| 生成kernel的相关代码为
 	| $(kernel): tools/kernel.ld
 	| $(kernel): $(KOBJS)
 	| 	@echo + ld $@
 	| 	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
 	| 	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
 	| 	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; \
 	| 		/^$$/d' > $(call symfile,kernel)
 	| 
 	| 为了生成kernel，需要kern和lib下个各种.o文件 kernel.ld init.o readline.o stdio.o kdebug.o
 	|	kmonitor.o panic.o clock.o console.o intr.o picirq.o trap.o
 	|	trapentry.o vectors.o pmm.o  printfmt.o string.o
 	| kernel.ld已存在

 	|>	obj/kern/*/*.o 
 	|	| 生成这些.o文件的相关makefile代码为
 	|	| $(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,\
 	|	|	$(KCFLAGS))
 	|	| 这些.o生成方式和参数均类似，仅举init.o为例，其余不赘述
 	|>	obj/kern/init/init.o
 	|	| 编译需要init.c
 	|	| 实际命令为
 	|	|	gcc -Ikern/init/ -fno-builtin -Wall -ggdb -m32 \
 	|	|		-gstabs -nostdinc  -fno-stack-protector \
 	|	|		-Ilibs/ -Ikern/debug/ -Ikern/driver/ \
 	|	|		-Ikern/trap/ -Ikern/mm/ -c kern/init/init.c \
 	|	|		-o obj/kern/init/init.o
 	| 
 
  生成一个有10000个块的文件，每个块默认512字节，用0填充
  dd if=/dev/zero of=bin/ucore.img count=10000
  把bootblock中的内容写到第一个块
  dd if=bin/bootblock of=bin/ucore.img conv=notrunc
  从第二个块开始写kernel中的内容
  dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

1.2 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?

从sign.c的代码来看，一个磁盘主引导扇区只有512字节, 
其中实际可以用的大小是510个字节，
第510个（倒数第二个）字节是0x55，
第511个（倒数第一个）字节是0xAA。


## Ex 2

2.1 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。

通过改写Makefile文件

```
	debug: $(UCOREIMG)
		$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -parallel stdio -hda $< -serial null"
		$(V)sleep 2
		$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
```

在调用qemu时增加`-d in_asm -D q.log`参数，便可以将运行的汇编指令保存在q.log中。

从q.log中看，计算机运行起来之后的第一条指令位置在0xfffffff0处，是一条跳转指令，BIOS的作用是将bootloader给搞进来，一共
有16000多行。


2.2  在初始化位置0x7c00 设置实地址断点,测试断点正常。

在tools/gdbinit结尾加上

```
	b *0x7c00  //在0x7c00处设置断点。此地址是bootloader入口点地址，可看boot/bootasm.S的start地址处
```
	
运行"make debug"便可得到

```
	Breakpoint 2, 0x00007c00 in ?? ()
	=> 0x7c00:      cli    
	   0x7c01:      cld    
	   0x7c02:      xor    %eax,%eax
	   0x7c04:      mov    %eax,%ds
	   0x7c06:      mov    %eax,%es
	   0x7c08:      mov    %eax,%ss 
	   0x7c0a:      in     $0x64,%al
	   0x7c0c:      test   $0x2,%al
	   0x7c0e:      jne    0x7c0a
	   0x7c10:      mov    $0xd1,%al
```

2.3 在调用qemu 时增加-d in_asm -D q.log 参数，便可以将运行的汇编指令保存在q.log 中。
将执行的汇编代码与bootasm.S 和 bootblock.asm 进行比较，看看二者是否一致。

在tools/gdbinit结尾加上
```
	b *0x7c00
	c
	x /10i $pc
```

便可以在q.log中读到"call bootmain"前执行的命令
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
	
```

	其与bootasm.S和bootblock.asm中的代码相同。

2.4 自己找一个bootloader或者内核中的代码的位置，设置断点并进行测试
	```
		make debug-nox 
		b init.c:30
		c
	```
	会停在 pmm_init()上

## Ex 3
分析bootloader 进入保护模式的过程。


* 查看bootloader.S 代码
* 首先清理环境：包括将flag置0和将段寄存器置0
```
	.code16
	    cli
	    cld
	    xorw %ax, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %ss
```

* 开启A20：通过将键盘控制器上的A20线置于高电位，全部32条地址线可用，
可以访问4G的内存空间。
```
	seta20.1:               # 等待8042键盘控制器不忙
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.1        #
	
	    movb $0xd1, %al     # 发送写8042输出端口的指令
	    outb %al, $0x64     # 
	
	seta20.2:               # 等待8042键盘控制器不忙
	    inb $0x64, %al      # 
	    testb $0x2, %al     #
	    jnz seta20.2        #
	
	    movb $0xdf, %al     # 打开A20
	    outb %al, $0x60     # 
```

* 初始化GDT表
```
	    lgdt gdtdesc
```

* 进入保护模式：通过将cr0寄存器PE位置1便开启了保护模式
```
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
```

* 通过长跳转更新cs的基地址
```
	 ljmp $PROT_MODE_CSEG, $protcseg
	.code32
	protcseg:
```

* 设置段寄存器，并建立堆栈
```
	    movw $PROT_MODE_DSEG, %ax
	    movw %ax, %ds
	    movw %ax, %es
	    movw %ax, %fs
	    movw %ax, %gs
	    movw %ax, %ss
	    movl $0x0, %ebp
	    movl $start, %esp
```
* 转到保护模式完成，进入boot主方法
```
	    call bootmain
```


## Ex 4
4.1 bootloader 是如何读取硬盘扇区的?
```
	查看readsect函数:
	它的功能是从设备的第secno扇区读取数据到dst位置
```

	> * 等待磁盘
```
	    waitdisk();
```
	
	> *	往0x1F2端口写1 表示读一个扇区，往剩下的一些串口写特定的标示符。 最后往0x1F7写0x20表示
	读一个扇区
```
	    outb(0x1F2, 1);                 
	    outb(0x1F3, secno & 0xFF);
	    outb(0x1F4, (secno >> 8) & 0xFF);
	    outb(0x1F5, (secno >> 16) & 0xFF);
	    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
	    outb(0x1F7, 0x20);                      // 0x20命令，读取扇区
```
	    waitdisk();

	> * 把扇区读到内存里面
```
	    insl(0x1F0, dst, SECTSIZE / 4);         // 读取到dst位置，
	                                            // 幻数4因为这里以DW为单位
```

4.2 bootloader 是如何加载ELF格式的OS的？

在bootmain函数中，
```
	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    // ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    // ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    // 根据ELF头部储存的入口信息，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```


## Ex 5
实现函数调用堆栈跟踪函数 

> * 参见代码。

ss:ebp指向的堆栈位置储存着caller的ebp，以此为线索可以得到所有使用堆栈的函数ebp。
ss:ebp+4指向caller调用时的eip，ss:ebp+8等是（可能的）参数。

输出中，堆栈最深一层为
```
	ebp:0x00007bf8 eip:0x00007d68 \
		args:0x00000000 0x00000000 0x00000000 0x00007c4f
```

其对应的是第一个使用堆栈的函数，bootmain.c中的bootmain。
bootloader设置的堆栈从0x7c00开始，使用"call bootmain"转入bootmain函数。
call指令压栈，所以bootmain中ebp为0x7bf8。


## Ex 6
完善中断初始化和处理

6.1 Ex 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

```
观察/kern/mm/mmu.h中的gatedesc结构可以发现：
中断向量表一个表项占用8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，
两者联合便是中断处理程序的入口地址。
```

6.2 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。
```
见代码
```

6.3 请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数

```
见代码
```



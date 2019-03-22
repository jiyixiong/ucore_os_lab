# <center>操作系统Lab1实验报告</center>

#### <center>计64 计逸雄 2016011326</center>

## 练习1

## 1.1

查找到ucore.img的生成逻辑如下代码，可以发现其依赖于kernel和bootblock，进而需要查找kernel和bootblock的生成。生成一个10000个512字节的块的文件，初始化为全0，第一个块写入bootblock，后面写入kernel的内容。

```makefile
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

kernel的生成代码如下，结合KOBJS的定义和make "V="的打印消息，可以知道需要先生成kernel.ld, init.o, stdio.o, readline.o, kdeug.o，kmonitor.o, panic.o, clock.o, console.o, intro.o, picirq.o, trap.o, trapentry.o, vector.o, pmm.o, printfmt.o, string.o等。

```makefile
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)
```

bootblock生成相关代码如下，可知其依赖于bootasm.o, bootmain.o, sign。先通过bootasm.o、bootmain.o生成bootblock.o, 拷贝至bootblock.out, 然后sign处理bootblock.out生成bootblock

```makefile
# create bootblock
bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```

bootasm.o，bootmain.o生成代码如下

```makefile
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))		
```

生成sign的代码如下

```makefile
# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

## 1.2

根据sign.c中的代码，可知其大小只有512字节，并且最后两个字节内容为0x55、0xAA.

## 练习2

## 2.1

参考实验指导书附录中的指导，修改gdbinit中的内容为：

```makefile
set architecture i8086
target remote :1234
```

然后make debug并在gdb界面使用si指令即可单步跟踪BIOS，部分结果如下

```asm
(gdb) x /2i 0xffff0
   0xffff0:     ljmp   $0x3630,$0xf000e05b
   0xffff7:     das 
(gdb) x /2i 0xfe062
   0xfe062:     jne    0xd241d416
   0xfe068:     mov    %ed,%ss
```

## 2.2

在gdb中输入如下，测试断点正常。

```asm
(gdb) b *0x7c00
Breakpoint 1 at 0x7c00
(gdb) c
Continuing.
Breakpoint 1, 0x00007c00 in ?? ()
(gdb) x /2i $pc
=> 0x7c00:      cli    
   0x7c01:      cld 
```

## 2.3

```asm
(gdb) x /10i 0x7c00
   0x7c00:      cli
   0x7c01:      cld
   0x7c02:      xor    %eax,%eax
   0x7c04:      mov    %eax,%ds
=> 0x7c06:      mov    %eax,%es
   0x7c08:      mov    %eax,%ss
   0x7c0a:      in     $0x64,%al
   0x7c0c:      test   $0x2,%al
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    $0xd1,%al
```

gdb中获取反编译的汇编代码如上，与bootasm.S和bootblock.asm中代码一致。

## 2.4

例如把断点设置在0x7c1c，测试断点正常，如下

```asm
(gdb) b *0x7c1c
Breakpoint 1 at 0x7c1c
(gdb) c
Continuing.
Breakpoint 1, 0x00007c1c in ?? ()
(gdb) x /2i $pc
=> 0x7c1c:      out    %al,$0x60
   0x7c1e:      lgdtl  (%esi)
```

## 练习3

首先关中断，并把ds，es，ss清零。

```asm
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment
    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment

    # Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
```

## 3.1

初始时A20为0，访问空间只有1MB，无法访问到更大的空间，只有A20为1时才能访问4G的内存空间。

开启A20的步骤为：

>1.等待8042的Input buffer不忙；
>
>2.发送写8042的输出端口的命令到Input buffer；
>
>3.等待8042的Input buffer不忙；
>
>将8042的输出端口的第二位置为1，并写入Input buffer。

```asm
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

## 3.2

见如下代码及注释

```asm
    lgdt gdtdesc       #载入gdt表，定义见本段最后部分
    movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0    #将cro寄存器的第0位（即PE位）置为1，及代表处于保护模式

    ljmp $PROT_MODE_CSEG, $protcseg  #长跳转更新cs的基地址

.code32                                             # Assemble for 32-bit mode
protcseg:
	#设置段寄存器
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment
	
	#并建立堆栈
    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain	#完成，调用bootmain主函数
gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
```

## 3.3

将cr0寄存器第0位（PE位）置为1即可。

## 练习4

## 4.1

 读取扇区的步骤为

>1.等待磁盘准备好；
>
>2.发出读取扇区的命令；
>
>3.等待磁盘准备好；
>
>4.把磁盘扇区数据读入到对应内存位置

如下为读取扇区的函数readsect及其注释

```c
static void readsect(void *dst, uint32_t secno) {
    waitdisk(); //等待磁盘准备好

    outb(0x1F2, 1);              		// 设置读取扇区的数量，为1
    outb(0x1F3, secno & 0xFF);   		// 设置读取的扇区编号
    outb(0x1F4, (secno >> 8) & 0xFF);  	// 所在柱面的低8位
    outb(0x1F5, (secno >> 16) & 0xFF); 	// 所在柱面的高8位
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0); //29~31设置为1，28位设置为0
    outb(0x1F7, 0x20);                  // 0x20命令，读取扇区

    waitdisk();

    insl(0x1F0, dst, SECTSIZE / 4);     // 读取数据到dst
}
```

readseg包装readset，支持读取多个段。

## 4.2

```c
readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0); //读取磁盘的第一个扇区

//如果e_magic位不等于ELF_MAGIC，也就是不为有效的ELF文件
if (ELFHDR->e_magic != ELF_MAGIC) { 
    goto bad;
}

struct proghdr *ph, *eph;
ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff); //第一段程序的头
eph = ph + ELFHDR->e_phnum; //最后一段程序的头
for (; ph < eph; ph ++) { 	//将程序都载入到内存中
    readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
}

// 根据ELF头部的入口信息找到内核入口并执行。
((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```

## 练习5

通过阅读实验指导书并参考注释的提示，实现如下

```c
void print_stackframe(void) {
    int i;    
    uint32_t ebp, eip;
    ebp = read_ebp();
    eip = read_eip();
    for(i = 0; i < STACKFRAME_DEPTH; ++i) {
    	cprintf("ebp:0x%08x eip:0x%08x ", ebp, eip);
    	cprintf("args:0x%08x 0x%08x 0x%08x 0x%08x", *((uint32_t*)ebp+2), *((uint32_t*)ebp+3), *((uint32_t*)ebp+4), *((uint32_t*)ebp+5));
    	cprintf("\n");
		print_debuginfo(eip-1);
		eip = *((uint32_t*)ebp+1);
		ebp = *((uint32_t*)ebp);
		if(ebp == 0) 	return;
    }
}
```

打印消息如下，可见正确完成了print_stackframe的实现

```
Kernel executable memory footprint: 64KB
ebp:0x00007b38 eip:0x00100a3d args:0x00010094 0x00010094 0x00007b68 0x0010007f
    kern/debug/kdebug.c:297: print_stackframe+22
ebp:0x00007b48 eip:0x00100d34 args:0x00000000 0x00000000 0x00000000 0x00007bb8
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b68 eip:0x0010007f args:0x00000000 0x00007b90 0xffff0000 0x00007b94
    kern/init/init.c:48: grade_backtrace2+19
ebp:0x00007b88 eip:0x001000a1 args:0x00000000 0xffff0000 0x00007bb4 0x00000029
    kern/init/init.c:53: grade_backtrace1+27
ebp:0x00007ba8 eip:0x001000be args:0x00000000 0x00100000 0xffff0000 0x00100043
    kern/init/init.c:58: grade_backtrace0+19
ebp:0x00007bc8 eip:0x001000df args:0x00000000 0x00000000 0x00000000 0x00103280
    kern/init/init.c:63: grade_backtrace+26
ebp:0x00007be8 eip:0x00100050 args:0x00000000 0x00000000 0x00000000 0x00007c4f
    kern/init/init.c:28: kern_init+79
ebp:0x00007bf8 eip:0x00007d6e args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007d6d --
```

最后一行对应的是第一个使用堆栈的函数，为bootmain.c中的bootmain函数。bootloader设置的堆栈从0x7c00开始，使用”call bootmain”转入bootmain函数。 call指令压栈，所以bootmain中ebp为0x7bf8。如下为0x7d6e附近的反汇编的代码，可见0x7d6e为bootmain执行完成后的返回地址。四个参数并不是bootasm.S中传入的，没有什么意义。

```asm
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
    7d62:	a1 18 00 01 00       	mov    0x10018,%eax
    7d67:	25 ff ff ff 00       	and    $0xffffff,%eax
    7d6c:	ff d0                	call   *%eax
}
static inline void
outw(uint16_t port, uint16_t data) {
    asm volatile ("outw %0, %1" :: "a" (data), "d" (port));
    7d6e:	ba 00 8a ff ff       	mov    $0xffff8a00,%edx
```

## 练习6

## 6.1

在kern/mm/mmu.h中中断描述符表定义如下：

```c
/* Gate descriptors for interrupts and traps */
struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
};
```

可知一个表项占8个字节，其中第0-1和6-7位拼成偏移量，第2~3位位段选择子，去GDT中便可找到对应的基地址，基地址加上偏移量便是中断处理代码的入口。

## 6.2

根据实验指导书和注释，实现如下

```c
void idt_init(void) {
    extern uintptr_t __vectors[]; //ISR
    int i;
    for (i = 0; i < 256; ++i) {  //根据__vector建立IDT表项
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
    }
    SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK],DPL_USER);
    lidt(&idt_pd);  //导入IDT表项给CPU使用
}
```

## 6.3

实现如下，每次遇到时钟中断clock中的ticks变量加1，然后判断是否为100的倍数，是的话调用print_ticks函数打印“100 ticks”.

```c
case IRQ_OFFSET + IRQ_TIMER:		
    ++ticks;
    if (ticks % TICK_NUM == 0) {
        print_ticks();
    }
    break;
```

qemu和Terminal中显示的打印消息说明实现正常，部分打印消息如下。

```
100 ticks
100 ticks
kbd [104] h
kbd [000] 
kbd [101] e
kbd [000] 
kbd [108] l
kbd [000] 
kbd [108] l
kbd [000] 
kbd [111] o
kbd [000] 
100 ticks
```
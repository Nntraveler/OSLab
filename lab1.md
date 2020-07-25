# Lab 1

## Exercise 1~4

* 配置环境，了解汇编语言
* 练习gdb使用
* 复习C语言指针

### Answer for Quetions in Exercise 3

> Q1 At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

* 从`boot.S` 的第57行，即.code32后开始执行32位代码

* ```assembly
  ljmp    $PROT_MODE_CSEG, $protcseg
  ```

  * 该行代码导致切换到32位模式

> Q2 What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?

* boot loader最后执行的指令为

  * ```assembly
    7d6b:	ff 15 18 00 01 00    	call   *0x10018
    ```

* kernal在load后执行的第一条指令是

  * ```assembly
    f010000c:	66 c7 05 72 04 00 00 	movw   $0x1234,0x472
    ```

> Q3 *Where* is the first instruction of the kernel?

* 在执行过程中 0x10018所存的值是
  * ![image-20200723193312187](D:\study\OS\lab\lab1\image-20200723193312187.png)
  * 所以kernal的第一条指令就是位于0x001000c处的指令

> Q4 How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?

* 通过读ELF头，获取第一个段以及总段数的信息

```c
	// read 1st page off disk
	readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);

	// is this a valid ELF?
	if (ELFHDR->e_magic != ELF_MAGIC)
		goto bad;

	// load each program segment (ignores ph flags)
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
	eph = ph + ELFHDR->e_phnum;
	for (; ph < eph; ph++)
		// p_pa is the load address of this segment (as well
		// as the physical address)
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);

```

## Exercise 5

> Trace through the first few instructions of the boot loader again and identify the first instruction that would "break" or otherwise do the wrong thing if you were to get the boot loader's link address wrong. Then change the link address in `boot/Makefrag` to something wrong, run make clean, recompile the lab with make, and trace into the boot loader again to see what happens. Don't forget to change the link address back and make clean again afterward!

* 在改动链接地址后，会造成链接地址与加载地址产生不一致。因此当执行` ljmp  $PROT_MODE_CSEG, $protcseg`时则会造成错误，因为boot loader尝试去按照链接地址寻找 `$protcseg`，而该地址的指令却不是所期望的，所以会产生问题。
* 下面将makefrag中链接地址改为0x7C04后 gdb给出信息如下
  * ![image-20200724103359331](D:\study\OS\lab\lab1\image-20200724103359331.png)
  * 证实的确是在该条指令处出错。

## Exercise 6

> Reset the machine (exit QEMU/GDB and start them again). Examine the 8 words of memory at 0x00100000 at the point the BIOS enters the boot loader, and then again at the point the boot loader enters the kernel. Why are they different? What is there at the second breakpoint? (You do not really need to use QEMU to answer this question. Just think.)

* 根据推测 在刚进入boot loader时，0x00100000处应该没有任何内容，而当进入kernel后，boot loader加载了kernel，修改了该处内容，因此存在数据。
* 验证如下
  * ![image-20200724104340498](D:\study\OS\lab\lab1\image-20200724104340498.png)

## Exercise 7

> Use QEMU and GDB to trace into the JOS kernel and stop at the `movl %eax, %cr0`. Examine memory at 0x00100000 and at 0xf0100000. Now, single step over that instruction using the stepi GDB command. Again, examine memory at 0x00100000 and at 0xf0100000. Make sure you understand what just happened.

* ![image-20200724124542156](D:\study\OS\lab\lab1\image-20200724124542156.png)

* 由于在该条语句前还没有开启分页，f0100000处还未映射到00100000，而在执行完毕后，两处的内容一致，因为f0100000已经被映射到物理地址00100000。

> What is the first instruction *after* the new mapping is established that would fail to work properly if the mapping weren't in place? Comment out the `movl %eax, %cr0` in `kern/entry.S`, trace into it, and see if you were right.

* 如果没有进行分页 那么`mov  $relocated, %eax`就会失败，因为$relocated是在内存空间外的，如果没有映射就会报错
* 验证如下
  * ![image-20200724125048145](D:\study\OS\lab\lab1\image-20200724125048145.png)
  * 可以看到 访问0xf010002c出错了

## Exercise 8

> We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment.

* 仿照case 'x': 修改如下

```c
        // (unsigned) octal
		case 'o':
			// Replace this with your code.
			num = getuint(&ap, lflag)
			base = 8;
			goto number;
			break;
```

### Answer for quetions in Exercise 8

> Q1 Explain the interface between `printf.c` and `console.c`. Specifically, what function does `console.c` export? How is this function used by `printf.c`?

* `console.c`提供了`cputchar` 当`printf.c`调用`putch`函数时会调用`cputchar`

> Q2 Explain the following from `console.c`:
>
> ```c
> 1      if (crt_pos >= CRT_SIZE) {
> 2              int i;
> 3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) * sizeof(uint16_t));
> 4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
> 5                      crt_buf[i] = 0x0700 | ' ';
> 6              crt_pos -= CRT_COLS;
> 7      }
> ```

* 该部分代码实现了 当内容超出屏幕范围后，滚动一行的功能。

> Q3 Trace the execution of the following code step-by-step:
>
> ```
> int x = 1, y = 3, z = 4;
> cprintf("x %d, y %x, z %d\n", x, y, z);
> ```
>
> - In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?
> - List (in order of execution) each call to `cons_putc`, `va_arg`, and `vcprintf`. For `cons_putc`, list its argument as well. For `va_arg`, list what `ap` points to before and after the call. For `vcprintf` list the values of its two arguments.

* 在调用cprintf()时，`fmt`指向格式字符串`"x %d, y %x, z %d\n"` `ap`指向变量x,y,z

* 调用过程如下

  * ```
    vcprintf (fmt=0xf0101ad2 "x %d, y %x, z %d\n", ap=0xf0115f64 "\001")
    cons_putc (c=120)
    cons_putc (c=32)
    va_arg(*ap, int) # ap = "\001"
    cons_putc (c=49)
    cons_putc (c=44)
    cons_putc (c=32)
    cons_putc (c=121)
    cons_putc (c=32)
    va_arg(*ap, int) # ap = "\003"
    cons_putc (c=51)
    cons_putc (c=44)
    cons_putc (c=32)
    cons_putc (c=122)
    cons_putc (c=32)
    va_arg(*ap, int) # ap = "\004"
    cons_putc (c=52)
    cons_putc (c=10)
    ```

> Q4 Run the following code.
>
> ```c
>     unsigned int i = 0x00646c72;
>     cprintf("H%x Wo%s", 57616, &i);
> ```

* 实际输出 `He110 World`，由于57616=0xe110，通过%x打印出来就是e110，将i拆分成 72 6c 64 00打印出来就是 rld，如果换成bit-endian 那么就需要将i倒置，57616无需变化

> Q5 In the following code, what is going to be printed after `'y='`? (note: the answer is not a specific value.) Why does this happen?
>
> ```c
>     cprintf("x=%d y=%d", 3);
> ```

* 打印出来的是随机的数字，由于缺少参数，所以就会打印第一个参数上的4字节内容，而该处内容是未定义的。

> Q6 Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?

* 需要增加一个表示个数的参数，从而保证顺序正确。

## Exercise 9

> Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

* 初始化栈位于entry.S文件中74~77行以及86~95行

  * ```assembly
    relocated:
    
    	# Clear the frame pointer register (EBP)
    	# so that once we get into debugging C code,
    	# stack backtraces will be terminated properly.
    	movl	$0x0,%ebp			# nuke frame pointer
    
    	# Set the stack pointer
    	movl	$(bootstacktop),%esp
    
    	# now to C code
    	call	i386_init
    
    	# Should never get here, but in case we do, just spin.
    spin:	jmp	spin
    
    
    .data
    ###################################################################
    # boot stack
    ###################################################################
    	.p2align	PGSHIFT		# force page alignment
    	.globl		bootstack
    bootstack:
    	.space		KSTKSIZE
    	.globl		bootstacktop   
    bootstacktop:
    ```

* 大小为KSTKSIZE，栈顶是0xf0110000，由高位向低位存储

## Exercise 10

> To become familiar with the C calling conventions on the x86, find the address of the `test_backtrace` function in `obj/kern/kernel.asm`, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of `test_backtrace` push on the stack, and what are those words?

* ![image-20200724184849782](D:\study\OS\lab\lab1\image-20200724184849782.png)
* 可以看到 两次调用 ebp变化了0x20即每次压入栈的数据量
* ebp保存上一层程序ebp寄存器的值，而esp则存放下一层子程序调用的参数

## Exercise 12

> Modify your stack backtrace function to display, for each `eip`, the function name, source file name, and line number corresponding to that `eip`.

* ![image-20200724221805952](D:\study\OS\lab\lab1\image-20200724221805952.png)
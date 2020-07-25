# Lab 1

## Exercise 1~4

* ���û������˽�������
* ��ϰgdbʹ��
* ��ϰC����ָ��

### Answer for Quetions in Exercise 3

> Q1 At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

* ��`boot.S` �ĵ�57�У���.code32��ʼִ��32λ����

* ```assembly
  ljmp    $PROT_MODE_CSEG, $protcseg
  ```

  * ���д��뵼���л���32λģʽ

> Q2 What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?

* boot loader���ִ�е�ָ��Ϊ

  * ```assembly
    7d6b:	ff 15 18 00 01 00    	call   *0x10018
    ```

* kernal��load��ִ�еĵ�һ��ָ����

  * ```assembly
    f010000c:	66 c7 05 72 04 00 00 	movw   $0x1234,0x472
    ```

> Q3 *Where* is the first instruction of the kernel?

* ��ִ�й����� 0x10018�����ֵ��
  * ![image-20200723193312187](D:\study\OS\lab\lab1\image-20200723193312187.png)
  * ����kernal�ĵ�һ��ָ�����λ��0x001000c����ָ��

> Q4 How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?

* ͨ����ELFͷ����ȡ��һ�����Լ��ܶ�������Ϣ

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

* �ڸĶ����ӵ�ַ�󣬻�������ӵ�ַ����ص�ַ������һ�¡���˵�ִ��` ljmp  $PROT_MODE_CSEG, $protcseg`ʱ�����ɴ�����Ϊboot loader����ȥ�������ӵ�ַѰ�� `$protcseg`�����õ�ַ��ָ��ȴ�����������ģ����Ի�������⡣
* ���潫makefrag�����ӵ�ַ��Ϊ0x7C04�� gdb������Ϣ����
  * ![image-20200724103359331](D:\study\OS\lab\lab1\image-20200724103359331.png)
  * ֤ʵ��ȷ���ڸ���ָ�����

## Exercise 6

> Reset the machine (exit QEMU/GDB and start them again). Examine the 8 words of memory at 0x00100000 at the point the BIOS enters the boot loader, and then again at the point the boot loader enters the kernel. Why are they different? What is there at the second breakpoint? (You do not really need to use QEMU to answer this question. Just think.)

* �����Ʋ� �ڸս���boot loaderʱ��0x00100000��Ӧ��û���κ����ݣ���������kernel��boot loader������kernel���޸��˸ô����ݣ���˴������ݡ�
* ��֤����
  * ![image-20200724104340498](D:\study\OS\lab\lab1\image-20200724104340498.png)

## Exercise 7

> Use QEMU and GDB to trace into the JOS kernel and stop at the `movl %eax, %cr0`. Examine memory at 0x00100000 and at 0xf0100000. Now, single step over that instruction using the stepi GDB command. Again, examine memory at 0x00100000 and at 0xf0100000. Make sure you understand what just happened.

* ![image-20200724124542156](D:\study\OS\lab\lab1\image-20200724124542156.png)

* �����ڸ������ǰ��û�п�����ҳ��f0100000����δӳ�䵽00100000������ִ����Ϻ�����������һ�£���Ϊf0100000�Ѿ���ӳ�䵽�����ַ00100000��

> What is the first instruction *after* the new mapping is established that would fail to work properly if the mapping weren't in place? Comment out the `movl %eax, %cr0` in `kern/entry.S`, trace into it, and see if you were right.

* ���û�н��з�ҳ ��ô`mov  $relocated, %eax`�ͻ�ʧ�ܣ���Ϊ$relocated�����ڴ�ռ���ģ����û��ӳ��ͻᱨ��
* ��֤����
  * ![image-20200724125048145](D:\study\OS\lab\lab1\image-20200724125048145.png)
  * ���Կ��� ����0xf010002c������

## Exercise 8

> We have omitted a small fragment of code - the code necessary to print octal numbers using patterns of the form "%o". Find and fill in this code fragment.

* ����case 'x': �޸�����

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

* `console.c`�ṩ��`cputchar` ��`printf.c`����`putch`����ʱ�����`cputchar`

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

* �ò��ִ���ʵ���� �����ݳ�����Ļ��Χ�󣬹���һ�еĹ��ܡ�

> Q3 Trace the execution of the following code step-by-step:
>
> ```
> int x = 1, y = 3, z = 4;
> cprintf("x %d, y %x, z %d\n", x, y, z);
> ```
>
> - In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?
> - List (in order of execution) each call to `cons_putc`, `va_arg`, and `vcprintf`. For `cons_putc`, list its argument as well. For `va_arg`, list what `ap` points to before and after the call. For `vcprintf` list the values of its two arguments.

* �ڵ���cprintf()ʱ��`fmt`ָ���ʽ�ַ���`"x %d, y %x, z %d\n"` `ap`ָ�����x,y,z

* ���ù�������

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

* ʵ����� `He110 World`������57616=0xe110��ͨ��%x��ӡ��������e110����i��ֳ� 72 6c 64 00��ӡ�������� rld���������bit-endian ��ô����Ҫ��i���ã�57616����仯

> Q5 In the following code, what is going to be printed after `'y='`? (note: the answer is not a specific value.) Why does this happen?
>
> ```c
>     cprintf("x=%d y=%d", 3);
> ```

* ��ӡ����������������֣�����ȱ�ٲ��������Ծͻ��ӡ��һ�������ϵ�4�ֽ����ݣ����ô�������δ����ġ�

> Q6 Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?

* ��Ҫ����һ����ʾ�����Ĳ������Ӷ���֤˳����ȷ��

## Exercise 9

> Determine where the kernel initializes its stack, and exactly where in memory its stack is located. How does the kernel reserve space for its stack? And at which "end" of this reserved area is the stack pointer initialized to point to?

* ��ʼ��ջλ��entry.S�ļ���74~77���Լ�86~95��

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

* ��СΪKSTKSIZE��ջ����0xf0110000���ɸ�λ���λ�洢

## Exercise 10

> To become familiar with the C calling conventions on the x86, find the address of the `test_backtrace` function in `obj/kern/kernel.asm`, set a breakpoint there, and examine what happens each time it gets called after the kernel starts. How many 32-bit words does each recursive nesting level of `test_backtrace` push on the stack, and what are those words?

* ![image-20200724184849782](D:\study\OS\lab\lab1\image-20200724184849782.png)
* ���Կ��� ���ε��� ebp�仯��0x20��ÿ��ѹ��ջ��������
* ebp������һ�����ebp�Ĵ�����ֵ����esp������һ���ӳ�����õĲ���

## Exercise 12

> Modify your stack backtrace function to display, for each `eip`, the function name, source file name, and line number corresponding to that `eip`.

* ![image-20200724221805952](D:\study\OS\lab\lab1\image-20200724221805952.png)
# Lab 3

## Exercise 1

> Modify `mem_init()` in `kern/pmap.c` to allocate and map the `envs` array. This array consists of exactly `NENV` instances of the `Env` structure allocated much like how you allocated the `pages` array. Also like the `pages` array, the memory backing `envs` should also be mapped user read-only at `UENVS` (defined in `inc/memlayout.h`) so user processes can read from this array.
>

### `mem_init`

```c
	//////////////////////////////////////////////////////////////////////
	// Make 'envs' point to an array of size 'NENV' of 'struct Env'.
	// LAB 3: Your code here.
	envs = (struct Env *)boot_alloc(NENV * sizeof(struct Env));
    memset(envs, 0, NENV * sizeof(struct Env));

//////////////////////////////////////////////////////////////////////
	// Map the 'envs' array read-only by the user at linear address UENVS
	// (ie. perm = PTE_U | PTE_P).
	// Permissions:
	//    - the new image at UENVS  -- kernel R, user R
	//    - envs itself -- kernel RW, user NONE
	// LAB 3: Your code here.
	boot_map_region(kern_pgdir, UENVS, PTSIZE, PADDR(envs), PTE_U);

```

## Exercise 2

> In the file `env.c`, finish coding the following functions:
>
> - `env_init()`
>
>   Initialize all of the `Env` structures in the `envs` array and add them to the `env_free_list`. Also calls `env_init_percpu`, which configures the segmentation hardware with separate segments for privilege level 0 (kernel) and privilege level 3 (user).
>
> - `env_setup_vm()`
>
>   Allocate a page directory for a new environment and initialize the kernel portion of the new environment's address space.
>
> - `region_alloc()`
>
>   Allocates and maps physical memory for an environment
>
> - `load_icode()`
>
>   You will need to parse an ELF binary image, much like the boot loader already does, and load its contents into the user address space of a new environment.
>
> - `env_create()`
>
>   Allocate an environment with `env_alloc` and call `load_icode` to load an ELF binary into it.
>
> - `env_run()`
>
>   Start a given environment running in user mode.

### `env_init`

```c
// LAB 3: Your code here.
	env_free_list = NULL;
	for(int i = NENV - 1; i >= 0; i--)
	{
		envs[i].env_status = ENV_FREE;
		envs[i].env_id = 0;
		envs[i].env_link = env_free_list;
		env_free_list = &envs[i];
	}
	// Per-CPU part of the initialization
	env_init_percpu();
```

采用后插法将envs表变为链表，如果不使用后插法顺序会出现错误

### `env_setup_vm`

```c
    // LAB 3: Your code here.
	p->pp_ref++;
    e->env_pgdir = (pde_t *)page2kva(p);
    memcpy(e->env_pgdir, kern_pgdir, PGSIZE);
	// UVPT maps the env's own page table read-only.
	// Permissions: kernel R, user R
	e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;
```

### `region_alloc`

```c
	// LAB 3: Your code here.
	// (But only if you need it for load_icode.)
	//
	// Hint: It is easier to use region_alloc if the caller can pass
	//   'va' and 'len' values that are not page-aligned.
	//   You should round va down, and round (va + len) up.
	//   (Watch out for corner-cases!)
	void* start = (void*)ROUNDDOWN((uint32_t)va, PGSIZE);
	void* end =  (void*)ROUNDUP((uint32_t)va + len, PGSIZE);
	int r;
	for (void *i = start; i < end; i += PGSIZE) 
	{
		struct PageInfo* p = page_alloc(0); //not initialized
		if(p == NULL)
			panic("region_alloc: allocation failed\n");
		r = page_insert(e->env_pgdir, p, i, PTE_U | PTE_W);
		if(r != 0)
			panic("region_alloc: %e\n", r);
	}
```

### `load_icode`

```c
	// LAB 3: Your code here.
	struct Elf *env_elf;
	struct Proghdr *ph, *eph;
	env_elf = (struct Elf*) binary;
	ph = (struct Proghdr*)((uint8_t*)(env_elf) + env_elf->e_phoff);
    eph = ph + env_elf->e_phnum;

    lcr3(PADDR(e->env_pgdir));
    for (; ph < eph; ph++) 
    {
        if(ph->p_type == ELF_PROG_LOAD) 
        {
            region_alloc(e, (void *)ph->p_va, ph->p_memsz);
            memcpy((void*)ph->p_va, (void *)(binary+ph->p_offset), ph->p_filesz);
            memset((void*)(ph->p_va + ph->p_filesz), 0, ph->p_memsz-ph->p_filesz);
        }
    }
    e->env_tf.tf_eip = env_elf->e_entry;
    lcr3(PADDR(kern_pgdir));
	// Now map one page for the program's initial stack
	// at virtual address USTACKTOP - PGSIZE.

	// LAB 3: Your code here.
	region_alloc(e, (void *)(USTACKTOP-PGSIZE), PGSIZE);
```

### `env_create`

```c
	// LAB 3: Your code here.
	struct Env *e;
	int r = env_alloc(&e, 0);
	if(r != 0)
	{
		panic("env_create: %e\n", r);
	}
	load_icode(e, binary);
	e->env_type = type;
```

### `env_run`

```c
	// LAB 3: Your code here.

	// panic("env_run not yet implemented");
	if(curenv != NULL && curenv->env_status == ENV_RUNNING)
	{
		e->env_status = ENV_RUNNABLE;
	}
	curenv = e;
	e->env_status = ENV_RUNNING;
	e->env_runs++;
	lcr3(PADDR(e->env_pgdir));
	env_pop_tf(&e->env_tf);
```

## Exercise 3

学习异常与中断相关知识

## Exercise 4

> Edit `trapentry.S` and `trap.c` and implement the features described above. The macros `TRAPHANDLER` and `TRAPHANDLER_NOEC` in `trapentry.S` should help you, as well as the T_* defines in `inc/trap.h`. You will need to add an entry point in `trapentry.S` (using those macros) for each trap defined in `inc/trap.h` ,and you'll have to provide ` _alltraps` which the `TRAPHANDLER` macros refer to. You will also need to modify `trap_init()` to initialize the `idt` to point to each of these entry points defined in `trapentry.S`; the `SETGATE` macro will be helpful here.

### `trapentry.s`

```assembly
/*
 * Lab 3: Your code here for generating entry points for the different traps.
 */
	TRAPHANDLER_NOEC(th0, 0)
	TRAPHANDLER_NOEC(th1, 1)
	TRAPHANDLER_NOEC(th3, 3)
	TRAPHANDLER_NOEC(th4, 4)
	TRAPHANDLER_NOEC(th5, 5)
	TRAPHANDLER_NOEC(th6, 6)
	TRAPHANDLER_NOEC(th7, 7)
	TRAPHANDLER(th8, 8)
	TRAPHANDLER_NOEC(th9, 9)
	TRAPHANDLER(th10, 10)
	TRAPHANDLER(th11, 11)
	TRAPHANDLER(th12, 12)
	TRAPHANDLER(th13, 13)
	TRAPHANDLER(th14, 14)
	TRAPHANDLER_NOEC(th16, 16)

	TRAPHANDLER_NOEC(th_syscall, T_SYSCALL)


/*
 * Lab 3: Your code here for _alltraps
 */

_alltraps:
	pushl %ds
	pushl %es
	pushal
	pushl $GD_KD
	popl %ds
	pushl $GD_KD
	popl %es
	pushl %esp
	call trap  

```

### `trap.c`

```c
void
trap_init(void)
{
	extern struct Segdesc gdt[];

	// LAB 3: Your code here.
	void th0();
	void th1();
	void th3();
	void th4();
	void th5();
	void th6();
	void th7();
	void th8();
	void th9();
	void th10();
	void th11();
	void th12();
	void th13();
	void th14();
	void th16();
	void th_syscall();
	SETGATE(idt[0], 0, GD_KT, th0, 0);
	SETGATE(idt[1], 0, GD_KT, th1, 0); 
	SETGATE(idt[3], 0, GD_KT, th3, 3);
	SETGATE(idt[4], 0, GD_KT, th4, 0);
	SETGATE(idt[5], 0, GD_KT, th5, 0);
	SETGATE(idt[6], 0, GD_KT, th6, 0);
	SETGATE(idt[7], 0, GD_KT, th7, 0);
	SETGATE(idt[8], 0, GD_KT, th8, 0);
	SETGATE(idt[9], 0, GD_KT, th9, 0);
	SETGATE(idt[10], 0, GD_KT, th10, 0);
	SETGATE(idt[11], 0, GD_KT, th11, 0);
	SETGATE(idt[12], 0, GD_KT, th12, 0);
	SETGATE(idt[13], 0, GD_KT, th13, 0);
	SETGATE(idt[14], 0, GD_KT, th14, 0);
	SETGATE(idt[16], 0, GD_KT, th16, 0);
	SETGATE(idt[T_SYSCALL], 0, GD_KT, th_syscall, 3);
	// Per-CPU setup 
	trap_init_percpu();
}
```

## Exercise 5

> Modify `trap_dispatch()` to dispatch page fault exceptions to `page_fault_handler()`. You should now be able to get make grade to succeed on the `faultread`, `faultreadkernel`, `faultwrite`, and `faultwritekernel` tests. If any of them don't work, figure out why and fix them. Remember that you can boot JOS into a particular user program using make run-*x* or make run-*x*-nox. For instance, make run-hello-nox runs the *hello* user program.

### `trap_dispatch`

```c
    // Handle processor exceptions.
	// LAB 3: Your code here.
	if (tf->tf_trapno == T_PGFLT) 
	{
		page_fault_handler(tf);
		return;
	}
```

## Exercise 6

> Modify `trap_dispatch()` to make breakpoint exceptions invoke the kernel monitor. You should now be able to get make grade to succeed on the `breakpoint` test.

### `trap_dispatch`

```c
	if (tf->tf_trapno == T_BRKPT) 
	{
		monitor(tf);
		return;
	}
```

## Exercise 7

> Add a handler in the kernel for interrupt vector `T_SYSCALL`. You will have to edit `kern/trapentry.S` and `kern/trap.c`'s `trap_init()`. You also need to change `trap_dispatch()` to handle the system call interrupt by calling `syscall()` (defined in `kern/syscall.c`) with the appropriate arguments, and then arranging for the return value to be passed back to the user process in `%eax`. Finally, you need to implement `syscall()` in `kern/syscall.c`. Make sure `syscall()` returns `-E_INVAL` if the system call number is invalid. You should read and understand `lib/syscall.c` (especially the inline assembly routine) in order to confirm your understanding of the system call interface. Handle all the system calls listed in `inc/syscall.h` by invoking the corresponding kernel function for each call.

### `trapentry.S`

在原有基础上加入该行

```assembly
	TRAPHANDLER_NOEC(th_syscall, T_SYSCALL)
```

### `trap_init`

在原有基础上对应位置加入两行

```c
	void th_syscall();
    // ...
	SETGATE(idt[T_SYSCALL], 0, GD_KT, th_syscall, 3);
```

### `trap_dispatch`

```c
	if (tf->tf_trapno == T_SYSCALL) 
	{
		tf->tf_regs.reg_eax = syscall(tf->tf_regs.reg_eax, tf->tf_regs.reg_edx, tf->tf_regs.reg_ecx,
			tf->tf_regs.reg_ebx, tf->tf_regs.reg_edi, tf->tf_regs.reg_esi);
		return;
	}
```

## Exercise 8

> Add the required code to the user library, then boot your kernel. You should see `user/hello` print "`hello, world`" and then print "`i am environment 00001000`". `user/hello` then attempts to "exit" by calling `sys_env_destroy()` (see `lib/libmain.c` and `lib/exit.c`). Since the kernel currently only supports one user environment, it should report that it has destroyed the only environment and then drop into the kernel monitor.

### `libmain`

```c
	// set thisenv to point at our Env structure in envs[].
	// LAB 3: Your code here.
	envid_t envid = sys_getenvid();  
	thisenv = envs + ENVX(envid);  
```

## Exercise 9

> Change `kern/trap.c` to panic if a page fault happens in kernel mode.
>
> Hint: to determine whether a fault happened in user mode or in kernel mode, check the low bits of the `tf_cs`.
>
> Read `user_mem_assert` in `kern/pmap.c` and implement `user_mem_check` in that same file.
>
> Change `kern/syscall.c` to sanity check arguments to system calls.

### `page_fault_handler`

```c
// LAB 3: Your code here.
	if ((tf->tf_cs & 3) == 0)
		panic("page_fault_handler(): page fault in kernel mode!\n");
```

### `user_mem_check`

```c
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
	// LAB 3: Your code here.
	cprintf("user_mem_check va: %x, len: %x\n", va, len);
	uint32_t begin = (uint32_t) ROUNDDOWN(va, PGSIZE); 
	uint32_t end = (uint32_t) ROUNDUP(va+len, PGSIZE);
	uint32_t i;
	for (i = (uint32_t)begin; i < end; i += PGSIZE) 
    {
		pte_t *pte = pgdir_walk(env->env_pgdir, (void*)i, 0);
		if ((i >= ULIM) || !pte || !(*pte & PTE_P) || ((*pte & perm) != perm))  // check rules
		{        
			user_mem_check_addr = (i < (uint32_t)va ? (uint32_t)va : i);    // check invalid linear addr
			return -E_FAULT;
		}
	}
	cprintf("user_mem_check success va: %x, len: %x\n", va, len);
	return 0;
}
```

### `sys_cputs`

```c
	// LAB 3: Your code here.
	user_mem_assert(curenv, s, len, 0);
```

### `syscall`

```c
    // Call the function corresponding to the 'syscallno' parameter.
	// Return any appropriate return value.
	// LAB 3: Your code here.
	int32_t ret;
	switch (syscallno) 
    { 
		case SYS_cputs:
			sys_cputs((char *)a1, (size_t)a2);
			ret = 0;
			break;
		case SYS_cgetc:
			ret = sys_cgetc();
			break;
		case SYS_getenvid:
			ret = sys_getenvid();
			break;
		case SYS_env_destroy:
			ret = sys_env_destroy((envid_t)a1);
			break;
		default:
			return -E_INVAL;
	}

	return ret;
```

### `debuginfo_eip`

```c
		// Make sure this memory is valid.
		// Return -1 if it is not.  Hint: Call user_mem_check.
		// LAB 3: Your code here.
		if (user_mem_check(curenv, usd, sizeof(struct UserStabData), PTE_U))
		{
			return -1;
		} 
		stabs = usd->stabs;
		stab_end = usd->stab_end;
		stabstr = usd->stabstr;
		stabstr_end = usd->stabstr_end;

		// Make sure the STABS and string table memory is valid.
		// LAB 3: Your code here.
		if (user_mem_check(curenv, stabs, stab_end - stabs, PTE_U))
		{
			return -1;
		}
		if (user_mem_check(curenv, stabstr, stabstr_end - stabstr, PTE_U))
		{
    		return -1;
		}
```

## Exercise 10

> Boot your kernel, running `user/evilhello`. The environment should be destroyed, and the kernel should not panic. You should see:
>
> ```bash
> 	[00000000] new env 00001000
> 	...
> 	[00001000] user_mem_check assertion failure for va f010000c
> 	[00001000] free env 00001000
> ```

![image-20200824162344210](D:\study\OS\lab\lab3\image-20200824162344210.png)

## Questions

### Q1

> What is the purpose of having an individual handler function for each exception/interrupt? (i.e., if all exceptions/interrupts were delivered to the same handler, what feature that exists in the current implementation could not be provided?)

A: 采用不同的处理函数可以用以区分处理不同的异常，根据类型决定函数。

### Q2

> Did you have to do anything to make the `user/softint` program behave correctly? The grade script expects it to produce a general protection fault (trap 13), but `softint`'s code says `int $14`. *Why* should this produce interrupt vector 13? What happens if the kernel actually allows `softint`'s `int $14` instruction to invoke the kernel's page fault handler (which is interrupt vector 14)?

A: 用户程序无法触发中断向量14，仅内核程序可以触发。

## Result

![image-20200821023114921](D:\study\OS\lab\lab3\image-20200821023114921.png)
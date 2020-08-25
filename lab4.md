# Lab 4
## Exercise 1
> Implement `mmio_map_region` in `kern/pmap.c`. To see how this is used, look at the beginning of `lapic_init` in `kern/lapic.c`. You'll have to do the next exercise, too, before the tests for `mmio_map_region` will run.

### `mmio_map_region`

```c
	// Your code here:
	// panic("mmio_map_region not implemented");
    size_t begin = ROUNDDOWN(pa, PGSIZE), end = ROUNDUP(pa + size, PGSIZE);
    size_t map_size = end - begin;
    if (base + map_size >= MMIOLIM) {
        panic("mmio_map_region: overflow MMIOLIM");
    }    
    boot_map_region(kern_pgdir, base, map_size, pa, PTE_PCD|PTE_PWT|PTE_W);
    uintptr_t result = base;
    base += map_size;
    return (void *)result;
```

## Exercise 2
> Read `boot_aps()` and `mp_main()` in `kern/init.c`, and the assembly code in `kern/mpentry.S`. Make sure you understand the control flow transfer during the bootstrap of APs. Then modify your implementation of `page_init()` in `kern/pmap.c` to avoid adding the page at `MPENTRY_PADDR` to the free list, so that we can safely copy and run AP bootstrap code at that physical address. Your code should pass the updated `check_page_free_list()` test (but might fail the updated `check_kern_pgdir()` test, which we will fix soon).

### `page_init`

```c
    size_t i, mp_page = PGNUM(MPENTRY_PADDR);
    for (i = 1; i < npages_basemem; i++) {
        if (i == mp_page) continue;
```

## Exercise 3
> Modify `mem_init_mp()` (in `kern/pmap.c`) to map per-CPU stacks starting at `KSTACKTOP`, as shown in `inc/memlayout.h`. The size of each stack is `KSTKSIZE` bytes plus `KSTKGAP` bytes of unmapped guard pages. Your code should pass the new check in `check_kern_pgdir()`.

### `mem_init_mp`

```c
	// LAB 4: Your code here:
    for (int i = 0; i < NCPU; i++) {
        int kstacktop_i = KSTACKTOP - KSTKSIZE - i * (KSTKSIZE + KSTKGAP);
        boot_map_region(kern_pgdir, kstacktop_i, KSTKSIZE, PADDR(percpu_kstacks[i]), PTE_W);
    }
```

## Exercise 4
> The code in `trap_init_percpu()` (`kern/trap.c`) initializes the TSS and TSS descriptor for the BSP. It worked in Lab 3, but is incorrect when running on other CPUs. Change the code so that it can work on all CPUs. (Note: your new code should not use the global `ts` variable any more.)

### `trap_init_percpu`

```c
	// LAB 4: Your code here:
    int cpu_id = thiscpu->cpu_id;
    struct Taskstate *this_ts = &thiscpu->cpu_ts;
    this_ts->ts_esp0 = KSTACKTOP - cpu_id * (KSTKSIZE + KSTKGAP);
    this_ts->ts_ss0 = GD_KD;
    this_ts->ts_iomb = sizeof(struct Taskstate);

    gdt[(GD_TSS0 >> 3) + cpu_id] = SEG16(STS_T32A, (uint32_t) (this_ts),
                    sizeof(struct Taskstate) - 1, 0); 
    gdt[(GD_TSS0 >> 3) + cpu_id].sd_s = 0;
    ltr(GD_TSS0 + (cpu_id << 3));
    lidt(&idt_pd);
```

## Exercise 5
> Apply the big kernel lock as described above, by calling `lock_kernel()` and `unlock_kernel()` at the proper locations.



```c
i386_init(void)
 
        // Acquire the big kernel lock before waking up APs
        // Your code here:
        lock_kernel();
 
        // Starting non-boot CPUs
        boot_aps();

mp_main(void)
        // only one CPU can enter the scheduler at a time!
        //
        // Your code here:
        lock_kernel();
        sched_yield();
 
        // Remove this after you finish Exercise 6
        // for (;;);


trap(struct Trapframe *tf)
                // Acquire the big kernel lock before doing any
                // serious kernel work.
                // LAB 4: Your code here.
                lock_kernel();
                assert(curenv);
                

env_run(struct Env *e)
        lcr3(PADDR(curenv->env_pgdir));
        unlock_kernel();
        env_pop_tf(&curenv->env_tf);
 }
```



## Exercise 6
> Implement round-robin scheduling in `sched_yield()` as described above. Don't forget to modify `syscall()` to dispatch `sys_yield()`.
>
> Make sure to invoke `sched_yield()` in `mp_main`.

### `sched_yield`

```c
	// LAB 4: Your code here.
	int i, index = 0;
	if (curenv)
		index = ENVX(curenv->env_id);
	else
		index = 0;
	for(i = index; i != index + NENV ; i++) {
		if (envs[i%NENV].env_status == ENV_RUNNABLE)
		{
			env_run(&envs[i%NENV]);
		}
	}
	if(curenv && curenv->env_status == ENV_RUNNING) {
		env_run(curenv);
	}

	// sched_halt never returns
	sched_halt();
```

并在`syscall`函数中加入对应语句。

## Exercise 7
> Implement the system calls described above in `kern/syscall.c` and make sure `syscall()` calls them. You will need to use various functions in `kern/pmap.c` and `kern/env.c`, particularly `envid2env()`. For now, whenever you call `envid2env()`, pass 1 in the `checkperm` parameter. Be sure you check for any invalid system call arguments, returning `-E_INVAL` in that case. Test your JOS kernel with `user/dumbfork` and make sure it works before proceeding.

### `sys_exofork`

```c
static envid_t
sys_exofork(void)
{
	// Create the new environment with env_alloc(), from kern/env.c.
	// It should be left as env_alloc created it, except that
	// status is set to ENV_NOT_RUNNABLE, and the register set is copied
	// from the current environment -- but tweaked so sys_exofork
	// will appear to return 0.

	// LAB 4: Your code here.
	//panic("sys_exofork not implemented");
	struct Env *e; 
    int ret = env_alloc(&e, curenv->env_id);
    if (ret) return ret;

    e->env_status = ENV_NOT_RUNNABLE;
    e->env_tf = curenv->env_tf;
    e->env_tf.tf_regs.reg_eax = 0;
    return e->env_id;
}
```

### `sys_env_set_status`

```c
	// LAB 4: Your code here.
	//panic("sys_env_set_status not implemented");
	struct Env *e;
    if (envid2env(envid, &e, 1)) return -E_BAD_ENV;
    
    if (status != ENV_NOT_RUNNABLE && status != ENV_RUNNABLE) return -E_INVAL;
    
    e->env_status = status;
    return 0;
```

### `sys_page_alloc`

```c
	// LAB 4: Your code here.
	//panic("sys_page_alloc not implemented");
	struct Env *e;
    if (envid2env(envid, &e, 1) < 0) return -E_BAD_ENV;

    int valid_perm = (PTE_U|PTE_P);
    if (va >= (void *)UTOP || (perm & valid_perm) != valid_perm) {
        return -E_INVAL;
    }

    struct PageInfo *p = page_alloc(1);
    if (!p) return -E_NO_MEM;

    int ret = page_insert(e->env_pgdir, p, va, perm);
    if (ret) {
        page_free(p);
    }
    return ret;
```

### `sys_page_map`

```c
	// LAB 4: Your code here.
	//panic("sys_page_map not implemented");
	struct Env *srcenv, *dstenv;
    if (envid2env(srcenvid, &srcenv, 1) || envid2env(dstenvid, &dstenv, 1)) {
        return -E_BAD_ENV;
    }

    if (srcva >= (void *)UTOP || dstva >= (void *)UTOP || PGOFF(srcva) || PGOFF(dstva)) {
        return -E_INVAL;
    }

    pte_t *pte;
    struct PageInfo *p = page_lookup(srcenv->env_pgdir, srcva, &pte);
    if (!p) return -E_INVAL;

    int valid_perm = (PTE_U|PTE_P);
    if ((perm&valid_perm) != valid_perm) return -E_INVAL;

    if ((perm & PTE_W) && !(*pte & PTE_W)) return -E_INVAL;

    int ret = page_insert(dstenv->env_pgdir, p, dstva, perm);
    return ret;
```

### `sys_page_unmap`

```c
	// LAB 4: Your code here.
	//panic("sys_page_unmap not implemented");
	struct Env *e;
    if (envid2env(envid, &e, 1)) return -E_BAD_ENV;

    if (va >= (void *)UTOP) return -E_INVAL;

    page_remove(e->env_pgdir, va);
    return 0;
```

## Exercise 8
> Implement the `sys_env_set_pgfault_upcall` system call. Be sure to enable permission checking when looking up the environment ID of the target environment, since this is a "dangerous" system call.

### `sys_env_set_pgfault_upcall`

```c
static int
sys_env_set_pgfault_upcall(envid_t envid, void *func)
{
	// LAB 4: Your code here.
	// panic("sys_env_set_pgfault_upcall not implemented");
	struct Env *e; 
    if (envid2env(envid, &e, 1)) return -E_BAD_ENV;
    e->env_pgfault_upcall = func;
    return 0;
}
```

## Exercise 9
> Implement the code in `page_fault_handler` in `kern/trap.c` required to dispatch page faults to the user-mode handler. Be sure to take appropriate precautions when writing into the exception stack. (What happens if the user environment runs out of space on the exception stack?)

### `page_fault_handler`

```c
	// LAB 4: Your code here.
    if (curenv->env_pgfault_upcall) {
        struct UTrapframe *utf;
        if (tf->tf_esp >= UXSTACKTOP-PGSIZE && tf->tf_esp <= UXSTACKTOP-1) {
            utf = (struct UTrapframe *)(tf->tf_esp - sizeof(struct UTrapframe) - 4); 
        } else {
            utf = (struct UTrapframe *)(UXSTACKTOP - sizeof(struct UTrapframe));
        }   

        user_mem_assert(curenv, (void*)utf, 1, PTE_W);
        utf->utf_fault_va = fault_va;
        utf->utf_err = tf->tf_err;
        utf->utf_regs = tf->tf_regs;
        utf->utf_eip = tf->tf_eip;
        utf->utf_eflags = tf->tf_eflags;
        utf->utf_esp = tf->tf_esp;

        curenv->env_tf.tf_eip = (uintptr_t)curenv->env_pgfault_upcall;
        curenv->env_tf.tf_esp = (uintptr_t)utf;
        env_run(curenv);
    } 
	// Destroy the environment that caused the fault.
	cprintf("[%08x] user fault va %08x ip %08x\n",
		curenv->env_id, fault_va, tf->tf_eip);
	print_trapframe(tf);
	env_destroy(curenv);
```

## Exercise 10

> Implement the `_pgfault_upcall` routine in `lib/pfentry.S`. The interesting part is returning to the original point in the user code that caused the page fault. You'll return directly there, without going back through the kernel. The hard part is simultaneously switching stacks and re-loading the EIP.

### `_pgfault_upcall`

```assembly
	// LAB 4: Your code here.
	movl 0x28(%esp), %eax	// trap-time eip
	subl $0x4, 0x30(%esp)	// we have to use subl now because we can't use after popfl
	movl 0x30(%esp), %ebx   // trap-time esp-4
	movl %eax, (%ebx)		// push trap-time eip to trap-time stack
	addl $0x8, %esp			// skip fault_va and error code

	// Restore the trap-time registers.  After you do this, you
	// can no longer modify any general-purpose registers.
	// LAB 4: Your code here.
	popal

	// Restore eflags from the stack.  After you do this, you can
	// no longer use arithmetic operations or anything else that
	// modifies eflags.
	// LAB 4: Your code here.
	addl $4, %esp
	popfl

	// Switch back to the adjusted trap-time stack.
	// LAB 4: Your code here.
	popl %esp
	// Return to re-execute the instruction that faulted.
	// LAB 4: Your code here.
	ret
```

## Exercise 11
> Finish `set_pgfault_handler()` in `lib/pgfault.c`.

### `set_pgfault_handler`

```c
    if (_pgfault_handler == 0) {
        // First time through!
        // LAB 4: Your code here.
        if (sys_page_alloc(0, (void *)(UXSTACKTOP - PGSIZE), PTE_W|PTE_U|PTE_P)) 
		{
            panic("set_pgfault_handler: page_alloc failed");
        }   
        if (sys_env_set_pgfault_upcall(0, _pgfault_upcall)) 
		{
            panic("set_pgfault_handler: set_pgfault_upcall failed");
        }   
    }   

    _pgfault_handler = handler;
```

## Exercise 12

> Implement `fork`, `duppage` and `pgfault` in `lib/fork.c`.

### `fork`

```c
	// LAB 4: Your code here.
	envid_t envid;
	uint32_t addr;
	int r;

	set_pgfault_handler(pgfault);

	envid = sys_exofork();
	if (envid < 0)
		panic("sys_exofork: %e", envid);
	if (envid == 0) 
	{
		thisenv = &envs[ENVX(sys_getenvid())];
		return 0;
	}


	for (addr = 0; addr < USTACKTOP; addr += PGSIZE)
		if((uvpd[PDX(addr)] & PTE_P) && (uvpt[PGNUM(addr)] & PTE_P) && (uvpt[PGNUM(addr)] & PTE_U))
			duppage(envid, PGNUM(addr));

	// allocate a new page
	if((r = sys_page_alloc(envid, (void *)(UXSTACKTOP - PGSIZE), PTE_U | PTE_W | PTE_P)) < 0)
		panic("sys_page_alloc: %e", r);

	extern void _pgfault_upcall();
	sys_env_set_pgfault_upcall(envid, _pgfault_upcall);

	// Start child environment
	if ((r = sys_env_set_status(envid, ENV_RUNNABLE)) < 0)
		panic("sys_env_set_status: %e", r);

	return envid;
```

### `duppage`

```c
	// LAB 4: Your code here.
	void* addr = (void *)(pn * PGSIZE);
	if((uvpt[pn] & PTE_COW) || (uvpt[pn] & PTE_W))
	{
		r = sys_page_map(0, addr, envid, addr, PTE_COW | PTE_U | PTE_P);
		if(r < 0)
			panic("duppage: sys_page_map fail\n");
		r = sys_page_map(0, addr, 0, addr, PTE_COW | PTE_U | PTE_P);
		if(r < 0)
			panic("duppage: sys_page_map fail\n");
	}
	else
	{
		r = sys_page_map(0, addr, envid, addr, PTE_U | PTE_P);
		if(r < 0)
			panic("duppage: sys_page_map fail\n");
	}

	return 0;
```

### `pgfault`

```c
	// LAB 4: Your code here.
	if (!((err & FEC_WR) && (uvpt[PGNUM(addr)] & PTE_COW) && (uvpd[PDX(addr)] & PTE_P) && (uvpt[PGNUM(addr)] & PTE_P)))
		panic("pgfault: not copy-on-write\n");
	// panic("pgfault");
	// Allocate a new page, map it at a temporary location (PFTEMP),
	// copy the data from the old page to the new page, then move the new
	// page to the old page's address.
	// Hint:
	//   You should make three system calls.
	//   No need to explicitly delete the old page's mapping.

	// LAB 4: Your code here.
	r = sys_page_alloc(0, (void *)PFTEMP, PTE_U | PTE_W | PTE_P);
	if (r < 0)
		panic("pgfault: sys_page_alloc fail\n");
	memcpy(PFTEMP, ROUNDDOWN(addr, PGSIZE), PGSIZE);
	r = sys_page_map(0, (void *)PFTEMP, 0, ROUNDDOWN(addr, PGSIZE), PTE_U | PTE_W | PTE_P);
	if (r < 0)
		panic("pgfault: sys_page_map fail\n");
	r = sys_page_unmap(0, (void *)PFTEMP);
	if (r < 0)
		panic("pgfault: sys_page_unmap fail\n");
	return ;
```

# Part C
## Exercise 13
> Modify `kern/trapentry.S` and `kern/trap.c` to initialize the appropriate entries in the IDT and provide handlers for IRQs 0 through 15. Then modify the code in `env_alloc()` in `kern/env.c` to ensure that user environments are always run with interrupts enabled.

### `trapentry.S`

```assembly
TRAPHANDLER_NOEC(handler32, IRQ_OFFSET + IRQ_TIMER)
TRAPHANDLER_NOEC(handler33, IRQ_OFFSET + IRQ_KBD)
TRAPHANDLER_NOEC(handler36, IRQ_OFFSET + IRQ_SERIAL)
TRAPHANDLER_NOEC(handler39, IRQ_OFFSET + IRQ_SPURIOUS)
TRAPHANDLER_NOEC(handler46, IRQ_OFFSET + IRQ_IDE)
TRAPHANDLER_NOEC(handler51, IRQ_OFFSET + IRQ_ERROR)
```

### `trap_init`

```c
SETGATE(idt[IRQ_OFFSET+IRQ_TIMER], 0, GD_KT, handler32, 0);
SETGATE(idt[IRQ_OFFSET+IRQ_KBD], 0, GD_KT, handler33, 0);
SETGATE(idt[IRQ_OFFSET+IRQ_SERIAL], 0, GD_KT, handler36, 0);
SETGATE(idt[IRQ_OFFSET+IRQ_SPURIOUS], 0, GD_KT, handler39, 0);
SETGATE(idt[IRQ_OFFSET+IRQ_IDE], 0, GD_KT, handler46, 0);
SETGATE(idt[IRQ_OFFSET+IRQ_ERROR], 0, GD_KT, handler51, 0);
```

### `trap_dispatch`

```c
// Handle clock interrupts. Don't forget to acknowledge the
// interrupt using lapic_eoi() before calling the scheduler!
// LAB 4: Your code here.
if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) {
       lapic_eoi();
       sched_yield();
       return;
}
```

## Exercise 14

> Modify the kernel's `trap_dispatch()` function so that it calls `sched_yield()` to find and run a different environment whenever a clock interrupt takes place.
>

```c
	// Handle clock interrupts. Don't forget to acknowledge the
	// interrupt using lapic_eoi() before calling the scheduler!
	// LAB 4: Your code here.
	if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) 
	{
       lapic_eoi();
       sched_yield();
       return;
	}
```

## Exercise 15

> Implement `sys_ipc_recv` and `sys_ipc_try_send` in `kern/syscall.c`. Read the comments on both before implementing them, since they have to work together. When you call `envid2env` in these routines, you should set the `checkperm` flag to 0, meaning that any environment is allowed to send IPC messages to any other environment, and the kernel does no special permission checking other than verifying that the target envid is valid.

### `ipc_recv`

```c
int32_t
ipc_recv(envid_t *from_env_store, void *pg, int *perm_store)
{
	// LAB 4: Your code here.
    if (pg == NULL) pg = (void *)UTOP;

    int r = sys_ipc_recv(pg);
    int from_env = 0, perm = 0;
    if (r == 0) {
        from_env = thisenv->env_ipc_from;
        perm = thisenv->env_ipc_perm;
        r = thisenv->env_ipc_value;
    } else {
        from_env = 0;
        perm = 0;
    }   

    if (from_env_store) *from_env_store = from_env;
    if (perm_store) *perm_store = perm;

    return r;
}
```

### `ipc_send`

```c
void
ipc_send(envid_t to_env, uint32_t val, void *pg, int perm)
{
	// LAB 4: Your code here.
    if (pg == NULL) pg = (void *)UTOP;

    int ret;
    while ((ret = sys_ipc_try_send(to_env, val, pg, perm))) {
        if (ret != -E_IPC_NOT_RECV) panic("ipc_send error %e", ret);
        sys_yield();
    }
}
```

### `sys_ipc_try_send`

```c
static int
sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
{
	// LAB 4: Your code here.
	    struct Env *e; 
    if (envid2env(envid, &e, 0)) return -E_BAD_ENV;

    if (!e->env_ipc_recving) return -E_IPC_NOT_RECV;

    if (srcva < (void *) UTOP) {
        if(PGOFF(srcva)) return -E_INVAL;

        pte_t *pte;
        struct PageInfo *p = page_lookup(curenv->env_pgdir, srcva, &pte);
        if (!p) return -E_INVAL;

        if ((*pte & perm) != perm) return -E_INVAL;

        if ((perm & PTE_W) && !(*pte & PTE_W)) return -E_INVAL;

        if (e->env_ipc_dstva < (void *)UTOP) {
            int ret = page_insert(e->env_pgdir, p, e->env_ipc_dstva, perm);
            if (ret) return ret;
            e->env_ipc_perm = perm;
        }   
    }   

    e->env_ipc_recving = 0;
    e->env_ipc_from = curenv->env_id;
    e->env_ipc_value = value;
    e->env_status = ENV_RUNNABLE;
    e->env_tf.tf_regs.reg_eax = 0;
    return 0;
}
```

### `sys_ipc_recv`

```c
static int
sys_ipc_recv(void *dstva)
{
	// LAB 4: Your code here.
    if ((dstva < (void *)UTOP) && PGOFF(dstva))
        return -E_INVAL;

    curenv->env_ipc_recving = 1;
    curenv->env_status = ENV_NOT_RUNNABLE;
    curenv->env_ipc_dstva = dstva;
    sys_yield();
    return 0;
}
```

并在`syscall`函数中加入对应语句

## Questions

### Q1

> Compare `kern/mpentry.S` side by side with `boot/boot.S`. Bearing in mind that `kern/mpentry.S` is compiled and linked to run above `KERNBASE` just like everything else in the kernel, what is the purpose of macro `MPBOOTPHYS`? Why is it necessary in `kern/mpentry.S` but not in `boot/boot.S`? In other words, what could go wrong if it were omitted in `kern/mpentry.S`?
> Hint: recall the differences between the link address and the load address that we have discussed in Lab 1.

A1: 由于mpentry.S被加载到0x7000，因此需要通过MPBOOTPHYS才能正确找到我们所需要的地址。

### Q2

> It seems that using the big kernel lock guarantees that only one CPU can run the kernel code at a time. Why do we still need separate kernel stacks for each CPU? Describe a scenario in which using a shared kernel stack will go wrong, even with the protection of the big kernel lock.

A2: 内核锁保证同一时刻在hi有一个CPU执行内核代码，CPU会将信息存储入内核栈，如果不将其分离就会产生错误，破坏数据

## Result

![image-20200825022535912](D:\study\OS\lab\lab4\image-20200825022535912.png)
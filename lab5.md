# Lab5

## Exercize 1

> `i386_init` identifies the file system environment by passing the type `ENV_TYPE_FS` to your environment creation function, `env_create`. Modify `env_create` in `env.c`, so that it gives the file system environment I/O privilege, but never gives that privilege to any other environment.

### `env_create`

```c
	if(type == ENV_TYPE_FS)
	{
		e->env_tf.tf_eflags |= FL_IOPL_3;
	}
```

## Exercize 2
> Implement the `bc_pgfault` and `flush_block` functions in `fs/bc.c`. `bc_pgfault` is a page fault handler, just like the one your wrote in the previous lab for copy-on-write fork, except that its job is to load pages in from the disk in response to a page fault. When writing this, keep in mind that (1) `addr` may not be aligned to a block boundary and (2) `ide_read` operates in sectors, not blocks.

### `bc_pgfault`

```c
	// LAB 5: you code here:
	addr = (void *)ROUNDDOWN(addr, PGSIZE);
	r = sys_page_alloc(0, addr, PTE_U | PTE_W | PTE_P);
	if (r < 0 )
		panic("bc_pgfault: sys_page_alloc fail\n");
	r = ide_read(blockno*BLKSECTS, addr, BLKSECTS);
	if (r < 0)
		panic("bc_pgfault: ide_read error\n");
```



### `flush_block`

```c
	// LAB 5: Your code here.
	// panic("flush_block not implemented");
	int r;
	if (!va_is_mapped(addr) || !va_is_dirty(addr))
		return ;
	addr = (void *)ROUNDDOWN(addr, PGSIZE);
	if (ide_write(blockno*BLKSECTS, addr, BLKSECTS) < 0)
		panic("flush_block: ide_write error\n");

	// Clear the dirty bit for the disk block page since we just read the
	// block from disk
	if ((r = sys_page_map(0, addr, 0, addr, uvpt[PGNUM(addr)] & PTE_SYSCALL)) < 0)
		panic("in bc_pgfault, sys_page_map: %e", r);
```

## Exercize 3
> Use `free_block` as a model to implement `alloc_block` in `fs/fs.c`, which should find a free disk block in the bitmap, mark it used, and return the number of that block. When you allocate a block, you should immediately flush the changed bitmap block to disk with `flush_block`, to help file system consistency.

### `alloc_block`

```c
	int bitmap_block_size = (super->s_nblocks + BLKBITSIZE - 1)/ BLKBITSIZE; 
	for (int blockno = 2 + bitmap_block_size; blockno < super->s_nblocks; blockno++) {
		if (block_is_free(blockno)) {
			bitmap[blockno/32] &= ~(1<<(blockno%32));
			flush_block(bitmap);
			return blockno;
		}
	}
	return -E_NO_DISK;
```

## Exercise 4

> Implement `file_block_walk` and `file_get_block`. `file_block_walk` maps from a block offset within a file to the pointer for that block in the `struct File` or the indirect block, very much like what `pgdir_walk` did for page tables. `file_get_block` goes one step further and maps to the actual disk block, allocating a new one if necessary.

### `file_block_walk`

```c
	int r;

	if (filebno >= NDIRECT + NINDIRECT)
		return -E_INVAL;

	if (filebno < NDIRECT) {
		if (ppdiskbno)
	 		*ppdiskbno = f->f_direct + filebno;
		return 0;
	}

	if (!f->f_indirect && !alloc)
		return -E_NOT_FOUND;

	if (!f->f_indirect) {
		if ((r = alloc_block()) < 0)
		  return -E_NO_DISK;
		f->f_indirect = r;
		memset(diskaddr(r), 0, BLKSIZE);
		flush_block(diskaddr(r));
	}

	if (ppdiskbno)
		*ppdiskbno = (uint32_t*)diskaddr(f->f_indirect) + filebno - NDIRECT;
	return 0;
```

### `file_get_block`

```c
	int r;
	uint32_t *pdiskno;

	if ((r = file_block_walk(f, filebno, &pdiskno, 1)) < 0)
	    return r;

	if (*pdiskno == 0) {
	    if ((r = alloc_block()) < 0)
	        return -E_NO_DISK;
	    *pdiskno = r;
		memset(diskaddr(r), 0, BLKSIZE);
		flush_block(diskaddr(r));
	}

	*blk = diskaddr(*pdiskno);
	return 0;
```

## Exercize 5~6

> Implement `serve_read` in `fs/serv.c`.
>
> Implement `serve_write` in `fs/serv.c` and `devfile_write` in `lib/file.c`.

### `serve_read`

```c
	struct OpenFile *o;
	int r;
	if ((r = openfile_lookup(envid, req->req_fileid, &o)) < 0)
		return r;

	int req_n = req->req_n > PGSIZE ? PGSIZE : req->req_n;
	if ((r = file_read(o->o_file, ret->ret_buf, req_n, o->o_fd->fd_offset)) < 0)
		return r;
	o->o_fd->fd_offset += r;
	return r;
```

### `serve_write`

```c
	struct OpenFile *o;
	int r;
	if ((r = openfile_lookup(envid, req->req_fileid, &o)) < 0)
		return r;

	int req_n = req->req_n > PGSIZE ? PGSIZE : req->req_n;
	if ((r = file_write(o->o_file, req->req_buf, req_n, o->o_fd->fd_offset)) < 0)
		return r;
	o->o_fd->fd_offset += r;
	return r;
```

## Exercize 7

> `spawn` relies on the new syscall `sys_env_set_trapframe` to initialize the state of the newly created environment. Implement `sys_env_set_trapframe` in `kern/syscall.c` (don't forget to dispatch the new system call in `syscall()`).

### `sys_env_set_trapframe`

```c
    struct Env *e;
    if (envid2env(envid, &e, 1)) 
	{
        return -E_BAD_ENV;
	}

    e->env_tf = *tf;
    e->env_tf.tf_eflags |= FL_IF;
    e->env_tf.tf_eflags &= ~FL_IOPL_MASK;
    return 0;
```

## Exercize 8
> Change `duppage` in `lib/fork.c` to follow the new convention. If the page table entry has the `PTE_SHARE` bit set, just copy the mapping directly. (You should use `PTE_SYSCALL`, not `0xfff`, to mask out the relevant bits from the page table entry. `0xfff` picks up the accessed and dirty bits as well.)

### `duppage`

```c
	// LAB 4: Your code here.
	void* addr = (void *)(pn * PGSIZE);
	if (uvpt[pn] & PTE_SHARE)
	{
		r = sys_page_map(0, addr, envid, addr, uvpt[pn]&PTE_SYSCALL);
		if (r < 0)
			return r;
	}
	else if((uvpt[pn] & PTE_COW) || (uvpt[pn] & PTE_W))
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

## Exercize 9
> In your `kern/trap.c`, call `kbd_intr` to handle trap `IRQ_OFFSET+IRQ_KBD` and `serial_intr` to handle trap `IRQ_OFFSET+IRQ_SERIAL`.

### `trap_dispatch`

```c
    if (tf->tf_trapno == IRQ_OFFSET + IRQ_KBD) 
	{
		lapic_eoi();
        kbd_intr();
        return;
    }
    if (tf->tf_trapno == IRQ_OFFSET + IRQ_SERIAL) 
	{
		lapic_eoi();
        serial_intr();
        return;
    }
```

## Exercize 10
> The shell doesn't support I/O redirection. It would be nice to run sh <script instead of having to type in all the commands in the script by hand, as you did above. Add I/O redirection for < to `user/sh.c`.

### `sh.c`

```c
			// LAB 5: Your code here.
			if ((fd = open(t, O_RDONLY)) < 0) 
			{
				cprintf("open %s for read: %e", t, fd);
				exit();
			}
			if (fd != 0) 
			{
				dup(fd, 0);
				close(fd);
			}
			break;
```

## Result

![image-20200825163656672](D:\study\OS\lab\lab5\image-20200825163656672.png)


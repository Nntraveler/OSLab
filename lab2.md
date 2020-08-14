# Lab 2

## Exercise 1

> In the file `kern/pmap.c`, you must implement code for the following functions (probably in the order given).
>
> ```c
> `boot_alloc()`
> `mem_init()` (only up to the call to `check_page_free_list(1)`)
> `page_init()`
> `page_alloc()`
> `page_free()
> ```

### boot_alloc

```c
	// Allocate a chunk large enough to hold 'n' bytes, then update
	// nextfree.  Make sure nextfree is kept aligned
	// to a multiple of PGSIZE.
	//
	// LAB 2: Your code here.
	result = nextfree;
	if(n == 0)
	{
		return result;
	}
	nextfree = ROUNDUP(nextfree + n, PGSIZE);
	return result;
```

### mem_init

```c
	//////////////////////////////////////////////////////////////////////
	// Allocate an array of npages 'struct PageInfo's and store it in 'pages'.
	// The kernel uses this array to keep track of physical pages: for
	// each physical page, there is a corresponding struct PageInfo in this
	// array.  'npages' is the number of physical pages in memory.  Use memset
	// to initialize all fields of each struct PageInfo to 0.
	// Your code goes here:
	pages = (struct PageInfo *)boot_alloc(sizeof(struct PageInfo) * npages);
```

### page_init

```c
	size_t i;
	for (i = 1; i < npages_basemem; i++) {
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}

	char *nextfree = boot_alloc(0);
    size_t free_pages = PGNUM(PADDR(nextfree));
	for (i = free_pages; i < npages; i++) {
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}
```

### page_alloc

```c
struct PageInfo *
page_alloc(int alloc_flags)
{
	// Fill this function in
	struct PageInfo* result = page_free_list;
	if(result)
	{
		page_free_list = page_free_list->pp_link;
		result->pp_link = NULL;
		if (alloc_flags & ALLOC_ZERO) {
            memset(page2kva(result), 0, PGSIZE);
        }
        return result;
	}
	return NULL;
}
```

### page_free

```c
void
page_free(struct PageInfo *pp)
{
	// Fill this function in
	// Hint: You may want to panic if pp->pp_ref is nonzero or
	// pp->pp_link is not NULL.
	if(pp->pp_ref != 0 || pp->pp_link != NULL)
	{
		panic("page free failure");
	}
    pp->pp_link = page_free_list;
    page_free_list = pp; 
}
```

## Exercise 2~3

熟悉80386手册5、6章，qemu调试命令

## Exercise 4

> In the file `kern/pmap.c`, you must implement code for the following functions.
>
> ```c
>         pgdir_walk()
>         boot_map_region()
>         page_lookup()
>         page_remove()
>         page_insert()
> ```

### pgdir_walk

```c
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
	// Fill this function in
	pde_t *pde = &pgdir[PDX(va)];
	if(!(*pde & PTE_P))
	{
		if(create)
		{
			struct PageInfo *page = page_alloc(ALLOC_ZERO);
			if(!page)
			{
				return NULL;
			}
			page->pp_ref++;
			*pde = page2pa(page) | PTE_P | PTE_U | PTE_W;
		}
		else
		{
			return NULL;
		}
		
	}
	pte_t *pte = (pte_t *) KADDR(PTE_ADDR(*pde));
	return &pte[PTX(va)];
}
```

### boot_map_region

```c
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
	// Fill this function in
	int pageNum = PGNUM(size);
	for(int i = 0; i < pageNum; ++i)
	{
		pte_t *pte = pgdir_walk(pgdir, (void*)va, 1);
		if(!pte)
		{
			panic("out of memory");
		}
		*pte = pa | perm | PTE_P;
		va += PGSIZE;
		pa += PGSIZE;
	}
}
```

### page_lookup

```c
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
	// Fill this function in
	pte_t *pte = pgdir_walk(pgdir, va, 0);
	if(pte && ((*pte) & PTE_P))
	{
		physaddr_t pa = PTE_ADDR(*pte);
		struct PageInfo* result = pa2page(pa);
		if(pte_store != 0)
		{
			*pte_store = pte;
		}
		return result;
	}
	return NULL;
}
```

### page_remove

```c
void
page_remove(pde_t *pgdir, void *va)
{
	// Fill this function in
	pte_t *pte;
	struct PageInfo *page = page_lookup(pgdir, va, &pte);
	if(!page || !(*pte & PTE_P))
	{
		return;
	}
	page_decref(page);
	tlb_invalidate(pgdir, va);
	*pte = 0;
}
```

### page_insert

```c
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
	// Fill this function in
	pte_t* pte = pgdir_walk(pgdir, va, 1);
	if(!pte)
	{
		return -E_NO_MEM;
	}
	pp->pp_ref++;
	if((*pte) & PTE_P)
	{
		page_remove(pgdir,va);
	}
	*pte = page2pa(pp) | PTE_P | perm;
	pgdir[PDX(va)] |= perm;
	return 0;
}
```

## Exercise 5

> Fill in the missing code in `mem_init()` after the call to `check_page()`.
>
> Your code should now pass the `check_kern_pgdir()` and `check_page_installed_pgdir()` checks.

### mem_init

```c
	//////////////////////////////////////////////////////////////////////
	// Map 'pages' read-only by the user at linear address UPAGES
	// Permissions:
	//    - the new image at UPAGES -- kernel R, user R
	//      (ie. perm = PTE_U | PTE_P)
	//    - pages itself -- kernel RW, user NONE
	// Your code goes here:
	boot_map_region(kern_pgdir, UPAGES, PTSIZE, PADDR(pages), PTE_U);
	//////////////////////////////////////////////////////////////////////
	// Use the physical memory that 'bootstack' refers to as the kernel
	// stack.  The kernel stack grows down from virtual address KSTACKTOP.
	// We consider the entire range from [KSTACKTOP-PTSIZE, KSTACKTOP)
	// to be the kernel stack, but break this into two pieces:
	//     * [KSTACKTOP-KSTKSIZE, KSTACKTOP) -- backed by physical memory
	//     * [KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE) -- not backed; so if
	//       the kernel overflows its stack, it will fault rather than
	//       overwrite memory.  Known as a "guard page".
	//     Permissions: kernel RW, user NONE
	// Your code goes here:
    boot_map_region(kern_pgdir, KSTACKTOP-KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);
	//////////////////////////////////////////////////////////////////////
	// Map all of physical memory at KERNBASE.
	// Ie.  the VA range [KERNBASE, 2^32) should map to
	//      the PA range [0, 2^32 - KERNBASE)
	// We might not have 2^32 - KERNBASE bytes of physical memory, but
	// we just set up the mapping anyway.
	// Permissions: kernel RW, user NONE
	// Your code goes here:
    boot_map_region(kern_pgdir, KERNBASE, -KERNBASE, 0, PTE_W);
```

## Questions

### Q1

> Assuming that the following JOS kernel code is correct, what type should variable `x` have, `uintptr_t` or `physaddr_t`?
>
> ```c
> 	mystery_t x;
> 	char* value = return_a_pointer();
> 	*value = 10;
> 	x = (mystery_t) value;
> ```

* `uintptr_t`

### Q2

> What entries (rows) in the page directory have been filled in at this point? What addresses do they map and where do they point? In other words, fill out this table as much as possible:
>
> | Entry | Base Virtual Address | Points to (logically):                |
> | ----- | -------------------- | ------------------------------------- |
> | 1023  | ?                    | Page table for top 4MB of phys memory |
> | 1022  | ?                    | ?                                     |
> | .     | ?                    | ?                                     |
> | .     | ?                    | ?                                     |
> | .     | ?                    | ?                                     |
> | 2     | 0x00800000           | ?                                     |
> | 1     | 0x00400000           | ?                                     |
> | 0     | 0x00000000           | [see next question]                   |

* | Entry | Base Virtual Address | Points to (logically):                |
  | ----- | -------------------- | ------------------------------------- |
  | 1023  | 0xefc00000           | Page table for top 4MB of phys memory |
  | 1022  | 0xefc00000           | ?                                     |
  | .     | ?                    | ?                                     |
  | .     | ?                    | ?                                     |
  | .     | ?                    | ?                                     |
  | 2     | 0x00800000           | ?                                     |
  | 1     | 0x00400000           | ?                                     |
  | 0     | 0x00000000           | [see next question]                   |

### Q3

> We have placed the kernel and user environment in the same address space. Why will user programs not be able to read or write the kernel's memory? What specific mechanisms protect the kernel memory?

* 利用分页保护机制，通过判定PTE_U对应位，来控制访问权限。

## Result

![image-20200815012658153](D:\study\OS\lab\lab2\image-20200815012658153.png)
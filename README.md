# 📑 **`Xv6 Memory Management Walkthrough`**

# **`Table of Content`**

- [Terminology](#terminology)
- [Overview of page table](#overview-of-page-table)
- [Memory Layout](#memory-layout)
- [Functions of interest {Part 1 : building blocks}](#functions-of-interest-part-1--building-blocks)
- [Functions of interest {Part 2 : core functions}](#functions-of-interest-part-2--core-functions)
- [How all of this works](#how-all-of-this-works)
- [Final Words](#final-words)
- [Acknowledgement](#resource)

# **`Terminology`**

- VA : Virtual Address
- PA : Physical Address
- PD : Page Directory
- PDE : Page Directory Entry
- PT : Page Table
- PTE : Page Table Entry
- PPN : Physical Page Number

# **`Overview of xv6 page table`**

All process works on virtual address. Machine’s RAM is physical address. Page Table maps virtual address to physical address. 

Xv6 does this in 2 steps. 

![Untitled](xv6%20memory%20management%20walkthrough%2052f10c25c9dd4de39e601e386e6a1788/Untitled.png)

Some things to note for xv6 :
- 32 bit virtual address (so, virtual address space 4GB)
- page size of 4KB
- The CPU register `CR3` contains **a pointer to the outer page directory** of the **current running process**.

![Untitled](xv6%20memory%20management%20walkthrough%2052f10c25c9dd4de39e601e386e6a1788/Untitled%201.png)

### **Caution**

xv6 **does not** do **demand paging**, so there is **no concept of virtual memory**. That is, all valid pages of a process are always allocated physical pages.


# **`Memory Layout`**

Every page table in xv6 has mappings for user pages as well as kernel pages. The part of the page table dealing with kernel pages is the same across all processes.

## **Kernel Only Memory**

![Untitled](xv6%20memory%20management%20walkthrough%2052f10c25c9dd4de39e601e386e6a1788/Untitled%202.png)

- virtual memory > `KERNBASE` (0x8000 0000) is for kernel
- always mapped as kernel-mode only
    - check `PTE_U` fag
    - **protection fault** for user-mode programs to access
- **physical memory address** `N` is mapped to `KERNBASE+N` or `0:PHYSTOP` to `KERNBASE:KERNBASE+PHYSTOP`
    - Note that although the size of physical memory is 4 GB, only 2 GB can be used by Xv6
- kernel code loaded into contiguous physical addresses

## **User Only Memory**

Each process has separate page table. For any process, user memory VA range is `0:KERNBASE` where `KERNBASE` is 0x80000000 i.e. 2 GB of memory is available to process.

![Untitled](xv6%20memory%20management%20walkthrough%2052f10c25c9dd4de39e601e386e6a1788/Untitled%203.png)

- The kernel code doesn’t exactly begin at `KERNBASE`, but a bit later at `KERNLINK`, to leave
some space at the start of memory for **I/O devices**. Next comes the kernel **code+read-only** data from the kernel binary. Apart from the memory set aside for kernel code and I/O devices, the **remaining memory is in the form of free pages** managed by the kernel. When any user process requests for memory to build up its user part of the address space, the kernel allocates memory to the user process from this free space list. That is, most physical memory can be mapped twice, once into the kernel part of the address space of a process, and once into the user part.

![Untitled](xv6%20memory%20management%20walkthrough%2052f10c25c9dd4de39e601e386e6a1788/Untitled%204.png)

## **`P2V` and `V2P`**

- `V2P(a)` (virtual to physical)
    - convert **kernel address** `a` to **physical address**
        - subtract `KERNBASE` (0x8000 0000)
    
    ```cpp
    #define V2P(a) (((uint) (a)) - KERNBASE)
    ```
    
- `P2V(a)` (physical to virtual)
    - convert **physical address** `a` to **kernel address**
        - add `KERNBASE` (0x8000 0000)
    
    ```cpp
    #define P2V(a) ((void *)(((char *) (a)) + KERNBASE))
    ```
    

# **`Functions of interest {Part 1 : building blocks}`**

## **`PGROUNDUP`**

What it does is round the address up as a multiple of page number i.e. do a CEIL type operation.

```cpp
#define PGROUNDUP(sz)  (((sz)+PGSIZE-1) & ~(PGSIZE-1))
```

`PGROUNDUP(620)` → ((620 + (1024 -1)) & ~(1023)) → 1024

## **`PGROUNDDOWN`**

another related function is `PGROUNDDOWN` , works similarly. just makes the address round down as a multiple of page number

```cpp
#define PGROUNDDOWN(a) (((a)) & ~(PGSIZE-1))
```

`PGROUNDDOWN(2400)` → (2400 & ~(1023)) → 2048

## **`switchuvm`**

- `u` here stands for user
- basically OS loads the user process information to run it
- loads process's Page Table to `%cr3`

## **`kalloc`**

This function is responsible to return an address of one **new, currently unused** page (4096 byte) in RAM. `kalloc` removes first free page from `kmem` and returns its (virtual!) address, where `kmem` points to the head of a list of free (that is, available) pages of memory.

**Failure :** If it **returns 0**, that means there are no available unused pages currently. 

## **`walkpgdir`**

Main job of this function is to get the content of 2nd level PTE. This is such an important function that we will go line by line.

```cpp
static pte_t *
walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
  pde_t *pde;
  pte_t *pgtab;

  pde = &pgdir[PDX(va)]; // PDX(va) returns the first 10 bit. pgdir is level 1 page table. So, pgdir[PDX(va)] is level 1 PTE where there is PPN and offset
  if(*pde & PTE_P){ // not NULL and Present
    pgtab = (pte_t*)P2V(PTE_ADDR(*pde)); // PTE_ADDR return the first 20 bit or PPN. PPN is converted to VPN for finding 2nd level PTE. pgtab is level 2 page table
  } else {
    if(!alloc || (pgtab = (pte_t*)kalloc()) == 0)
      return 0;
    // Make sure all those PTE_P bits are zero.
    memset(pgtab, 0, PGSIZE);
    // The permissions here are overly generous, but they can
    // be further restricted by the permissions in the page table
    // entries, if necessary.
    *pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;
  }
  return &pgtab[PTX(va)]; // PDX(va) returns the second 10 bit. So, pgtab[PTX(va)] is level 2 PTE where there is PPN and offset
}
```

### **Case 1 : when no allocation needed (`alloc = 0`)**

Recall that

```cpp
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |      Index     |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(va) --/ \--- PTX(va) --/
```

![Untitled](xv6%20memory%20management%20walkthrough%2052f10c25c9dd4de39e601e386e6a1788/Untitled%205.png)

- `PDX(va)` returns the first 10 bit. `pgdir` is level 1 page table. So, `pgdir[PDX(va)]` is level 1 PTE where there is PPN and offset
    
    ```cpp
    pde = &pgdir[PDX(va)];
    ```
    
- `PTE_ADDR` return the first 20 bit or PPN. PPN is converted to VPN using `P2V` for finding 2nd level PTE. `pgtab` is level 2 page table
    
    ```cpp
    pgtab = (pte_t*)P2V(PTE_ADDR(*pde));
    ```
    
- `PDX(va)` returns the second 10 bit. So, `pgtab[PTX(va)]` is level 2 PTE where there is PPN and offset
    
    ```cpp
    &pgtab[PTX(va)];
    ```
    

### **Case 2 : creating second-level page tables (When `alloc = 1`)**

- return **NULL** if **not trying to make new page table** otherwise use `kalloc` to allocate it
    
    ```cpp
    if(!alloc || (pgtab = (pte_t*)kalloc()) == 0)
        return 0;
    ```
    
- clear the new second-level page table. Make sure all those **`PTE_P`** bits are zero.
    
    ```cpp
    memset(pgtab, 0, PGSIZE);
    ```
    
- now that we have made the 2nd level page table, we have to put that address in 1st level page table. plus add some flags
    
    ```cpp
    *pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;
    ```
    

## **`mappages`**

```cpp
static int
mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
  char *a, *last;
  pte_t *pte;

  a = (char*)PGROUNDDOWN((uint)va);
  last = (char*)PGROUNDDOWN(((uint)va) + size - 1);
  for(;;){
    if((pte = walkpgdir(pgdir, a, 1)) == 0)
      return -1;
    if(*pte & PTE_P)
      panic("remap");
    *pte = pa | perm | PTE_P;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```

As the name suggests, `mappages` adds a mapping from the virtual address (starting from given `VA` to `VA+SIZE`) to the physical address (starting from given `PA` to `PA+SIZE`) 

- for each virtual page in range, **get its page table entry**. if 2nd level not present allocate it (recall `alloc=1` case of `walkpgdir`). failure happens if runs out of memory
    
    ```cpp
    if((pte = walkpgdir(pgdir, a, 1)) == 0)
          return -1;
    ```
    
- make sure it’s not already set
    
    ```cpp
    if(*pte & PTE_P)
    		panic("remap");
    ```
    
- set page table entry to valid value pointing to physical page at `PA` with specified permission (`perm`) and `P` for present
    
    ```cpp
    *pte = pa | perm | PTE_P; // pa is first 20 bit, perm is flags, PTE_P is done to mark as present
    ```
    
- advance to next physical page (`PA`) and next virtual page (`VA`)
    
    ```cpp
    a += PGSIZE;
    pa += PGSIZE;
    ```
    

<!-- ## **`growproc`** -->

# **`Functions of interest {Part 2 : core functions}`**

## **`allocuvm`**

Allocates user virtual memory (page tables and physical memory) to grow process. This function is responsible to **increase** the user's virtual memory in a specific page directory  from `oldsz` to `newsz` . This function used for initial allocation plus expanding heap on request

```cpp
int
allocuvm(pde_t *pgdir, uint oldsz, uint newsz)
{
  char *mem;
  uint a;

  if(newsz >= KERNBASE)
    return 0;
  if(newsz < oldsz)
    return oldsz;

  a = PGROUNDUP(oldsz);
  for(; a < newsz; a += PGSIZE){
    mem = kalloc();
    if(mem == 0){
      cprintf("allocuvm out of memory\n");
      deallocuvm(pgdir, newsz, oldsz);
      return 0;
    }
    memset(mem, 0, PGSIZE);
    if(mappages(pgdir, (char*)a, PGSIZE, V2P(mem), PTE_W|PTE_U) < 0){
      cprintf("allocuvm out of memory (2)\n");
      deallocuvm(pgdir, newsz, oldsz);
      kfree(mem);
      return 0;
    }
  }
  return newsz;
}
```

Returns `newsz` if succeeded, 0 otherwise.

- walks the virtual address space between the old size and new size in page-sized chunks. For each new logical page to be created, it allocates a new free page from the kernel, and adds a mapping from the virtual address to the physical address by calling `mappages`
- allocate a new, zero page
    
    ```cpp
    mem = kalloc();
    ```
    
- add page to second-level page table, also do the allocation. recall that `mappages` call `walkpgdir` with `alloc = 1`
    
    ```cpp
    mappages(pgdir, (char*)a, PGSIZE, v2p(mem), PTE_W|PTE_U);
    ```
    
- There are indeed 2 cases where **this function can fail**:
    
    Case 1: `kalloc` (kernel allocation) function failed. This function is responsible to return an address of a **new, currently unused** page in RAM. If it **returns 0**, that means there are no available unused pages currently.
    
    Case 2: `mappages` function failed. This function is responsible of making the **new allocated page to be accessible by the process** who uses the given page directory by **mapping that page with the next virtual address available in the page directory**. If this function fails that means it failed in doing so, probably due to the page directory being already full.
    
    In both cases, `allocuvm` didn't managed to increase the user's memory to the size requested, Therefore, `deallocuvm` is undoing all allocations until the point of failure, so the virtual memory will remain unchanged, and returns an error it self.

## **`deallocuvm`**
deallocuvm looks at all the logical pages from the (bigger) old size of the process to the (smaller) new size, locates the corresponding physical pages, frees them up, and zeroes out the corresponding PTE as well.


## **`copyuvm`**
Once a child process is allocated, its memory image is setup as a complete copy of the parent’s memory image by a call to `copyuvm`

- `setupkvm` : sets up kernel virtual memory
- walks through the entire address space of the parent in page-sized chunks, gets
the physical address of every page of the parent using a call to `walkpgdir`
- allocates a new physical page for the child using `kalloc`
- copies the contents of the parent’s page into the child’s page, **adds an entry to the child’s page table** using `mappages`
- returns the child’s page table
- has similar failure cases as discussed in `allocuvm`

<!-- ## `inituvm`

## `exec` -->
<!-- ![Untitled](xv6%20memory%20management%20walkthrough%2052f10c25c9dd4de39e601e386e6a1788/Untitled%206.png) -->

<!-- ## `sbrk` -->

<!-- # **`User Process Memory Management initialization`**

- `userinit(void)` → creates the first user process known as `init`
    - `setupkvm`  : Set up kernel part of a page table
    - `inituvm` : allocates one physical page of memory, copies the `init` executable into that memory, and sets up a page table entry for the first page of the user virtual address space
    - When the `init` process runs, it executes the `init` executable, whose main function **forks** a shell and starts listening to the user

  I guess this explain why we get this output when we do `control + P` after xv6 boots

  ```cpp
  1 sleep  init 80104347 801043f5 80104f5d 80106051 80105d93
  2 sleep  sh 80104310 801002ea 80101030 801050b6 80104f5d 80106051 80105d93
  ```

- all other case (meaning for all other user processes)
    - created by the `fork` system call
        - calls `copyuvm` : memory image is setup as a complete copy of the parent’s memory image. it return the child’s page table
        - after `copyuvm` entire memory of the parent has been cloned for the child, and the child’s new page table points to its newly allocated physical memory -->

<!-- # **`Grow/shrink the userspace part of the memory image`**

- sbrk → sys_sbrk [SYSTEM CALL]
    - growproc
        - allocuvm : to grow
        - deallocuvm : to shrink
        - switchuvm -->

<!-- # running executable (`exec`) -->

# **`How all of this works`**

Now that we are done with the function basics, let’s focus on how things actually work. Most of the findings and intuition gained are from a shit ton of debug statements and by reading a few books and notes. So, take it with a grain of salt.

&nbsp;

The first user process is the `init` function. It is initiated by `userinit` inside `main` of `main.c`.

Let’s focus on this first.

## **`init`**

### **`userinit` function**

```cpp
//PAGEBREAK: 32
// Set up first user process.
void
userinit(void)
{
  struct proc *p;
  extern char _binary_initcode_start[], _binary_initcode_size[];

  p = allocproc();
  
  initproc = p;
  if((p->pgdir = setupkvm()) == 0)
    panic("userinit: out of memory?");
  inituvm(p->pgdir, _binary_initcode_start, (int)_binary_initcode_size);
  p->sz = PGSIZE;
  memset(p->tf, 0, sizeof(*p->tf));
  p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
  p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
  p->tf->es = p->tf->ds;
  p->tf->ss = p->tf->ds;
  p->tf->eflags = FL_IF;
  p->tf->esp = PGSIZE;
  p->tf->eip = 0;  // beginning of initcode.S

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  // this assignment to p->state lets other cores
  // run this process. the acquire forces the above
  // writes to be visible, and the lock is also needed
  // because the assignment might not be atomic.
  acquire(&ptable.lock);

  p->state = RUNNABLE;

  release(&ptable.lock);
}
```

Now let’s see what things are done step by step

- `allocproc` : for every process `pid` needs to be assigned, `proc` structure needs to be initialized. all of this is done here. for the `init` process `pid` is returned 1 by `allocproc`
    - `kalloc` : `allocproc` calls `kalloc` which sets up **data** on the **kernel stack**
- `setupkvm` : creates **kernel page table** of this `init` process
- `inituvm` : allocates **one page** of physical memory, copies the **init executable** into that memory, sets up a page table entry (PTE) for the first page of the user virtual address space.
    
    Let’s also keep track of how many pages are being allocated where, this will be key to our understanding when we want to find what which pages are being deallocated in `deallocuvm` in later part.
    
    ```cpp
    PAGE ALLOCATED HERE = 1
    ```
    

### **`init executable`**

now the **init executable** that has been loaded runs, which to my understanding points to this part inside code

![Untitled](How%20all%20of%20this%20works%207868ca41311a42b5ba45aa4e1b1033fd/Untitled.png)

### **`exec`**

`SYS_exec` in turn calls `exec` . The `exec` function looks like this -

```cpp
int
exec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint argc, sz, sp, ustack[3+MAXARG+1];
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pde_t *pgdir, *oldpgdir;
  struct proc *curproc = myproc();
  
  begin_op();

  if((ip = namei(path)) == 0){
    end_op();
    cprintf("exec: fail\n");
    return -1;
  }
  ilock(ip);
  pgdir = 0;

  // Check ELF header
  if(readi(ip, (char*)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pgdir = setupkvm()) == 0)
    goto bad;

  // Load program into memory.
  sz = 0;
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, (char*)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    if((sz = allocuvm(pgdir, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    if(loaduvm(pgdir, (char*)ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;

  // Allocate two pages at the next page boundary.
  // Make the first inaccessible.  Use the second as the user stack.
  sz = PGROUNDUP(sz);
  if((sz = allocuvm(pgdir, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
 
  clearpteu(pgdir, (char*)(sz - 2*PGSIZE));
  sp = sz;

  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp = (sp - (strlen(argv[argc]) + 1)) & ~3;
    if(copyout(pgdir, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    ustack[3+argc] = sp;
  }
  ustack[3+argc] = 0;

  ustack[0] = 0xffffffff;  // fake return PC
  ustack[1] = argc;
  ustack[2] = sp - (argc+1)*4;  // argv pointer

  sp -= (3+argc+1) * 4;
  if(copyout(pgdir, sp, ustack, (3+argc+1)*4) < 0)
    goto bad;

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(curproc->name, last, sizeof(curproc->name));

  // Commit to the user image.
  oldpgdir = curproc->pgdir;
  curproc->pgdir = pgdir;
  curproc->sz = sz;
  curproc->tf->eip = elf.entry;  // main
  curproc->tf->esp = sp;

  switchuvm(curproc);

  freevm(oldpgdir);

  return 0;

 bad:
  if(pgdir)
    freevm(pgdir);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}
```

- `setupkvm` : The thing that I understood so far is that, the 1 page allocated inside `inituvm` is responsible for executing this exec function. **Whatever this exec function now wants to execute**, will be in a separate new kernel page table, hence a `setupkvm` is called once more. (what I exactly mean here will be more clear when I talk about other user processes)
- `allocuvm` : allocates pages for the executable it wants to run, here 1 page is sufficient for `init`
    
    ```cpp
    PAGE ALLOCATED HERE = 1
    ```
    
- `loaduvm` : loads the executable from disk into the newly allocated pages in `allocuvm`
- `allocuvm` : the new memory image so far only has executable **code and data**, now we also need **stack** space. Rather than allocating 1 page for the stack, it allocates 2. The 2nd one is the actual stack, 1st one serves as a guard page. This is the only page in the user’s memory which is marked as present but not user-accessible (`PTE_U` is cleared)
    
    ***Reason for having a guard page :*** To guard a stack growing oﬀ the stack page, xv6 places a guard page right below the stack. As the guard page is not mapped, if the stack runs oﬀ the stack page, the **hardware** will generate an **exception** because it cannot translate the faulting address.
    
    Strings containing the **command-line arguments**, as well as an **array of pointers** to them, are at the very top of the stack
    
    ![Untitled](How%20all%20of%20this%20works%207868ca41311a42b5ba45aa4e1b1033fd/Untitled%201.png)
    
    ```cpp
    PAGE ALLOCATED HERE = 2
    ```
    
- **trap frame update** : It is important to note that exec **does not replace/reallocate** the kernel stack. The exec system call only replaces the **user part of the memory image**, and does nothing to the kernel part. And if you think about it, there is no way the process can replace the kernel stack, because the process is executing in kernel mode on the kernel stack itself, and has important information like the trap frame stored on it.
    
    Recall that a process that makes the exec system call has moved into kernel mode to service
    the software interrupt of the system call. Normally, when the process moves back to user mode again (**by popping the trap frame on the kernel stack**), it is expected to return to the instruction after the system call. However, in the case of exec, the process doesn’t have to return to the instruction after exec when it gets the CPU next, but instead must start executing the new executable it just loaded from disk. So, the code in exec changes the return address in the trap frame to point to the entry address of the binary. It also sets the stack pointer in the trap frame to point to the top of the newly created user stack.
    
    ```cpp
    curproc->tf->eip = elf.entry;  // main
    curproc->tf->esp = sp;
    ```
    
- `switchuvm` : Finally, once all these operations succeed, `exec` **switches page tables** to start using the new memory image. That is, it writes the address of the new page table into the CR3 register, so that it can start accessing the new memory image when it goes back to userspace.
- `freevm` : the process that called exec i.e. `init` , was still using the old memory image. we free the old memory image that it was pointing to after updating
    - `deallocuvm` : The pages that are deleted here are the pages that were created in `inituvm` . So,
        
        ```cpp
        PAGE DEALLOCATED HERE = 1
        ```
        

The new memory image created by exec looks like this

![Untitled](How%20all%20of%20this%20works%207868ca41311a42b5ba45aa4e1b1033fd/Untitled%202.png)

At this point, the process that called `exec` can start executing on the new memory image
when it returns from trap

**Note** : exec waits until the end to do this switch of page tables, because if anything went wrong in the system call, exec returns from trap into the old memory image and prints out an error

## `sh`

Apart from `init`, all other processes are created by the fork system call. When the `init` process runs, it executes the **init executable**, whose main function forks a shell (`sh`) and starts listening to the user

### `fork`

- `allocproc` : returns `pid = 2` for `sh`
- `copyuvm` : once a child process is allocated, its memory image is setup as a complete copy of the parent’s memory. This is done using `copyuvm` .
    - `setupkvm` : inside copyuvm, a new kernel page table is created, and parent memory is allocated here.
    
    Recall that 3 pages were created in `exec` of `init` . These 3 pages are copied here
    
    ```cpp
    PAGES ALLOCATED HERE = 3
    ```
    

### `exec`

The description of exec for shell is the same as discussed for `init` . So lets just find out how many pages are allocated where

- `setupkvm`
- `allocuvm`
    
    The executable that `sh` wants to run needs more memory than `init` needed. So 2 pages are allocated here.
    
    ```cpp
    PAGES ALLOCATED HERE = 2
    ```
    
- `loaduvm`
- `allocuvm`
- `freevm`
    - `deallocuvm` The pages that were allocated (copied) using `copyuvm` are deallocated as they are pointing to old page directory
        
        ```cpp
        PAGE DEALLOCATED HERE = 3
        ```
        

Now let’s look at a general user process. Every other user process will fork `sh` . Let’s look at `echo`

## `echo`

### `fork`

- `allocproc`
- `copyuvm` : here 4 page will be copied (2 + 2 pages that were allocated inside `sh` exec)
    
    ```cpp
    PAGE ALLOCATED HERE = 4
    ```
    

### `growproc`

After fork, `growproc` is called to grow userspace memory image. `growproc` can be called by the `sbrk` system call. It basically increases the heap size

```cpp
PAGE ALLOCATED HERE = 8
```

### `exec`

- `allocuvm`
    
    ```cpp
    PAGE ALLOCATED HERE = 1
    ```
    
- `loaduvm`
- `allocuvm`
    
    ```cpp
    PAGE ALLOCATED HERE = 2
    ```
    
- `freevm`
    - `deallocuvm` : 4+8 pages deleted of old memory image
        
        ```cpp
        PAGE DEALLOCATED HERE = 12
        ```
        

### `exit`

- `freevm`
    - `deallocuvm` : 1+2 pages deleted of new memory image
        
        ```cpp
        PAGE DEALLOCATED HERE = 3
        ```

Finally, an overview of all the things said in a picture - because ofcourse a picture speaks more than a thousand words!        

![Untitled](How%20all%20of%20this%20works%207868ca41311a42b5ba45aa4e1b1033fd/Untitled%203.png)

---

# **`Final Words`**

Thanks for reading this far ! I am not sure how much helpful this has been for you, but I hope you can start navigating the memory management part of the xv6 codebase after reading this. Go through this writeup multiple times if you don't undersand anything and if you have some more time to spare, please go through [this](https://www.cse.iitb.ac.in/~mythili/os/notes/old-xv6/xv6-memory.pdf) doc. A lot of the part of this writeup has been taken or inspired from this.

May the force be with you so that you survive this offline! 

# **`Resource`**

- Xv6 book
- [Memory Management in Xv6](https://www.cse.iitb.ac.in/~mythili/os/notes/old-xv6/xv6-memory.pdf)
- [https://www.cs.virginia.edu/~cr4bd/4414/F2018/slides/20181011--slides-1up.pdf](https://www.cs.virginia.edu/~cr4bd/4414/F2018/slides/20181011--slides-1up.pdf)
- [https://www.cs.virginia.edu/~cr4bd/4414/F2019/slides/20191015--slides-1up.pdf](https://www.cs.virginia.edu/~cr4bd/4414/F2019/slides/20191015--slides-1up.pdf)

- [https://iitd-plos.github.io/os/2020/lec/l15.html](https://iitd-plos.github.io/os/2020/lec/l15.html)

- [https://pdos.csail.mit.edu/6.828/2009/lec/l5.html](https://pdos.csail.mit.edu/6.828/2009/lec/l5.html)
- [http://course.ece.cmu.edu/~ece447/s13/lib/exe/fetch.php?media=onur-447-spring13-lecture18-virtual-memory-iii-afterlecture.pdf](http://course.ece.cmu.edu/~ece447/s13/lib/exe/fetch.php?media=onur-447-spring13-lecture18-virtual-memory-iii-afterlecture.pdf)
- [https://github.com/YehudaShapira/xv6-explained](https://github.com/YehudaShapira/xv6-explained)
- [https://stackoverflow.com/questions/56258056/what-does-deallocation-function-in-xv6s-allocation-function](https://stackoverflow.com/questions/56258056/what-does-deallocation-function-in-xv6s-allocation-function)
- [https://www.cs.columbia.edu/~junfeng/11sp-w4118/lectures/mem.pdf](https://www.cs.columbia.edu/~junfeng/11sp-w4118/lectures/mem.pdf)
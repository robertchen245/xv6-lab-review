# Lab6: Copy-on-Write Fork for xv6
## The problem
The fork() system call in xv6 copies all of the parent process's user-space memory into the child.  
If the parent is large, copying can take a long time. Worse, the work is often largely wasted;  
for example, a fork() followed by exec() in the child will cause the child to discard the copied memory,  
probably without ever using most of it. On the other hand,  
if both parent and child use a page, and one or both writes it, a copy is truly needed.
## The solution
The goal of copy-on-write (COW) fork() is to defer allocating and copying physical memory pages for the child until the copies are actually needed, if ever.  
COW fork() creates just a pagetable for the child, with PTEs for user memory pointing to the parent's physical pages.   
COW fork() marks all the user PTEs in both parent and child as not writable. When either process tries to write one of these COW pages, the CPU will force a page fault.  
The kernel page-fault handler detects this case, allocates a page of physical memory for the faulting process, copies the original page into the new page, and modifies the relevant PTE in the faulting process to refer to the new page, this time with the PTE marked writeable.  
When the page fault handler returns, the user process will be able to write its copy of the page.

COW fork() makes freeing of the physical pages that implement user memory a little trickier. A given physical page may be referred to by multiple processes' page tables, and should be freed only when the last reference disappears. 

## 1.Implement copy-on write (hard)
```
Your task is to implement copy-on-write fork in the xv6 kernel. You are done if your modified kernel executes both the cowtest and usertests programs successfully. 
The "simple" test allocates more than half of available physical memory, and then fork()s. The fork fails because there is not enough free physical memory to give the child a complete copy of the parent's memory. 
When you are done, your kernel should pass all the tests in both cowtest and usertests. That is:
```
```
Here's a reasonable plan of attack.

Modify uvmcopy() to map the parent's physical pages into the child, instead of allocating new pages. Clear PTE_W in the PTEs of both child and parent.
Modify usertrap() to recognize page faults. When a page-fault occurs on a COW page, allocate a new page with kalloc(), copy the old page to the new page, and install the new page in the PTE with PTE_W set.
Ensure that each physical page is freed when the last PTE reference to it goes away -- but not before. A good way to do this is to keep, for each physical page, a "reference count" of the number of user page tables that refer to that page. Set a page's reference count to one when kalloc() allocates it. Increment a page's reference count when fork causes a child to share the page, and decrement a page's count each time any process drops the page from its page table. kfree() should only place a page back on the free list if its reference count is zero. It's OK to to keep these counts in a fixed-size array of integers. You'll have to work out a scheme for how to index the array and how to choose its size. For example, you could index the array with the page's physical address divided by 4096, and give the array a number of elements equal to highest physical address of any page placed on the free list by kinit() in kalloc.c.
Modify copyout() to use the same scheme as page faults when it encounters a COW page.
Some hints:

The lazy page allocation lab has likely made you familiar with much of the xv6 kernel code that's relevant for copy-on-write. However, you should not base this lab on your lazy allocation solution; instead, please start with a fresh copy of xv6 as directed above.
It may be useful to have a way to record, for each PTE, whether it is a COW mapping. You can use the RSW (reserved for software) bits in the RISC-V PTE for this.
usertests explores scenarios that cowtest does not test, so don't forget to check that all tests pass for both.
Some helpful macros and definitions for page table flags are at the end of kernel/riscv.h.
If a COW page fault occurs and there's no free memory, the process should be killed.
```
1.modify **uvmcopy()** parent page mapped to child. remove PTE_W 

2.modify **usertrap** 

3.released when all reference is removed 

4.use spinlock to protect operation of refcount

5.when using copyout you use data in kernel to modify processes' data, so there should be implementation same as usertrap

```c
struct refcount
{
  struct  spinlock rclock;
  unsigned char pa_counter[PHYSTOP/PGSIZE];
}ref;

void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
  {
    acquire(&ref.rclock);
    ref.pa_counter[(uint64)p/PGSIZE]=1; //set all free pages' reference count = 1
    release(&ref.rclock);
    kfree(p);
  }
}
```

```c
//uvmcopy() called by fork in vm.c
 for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0) // 
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    *pte=(*pte)&(~PTE_W); //set the PTE_W to zero
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    /*if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);*/
    if(mappages(new, i, PGSIZE, pa, flags) != 0){
      //kfree(mem);
      goto err;
    }
  }
```
```
the make qemu to run echo c
show there are scause 15 2 12 
instuction pagefault, store/AMO pagefault
mind that stval is the fault virtual address
```
```c
//in kalloc.c: freerange()
void
freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE)
    acquire(&ref.rclock);
    ref.pa_counter[(uint64)p/PGSIZE]=1; //set all free pages' reference count = 1
    release(&ref.rclock);
    kfree(p);
}
```
then define a global array and lock struct to count pa's reference times
```c
struct refcount
{
  struct  spinlock rclock;
  unsigned char pa_counter[PHYSTOP/PGSIZE];
}ref;

// init in kinit
void
kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&ref.rclock,"rclock");
  freerange(end, (void*)PHYSTOP);
}
```

```c
//in usertrap()
 else if (r_scause() == 15){ 
    uint64 va = r_stval();
    pte_t *pte;
    if((pte= walk(p->pagetable,va, 0))==0)
    panic("cow error");
    if (va >= MAXVA || (va <= PGROUNDDOWN(p->trapframe->sp) && va >= PGROUNDDOWN(p->trapframe->sp) - PGSIZE))// just like in lazy allocation. notice the valid of va
                                // va might out of stack frame
    {
      p->killed = 1;
    }
    else if((*pte)|PTE_RSW){
      if(Copy_On_Write(p->pagetable, pte) == -1)
      p->killed = 1;
    }else
    {
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
    }
  }
```
```c
// in vm.c
int
Copy_On_Write(pagetable_t pagetable,pte_t *pte)
{
  uint64 oldpa=PTE2PA(*pte);
  uint64 new;
  if((new=(uint64)kalloc())==0)
  {
    return -1;
  }
  acquire(&ref.rclock);
  if(ref.pa_counter[oldpa/PGSIZE]>1) //引用次数大于1
  {
    memmove((void*)new,(void*)oldpa,PGSIZE);
    *pte=PA2PTE(new)| PTE_W | PTE_R | PTE_X | PTE_U | PTE_V;
    ref.pa_counter[oldpa/PGSIZE]--;
    release(&ref.rclock);
    return 0;
  }else if(ref.pa_counter[oldpa/PGSIZE]==1) //只引用了一次
  {
    release(&ref.rclock);
    kfree((void*)new); //只是 修改写入权限，不需要额外内存
    *pte=(*pte)|PTE_W;
    return 0;
  }else{
    release(&ref.rclock);
    return -1;
  }
}
```
## 2.Result
### cowtest
<image src="cowtest.PNG"> 

### usertests
<image src="usertests.PNG"> 

### make grade
<image src="grade.PNG">
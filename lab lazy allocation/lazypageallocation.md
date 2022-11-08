# Lab4: Lazy Page Allocation
## 1.Eliminate allocation from sbrk() (easy)
```
Your first task is to delete page allocation from the sbrk(n) system call implementation, which is the function sys_sbrk() in sysproc.c. The sbrk(n) system call grows the process's memory size by n bytes, and then returns the start of the newly allocated region (i.e., the old size). Your new sbrk(n) should just increment the process's size (myproc()->sz) by n and return the old size. It should not allocate memory -- so you should delete the call to growproc() (but you still need to increase the process's size!).
```
```c
//in sysproc.c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz=myproc()->sz+n;
  /*if(growproc(n) < 0)
    return -1;*/
  return addr;
}
//modify like this
```
```
make qemu and echo hi:
terminal prints out:
usertrap(): unexpected scause 0x000000000000000f pid=3
            sepc=0x00000000000012ac stval=0x0000000000004008
panic: uvmunmap: not mapped
```
```
The "usertrap(): ..." message is from the user trap handler in trap.c; it has caught an exception that it does not know how to handle. Make sure you understand why this page fault occurs. The "stval=0x0..04008" indicates that the virtual address that caused the page fault is 0x4008.(va=0x4008)
```
## 2.Lazy allocation
```
Modify the code in trap.c to respond to a page fault from user space by mapping a newly-allocated page of physical memory at the faulting address, and then returning back to user space to let the process continue executing. You should add your code just before the printf call that produced the "usertrap(): ..." message. Modify whatever other xv6 kernel code you need to in order to get echo hi to work.
```
```
hints:
You can check whether a fault is a page fault by seeing if r_scause() is 13 or 15 in usertrap().
r_stval() returns the RISC-V stval register, which contains the virtual address that caused the page fault.
Steal code from uvmalloc() in vm.c, which is what sbrk() calls (via growproc()). You'll need to call kalloc() and mappages().
Use PGROUNDDOWN(va) to round the faulting virtual address down to a page boundary.
uvmunmap() will panic; modify it to not panic if some pages aren't mapped.
If the kernel crashes, look up sepc in kernel/kernel.asm
Use your vmprint function from pgtbl lab to print the content of a page table.
If you see the error "incomplete type proc", include "spinlock.h" then "proc.h".
```
```c
//in vm.c
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      panic("uvmunmap: walk");
    if((*pte & PTE_V) == 0)
      continue;// continue： ignore the unmapped pages
      //panic("uvmunmap: not mapped");
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
  }
}

```c
// in trap.c
  else if((which_dev = devintr()) != 0){
    // ok
    } else if ((r_scause()==15)||(r_scause()==13))// handle the pagefault here
    {
      //printf("usertrap(): page fault %p pid=%d\n", r_scause(), p->pid);
      uint64 va;
      uint64 mem;
      va=PGROUNDDOWN(r_stval());
      if(r_stval()>p->sz||r_stval()<PGROUNDUP(p->trapframe->sp) - 1)
      {
        p->killed=1;
      }
      else if((mem=(uint64)kalloc())==0) // kalloc then immediately judge because when stval is not vaild, you don't have to kalloc!!!
      {
        p->killed=1;
      }else
      {
        memset((void*)mem,0,PGSIZE);
        if(mappages(p->pagetable,va,PGSIZE,mem,PTE_R | PTE_W | PTE_X | PTE_U)!=0)//map failed
        {
          kfree((void*)mem);
          p->killed=1;
        }
      }

    }
```
### Echo hi result
<image src="echo hi.PNG"> 

## 3.Lazytests and Usertests (moderate)
```
Handle negative sbrk() arguments.
Kill a process if it page-faults on a virtual memory address higher than any allocated with sbrk().
Handle the parent-to-child memory copy in fork() correctly.
Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.
Handle out-of-memory correctly: if kalloc() fails in the page fault handler, kill the current process.
Handle faults on the invalid page below the user stack.
```
```
for negative sbrk() arguments
we have to shrink the proc heap size,
by which function can we do this?
unmdealloc(make sure modified size is above 0)
```
```c
//in sysproc.c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  if(n>0){
  myproc()->sz=myproc()->sz+n;
  }else if(myproc()->sz+n>0)
  {
    myproc()->sz=uvmdealloc(myproc()->pagetable,myproc()->sz,myproc()->sz+n); //pagetable oldsize newsize
  }else// if total is below 0
  {
    return -1;
  }
  /*if(growproc(n) < 0)
    return -1;*/
  return addr;
}
```
```
We should notice that though allocation is lazy, the deallocation on the contrary,is eager.
makes sense since we tend to save memory
```
```
the first lazy test raise panic("uvmunmap: walk");
add continue
```
```c
for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0) // walk get pte of this va in given pagetable
       continue;
      //panic("uvmunmap: walk");
```
```c
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;// pte is PPN+FLAGbits
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      continue;//don't care about unmapped pages
      //panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      continue;// same though
      //panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    if((mem = kalloc()) == 0)
      goto err;
    memmove(mem, (char*)pa, PGSIZE);
    if(mappages(new, i, PGSIZE, (uint64)mem, flags) != 0){
      kfree(mem);
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i / PGSIZE, 1);
  return -1;
}

```
```
Handle the case in which a process passes a valid address from sbrk() to a system call such as read or write, but the memory for that address has not yet been allocated.
pagefault: r_scause=13  syscall:r_scause=8
handle unmapped addr when pagefault hasn't occur.
write and read syscall need argaddr.
in argaddr check if the addr has mapped.
if not do alloc and mapping just like trap.c
```
```c
int
argaddr(int n, uint64 *ip) //ip as addr do alloc and map like in trap.c
{
  *ip = argraw(n);
  struct proc* p=myproc();
  
  if(walkaddr(p->pagetable,*ip)==0)//unmapped addr
  {
    if(PGROUNDDOWN(p->trapframe->sp)<*ip&&*ip<p->sz)
    {
    uint64 va=PGROUNDDOWN(*ip);
    uint64 mem=(uint64)kalloc();
    if(mem==0)
    {
      return -1;
    }
    else if(mappages(p->pagetable,va,PGSIZE,mem,PTE_R | PTE_W | PTE_X | PTE_U)!=0)// wrongly "=="
    {
      kfree((void*)mem);
      return -1;
    }
    }
    else{
      return -1;
    }
  }
  return 0;
}
```
## Result
<image src="lazygrade.PNG"> 

## Conclusion
```
we have used 4 continues,2 in uvmcopy and 2 in uvmunmap
in usertrap(): mind that do not hustle to kalloc before check the va's validity
(below p->sz , above userstack(always one page irrc))
Always remember to PGROUNDUP/DOWN before mapping
```

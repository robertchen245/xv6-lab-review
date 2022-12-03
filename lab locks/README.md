# Lab8: Locks
In this lab you'll gain experience in re-designing code to increase parallelism. A common symptom of poor parallelism on multi-core machines is high lock contention. 
Improving parallelism often involves changing both data structures and locking strategies in order to reduce contention. 
You'll do this for the xv6 memory allocator and block cache.
## 1.Memory Allocator(moderate)
```
The program user/kalloctest stresses xv6's memory allocator: three processes grow and shrink their address spaces, resulting in many calls to kalloc and kfree. kalloc and kfree obtain kmem.lock. 
kalloctest prints (as "#fetch-and-add") the number of loop iterations in acquire due to attempts to acquire a lock that another core already holds, for the kmem lock and a few other locks. 
The number of loop iterations in acquire is a rough measure of lock contention. The output of kalloctest looks similar to this before you complete the lab:

$ kalloctest
start test1
test1 results:
--- lock kmem/bcache stats
lock: kmem: #fetch-and-add 83375 #acquire() 433015
lock: bcache: #fetch-and-add 0 #acquire() 1260
--- top 5 contended locks:
lock: kmem: #fetch-and-add 83375 #acquire() 433015
lock: proc: #fetch-and-add 23737 #acquire() 130718
lock: virtio_disk: #fetch-and-add 11159 #acquire() 114
lock: proc: #fetch-and-add 5937 #acquire() 130786
lock: proc: #fetch-and-add 4080 #acquire() 130786
tot= 83375
test1 FAIL
acquire maintains, for each lock, the count of calls to acquire for that lock, and the number of times the loop in acquire tried but failed to set the lock. 
kalloctest calls a system call that causes the kernel to print those counts for the kmem and bcache locks (which are the focus of this lab) and for the 5 most contended locks. 
If there is lock contention the number of acquire loop iterations will be large. The system call returns the sum of the number of loop iterations for the kmem and bcache locks.

For this lab, you must use a dedicated unloaded machine with multiple cores. If you use a machine that is doing other things, the counts that kalloctest prints will be nonsense. 
You can use a dedicated Athena workstation, or your own laptop, but don't use a dialup machine.

The root cause of lock contention in kalloctest is that kalloc() has a single free list, protected by a single lock. 
To remove lock contention, you will have to redesign the memory allocator to avoid a single lock and list. The basic idea is to maintain a free list per CPU, each list with its own lock. 
Allocations and frees on different CPUs can run in parallel, because each CPU will operate on a different list. The main challenge will be to deal with the case in which one CPU's free list is empty, but another CPU's list has free memory; in that case, the one CPU must "steal" part of the other CPU's free list. 
Stealing may introduce lock contention, but that will hopefully be infrequent.

Your job is to implement per-CPU freelists, and stealing when a CPU's free list is empty. You must give all of your locks names that start with "kmem". 
That is, you should call initlock for each of your locks, and pass a name that starts with "kmem". Run kalloctest to see if your implementation has reduced lock contention. 
o check that it can still allocate all of memory, run usertests sbrkmuch. Your output will look similar to that shown below, with much-reduced contention in total on kmem locks, although the specific numbers will differ. Make sure all tests in usertests pass. make grade should say that the kalloctests pass.
```
```c
#define NCPU 8
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];
// initialize 8 kmem list for each CPU
```
```c
void
kinit()
{
  char lockname[8];
  for(int i=0;i<NCPU;i++)
  {
    snprintf(lockname,8,"kmem_%d",i);
    initlock(&kmem[i].lock,lockname);
  }
  freerange(end, (void*)PHYSTOP);
}
// everylocks hold their own lock
```
```c
void *
kalloc(void)
{
  struct run *r;
  int cur_cpu;
  push_off(); // turn off intr 
  cur_cpu=cpuid();
  acquire(&kmem[cur_cpu].lock);
  r = kmem[cur_cpu].freelist;
  if(r)
    kmem[cur_cpu].freelist = r->next;
  release(&kmem[cur_cpu].lock);
  pop_off();// turn on intr
  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  //when selflist is empty, go to steal others'
  if(!r)
  {
    for(int i=0;i<NCPU;i++)
    {
      acquire(&kmem[i].lock);
      r=kmem[i].freelist;
      if(r)
      {
        kmem[i].freelist = r->next;
      }
      release(&kmem[i].lock);
      if(r)// found one
      {
        memset((char*)r, 5, PGSIZE); // fill with junk
        return (void*)r;
      }
    }
  }
    return (void*)r; //return anyway
}

```
```c
void
kfree(void *pa)
{
  struct run *r;
  int cur_cpu;
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;
  push_off();
  cur_cpu=cpuid();
  acquire(&kmem[cur_cpu].lock);
  r->next = kmem[cur_cpu].freelist;
  kmem[cur_cpu].freelist = r;
  release(&kmem[cur_cpu].lock);
  pop_off();
}
```
<image src="kalloctest.PNG"> 

## 2.Buffer Cache(hard)

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
```
Modify the block cache so that the number of acquire loop iterations for all locks in the bcache is close to zero when running bcachetest. 
Ideally the sum of the counts for all locks involved in the block cache should be zero, but it's OK if the sum is less than 500. 
Modify bget and brelse so that concurrent lookups and releases for different blocks that are in the bcache are unlikely to conflict on locks 
(e.g., don't all have to wait for bcache.lock). You must maintain the invariant that at most one copy of each block is cached. 
When you are done, your output should be similar to that shown below
 (though not identical). Make sure usertests still passes. make grade should pass all tests when you are done.
```
use hashtable/bucket to speed up buffer cache operation
```c
//param.h
#define NBUCKET      17 //number of hashtable
#define NSIZE        4 //size of each bucket
// or just use NBUF number
```
```c
// init all hashtables of bufs
void
binit(void)
{
  struct buf *b;
  for(int i=0;i<NBUCKET;i++)
  {
    initlock(&bcache[i].lock,"bcache");
    for(int j=0;j<NSIZE;j++)
    {// global var's value is 0 already
      b=&bcache[i].buf[j];
      initsleeplock(&b->lock,"sleeplock");
      b->timestamp=ticks;
    }
  }
}
int hashed(uint blockno)
{
  return blockno%NBUCKET;
}

static struct buf*
bget(uint dev, uint blockno)
{
  int id=hashed(blockno);
  acquire(&bcache[id].lock);
  struct buf *b;
  for(int i=0;i<NSIZE;i++)
  {
    b=&bcache[id].buf[i];
    if (b->dev==dev&&b->blockno ==blockno)
    {
      b->refcnt++;
      release(&bcache[id].lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  // find the LRU
  struct buf *temp=0;
  uint temptime=-1;// max
  //temptime=bcache[id].buf[0].timestamp;
  for(int i=0;i<NSIZE;i++)
  {
    if(bcache[id].buf[i].refcnt==0&&bcache[id].buf[i].timestamp<=temptime)
    temp = &bcache[id].buf[i];
  }
  if(temp)
  {
    temp->dev=dev;
    temp->blockno=blockno;
    temp->valid=0;
    temp->refcnt=1;
    release(&bcache[id].lock);
    acquiresleep(&temp->lock);
    return temp;
  }
  panic("bget: no buffers");
}
```
```c
void
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  b->refcnt--;
  if (b->refcnt == 0) {
    // no one is waiting for it.
    /*b->next->prev = b->prev;
    b->prev->next = b->next;
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    bcache.head.next->prev = b;
    bcache.head.next = b;*/
    b->timestamp=ticks;// get a stamp of release time
  }
}
```
```
notice the above brelse, since we apply hashtable instead and implement LRU 
strategy by timestamp, so there is no need to acquire the table lock because the 
structure won't be modified like Linked-List.
```
<image src="grade.PNG"> 

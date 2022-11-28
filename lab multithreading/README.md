# Lab7: Multithreading
This lab will familiarize you with multithreading. 

You will implement switching between threads in a user-level threads package, 

use multiple threads to speed up a program, and implement a barrier.


## 1.Uthread: switching between threads (moderate)

```
In this exercise you will design the context switch mechanism for a user-level threading system, and then 
implement it. To get you started, your xv6 has two files user/uthread.c and user/uthread_switch.S, and a 
rule in the Makefile to build a uthread program. uthread.c contains most of a user-level threading 
package, and code for three simple test threads. The threading package is missing some of the code to 
create a thread and to switch between threads.

Your job is to come up with a plan to create threads and save/restore registers to switch between 
threads, and implement that plan. When you're done, make grade should say that your solution passes the 
uthread test.
```
```c
// define a user context structure
struct ucontext
{
  uint64 ra;
  uint64 sp;
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct ucontext uctext;
};
struct thread all_thread[MAX_THREAD];
struct thread *current_thread;
extern void thread_switch(uint64, uint64); // uthreadswitch.S defined in asm file
```
```S
#the Switch function same as kernel switch
	.text

	/*
         * save the old thread's registers,
         * restore the new thread's registers.
         */

	.globl thread_switch
thread_switch:
	 	sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
	/* YOUR CODE HERE */
	ret    /* return to ra */

```
```c
void 
thread_schedule(void)
{
  struct thread *t, *next_thread;

  /* Find another runnable thread. */
  next_thread = 0;
  t = current_thread+1 ;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;//found one
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread) {         /* switch threads?  */
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    thread_switch((uint64)&t->uctext,(uint64)&next_thread->uctext);//old and next exchange //register -> context1     context2 -> register
    /* YOUR CODE HERE
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
  } else
    next_thread = 0;
}
```
```c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;//find a free structure to fill in
  }
  t->state = RUNNABLE;
  t->uctext.ra=(uint64)func;
  t->uctext.sp=(uint64)&t->stack+(STACK_SIZE-1);
  // YOUR CODE HERE
}
// save switchable threads into the the queue
```
## 2. Using Thread
in this assignment you will explore parallel programming with threads and locks using a hash table. You should do this assignment on a real Linux or MacOS computer (not xv6, not qemu) that has multiple cores. Most recent laptops have multicore processors.

This assignment uses the UNIX pthread threading library. You can find information about it from the manual page, with man pthreads, and you can look on the web, for example here, here, and here.

The file notxv6/ph.c contains a simple hash table that is correct if used from a single thread, but incorrect when used from multiple threads. In your main xv6 directory (perhaps ~/xv6-labs-2020), type this:

$ make ph
$ ./ph 1
Note that to build ph the Makefile uses your OS's gcc, not the 6.S081 tools. The argument to ph specifies the number of threads that execute put and get operations on the the hash table. After running for a little while, ph 1 will produce output similar to this:
100000 puts, 3.991 seconds, 25056 puts/second
0: 0 keys missing
100000 gets, 3.981 seconds, 25118 gets/second
```c
//use pthread lib func and type to utilize mutex(lock used for multithread communicating)
pthread_mutex_t lock;            // declare a lock
pthread_mutex_init(&lock, NULL); // initialize the lock
pthread_mutex_lock(&lock);       // acquire lock
pthread_mutex_unlock(&lock);     // release lock
pthread_mutex_init();
```
```c
//add coarse grained lock to the buckets
static 
void put(int key, int value)
{
    pthread_mutex_lock(&lock);
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&lock);
}

```
to speed up the process, we reduce the grain of the lock
make it more and separately obtained by individual bucket.
```c
static 
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    pthread_mutex_lock(&lock[i]);  //5 locks. when current bucket is locked,other threads can visit other bucket.
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&lock[i]);
  }
}
```
result of single thread:

<image src="single thread.PNG"> 

result of double thread: 

<image src="double thread.PNG"> 

## 3. barrier


In this assignment you'll implement a barrier: a point in an application at which all participating threads must wait until all other participating threads reach that point too. 
You'll use pthread condition variables, which are a sequence coordination technique similar to xv6's sleep and wakeup.

You should do this assignment on a real computer (not xv6, not qemu).

The file notxv6/barrier.c contains a broken barrier.

$ make barrier
$ ./barrier 2
barrier: notxv6/barrier.c:42: thread: Assertion `i == t' failed.
The 2 specifies the number of threads that synchronize on the barrier ( nthread in barrier.c). Each thread executes a loop. 
In each loop iteration a thread calls barrier() and then sleeps for a random number of microseconds. The assert triggers, because one thread leaves the barrier before the other thread has reached the barrier. 
The desired behavior is that each thread blocks in barrier() until all nthreads of them have called barrier().
Your goal is to achieve the desired barrier behavior. 
In addition to the lock primitives that you have seen in the ph assignment, you will need the following new pthread primitives; look here and here for details.

pthread_cond_wait(&cond, &mutex);  // go to sleep on cond, releasing lock mutex, acquiring upon wake up
pthread_cond_broadcast(&cond);     // wake up every thread sleeping on cond
Make sure your solution passes make grade's barrier test.
```c
//additional use these
pthread_cond_wait(&cond, &mutex); //sleep
pthread_cond_broadcast(&cond);//wakeup
```
```c
static void 
barrier()
{
  pthread_mutex_lock(&bstate.barrier_mutex); // notice the sequence   acquire sleep wakeup release
  bstate.nthread++;// every thread reach this round of barrier count
  if(bstate.nthread==nthread)   // when all threads reach this round
  {
  pthread_cond_broadcast(&bstate.barrier_cond);// the latest thread wake them all up.
  bstate.round++; //let's move to next round
  bstate.nthread=0;//new round do thread recount
  }
  else
  pthread_cond_wait(&bstate.barrier_cond,&bstate.barrier_mutex); //sleeper sleeps
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_unlock(&bstate.barrier_mutex);release the mutex
}
```

result(1,2,3,4 threads):
<image src="barrier thread.PNG"> 

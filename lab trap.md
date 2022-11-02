 # Lab4: Trap
 ## 1.Risc-V Assembly (easy)
 ## call.c as follows
 ```c
#include "kernel/param.h"
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int g(int x) {
  return x+3;
}

int f(int x) {
  return g(x);
}

void main(void) {
  printf("%d %d\n", f(8)+1, 13);
  exit(0);
}
```
 ### Which registers contain arguments to functions? For example, which register holds 13 in main's call to printf?
in XV6-Calling: ra for return address,s0 for saved register
in void main
``` 
    void main(void) {
    1c:	1141                	addi	sp,sp,-16
    1e:	e406                	sd	ra,8(sp)
    20:	e022                	sd	s0,0(sp)
    22:	0800                	addi	s0,sp,16
    printf("%d %d\n", f(8)+1, 13);
    24:	4635                	li	a2,13
    26:	45b1                	li	a1,12
    28:	00000517          	auipc	a0,0x0
    2c:	7b050513          	addi	a0,a0,1968 # 7d8 <malloc+0xea>
    30:	00000097          	auipc	ra,0x0
    34:	600080e7          	jalr	1536(ra) # 630 <printf>
    exit(0);
    38:	4501                	li	a0,0
    3a:	00000097          	auipc	ra,0x0
    3e:	27e080e7          	jalr	638(ra) # 2b8 <exit>
```
```
obviously in 24~26 a1 and a2 holds 13 and 12
```
### Where is the call to function f in the assembly code for main? Where is the call to g? (Hint: the compiler may inline functions.)
```
in line 0x26, we could indicate that compiler has done some pretreatment to f, send immediate number 12 (namely result of f(8)+1) to argument
for call to g, we can find that in 
```
```
int g(int x) {
   0:	1141                	addi	sp,sp,-16
   2:	e422                	sd	s0,8(sp)
   4:	0800                	addi	s0,sp,16
  return x+3;
}
   6:	250d                	addiw	a0,a0,3
   8:	6422                	ld	s0,8(sp)
   a:	0141                	addi	sp,sp,16
   c:	8082                	ret

000000000000000e <f>:

int f(int x) {
   e:	1141                	addi	sp,sp,- 16
  10:	e422                	sd	s0,8(sp)
  12:	0800                	addi	s0,sp,16
  return g(x);
}
  14:	250d                	addiw	a0,a0,3
  16:	6422                	ld	s0,8(sp)
  18:	0141                	addi	sp,sp,16
  1a:	8082                	ret
```
```
There is also a pretreatment: return g(x) directly use the same code in g func, avoiding to go into g to save some time
```
### What value is in the register ra just after the jalr to printf in main?
in
(ra for return address)
```asm
  2c:	7b050513          	addi	a0,a0,1968 # 7d8 <malloc+0xea>
  30:	00000097          	auipc	ra,0x0
  34:	600080e7          	jalr	1536(ra) # 630 <printf>
  exit(0);
  38:	4501                	li	a0,0
```
```
PC:30h
auipc ra,0x0 ---> ra=0x00000030 0x0 as imme num set to high 20bits in ra  
jalr 1536(ra)---> jump to ra+1536(set PC=30h+1536(600h))==630h  
in main, the next instuction's addr is 0x38 
so, register ra is 0x00000038 (when printf finished ra must be the next addr of instruction in caller function) 
```
```
The indirect jump instruction JALR (jump and link register) uses the I-type encoding. The target  
address is obtained by adding the sign-extended 12-bit I-immediate to the register rs1, then setting  
the least-significant bit of the result to zero. The address of the instruction following the jump  
(pc+4) is written to register rd. Reg
```
representing 0x600080e7 0x600 for offset(or imme) 00001 for ra ,000 for func code 00001 for ra itself 11100111 for jalr  

    
### Run the following code.
```c
unsigned int i = 0x00646c72;
printf("H%x Wo%s", 57616, &i);
```
      
### What is the output? Here's an ASCII table that maps bytes to characters.
### The output depends on that fact that the RISC-V is little-endian. If the RISC-V were instead big-endian what would you set i to in order to yield the same output? Would you need to change 57616 to a different value?

### Here's a description of little- and big-endian and a more whimsical description.
```
The output is "HE110 World".(no line feed)
in C print format, %s for string.
These two differ from %d, that they receive an addr rather and variable name.(addr+0 to addr+n)
'r'=0111 0010b 
'l'=0110 1100b
'd'=0110 0100b
in big endian we'll set 0x726c6400 instead
immediate num 57616 in hex is E110
since RISC-V is little Endian, when format printing 57616 in hex(%x), E110 will be printed out as so.
However we don't have to change 57616 because 57616D IS E110, nothing to do with What-Endian
```
### In the following code, what is going to be printed after 'y='? (note: the answer is not a specific value.) Why does this happen?

	printf("x=%d y=%d", 3);
```
we set the void main as:
```
```c
void main(void) {
  //printf("%d %d\n", f(8)+1, 13);
  printf("x=%d,y=%d",3);
  exit(0);
}
```
```
the output is
x=3,y=5221
let's look into asm code
```
```
  printf("x=%d,y=%d",3);
  24:	458d                	li	a1,3
  26:	00000517          	auipc	a0,0x0
  2a:	7aa50513          	addi	a0,a0,1962 # 7d0 <malloc+0xe4>
  2e:	00000097          	auipc	ra,0x0
  32:	600080e7          	jalr	1536(ra) # 62e <printf>
```
```
no li a2,data existing
indicated that printf function will use a2 register no matter what
```
## 2. Backtrace(moderate)
### For debugging it is often useful to have a backtrace: a list of the function calls on the stack above the point at which the error occurred.
```
in bttest.c
```
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  sleep(1);
  exit(0);
}
```
```
it calls sleep()// namely sys_sleep
```
```
The GCC compiler stores the frame pointer of the currently executing function in the register s0. Add the following function to kernel/riscv.h:
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```
```
frame pointer:fp
stack pointer:sp
watch lecture 5 to understand it

Review ASM by RISCV
Stack pointer from high to low.
allocate 0------------------------------160 of stack sp=160. do addi sp,sp,16 to
                                         sp
create   0---------------------144======160
                                sp
store ra,fp in the stack(s0 reg is fp reg)
```
```
the ra and previous fp value is in the offset-8 and offset-16
function traceback in printf.c follows
```
```c
void backtrace(void)
{
  uint64 s0=r_fp(); //get current framepage's addr
  printf("backtrace:\n");
  while(PGROUNDUP(s0)-PGROUNDDOWN(s0)==PGSIZE)//check whether allocated a full pages or not
  {
    printf("%p\n",*(uint64*)(s0-8));// current framepage's return addr(pushed ra+4) print it out in hex
    s0=*(uint64*)(s0-16);//get the previous frame pages addr 
  }
}
```
```
result in qemu
```
<image src="tracebackresult.png">

## 3.Alarm(hard)
```
In this exercise you'll add a feature to xv6 that periodically alerts a process as it uses CPU time. This might be useful for compute-bound processes that want to limit how much CPU time they chew up, or for processes that want to compute but also want to take some periodic action. More generally, you'll be implementing a primitive form of user-level interrupt/fault handlers; you could use something similar to handle page faults in the application, for example. Your solution is correct if it passes alarmtest and usertests.
```
```
To add a syscall
1.modify usys.pl to add usertrap
2.Modify the syscall() function in kernel/syscall.c to print the trace output. You will need to add an array of syscall names to index into.
```
```c
void
periodic()
{
  count = count + 1;
  printf("alarm!\n");
  sigreturn();
}

// tests whether the kernel calls
// the alarm handler even a single time.
void
test0()
{
  int i;
  printf("test0 start\n");
  count = 0;
  sigalarm(2, periodic);
  //noticed that fn call sigreturn to return
```
```
idea:
trap return to handler, but remember to protect register
procstruct init by allocproc, recycled in by freeproc
```
```c
if(which_dev == 2) //handle the time interrupt
    yield();
```
```
every a definite time , cpu will raise a time interrupt in
this interrupt we can set interval increase
```
```c
uint64
sys_sigalarm(void)
{
  int interval;
  uint64 handlerAddr;
  if(argint(0,&interval)<0||argaddr(1, &handlerAddr)<0)//get the handler function addr in a1 //get the tick times in a0
  return -1;
  struct proc *p=myproc();
  p->alarm_interval=interval;
  p->alarm_handler=(void*)handlerAddr;
  return 0;
}
uint64
sys_sigreturn(void)
{
  struct proc* p=myproc();
  memmove(p->trapframe,p->alarm_trapframe,sizeof(struct trapframe));
  p->ishandling=0;
  return 0; //test 0 for now
}

```
```c
//in proc.h
  int alarm_interval;          //interval of every tick
  void (*alarm_handler)();     //handler function
  int ticks_count;             //ticks count
  int ishandling;              // is cpu handling?
  struct trapframe* alarm_trapframe; // saved trapframe ,used for return to pc
```
```
alarmtest/usertests result and make grade as follow:
```
<image src="alarmresult.PNG">
<image src="usertestresult.PNG">
<image src="trapgrade.PNG">
	
finished usertrap in trap.c
```c
void
usertrap(void)
{
  int which_dev = 0;

  if((r_sstatus() & SSTATUS_SPP) != 0)
    panic("usertrap: not from user mode");

  // send interrupts and exceptions to kerneltrap(),
  // since we're now in the kernel.
  w_stvec((uint64)kernelvec);

  struct proc *p = myproc();
  
  // save user program counter.
  p->trapframe->epc = r_sepc();
  
  if(r_scause() == 8){
    // system call

    if(p->killed)
      exit(-1);

    // sepc points to the ecall instruction,
    // but we want to return to the next instruction.
    //if called by sigalarm , epc has to change to handlerAddr
    p->trapframe->epc += 4;

    // an interrupt will change sstatus &c registers,
    // so don't enable until done with those registers.
    intr_on();

    syscall();
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
    printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
    p->killed = 1;
  }

  if(p->killed)
    exit(-1);

  // give up the CPU if this is a timer interrupt. 时间中断
  if(which_dev == 2){
    if(++p->ticks_count==p->alarm_interval&&p->ishandling==0){//tick reach set count and proc is not handling
        memmove(p->alarm_trapframe,p->trapframe,sizeof(struct trapframe));// save trapframe to alarm_trapframe
        p->ishandling=1;//start handling
        p->trapframe->epc = (uint64)p->alarm_handler; //cast to timeinterrupt
        p->ticks_count = 0;// 
  }
    yield();
  }
  usertrapret();
}
```

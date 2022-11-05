# Lab3: Pgtbl
## 1.Print a page table (easy)
```
To help you learn about RISC-V page tables, and perhaps to aid future debugging, your first task is to write a function that prints the contents of a page table.

Define a function called vmprint(). It should take a pagetable_t argument, and print that pagetable in the format described below. Insert if(p->pid==1) vmprint(p->pagetable) in exec.c just before the return argc, to print the first process's page table. You receive full credit for this assignment if you pass the pte printout test of make grade.
```
```c
//in exec.c add 
if(p->pid==1) vmprint(p->pagetable)
```
```c
//define vmprint in vm.c
void Walk_Print(pagetable_t pagetable,int level):

void vmprint(pagetable_t pagetable)
{
  printf("page table %p\n",pagetable);
  Walk_Print(pagetable,1);
}


void Walk_Print(pagetable_t pagetable,int level)
{//there are 512 pte in a pagetable
  for(int i=0;i<512;i++)
  {
    pte_t pte=pagetable[i];
    if(pte&PTE_V)
    {
      for(int j=0;j<level;j++)
      {
        printf("..");
        if(j!=level-1) printf(" ");//Last level don't need space
      }
      printf("%d: pte %p pa %p\n",i,pte,PTE2PA(pte));
      if((pte & (PTE_R|PTE_W|PTE_X))==0)
      {
        uint64 child = PTE2PA(pte);// has to trans to pa
        Walk_Print((pagetable_t)child,level+1);
      }
    }
}
}
```
```c
//Define the prototype for vmprint in kernel/defs.h so that you can call it from exec.c.
void            vmprint(pagetable_t pagetable);
void            Walk_Print(pagetable_t pagetable,int level);
```
result:
<<image src="printpgtbl.png">>

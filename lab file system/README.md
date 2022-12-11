# Lab9: File System
## 1.Large files(moderate)
```
In this assignment you'll increase the maximum size of an xv6 file. Currently xv6 files are limited to 268 blocks, or 268*BSIZE bytes (BSIZE is 1024 in xv6). This limit comes from the fact that an xv6 inode contains 12 "direct" block numbers and one "singly-indirect" block number, which refers to a block that holds up to 256 more block numbers, for a total of 12+256=268 blocks.

The bigfile command creates the longest file it can, and reports that size:

$ bigfile
..
wrote 268 blocks
bigfile: file is too small
$
The test fails because bigfile expects to be able to create a file with 65803 blocks, but unmodified xv6 limits files to 268 blocks.
You'll change the xv6 file system code to support a "doubly-indirect" block in each inode, containing 256 addresses of singly-indirect blocks, 
each of which can contain up to 256 addresses of data blocks. The result will be that a file will be able to consist of up to 65803 blocks, 
or 256*256+256+11 blocks (11 instead of 12, because we will sacrifice one of the direct block numbers for the double-indirect block).
```
```
Your Job
Modify bmap() so that it implements a doubly-indirect block, in addition to direct blocks and a singly-indirect block. 
You'll have to have only 11 direct blocks, rather than 12, to make room for your new doubly-indirect block; 
you're not allowed to change the size of an on-disk inode. 
The first 11 elements of ip->addrs[] should be direct blocks; 
the 12th should be a singly-indirect block (just like the current one); 
the 13th should be your new doubly-indirect block. 
You are done with this exercise when bigfile writes 65803 blocks and usertests runs successfully: 
```
```
mkfs use fs.c fs.h to build up a virtual disk(on disk)
```
```c
//the prototype of bmap
static uint
bmap(struct inode *ip, uint bn) //the block no is logical bn（from 0 to inf）
{
  uint addr, *a;
  struct buf *bp;// the buf is buf cache

  if(bn < NDIRECT){ //from b0 to b11 ,12 blocks can be found in inode->addrs[0-11], when bn is 12-267 (bn-12) go to addrs[12] to find()
    if((addr = ip->addrs[bn]) == 0)// NINDIRECT = 1024/4=256
      ip->addrs[bn] = addr = balloc(ip->dev);//inode struct is the mapping of dinode(inside disk)
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range"); 
}
```
```
routine:
make some places for 2-layer indirectblocks
modify NDIRECT NINDIRECT
```
```c
#define NDIRECT 11 //from 12 to 11 and addr remains 13
#define NINDIRECT (BSIZE / sizeof(uint))
#define N2INDIRECT (BSIZE/ sizeof(uint))*(BSIZE / sizeof(uint))
#define MAXFILE (NDIRECT + NINDIRECT+N2INDIRECT) // OS only allocate blocks below MAXFILE

// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+2];   // Data block addresses
};
// also in file.h synchronize the inode struct
```
after changes above, runs bigfiles shows wrote 267 block ratherthan 268, meaning losing 1 block (12->11)
```c
//finished bmap
static uint
bmap(struct inode *ip, uint bn) //the block no is logical bn（from 0 to inf）
{
  uint addr, *a,*b;
  struct buf *bp,*bp2;// the buf is buf cache

  if(bn < NDIRECT){ //from b0 to b11 ,12 blocks can be found in inode->addrs[0-11], when bn is 12-267 (bn-12) go to addrs[12] to find()
    if((addr = ip->addrs[bn]) == 0)// NINDIRECT = 1024/4=256
      ip->addrs[bn] = addr = balloc(ip->dev);//inode struct is the mapping of dinode(inside disk)
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
  bn-=NINDIRECT;
  if(bn < N2INDIRECT){
    // Load 2-indirect block, allocating if necessary.
    if((addr=ip->addrs[NDIRECT+1])==0)
      ip->addrs[NDIRECT+1] = addr = balloc(ip->dev);
    bp = bread(ip->dev,addr);
    a = (uint*)bp->data;
    if((addr = a[bn/256])==0){
      a[bn/256] = addr=balloc(ip->dev); // allocate blocks for 2-indirect table
      log_write(bp);//if modified, need log_write. (commit changes)
    }
    bp2 = bread(ip->dev,addr);
    b = (uint*)bp2->data;
    if((addr=b[bn%256])==0){
      b[bn%256] = addr=balloc(ip->dev);
      log_write(bp2);
    }
    brelse(bp2);
    brelse(bp);
    return addr;
  }
  panic("bmap: out of range"); 
}
```
<image src="largefile.PNG"> 

## 2.Symbolic Link
```
In this exercise you will add symbolic links to xv6. Symbolic links (or soft links) refer to a linked file by pathname; 
when a symbolic link is opened, the kernel follows the link to the referred file. 
Symbolic links resembles hard links, but hard links are restricted to pointing to file on the same disk, 
while symbolic links can cross disk devices. 
Although xv6 doesn't support multiple devices, 
implementing this system call is a good exercise to understand how pathname lookup works.
```
```
Your job
You will implement the symlink(char *target, char *path) system call, which creates a new symbolic link at path that refers to file named by target. 
For further information, see the man page symlink. 
To test, add symlinktest to the Makefile and run it. 
Your solution is complete when the tests produce the following output (including usertests succeeding).
```
```
add a syscall sys_symlock()
regular routine:
usys.pl
sysfile.c
syscall.c
syscall.h
user.h
```
```c
uint64
sys_symlink(void)
{
  return 0;
}
```
```c
//stat.h
//the type of file
#define T_DIR     1   // Directory
#define T_FILE    2   // File
#define T_DEVICE  3   // Device
#define T_SYMLINK 4   // soft(symbolic) link
```
```c
//fcntl.h
#define O_RDONLY  0x000
#define O_WRONLY  0x001
#define O_RDWR    0x002
#define O_CREATE  0x200
#define O_TRUNC   0x400
#define O_NOFOLLOW 0x800 //the 12th bit

```c
uint64
sys_symlink(void)
{
  char target[MAXPATH]; //we are going to create a symlink in target.
  char path[MAXPATH];
  struct inode *ip;
  if(argstr(0,target,MAXPATH)<||argstr(1,path,MAXPATH))
    return -1;
  begin_op();
  if((ip=create(path,T_SYMLINK,0,0))==0) //在path的地方新建一个inode (locked)
  {
    end_op();
    return -1;
  }
  if(writei(ip,0,(uint64)target,0,MAXPATH)<MAXPATH)
  {
    iunlockput(ip); // put ref-1 , just create ,don't have to keep inode in mem
    end_op();
    return -1;
  } // write target into inode
  iunlockput(ip);
  end_op();
  return 0;
}
```
```c
// beware that writei aims to copy src content in to the inode's correspond bcache then commit changes
int
writei(struct inode *ip, int user_src, uint64 src, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > MAXFILE*BSIZE)
    return -1;

  for(tot=0; tot<n; tot+=m, off+=m, src+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));// bp read ip's blocks' buffercache read every mapped cache
    m = min(n - tot, BSIZE - off%BSIZE);
    if(either_copyin(bp->data + (off % BSIZE), user_src, src, m) == -1) {
      brelse(bp);
      break;
    }
    log_write(bp);
    brelse(bp);
  }

  if(off > ip->size)
    ip->size = off;

  // write the i-node back to disk even if the size didn't change
  // because the loop above might have called bmap() and added a new
  // block to ip->addrs[].
  iupdate(ip);

  return tot;
}
```
```c
if(ip->type==T_SYMLINK&&!(omode & O_NOFOLLOW))// follow 
  {
    for(int i = 0; i < 10; ++i) {
      if(readi(ip, 0, (uint64)path, 0, MAXPATH) != MAXPATH) {
        iunlockput(ip);
        end_op();
        return -1;
      }
      iunlockput(ip);
      ip = namei(path);
      if(ip == 0) {
        end_op();
        return -1;
      }
      ilock(ip);
      if(ip->type != T_SYMLINK)
        break;
    }
    if(ip->type == T_SYMLINK) {// if still symlink then error
      iunlockput(ip);
      end_op();
      return -1;
    }
  }
```
<image src="symlink.PNG"> 
<image src="fsgrade.PNG"> 

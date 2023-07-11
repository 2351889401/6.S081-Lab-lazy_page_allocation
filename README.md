## 虽然本次实验的难度仅为 moderate ，但是需要耐心的调试，通过全部的 usertests；此外，本次实验有很多值得思考的点。
## 总体来说，细节的完成全部内容并不轻松，需要仔细考虑虚拟内存、页表相关的内容。不过经过这次实验，确实对虚拟内存、页表的掌握更加深刻。  
  
### 实验主题：堆内存的延迟分配（xv6 lazy page allocation：https://pdos.csail.mit.edu/6.S081/2020/labs/lazy.html ）
### 在 xv6 内核中，sbrk()系统调用总是会立即分配应用程序申请的内存。如果申请的空间很大，那么内存分配是一件比较耗时间的事情；此外，应用程序申请的内存可能并不立即使用；而且可能申请的内存根本用不到，因为申请的最大内存常常是为了程序处理的最坏情况考量的。
### 因此，堆内存延迟分配，在真正使用到这一块内存区域时才进行分配，应该是一个更合理的选择。实现延迟分配的做法是：sbrk()系统调用仅仅增加地址空间（内存大小），而不在页表中注册相应的内存区域，在真正使用到这些内存区域时，产生 “缺页错误（page fault exception）”，让内核中的异常处理程序（“kernel/trap.c/usertrap()”）再去申请物理内存给进程使用。  

**1.** lazy allocation 第一步，将sys_sbrk()函数中仅仅增加地址空间（内存大小），而不在页表中注册相应的内存区域，也就是sys_sbrk()中如下的代码段  
```
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz += n;
  // if(growproc(n) < 0)
  //   return -1;

  return addr;
}
```
此时，在命令行中输入“**echo hi**”，会出现下面的错误：  
![](https://github.com/2351889401/6.S081-Lab-lazy_page_allocation/blob/main/images/echo_hi_fault.png)
  
分析为什么修改了“**sys_sbrk()**”会导致命令行输入“**echo hi**”出现上图的错误？  
（1）上图中**scause**值为**15**，查阅《**RISC-V Instruction Set Manual**》，发现这个原因是“**store/AMO page fault**”，也就是是一条存储指令引发的异常  
  
（2）**sepc**寄存器的值是“**0x12ac**”，标志着发生异常时应用程序的**PC**值。我们是执行“**shell**”程序时发生的异常，所以应当去“**sh.asm**”中查看“**0x12ac**”地址发生了什么。  
```
void*
malloc(uint nbytes)
{
    ...
  p = sbrk(nu * sizeof(Header));
    ...
  hp->s.size = nu;
    12ac:	01652423          	sw	s6,8(a0)
    ...
}
```  
发现这确实是一条“**store**”指令，与前面**scause**值15对应上了。再看一下，该指令是属于“**malloc()**”函数的一部分。并且**malloc**函数中调用了“**sbrk()**”。  
  
（3）所以我们现在考虑的问题流程是这样的（**pid**和**stval**的分析在后面）：  
![](https://github.com/2351889401/6.S081-Lab-lazy_page_allocation/blob/main/images/think1.png)  
我们考虑 “**命令行程序**” 的执行流程：  
“**fork()**” **->** “**exec()**”  
经过测试和最开始的图片中的“**pid=3**”，说明子进程成功完成了“**fork()**”这一步，而“**exec()**”中的测试语句没有输出，说明问题发生在“**fork**”之后，“**exec**”之前。  

（4）我们回到“**shell**”程序中，“**main**”函数中主要是下面的循环：  
```
// Read and run input commands.
  while(getcmd(buf, sizeof(buf)) >= 0){
    if(buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' '){
      // Chdir must be called by the parent, not the child.
      buf[strlen(buf)-1] = 0;  // chop \n
      if(chdir(buf+3) < 0)
        fprintf(2, "cannot cd %s\n", buf+3);
      continue;
    }
    if(fork1() == 0)
      runcmd(parsecmd(buf));
    wait(0);
  }
```
其中的“**fork1()**”完成“**fork()**”的任务，所以问题定位可能就在于下面的 “**runcmd(parsecmd(buf));**” 语句。  
在“**parsecmd()**”函数的执行流程中，最终定位到“**execcmd()**”函数  
```
struct cmd*
execcmd(void)
{
  struct execcmd *cmd;

  cmd = malloc(sizeof(*cmd));
  memset(cmd, 0, sizeof(*cmd));
  cmd->type = EXEC;
  return (struct cmd*)cmd;
}
```
该函数的作用是申请空间存放 “**命令行参数**”。可以看到里面存在“**malloc()**”函数，至此，可以解决（3）的图中的第1个“**?**”了。  
  
（5）未修改的**xv6**内核对异常的反映是直接杀死出现异常的进程。在“**kernel/trap.c/usertrap()**”中：  
```
else {
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
```
因此，该进程的内存空间会被回收。除了“fork”父进程的4页内存（下面会解释为什么是4页）可以正常回收，在该子进程中通过“**sbrk**”申请的内存都无法正常释放，也就是在“**kernel/vm.c/uvmunmap()**”函数中会走到这一步：
```
if((*pte & PTE_V) == 0) 
    panic("uvmunmap: not mapped");
```
至此，（3）中的图中的第2个“**?**”解决了。  

（6）这里解释为什么父进程是4页内存？  
修改“**kernel/vm.c/uvmunmap()**”函数如下图：  
![](https://github.com/2351889401/6.S081-Lab-lazy_page_allocation/blob/main/images/xiugai.png)   
命令行输入“**echo hi**”，输出为：  
![](https://github.com/2351889401/6.S081-Lab-lazy_page_allocation/blob/main/images/xiugai_result.png)  
是父进程在回收子进程的内容，并且有4页已经成功回收了，说明父进程原来就是4页内存。  

（7）**pid**为1是“**init**”进程，**pid**为2是“**shell**”进程  

（8）最终可以将（3）中的图变成如下的流程：  
![](https://github.com/2351889401/6.S081-Lab-lazy_page_allocation/blob/main/images/think2.png)   


**2.** 修改“**kernel/trap.c/usertrap()**”和内核中的代码让**echo hi**正常输出  
  
这里主要给出的是“**trap.c/usertrap()**”中的修改，而且修改的并不完善；而“**kernel/vm.c**”中的修改主要在下一个实验内容介绍。因为这里仅仅让**echo hi**工作，但实际上想让内核正常运行还有很多的情况没有考虑到。  
主要的思路就是：读取引起异常的虚拟地址；分配内存；在页表中注册；返回应用程序继续执行。
```
if(r_scause() == 8){
    // system call

    ...
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    if(r_scause() == 13 || r_scause() == 15) {
        uint64 va_fault = r_stval(); //获取引起缺页异常的虚拟地址
        uint64 down = PGROUNDDOWN(va_fault); //取其基址
        char* mem;
        mem = (char*)kalloc();
        memset(mem, 0, PGSIZE); //分配物理内存并且初始化
        mappages(p->pagetable, down, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U); //在页表中注册 虚拟地址基址与物理地址对应上
        printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid); 
        printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
        vmprint(p->pagetable, 1); // 页表实验中的函数 可以查看当前页表的内容
        printf("p->sz: %d\n", p->sz/PGSIZE); //查看当前进程一共的内存大小 可能并没有分配那么多
    }  
    else {
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
}
```  
此时，在命令行输入“**echo hi**”，其结果如下图：  
![](https://github.com/2351889401/6.S081-Lab-lazy_page_allocation/blob/main/images/echo_hi_ok.png)    
  
可以从图中看到，发生了2次缺页异常（初始时进程是4页内存）。2次缺页的地址分别是“**0x4008和0x13f48**”。  
而“**0x4008**”的基址（后12位置零）是“**0x4000**”，是第4页（从第0页开始）的内存。4正好是图中第一个大方框中三级页表表示的值。  
而“**0x13f48**”的基址（后12位置零）是“**0x13000**”，转换为10进制是第19页的内存。19正好是图中第二个大方框中三级页表表示的值。  
还可以看到的一点是，进程的内存大小一共有20页左右，但实际上分配且使用到的最终只有6页。因此，**lazy allocation**很有作用。  
  

**3.** 修改内核代码通过一些压力测试。  
主要需要解决的情况为：  
（1）处理“**sbrk()**”系统调用参数为负的情况  
（2）如果发生异常的虚拟地址 高于 最大的地址空间大小，杀死该进程。  
（3）正确处理“**fork()**”过程中 父进程向子进程的内存拷贝。  
（4）在“**sbrk()**”后（我们知道这时并没有实际分配内存），如果此时在系统调用（比如“**sys_write**、**sys_read**、**sys_pipe**”）中使用到新申请的地址空间，需要在这些系统调用里面分配内存，并在页表中注册。   
（5）“**kalloc()**”如果已经没有内存可以分配，杀死进程。  
（6）处理虚拟地址位于“**guard page**”引起的异常。  

对代码的修改主要集中在下面4个文件中：  
* **kernel/sysproc.c/sys_sbrk()**
主要添加了“**sbrk()**”参数为负的处理，处理过程可以参考“**kernel/vm.c/uvmdealloc()**”的实现
```
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  uint64 oldsz;
  oldsz = myproc()->sz;

  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz += n;
  // if(growproc(n) < 0)
  //   return -1;
  if(n < 0) {
    uint64 newsz = myproc()->sz;
    //回收一些内存
    if(PGROUNDUP(newsz) < PGROUNDUP(oldsz)) {
      int npages = (PGROUNDUP(oldsz) - PGROUNDUP(newsz))/PGSIZE;
      uvmunmap(myproc()->pagetable, PGROUNDUP(newsz), npages, 1);
    }
  }
  return addr;
}
```

  
* **kernel/trap.c/usertrap()**
处理 *发生异常的虚拟地址高于最大的地址空间大小* + *没有内存可以分配则杀死进程* + *处理虚拟地址位于“guard page”引起的异常*  
需要注意的是，这里存在一个问题，就是：如何判断这是一个普通的缺页地址、还是处于“**guard page**”的非法地址？
在“**kernel/vm.c**”中存在函数“**walkaddr(pagetable, va)**”可以用来根据页表读取该虚拟地址对应的物理地址。
```
// Look up a virtual address, return the physical address,
// or 0 if not mapped.
// Can only be used to look up user pages.
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  if((*pte & PTE_U) == 0) //注释掉
    return 0; //注释掉
  pa = PTE2PA(*pte);
  return pa;
}
```
可以代码中标记“注释掉”的这2行注释掉。这样，当使用“**walkaddr()**”函数时，如果是正常的缺页地址则返回0；而“**guard page**”中的地址可以返回正常的地址了。  
为什么“**guard page**”有正常的物理地址呢？  
因为应用程序的执行流程“**fork -> exec**”，在“**exec**”中为应用程序进程分配了“**user stack + guard page**”，而仅仅使用了如下的函数将“**guard page**”在页表的页表项中的“**PTE_U**”置零了：  
```
// mark a PTE invalid for user access.
// used by exec for the user stack guard page.
void
uvmclear(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  
  pte = walk(pagetable, va, 0);
  if(pte == 0)
    panic("uvmclear");
  *pte &= ~PTE_U;
}
```

  
到此，可以在“**kernel/trap.c/usertrap()**”中进行修改啦：  
```
if(r_scause() == 8){
    // system call

    ...
  } else if((which_dev = devintr()) != 0){
    // ok
  } else {
    if(r_scause() == 13 || r_scause() == 15) {
      uint64 va_fault = r_stval();
      if(walkaddr(p->pagetable, va_fault) != 0) { //guard page中的地址
        p->killed = 1;
      }
      else { //正常的缺页地址
        if(va_fault >= p->sz) p->killed = 1; //不能超过地址空间
        else {
          uint64 down = PGROUNDDOWN(va_fault);
          char* mem;
          if((mem = (char*)kalloc()) == 0) p->killed = 1; //内存不够要杀死进程
          else {
            memset(mem, 0, PGSIZE);
            mappages(p->pagetable, down, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U);
            // printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid); //make grade的过程这些 printf 语句会导致 Timeout!
            // printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
          }
          // vmprint(p->pagetable, 1);
          // printf("p->sz: %d\n", p->sz/PGSIZE);
        }
      }
    }
    else {
      printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
      printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
      p->killed = 1;
    }
  }
```  
  
* **kernel/vm.c**
处理 *不再需要panic的情况* + *fork时父进程向子进程内存的拷贝*  
因为修改的点挺多得，这里直接给出修改的函数：  
“**walkaddr()**”
```
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  // if((*pte & PTE_U) == 0) //这里注释掉的原因是 对于user stack下的guard page的虚拟地址下的内容 我们会在trap.c中进行判断 使其walkaddr的地址不为0 不作为缺页处理
  //   return 0;
  pa = PTE2PA(*pte);
  return pa;
}
```  
“**uvmunmap()**”  
修改原因：虚拟地址没有映射，不应该再“panic”
```
// Remove npages of mappings starting from va. va must be
// page-aligned. The mappings must exist.
// Optionally free the physical memory.
void
uvmunmap(pagetable_t pagetable, uint64 va, uint64 npages, int do_free)
{
  uint64 a;
  pte_t *pte;

  if((va % PGSIZE) != 0)
    panic("uvmunmap: not aligned");
  
  // int pages = 0;

  for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      // panic("uvmunmap: walk");
      continue; //这里是为什么呢? 就是说在三级页表里面遍历的时候 会出现比如说顶级页表中的页表项为0的情况 因为没有映射该虚拟地址
    if((*pte & PTE_V) == 0) {
      // printf("pid: %d hui shou %d pages\n", myproc()->pid, pages);
      // panic("uvmunmap: not mapped");
      continue; //这里是为什么呢? 能够正常走完三级页表的遍历 但是最后一级页表项为0 也是没有映射该虚拟地址导致的
    }
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
    if(do_free){
      uint64 pa = PTE2PA(*pte);
      kfree((void*)pa);
    }
    *pte = 0;
    // pages++;
  }
}
```
“**uvmcopy()**”  
```
// Given a parent process's page table, copy
// its memory into a child's page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  char *mem;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      // panic("uvmcopy: pte should exist");
      continue; //这儿是为什么呢? 因为父进程的虚拟地址可能没有映射 所以会导致页表项为空 这里属于未映射的虚拟地址 直接continue即可
    if((*pte & PTE_V) == 0)
      // panic("uvmcopy: page not present");
      continue; //这里也是属于未映射的虚拟地址
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

* **kernel/sysfile.c**中的“**sys_read、sys_write、sys_pipe**”  
因为在这些系统调用中都可能使用新申请而尚未分配内存的地址空间
“**sys_read()**”
```
uint64
sys_read(void)
{
  struct file *f;
  int n;
  uint64 p;

  if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argaddr(1, &p) < 0)
    return -1;
  
  if(p < myproc()->sz && p + n - 1 < myproc()->sz) { //给出的地址应当小于可用的地址空间 并且 最后的地址也应当小于可用的地址空间
    if(walkaddr(myproc()->pagetable, p) == 0) { //在这里查看地址p是否缺页
      uint64 down = PGROUNDDOWN(p);
      while(n > 0) {
        char* mem = (char*)kalloc();
        memset(mem, 0, PGSIZE);
        mappages(myproc()->pagetable, down, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U);
        n -= (PGSIZE - (p - down)); // (PGSIZE - (p - down)) 为这一页内存可以存入多少字节数据
        down += PGSIZE;
        p += PGSIZE;
        p = (p / PGSIZE) * PGSIZE; // 从分配的第2页开始对齐（这里感觉应该是对的吧 有点不太确定）
      }
    }
  }
  return fileread(f, p, n);
}
```
“**sys_write()**” 和上面的函数一样
```
uint64
sys_write(void)
{
  struct file *f;
  int n;
  uint64 p;

  if(argfd(0, 0, &f) < 0 || argint(2, &n) < 0 || argaddr(1, &p) < 0)
    return -1;

  if(p < myproc()->sz && p + n - 1 < myproc()->sz) {
    if(walkaddr(myproc()->pagetable, p) == 0) {
      uint64 down = PGROUNDDOWN(p);
      while(n > 0) {
        char* mem = (char*)kalloc();
        memset(mem, 0, PGSIZE);
        mappages(myproc()->pagetable, down, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U);
        n -= (PGSIZE - (p - down));
        down += PGSIZE;
        p += PGSIZE;
        p = (p / PGSIZE) * PGSIZE;
      }
    }
  }
  return filewrite(f, p, n);
}
```  
“**sys_pipe()**” 为什么会修改这个函数，因为在“**usertests**”中有测试到这里的情况  
```
uint64
sys_pipe(void)
{
  uint64 fdarray; // user pointer to array of two integers
  struct file *rf, *wf;
  int fd0, fd1;
  struct proc *p = myproc();

  if(argaddr(0, &fdarray) < 0)
    return -1;
  
  if(pipealloc(&rf, &wf) < 0)
    return -1;
  fd0 = -1;
  if((fd0 = fdalloc(rf)) < 0 || (fd1 = fdalloc(wf)) < 0){
    if(fd0 >= 0)
      p->ofile[fd0] = 0;
    fileclose(rf);
    fileclose(wf);
    return -1;
  }

//主要的修改是下面这里 从fdarray地址开始分配2个int（也就是8字节）的地址空间 
  int n = 8;
  uint64 _p = fdarray; // va
  if(fdarray < p->sz && fdarray + n - 1 < p->sz) {
    if(walkaddr(myproc()->pagetable, _p) == 0) {
      uint64 down = PGROUNDDOWN(_p);
      while(n > 0) {
        char* mem = (char*)kalloc();
        memset(mem, 0, PGSIZE);
        mappages(myproc()->pagetable, down, PGSIZE, (uint64)mem, PTE_W|PTE_X|PTE_R|PTE_U);
        n -= (PGSIZE - (_p - down));
        down += PGSIZE;
        _p += PGSIZE;
        _p = (_p / PGSIZE) * PGSIZE;
      }
    }
  } 
  
  if(copyout(p->pagetable, fdarray, (char*)&fd0, sizeof(fd0)) < 0 ||
     copyout(p->pagetable, fdarray+sizeof(fd0), (char *)&fd1, sizeof(fd1)) < 0){
    p->ofile[fd0] = 0;
    p->ofile[fd1] = 0;
    fileclose(rf);
    fileclose(wf);
    return -1;
  }
  return 0;
}
```
  
到此，所有的代码修改完毕，结果如下图：  
![](https://github.com/2351889401/6.S081-Lab-lazy_page_allocation/blob/main/images/lazytests.png)  
![](https://github.com/2351889401/6.S081-Lab-lazy_page_allocation/blob/main/images/usertests.png) 

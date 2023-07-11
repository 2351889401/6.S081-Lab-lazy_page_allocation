## 虽然本次实验的难度仅为 moderate ，但是需要耐心的调试，通过全部的 usertests；此外，本次实验有很多值得思考的点。
## 总体来说，细节的完成全部内容并不轻松，需要仔细考虑虚拟内存、页表相关的内容。不过经过这次实验，确实对虚拟内存、页表的掌握更加深刻。  
  
### 实验主题：堆内存的延迟分配（xv6 lazy page allocation：https://pdos.csail.mit.edu/6.S081/2020/labs/lazy.html）
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

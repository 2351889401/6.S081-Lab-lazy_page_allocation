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
  
分析为什么修改了“**sys_sbrk()**”会导致命令行输入“**echo hi**”出现上图的错误  
（1）**scause**值为**15**，查阅《**RISC-V Instruction Set Manual**》，发现这个原因是“**store/AMO page fault**”  
（2）

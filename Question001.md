2023.12.19
---------------------------------------
> https://zhuanlan.zhihu.com/p/67894878 文中有如下描述：
如果使用mmap()，则在磁盘数据加载到page cache后，用户进程可以通过指针操作直接读写page cache，不再需要系统调用和内存拷贝。看起来mmap()好像完胜read()有没有？<br />
其实，mmap()在数据加载到page cache的过程中，会触发大量的page fault和建立页表映射的操作，开销并不小。另一方面，随着硬件性能的发展，内存拷贝消耗的时间已经大大降低了。所以啊，很多情况下，mmap()的性能反倒是比不过read()和write()的。<br />
Q ：具体哪些情况下会是mmap比read更快呢？ <br/>
A : 这里比较的应该是文件已经被其他的进程读入到内存，也就是page cache中已经存在了。此时如果是调用read方法，开销为系统调用由内核上下文进行切换，数据通过copy_to_user的拷贝时间；mmap消耗的时间为page fault的执行时间，因为创建页表最多可能有3级（pgd这一级肯定已经有了），因为内存访问延迟都是比较大的，而数据读写由于DDR吞吐量比较大，可能读写的时间反而比较短（当前DDR5每秒速度应该大于1GB的，小数据量跟DDR的核心频率有关系）。

---------------------------------------
> https://zhuanlan.zhihu.com/p/616941834 <br />
在／include/linux/sched.h中定义了如下一个联合结构：
 ```
union task_union {
  struct task_struct task;
  unsigned long stack[2408];
};
```
从这个结构可以看出，内核栈占8kb的内存区。实际上，进程的task_struct结构所占的内存是由内核动态分配的，更确切地说，内核根本不给task_struct分配内存，而仅仅给内核栈分配8K的内存，并把其中的一部分给task_struct使用。<br />
Q : 根据这篇文章的解释，gtaskkernelstack是union task_union的地址，那不是把struct task_struct task数据给覆盖掉了<br />
A : 重新理解了下，应该是内核堆栈和struct task_struct task共用8K的内存，struct task_struct task占用的低地址，内核栈顶为高地址，刚开始的时候两个是离的比较远的。也就是说内核栈虽然设置的有8K，但是实际能够使用的需要减掉struct task_struct task的大小，否则就把struct task_struct task的数据给覆盖掉了。

---------------------------------------
> Q ： mmap在内核的虚拟地址是线性地址还是vmalloc的地址？ <br />
kernel不会产生page fault， 估计也对的。 vmalloc 分配虽然记录到页表了，但是不会有lazy 分配
不会major page fault 有可能会minor page fault，两者的区别是前者会分配物理内存并映射到页表，后者不去分配物理内存仅仅去做个页表映射。一般说的page fault指前者，所以不会page fault，如果严谨的说把minor也算上，那就是会有page fault
vmalloc申请的内存，不一定有对应物理内存，以前打印过来vmalloc调试信息

---------------------------------------
> Q ： 使用mmap映射的文件共享内存，在进程A和进程B的用户空间、内核空间分别有3个虚拟地址，当进程A内存页表完成映射以后，在内核和进程B中第一次访问，是不是还是会出现TLB cache miss <br />

---------------------------------------
> 每个page frame都需要一个struct page来描述，一个page frame占4KB，一个struct page占32字节，那所有的struct page需要消耗的内存占了整个系统内存的32/4096，不到1%的样子，说小也小，但一个拥有4GB物理内存的系统，光这一项的开销最大就可达30多MB <br />
Q ： struct page结构体是存储在哪里的，MMU查找的页表跟这个应该是没有关系，但是一个页表应该也有一个struct page的，但是感觉MMU查表的时候应该用不上？

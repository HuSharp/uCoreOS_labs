##基础知识

Q1: 除了存储进程的trapframe，内核堆栈还有什么用呢？ 
我看进程拷贝的时候只拷贝了trapframe，并没有拷贝整个内核堆栈

A1:
堆栈是两个东西，heap 和 stack. 
Stack是用来维护execution flow. stack存储了一些临时数据，包括 local data and return address. 
Kernel stack 就是在kernel mode 下的 stack, 也一样包括了 local data and return 
address(包括切换回user mode的). Kernel stack也是用来维护execution flow的，kernel mode 下的execution flow 包括 thread 和 interrupt handler. 
至于是不是每一个thread都需要一个stack, interrupt handler 使用谁的stack, 这个在实现时候都可以按需要考虑。 
肤浅的理解，欢迎纠正 :)

哦，我对你的提问的理解是：程序的运行总是需要 stack 的，比如函数 call、局部变量等等，在内核里面也一样，从用户程序进入内核以后，执行内核代码所使用的 stack 就是你在进程创建的时候分配的 stack。其次，在由用户程序进入内核的时候，这个 stack 还有一个作用就是保存当前所有的 寄存器 的值，参见 trapasm.S。当然，从内核回到用户程序，也需要类似的操作。所以，这个 stack 很有用。
至于你说为什么不拷贝内核堆栈，是指 fork 的时候吧，不拷贝，因为确实是没有用。这个 stack 对于用户程序而言，就是一个临时的东西。 

可以从下一个进程是否会重用kernel stack来理解fork时拷贝。只要下一个进程不会占用的，就可以不复制。这样，就只有寄存器是必须复制的了，其他的都可以不复制。


Q2: 看到考试上的一些题目问到某个页式管理机制最多一共有多少个页表。 
那如果考虑自映射机制的话，一级页表的其中一个表项指向的是一级页表本身，是否应该在统计的时候减1？

A2:
你的理解是正确的。自映射的页表占了两个位置。 


Q3: 线性地址、物理地址、虚拟地址具体的联系和区别是什么？

A3:
这个，看IA32手册吧。
简单来说
Virtual Address ------ Segment ------>
Linear Address ------ Paging --------->
Physical Address
最后Physical Address 即是访问物理内存的地址。


##Ucore实验相关

Q4:
lab2的ucore的小os的load addr和link addr分别是多少？ 
我从kernel.ld得到load addr是0xC0100000，但不知道link addr从哪里得到，需要虚拟地址还是物理地址？希望老师给出答案。 

A4:
ucore 设定了ucore运行中的虚地址空间，具体设置可看 memlayout.h "Virtual memory map: "图 
所以 tools/kernel.ld描述的是 link_addr , link addr是0xC0100000 是一个虚地址，在ucore建立内核用页表时，设定的虚实映射关系是： 
phy addr + 0xC0000000 = virtual addr 
但bootloader把OS load到内存，那时还没有启动页表映射，用的是bootloader阶段设置的段模式，其映射关系(bootasm.S最后有段描述符表)是： 
linear addr = phy addr = virtual addr 
所以查看 bootloader的实现代码 bootmain::bootmain.c 
readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset); 
p_va=0xC0XXXXXX 
但 ph->p_va & 0xFFFFFF= 0x0XXXXXX 
了所以bootloader load OS的load addr是0x0XXXXXX, 是物理地址，这个是通过分析bootmain函数的实现得到的。 
简言之： OS的link addr 在tools/kernel.ld中设置好了，是一个虚地址 
OS的load addr在bootmain函数中指定了，是一个物理地址 

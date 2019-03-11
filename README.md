# mykernel  完成一个简单的时间片轮转多道程序内核代码



######  235 + 原创作品转载请注明出处 + 中科大孟宁老师的linux操作系统分析： <https://github.com/mengning/linuxkernel/>  

### 实验要求： 

完成一个简单的时间片轮转多道程序内核代码； 

仔细分析进程的启动和进程的切换机制；

阐明自己对“操作系统是如何工作的”理解。

### 实验内容：

#### 一、实验过程及截图

实验环境：VMware 虚拟机+Ubuntu 16.04.10（Linux4.15）

下载并解压kernel内核linux-3.9.4及kernel补丁包mykernel_for_linux3.9.4sc.patch：



![1552233199544](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1552233199544.png)



```
sudo apt-get install qemu # install QEMU
sudo ln -s /usr/bin/qemu-system-i386 /usr/bin/qemu

wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.9.4.tar.xz # download Linux Kernel 3.9.4 source code

wget https://raw.github.com/mengning/mykernel/master/mykernel_for_linux3.9.4sc.patch # download mykernel_for_linux3.9.4sc.patch

xz -d linux-3.9.4.tar.xz
tar -xvf linux-3.9.4.tar
cd linux-3.9.4
patch -p1 > ../mykernel_for_linux3.9.4sc.patch  //打补丁
make allnoconfig
make
qemu -kernel arch/x86/boot/bzImage
```

![1552305911862](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1552305911862.png)



![1552305944937](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1552305944937.png)

​	安装QEMU，编译运行kernel：

​	需要注意的一点是make的时候可能会产生错误，在/include/linux下找不到compiler-gcc5.h导致make失败。这是因为mykernel是基于linux原来的3.9.4内核写的，当时gcc的版本还没有到5，在对应的文件夹下只有compiler-gcc4.h compiler-gcc3.h compiler-gcc.h。因此需要自己下载compiler-gcc5.h并将它放在make目录的文件夹中的/include/linux中，注意不要错放在系统/usr/include下。

​	部署compiler-gcc5.h再运行make即可成功make，接下来使用qemu -kernel arch/x86/boot/bzImage可查看文件的输出结果，如下图。



![1552305964168](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1552305964168.png)



时间转轮片的实现：

拷贝源码替换到linux-3.9.4/mykernel/内编译运行

![Ubuntu 64 位-2019-03-11-20-23-11](C:\Users\Administrator\Desktop\Ubuntu 64 位-2019-03-11-20-23-11.png)

![Ubuntu 64 位-2019-03-11-20-20-12](C:\Users\Administrator\Desktop\Ubuntu 64 位-2019-03-11-20-20-12.png)

​	总共有4个状态，process0-3，系统开始在process 0中运行，在出现my_timer_handler后跳转到process 1，之后在process 1中运行，在出现my_timer_handler后跳转到process 2，之后在process 2中运行，在出现my_timer_handler后跳转到process 3，之后在process 3中运行，在出现my_timer_handler后跳转到process 0，依次循环。

### 二、源码分析

mypcb.h -- 定义了一个PCB结构体，也就是所谓的进程管理块，用来记录进程的有关信息。

```
#define MAX_TASK_NUM        4    
/*定义了最大任务数量*/
#define KERNEL_STACK_SIZE   1024*8  
//定义了内核堆栈的大小

/* CPU-specific state of this task */
struct Thread {
    unsigned long       ip;   //eip
    unsigned long       sp;    //esp
};     //定义了一个结构体用来保存当前ip和sp

typedef struct PCB{
    int pid;    //进程号
    volatile long state;    /* -1 unrunnable, 0 runnable, >0 stopped */
    char stack[KERNEL_STACK_SIZE]; 
    //内核堆栈大小
    /* CPU-specific state of this task */
    struct Thread thread; 
    //一个Thread类型的结构体变量thread
    unsigned long   task_entry; //程序入口
    struct PCB *next; //用来将PCB链接起来的链表
}tPCB;

void my_schedule(void); //声明进程调度函数
```

mymain.c -- 完成了内核的初始化工作，并且创建了4个进程，进程从0号开始执行，并且根据标志位判断进程是否需要调度，当标志位为1时表示进程需要调度，此时会执行my_schedule()方法来完成相应的调度。

```
#include "mypcb.h"

tPCB task[MAX_TASK_NUM];  
//定义了一个类型为PCB的task数组
tPCB * my_current_task = NULL;
//定义了一个PCB类型的指针变量并且初值为空
volatile int my_need_sched = 0;  //进程是否需要调度的标志位

void my_process(void); 
//声明函数
void __init my_start_kernel(void) //对内核进行初始化
{
    int pid = 0;  //定义0号进程
    int i;
    /* Initialize process 0*/  
    task[pid].pid = pid;
    task[pid].state = 0;/* -1 unrunnable, 0 runnable, >0 stopped */
    task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;  //定义了0号进程的入口为my_process方法的首地址
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
    task[pid].next = &task[pid];
    /*fork more process */
    for(i=1;i<MAX_TASK_NUM;i++)
    {
        memcpy(&task[i],&task[0],sizeof(tPCB));
        task[i].pid = i;
        task[i].state = -1;
        task[i].thread.sp = (unsigned long)&task[i].stack[KERNEL_STACK_SIZE-1];
        task[i].next = task[i-1].next;
        task[i-1].next = &task[i];
    }
    /* start process 0 by task[0] */ 
    pid = 0;
    my_current_task = &task[pid];
    asm volatile(
        "movl %1,%%esp\n\t"     /* set task[pid].thread.sp to esp */
        "pushl %1\n\t"          /* push ebp */
        "pushl %0\n\t"          /* push task[pid].thread.ip */
        "ret\n\t"               /* pop task[pid].thread.ip to eip */
        "popl %%ebp\n\t"
        : 
        : "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)   /* input c or d mean %ecx/%edx*/
    ); //将0号进程交给cpu进行处理
}   
void my_process(void)
{
    int i = 0;
    while(1)
    {
        i++;
        if(i%10000000 == 0)
        {
            printk(KERN_NOTICE "this is process %d -\n",my_current_task->pid);
            if(my_need_sched == 1)  //
            {
                my_need_sched = 0; //将标志位置0
                my_schedule();  //执行my_schedule()方法
            }
            printk(KERN_NOTICE "this is process %d +\n",my_current_task->pid);
        }     
    }
}
```

my_interrupt.c -- 产生时钟中断，用一个时间计数器周期性的检查循环条件，当条件满足时便会产生中断，并将进程调度标志位置1，当标志位为1时，进程便会执行my_schedule()方法，首先将当前进程的信息通过嵌入式汇编语句保存到堆栈当中，然后将下一个进程的地址赋给当前运行程序的指针，完成调度。

```
#include "mypcb.h"

extern tPCB task[MAX_TASK_NUM];
extern tPCB * my_current_task;
extern volatile int my_need_sched;
volatile int time_count = 0;  //时间计数器

/*
 * Called by timer interrupt.
 * it runs in the name of current running process,
 * so it use kernel stack of current running process
 */
void my_timer_handler(void) //时间中断程序
{
#if 1
    if(time_count%1000 == 0 && my_need_sched != 1)
    {
        printk(KERN_NOTICE ">>>my_timer_handler here<<<\n"); //如果计数器为1000并且标志位不等于1那么执行此条语句
        my_need_sched = 1; 标志位置1
    } 
    time_count ++ ;  
#endif
    return;     
}

void my_schedule(void)
{
    tPCB * next; //表示下一个运行的进程
    tPCB * prev; //表示当前正在运行的进程

    if(my_current_task == NULL 
        || my_current_task->next == NULL)
    {
        return;
    } //如果当前任务为空或者没有其他进程需要执行返回
    printk(KERN_NOTICE ">>>my_schedule<<<\n");
    /* schedule */
    next = my_current_task->next;
    prev = my_current_task;
    if(next->state == 0)/* -1 unrunnable, 0 runnable, >0 stopped */
    {
        /* switch to next process */
        asm volatile(   
            "pushl %%ebp\n\t"       /* save ebp */
            "movl %%esp,%0\n\t"     /* save esp */
            "movl %2,%%esp\n\t"     /* restore  esp */
            "movl $1f,%1\n\t"       /* save eip */ 
            "pushl %3\n\t" 
            "ret\n\t"               /* restore  eip */
            "1:\t"                  /* next process start here */
            "popl %%ebp\n\t"
            : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
            : "m" (next->thread.sp),"m" (next->thread.ip)
        );   //嵌入式汇编保存了当前进程的PCB信息用来为进程调度做准备
        my_current_task = next;  //调度下一进程
        printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);      
    }
    else
    {
        next->state = 0;
        my_current_task = next;
        printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);
        /* switch to new process */
        asm volatile(   
            "pushl %%ebp\n\t"       /* save ebp */
            "movl %%esp,%0\n\t"     /* save esp */
            "movl %2,%%esp\n\t"     /* restore  esp */
            "movl %2,%%ebp\n\t"     /* restore  ebp */
            "movl $1f,%1\n\t"       /* save eip */ 
            "pushl %3\n\t" 
            "ret\n\t"               /* restore  eip */
            : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
            : "m" (next->thread.sp),"m" (next->thread.ip)
        );          
    }   
    return; 
}
```



### 三、实验总结

​	操作系统的核心功能就是：进程调度和中断机制，计算机多道程序的处理的关键机制是中断机制，cpu为每一个进程创建了一个pcb，此pcb中保存了所有关于进程的相关信息，cpu有一个固定大小的时间片，当时间片结束的时候cpu会产生时钟中断，并且将进程有关信息通过嵌入式汇编将进程交给cpu处理，保存上下文环境到堆栈当中，当程序执行完毕后，从堆栈当中恢复上下文，当进程执行完毕后或者没有进程将要执行那么程序结束。


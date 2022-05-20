<center><h1>Operating System Lab04</h1></center>

<center><h4>10192100571 俞辰杰</h4></center>

------

#### 练习一、请简单说明proc_struct中各个成员变量含义以及作用。它与理论课中讲述的PCB结构有何不同之处？

​	**proc_struct 数据结构内容：**

| 变量名                                         | 作用                                                    |
| ---------------------------------------------- | ------------------------------------------------------- |
| int                                  **pid**   | 进程标识符                                              |
| char[15+1]                   **name**          | 进程名字                                                |
| enum proc_state        **state**               | 进程状态                                                |
| int                                  **runs**  | 进程被调度次数                                          |
| volatile bool                 **need_resched** | 非抢占式调度中，true代表运行结束，false代表运行结束     |
| uint32_t                        **flags**      | 进程flags                                               |
| uintptr_t                       **kstack**     | 进程内核堆栈                                            |
| uintptr_t                       **cr3**        | 进程页表PDT起始地址（uCore中支持内核线程，cr3地址一样） |
| struct mm _struct*     **mm**                  | 管理进程虚拟内存空间                                    |
| struct context              **context**        | 上下文（EIP，通用寄存器......）                         |
| struct trapframe*       **tf**                 | 中断帧，模拟中断前的入栈操作                            |
| struct proc_struct *    **parent**             | 父进程                                                  |
| list_entry_t                   **list_link**   | 进程链表【就绪队列和阻塞队列被连接同一个链表】          |
| list_entry_t                   **hash_link**   | 相同ID的进程的hash链表                                  |

​	**不同：**

1. proc_struct 的进程状态不是就绪态，运行态，阻塞态三种状态；而是用 runnable 来替代就绪态和运行态 

2. proc_struct 的进程将所有的进程连接成一个链表list_link，以及次级链表hash_link

3. proc_struct 使用 trapframe来模拟中断的入栈过程而非使用硬件实现

4. proc_struct 缺少对于用户信息的存储

   

#### 练习二、试简单分析 uCore中 内核线程的创建过程。

​	线程从**proc_init()** 作为入口创建，在此函数中包含**kernel_thread()**来设置中断帧的相关数据，设置程序入口EIP和堆栈ESP。

​	**kernel_thread()** 指定了对应的执行流，使用 **do_fork()** 函数来创建线程，其中**do_fork()**的内部调用如下：

- **alloc_proc**：为某个线程分配线程控制块，进行初始化TCB

  - proc_init可以直接调用alloc_proc，此时是为用于已经启动的零号线程分配线程控制块

- **setup_kstack**：为线程创建内核堆栈区域

- **copy_mm**： 与父线程共享虚拟内存空间（由于uCore是内核线程，使用都是kernel的内存空间）

- **copy_thread**：在kernel_thread()创建的中断帧放入堆栈中

- **get_pid**：为线程分配唯一的PID

- **wakeup_proc**：唤醒线程，将其状态设置为runnable

  在kernel_thread()执行完成后返回对应的线程PID，根据PID使用**find_proc()**找到对应线程控制块。然后使用**set_proc_name()**为这个线程设置一个名字

  同时，在进行线程切换的时候，整个线程队列应该是不能被其他进程所使用的，所以需要**互斥访问**，使用**local_intr_save**和**local_intr_restore**实现PV操作



**重要源程序分析：**

~~~C
void proc_init(void) {
    int i;
	//初始化list_link表头
    list_init(&proc_list);
    
    // 初始化hash_link链表，并把得到的链表也放入list_link中
    for (i = 0; i < HASH_LIST_SIZE; i ++) {
        // hash_link每个子列表的表头初始化
        list_init(hash_list + i);
    }
	
    // 为第一个线程分配一个TCB，idleproc指向0号TCB
    // alloc_proc所进行的初始化分配都会被覆盖，只有分配的内存不会更改
    if ((idleproc = alloc_proc()) == NULL) {
        panic("cannot alloc idleproc.\n");
    }
	
    // 零号线程的PID为0
    idleproc->pid = 0;
    // 零号线程已经处于运行状态
    idleproc->state = PROC_RUNNABLE;
    idleproc->kstack = (uintptr_t)bootstack;
    idleproc->need_resched = 1;
    set_proc_name(idleproc, "idle");
    
    // 当前运行的线程数加1
    nr_process ++;
	// 当前运行的线程为零号线程
    current = idleproc;
	
    
    // ===================以下为一般线程的创建过程===================//
    // kernel_thread传入执行流（函数），返回得到的线程PID
    int pid = kernel_thread(init_main, "Hello world!!", 0);
    if (pid <= 0) {
        panic("create init_main failed.\n");
    }
	// 根据pid找到创建的线程；find_proc是通过pid在hash队列中进行的查找
    initproc = find_proc(pid);
    // 给线程的名字取名为init
    set_proc_name(initproc, "init");

    assert(idleproc != NULL && idleproc->pid == 0);
    assert(initproc != NULL && initproc->pid == 1);
}
~~~

~~~C
int kernel_thread(int (*fn)(void *), void *arg, uint32_t clone_flags) {
    // 创建一个中断帧的变量，并且分配内存 
    struct trapframe tf;
    memset(&tf, 0, sizeof(struct trapframe));
    
    // 代码段的选择值和kernel内核代码段一致
    tf.tf_cs = KERNEL_CS;
    // 数据段，附加数据段，堆栈段的选择值和kernel内核数据段一致
    // 可以看出所有线程的指针都是和内核一致，符合内核级线程的要求
    tf.tf_ds = tf.tf_es = tf.tf_ss = KERNEL_DS;
    
    // 将通用寄存器的ebx和edx分别设置为要执行的函数和传入的参数，相当于设置了程序入口
    tf.tf_regs.reg_ebx = (uint32_t)fn;
    tf.tf_regs.reg_edx = (uint32_t)arg;
    // 先执行kernel_thread_entry，将调用的函数及其参数直接运行
    tf.tf_eip = (uint32_t)kernel_thread_entry;
    // 以上对于tf的设置，在iret之后会被保存在线程的中断帧中以供使用
    
    // 之后对于线程的设置，都需要在do_fork中进行，相当于kernel_thread只进行了tf的设置
    return do_fork(clone_flags | CLONE_VM, 0, &tf);
}
~~~

 <div STYLE="page-break-after: always;"></div>

~~~C
int do_fork(uint32_t clone_flags, uintptr_t stack, struct trapframe *tf) {
    struct proc_struct *proc;
	// 设置该线程的父线程是当前正在运行的线程
    proc->parent = current;
	// 分配两个共8K的物理块，kstack是物理块的起始地址
    setup_kstack(proc);
    // 由于uCore使用内核线程，所以不需要分配独立虚拟地址空间
    copy_mm(clone_flags, proc);
    
    // *********************************** // 
    // 将新建的线程的trapframe设置到堆栈区中
    copy_thread(proc, stack, tf);
    // *********************************** // 
    
    bool intr_flag;
    // 互斥的进行操作，类似于P
    local_intr_save(intr_flag);
    {
        // 为新的线程分配一个独立的PID
        proc->pid = get_pid();
        // 根据PID将线程放入对应的hash列表
        hash_proc(proc);
        // 还需要将线程放入全体线程的列表中
        list_add(&proc_list, &(proc->list_link));
        nr_process ++;
    }
    // 互斥的进行操作，类似于V
    local_intr_restore(intr_flag);
    // 唤醒线程，将其设置为runnable
    wakeup_proc(proc);
    return proc->pid;
}
~~~

~~~C
static void
copy_thread(struct proc_struct *proc, uintptr_t esp, struct trapframe *tf) {
    // 新线程所对应的tf进行初始化，大小同kstack，并且指向传入的tf
    proc->tf = (struct trapframe *)(proc->kstack + KSTACKSIZE) - 1;
    *(proc->tf) = *tf;
    
    proc->tf->tf_regs.reg_eax = 0;
    proc->tf->tf_esp = esp;
    proc->tf->tf_eflags |= FL_IF;
	
    // 设置线程的上下文的EIP和ESP两个入口地址
    // EIP指向所需要运行的程序，得到的值在do_fork()的最后返回
    proc->context.eip = (uintptr_t)forkret;
    // ESP指向所需要tf的堆栈，将其中的数据不断pop出来用于设置本线程
    proc->context.esp = (uintptr_t)(proc->tf);
}
~~~

 <div STYLE="page-break-after: always;"></div>

#### 练习三、试简单分析 uCore中 内核线程的切换过程。

​	在cpu_idle() 创建完一系列线程之后，需要使用**schedule()**函数将线程调度进行使用。其中schedule()是使用proc_struct中的list_link来进行查询调度的。

​	**schedule()** 首先使用 **link_next()** 函数得到线程队列中合适的，处于runnable状态的线程。

​	找到合适的线程后，使用proc_run() 将新的进程和原先的线程切换，在切换的过程中调度的函数如下

- **load_esp0**：设置TSS(Task State Segment)，放入当前线程的重要数据【当前线程内核堆栈指针】

- **lcr3**：将新的线程的地址设置到cr3的位置

- **switch_to**：进行线程的上下文切换

  同时，在进行线程切换的时候，整个线程队列应该是不能被其他进程所使用的，所以需要**互斥访问**，使用**local_intr_save**和**local_intr_restore**实现PV操作



​	**重要源程序分析：**

~~~C
void schedule(void) {
    bool intr_flag;
    list_entry_t *le, *last;
    struct proc_struct *next = NULL;
    
	// 互斥操作
    local_intr_save(intr_flag);
    {
        // 结束运行后，将本线程的需要调度位设为0
        current->need_resched = 0;
        // 从链表开始找下一个线程，如果是零号线程则从头开始找【零号线程不在链表中】
        last = (current == idleproc) ? &proc_list : &(current->list_link);
        le = last;
        do {
            if ((le = list_next(le)) != &proc_list) {
                next = le2proc(le, list_link);
                // 找到一个处于就绪态的线程
                if (next->state == PROC_RUNNABLE) {
                    break;
                }
            }
        } while (le != last);
        // 如果没有下一个线程，或者没有一个就绪的线程，则重新运行零号线程
        if (next == NULL || next->state != PROC_RUNNABLE) {
            next = idleproc;
        }
        // 下一个线程运行次数加一
        next->runs ++;
        // 保证下一个线程不是当前的线程才运行
        if (next != current) {
            proc_run(next);
        }
    }
    local_intr_restore(intr_flag);
}

~~~

 <div STYLE="page-break-after: always;"></div>

~~~C
void proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        
        // 互斥操作
        local_intr_save(intr_flag);
        {
            current = proc;
            // 将tf设为下一个线程的任务状态段的内核堆栈的位置
            load_esp0(next->kstack + KSTACKSIZE);
            // 将选择到的线程的TCB的起始地址保存在tf的cr3中
            lcr3(next->cr3);
            
            // *********************************** // 
            // 保存prev线程的现场，切换到next线程的内容
            // 从高位到低位，将prev线程的堆栈保存，将next线程的堆栈加载
            switch_to(&(prev->context), &(next->context));
            // *********************************** // 
        }
        local_intr_restore(intr_flag);
    }
}
~~~


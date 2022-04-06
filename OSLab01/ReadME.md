# Operating System Lab01

### 10192100571 俞辰杰

#### 阅读、理解Lab1基准代码，特别要重点关注以下源程序代码：bootasm.S、bootmain.c、init.c、trap.c、trapentry.S、vectors.S等。



#### 练习1∶理解通过make生成执行文件的过程。(要求在报告中写出对下逑问题的回答)

列出本实验各练习中对应的OS原理的知识点﹐并说明本实验中的实现部分如何对应和体现了原理中的基本概念和关键知识点。在此练习中，大家需要通过静态分析代码来了解︰

1. **操作系统镜像文件 ucore.img 是如何一步一步生成的?**

   (需要比较详细地解释 Makefile 中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

   ```makefile
   # create ucore.img
   UCOREIMG	:= $(call totarget,ucore.img)
   #将UCORIMG定义为一个函数，返回变量“togarget”为，输入参数为ucore.img
   
   $(UCOREIMG): $(kernel) $(bootblock)
   #生成UCOREIMG所需要的条件是生成kernel和生成bootblock
   	$(V)dd if=/dev/zero of=$@ count=10000
   	#从一个空的有10000个块的文件读入，向UCOREIMG文件输出
   	$(V)dd if=$(bootblock) of=$@ conv=notrunc
   	#从bootblock文件中读入，向UCOREIMG文件输出到第一个块
   	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
   	#从kernel文件中跳过第一个块，向UCOREIMG文件中输出
   
   $(call create_target,ucore.img)
   #定义一个函数，返回变量“create_target”，输入参数为ucore.img
   ```

   1. **kernel生成代码**

      ```makefile
      # create kernel target
      kernel = $(call totarget,kernel)
      #kernel通过定义函数来进行创造
      
      $(kernel): tools/kernel.ld
      #生成kernel文件需要kernel的连接文件kernel.ld
      $(kernel): $(KOBJS)
      #生成kernel还需要KOBJS文件
      	@echo + ld $@
      	#打印kernel目标文件的名字
      	
      	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
      	# V定义为为@，LD定义为GCC前缀为ld的变量名
      	# 将上一步生成得到的连接文件连接起来，输出KOBJS的.o文件
      	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
      	#使用OBJDUMP对kernel文件进行反汇编，调用asmfile函数输出kernel的汇编代码
      	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
      	#使用OBJDUMP解析kernel得到符号表，返回kernel.sym
      
      $(call create_target,kernel)
      ```

   2. **bootblock生成代码**

      ```makefile
      # create bootblock
      bootfiles = $(call listf_cc,boot)
      $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
      bootblock = $(call totarget,bootblock)
      
      $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
      #生成bootblock需要生成bootasm.o,bootmain.o和sign
      	@echo + ld $@
      	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
      	# V定义为为@，LD定义为GCC前缀为ld的变量名
      	# 将上一步生成得到的连接文件连接起来，输出bootblock的.o文件
      	# 并且设置bootblock的入口地址为）0x7C00
      	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
      	#使用OBJDUMP对kernel文件进行反汇编，调用asmfile函数输出bootblock的汇编代码
      	@$(OBJDUMP) -t $(call objfile,bootblock) | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,bootblock)
      	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
      	#使用objcopy将bootblock.o转换成bootblock.out，并且去除重定位和符号信息
      	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
      	#使用sign工具将bootblock转换生成bootblock目标文件
      
      $(call create_target,bootblock)
      ```

   3. **ucore.img生成过程**

      - 编译libs和kern目录下的.c和.S文件，生成.o文件，并链接得到bin/kernel文件
      - 编译boot目录下的.c和.S文件，生成.o文件，并链接得到bin/bootblock.out文件
      - 编译tools/sign文件，得到bin/sign文件
      - 使用sign将bin/bootblock.out文件转化为512字节的bin/bootblock文件，并将其最后两字节设置为0x55AA
      - 为bin/ucore.img分配5000MB的内存空间，并将bootblock放在img文件的第一个block，接着将kernel复制到ucore.img第二个block开始的位置


2. **一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?**

   在上一题中可知，扇区引导盘bootblock是由sign所产生的，在sign.c之中可以看到主引导扇区的要求：

   ```C
   char buf[512];
   memset(buf, 0, sizeof(buf));
   FILE *ifp = fopen(argv[1], "rb");
   int size = fread(buf, 1, st.st_size, ifp);
   if (size != st.st_size) {
       fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
       return -1;
   }
   fclose(ifp);
   buf[510] = 0x55;
   buf[511] = 0xAA;
   ```

   在 sign.c 中，分配了一个具有512个字节长度的内存空间，并且在内存空间的最后两个字节分别为 0x55 和 0xAA 作为结束。




####  练习2∶使用qemu执行并调试lab1中的软件。(要求在报告中简要写出练习过程)

为了熟悉使用qemu和gdb进行的调试工作﹐我们进行如下的小练习∶

1. 从CPU加电后执行的第一条指令开始﹐单步跟踪BIOS的执行。

   ~~~makefile
   set architecture i8086
   target remote :1234
   break start	#在开始的时候就设置断点，以便调试代码
   continue
   ~~~

   在[~/moocos/ucore_lab/labcodes/lab1]下输入make debug，得到如下界面，停留在0x0000fff0的位置

   ![image-20220309204852590](C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220309204852590.png)


2. 在初始化位置0x7C00设置实地址断点,测试断点正常。

   ```makefile
   set architecture i8086
   target remote :1234
   break *0x7C00	#在0x7C00处设置断点
   continue
   ```

   <img src="C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220309205406849.png" alt="image-20220309205406849" style="zoom: 40%;" />

3. 从0x7C00开始跟踪代码运行,将单步跟踪反汇编得到的代码与 bootasm.S 和 bootblock.asm 进行比较。

   <img src="C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220309213531391.png" alt="image-20220309213531391" style="zoom:40%;" />

   <center><h5>单步反汇编得到的结果</h5></center>

   <img src="C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220309214018359.png" alt="image-20220309214018359" style="zoom: 40%;" />

   <center><h5>bootasm.S的代码</h5></center>

   <img src="C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220309213842407.png" alt="image-20220309213842407" style="zoom: 40%;" />

   <center><h5>bootblock.asm的代码</h5></center>



4. 自己找一个bootloader或内核中的代码位置﹐设置断点并进行测试。

   ![image-20220309220501224](C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220309220501224.png)

   ![image-20220309220923660](C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220309220923660.png)

   

#### 练习3∶分析bootloader进入保护模式的过程。(要求在报告中写出分析)

BIOS将通过读取硬盘主引导扇区到内存﹐并转跳到对应内存中的位置执行bootloader ·请分析bootloader是如何完成从实模式进入保护模式的。

提示︰需要阅读小节“保护模式和分段机制"和lab1/bootlbootasm.S源码﹐了解如何从实模式切换到保护模式﹐需要了解︰

- **为何开启A20·以及如何开启A20**

  1. 在老旧的计算机系统中，内存空间为1MB，也即$2^{20}$大小的内存空间，A20超过了1MB会直接设为0。但是在现在计算机内存空间都大于1MB的情况下仍旧使用A20作为寻址空间会导致超过$2^{20}$大小的内存仍旧会被当作0，从而使得只有奇数兆的内存空间可以被读取使用。所以需要开启A20来避免这种情况发生。

     

     2. 通过8042键盘控制器来开启A20，其中0x64是命令寄存器，0x60是数据寄存器。

  ​	bootasm.S源码如下：

  ![](C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220310110905302.png)

  Line 34：等待8042输入缓冲为空

  Line 43：发送“向P2端口写数据命令”到8042的输入端口

  Line 49：等待8042输入端口为空

  Line 58：将立即数10111111传入数据寄存器，将输入端口的第二位置1，写入输入缓冲区的内容

  

- **如何初始化GDT表**

  bootasm.S中关于gdt表的内容：

  ![image-20220310111031060](C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220310111031060.png)

  gdt一共有 0x17 + 1 = 0x18， 即24个字节，每8个字节为一段

  1. 第一段为空段
  2. 第二段为代码段，长度为32位，地址空间大小是4GB，可读可执行
  3. 第三段为数据段，长度为32位，地址空间大小是4GB，可写

  

- **如何使能和进入保护模式**

  bootasm.S中进入保护模式的代码：

  ![](C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220310110927456.png)

  1. 将cr0寄存器的内容复制到eax寄存器
  2. 将eax寄存器的内容和0x000...001做或运算，得到的结果第一位置为1，其余为原来的值
  3. 将eax寄存器的内容复制到cr0寄存器

  此时cr0寄存器的第一位保护模式位被设为了1，其余位数没变。

  

#### 练习4∶bootloader加载ELF格式的OS的过程。(要求在报告中写出分析)

- **bootloader如何读取硬盘扇区的?**

  bootmain.C的读取扇区代码如下：

  ![image-20220310113326916](C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220310113326916.png)

  

  **bootloader读取硬盘扇区的过程大体如下：**

  1. **等待磁盘准备好：（Line 46）**

     ~~~C
     static void
     waitdisk(void) {
         while ((inb(0x1F7) & 0xC0) != 0x40)
             /* do nothing */;
         // 0xC0 = 1100 0000
         // 0x40 = 0100 0000
     }
     ~~~

     不断查询0x1F7寄存器的最高两位，直到最高和次高位为01的时候结束循环。

  2. **发出读取扇区的命令：（Line 48 ~ Line 53）**

     - 0x1F2 为读取扇区数，置为1
     - 0x1F3 ~ 0x1F6 为读取扇区的起始编号
     - 0x1F7 为命令字0x20

  3. **等待磁盘准备好：（Line 56）**

  4. **把磁盘扇区数据读到指定内存：（Line 59）**

     从0x1F0开始以4字节读取数据

  

- **bootloader是如何加载ELF格式的OS?**

  bootmain加载OS的代码如下：

  ![image-20220310114856491](C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220310114856491.png)

  - 首先读取ELF的头部，也即扇区的第一分页
  - 通过ELF头部的e_magic字段来判断是否是一个ELF的OS文件
  - 读取ELF头部的e_phoff字段，得到Program Header的起始地址
  - 读取ELF头部的e_phnum字段，得到Program Header的元素数目
  - 遍历每个PH表中的元素，得到每个分段在文件中的偏移，要加载的虚拟地址，以及分段长度



#### 练习5∶uCore OS是如何实现中断机制的？

​	uCore OS的在正常发生中断的时候会产生一个中断信号IRQ，起始数是0x80

​	在此基础之上，每增加1，则代表一种预设的中断类型，例如时钟中断为32，键盘中断为33....通过查询中断向量表**IDT**来查询发生的中断类型。

​	通过中断描述符来表示一个中断程序，包括offset，段选择值，中断类型，特权值等。而中断描述符在向量表的初始化阶段进行创建。在一个中断发生的时候，首先通过IDT取得中断向量号，然后在GDT段表中查询该向量中断号所对应的分段的相关信息，从基址加上IDT的offset从而找到中断所对应的内容。

​	在中断的时候，会在栈中存放入EFLAGS（程序状态字寄存器），CS（程序起始地址指针），EIP（段内偏移），ERRORCODE（错误码，正常中断为0），在堆栈数据存放完毕之后，再调用trap进行中断程序的操作



### 编程作业：利用uCore的中断机制，编程实现一个计时器，按“S”开始计时，再按一下“S”停止计时并显示计时时间值。

![](C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220324190757217.png)

在键盘中断的时候，会产生一个键盘中断信号，此时读取字符，如果为S，则进行中断程序操作。

维护一个全局时间变量nanotime，记录了暂停时候的时间戳，每次按下s的时候都会进行一次判断。

- 如果nanotime = -1，则开始计时，nanotime 设为当前 ticks

- 如果nanotime > -1，则停止计时，暂停时间为 当前ticks - nanotime

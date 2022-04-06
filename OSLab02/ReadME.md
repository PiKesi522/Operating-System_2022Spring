<center><h1>Operating System Lab02</h1></center>

<center><h4>10192100571 俞辰杰</h4></center>



#### 练习1、uCore如何探测物理内存布局，其返回结果e820映射结构是什么样的？分别表示什么？

​		uCore使用系统内存映射访问物理内存。uCore的代码段起始地址是－0xC0000000，逻辑地址起始为 0xC0000000，而实际的物理地址起始是0x00000000，通过**逻辑地址KERNBASE 0xC0000000** 加上**起始地址 -0xC0000000**，得到实际的物理地址 0x00000000。

​		e820地址描述符返回的格式是：

​		**8字节的基址 + 8字节的内存大小 + 4字节的内存类型（表示内存可用？保留？等状态）**



#### 练习2、uCore中的物理内存空间管理采用什么样的方案？跟我们理论课中的哪个方案相似？有何不同之处？

​		uCore的物理内存空间管理类似于**段页式存储方案**，将逻辑地址和实际物理地址分开，在实际运行的时候再进行地址的转换。

​		但是uCore的Kernel将分段的功能弱化，**在分段层次上是一个对等映射，也即逻辑地址到线性地址是相等的**。而在分段之后还是采用正常的分页式映射。



#### 练习3、Page数据结构中每个字段含义与作用是什么？如何表示某物理块分配与否？字段property的作用是什么？如何表示 property是否有效？

​		**Page**结构体中包含四个部分，其中：

​			1. **ref**表示该内存块被引用的数量，一般为1，大于1则为公共内存块。

​			2. **flags**有两个作用，分别用flags的后两位表示：

​				最后一位表示该物理块是否已经被使用，1表示已被分配，0表示未被分配

​				倒数第二位表示property字段是否有效，1则有效，0则无效

​			3. **property**是**在此物理块是空闲块的情况下才有效，也即flags的最后一位是0的情况下才有效**，采用一种				类似于链表式存储的方案，将几个连续的空闲内存块作为一块大块的内存空间进行使用，此时**首个空闲				的property标注后面还留有多少空闲的内存块，在此之后的内存块的property仍旧是0**。

​			4. page_link 包含一个前向指针和后向指针，可以将一些空闲块通过列表指针联系在一起



#### 练习4、uCore现有代码已实现的物理块分配首次适应算法以及物理块回收有没有问题或者错误？有没有优化的可能？如有的话请修改、优化相应代码，并说明相关的设计思路。

##### 	

#### 	分配内存

​		**优化：**现有的方法只能返回一个连续的满足大小的内存块，但是如果程序并不要求使用连续的内存块的时候，我们可以多次调用此函数，并且查询n - 1，n - 2的时候是否可以满足条件，从而返回一个地址块数组。这个数组中会尽量多的来满足更大的内存块需求，但是在条件不允许的情况下，也会返回一系列小的内存块碎片。

~~~C
/* PageSeq结构体表示零散内存链表
	mem是Page类型，代表一个地址的起始段
	size是整型，代表mem的内存长度
	*next是链表的用法，将后续的内存连接起来*/
struct PageSep{
    struct Page mem;
    int size;
    struct PageSep *next;
}
PageSep* head;
PageSep* tail = head;

// getMem表示某次函数调用是否得到了一块内存块
bool getMem = false;

static struct PageSep *
default_alloc_pages(size_t n) {
    if (n == 0){
        return head;
    }
    
// ====================== 下方不变 ======================
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    list_entry_t *le, *len;
    le = &free_list;

    while((le=list_next(le)) != &free_list) {
        struct Page *p = le2page(le, page_link);
        if(p->property >= n){
            int i;
            for(i=0;i<n;i++){
                len = list_next(le);
                struct Page *pp = le2page(le, page_link);
                SetPageReserved(pp);
                ClearPageProperty(pp);
                list_del(le);
                le = len;
            }
            if(p->property>n){
                (le2page(le,page_link))->property = p->property - n;
            }
            ClearPageProperty(p);
            SetPageReserved(p);
            nr_free -= n;
// ====================== 上方不变 ======================
            
            // 如果拿到了一块内存块，那么将这块内存的信息放入链表之中，同时更新tail
            PageSep* ans = (PageSep *)malloc(sizeof(PageSep));
            ans->mem = p;
            ans->size = n;
            ans->next = NULL;
            tail->next = ans;
            tail = tail->next;
            getMem = true;

            return head;
        }
    }
    
    if(getMem){
        getMem = false;
        // 如果拿到了一块内存，那么我们还需要的内存数可以减少 size 个
        head = default_alloc_pages(n - tail->size);
    }else{
        // 如果没有拿到内存，那么通过减少一个最大内存块进行调用
        head = default_alloc_pages(n - 1);
    }
    return head;
}
~~~



#### 释放内存

​		**更正：**在释放内存的时候，只对首个内存块进行了恢复操作，而对于之后的内存块的ref引用位，flag使用位，property位，都没有进行还原操作。所以需要改正，使得首位之后的内存块也可以正确的被修正。

~~~C
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    assert(PageReserved(base));

    list_entry_t *le = &free_list;
    struct Page * p;
    while((le=list_next(le)) != &free_list) {
      p = le2page(le, page_link);
      if(p>base){
        break;
      }
    }
    
//====================== 以下为修正内容 ======================
    for(p = base; p < base+n; p++){
        list_add_before(le, &(p->page_link));
        
        // 被释放的内存块的flag的最后一位都应该是0
        p->flag = 0;
        
        // 被释放的内存块的ref都应该是0
      	set_page_ref(p, 0);
        
        // 除了首个内存块，其余内存块的property位都是无效的
        p->property = 0;
    }
    SetPageProperty(base);
    base->property = n;
//====================== 以上为修正内容 ======================
    
    p = le2page(le,page_link) ;
    if( base+n == p ){
      base->property += p->property;
      p->property = 0;
    }
    le = list_prev(&(base->page_link));
    p = le2page(le, page_link);
    if(le!=&free_list && p==base-1){
      while(le!=&free_list){
        if(p->property){
          p->property += base->property;
          base->property = 0;
          break;
        }
        le = list_prev(le);
        p = le2page(le,page_link);
      }
    }

    nr_free += n;
    return ;
}
~~~



#### 练习5、get_pte、get_page函数的作用是什么？它们的输入与返回分别是什么？

​	**get_pte：（get Page Table Entry）**获取页表表项入口，由于uCore使用多级页表进行地址映射，所以会包含页表目录和二级页表，需要分别进行查找

- 输入参数：

  - pgdir：页表目录表的起始地址，定位到页表目录
  - la：需要使用的页表逻辑地址。10位表示目录页表位置偏移，其次10位表示二级页表位置偏移，最后12位表示物理块内位置偏移
  - create：是否页表已经被调入了内存，如果没被调入则需要page_alloc()调入内存

- 返回值：

  - **la所对应的页表表项的起始地址**（将PDE的高20位所对应的块转为PTE的起始物理地址，再将骑士的物理地址转为PTE起始逻辑地址，再将逻辑地址加上Offset得到实际的页表表项）

    

  **get_page：**需要得到la地址所对应的物理块

- 输入参数：

  - pgdir：页表目录表的起始地址，定位到页表目录
  - la：需要使用的页表逻辑地址。10位表示目录页表位置偏移，其次10位表示二级页表位置偏移，最后12位表示物理块内位置偏移
  - ptep_store：用于存储la所对应的页表表项的32位内容

- 返回值：

  - la所对应的物理块的物理块描述符（将PTE的前20位用于寻找物理块的地址，后12位寻找物理块内的偏移找到真实起始地址，再返回物理块描述符）

  

## 编程作业

##### 	**在现有的物理块分配首次适应算法基础上，实现循环首次适应算法或者最佳适应算法**	

#### 最佳适应内存分配Best Fit

~~~C
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    list_entry_t *le, *len;
    le = &free_list;
    
    // 记录某个页面是最佳内存块，初始化是NULL
    struct Page *bestMatch = NULL;
    // 记录最佳内存块的内存链表位置，初始化是NULL
	list_entry_t *bestLink = NULL;
    
    while((le=list_next(le)) != &free_list) {
    	struct Page *p = le2page(le, page_link);
        
    	if(p->property == n){
    	// 如果某个内存块的property数量和要求一致，那么直接退出寻找最有页面，进入下一步
            bestMatch = p;
            bestLink = le;
    		break;
    	}else if(p->property > n){
        // 如果某个内存块的property数量比需求的多，进行下一步比较
        // 如果当前bestMatch拥有的空闲内存块比找到的更多，那么替换bestMatch位当前的内存块
    		if(bestMatch == NULL || bestMatch->property > p->property){
    			bestMatch = p;
                bestLink = le;
    		}
    	}
    }

    // 如果找到了某一块符合要求的内存块，那么开始进行下一步的处理
    // ======================以下内容同First Fit一致======================
    if(bestMatch != NULL){
    	int i;
    	for(i=0; i<n; i++){
			len = list_next(bestLink);
			struct Page *pp = le2page(bestLink, page_link);
			SetPageReserved(pp);
			ClearPageProperty(pp);
			list_del(bestLink);
			bestLink = len;
    	}
    	if(bestMatch->property > n){
    		(le2page(bestLink,page_link))->property = bestMatch->property - n;
    	}
        ClearPageProperty(bestMatch);
        SetPageReserved(bestMatch);
        nr_free -= n;
        return bestMatch;
    }
    return NULL;
}
~~~

<img src="C:\Users\86008\AppData\Roaming\Typora\typora-user-images\image-20220331124855190.png" alt="image-20220331124855190" style="zoom:50%;" />

<center>最佳适应算法成功启动


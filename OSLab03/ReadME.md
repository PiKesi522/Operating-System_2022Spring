<center><h1>Operating System Lab03</h1></center>

<center><h4>10192100571 俞辰杰</h4></center>



#### 练习1、试简述uCore中缺页异常的处理过程，它与理论课中讲述的过程有何不同？

1. ​	uCore处理缺页异常的函数“do_pgfault”包含两个传入参数：		

   - 一个对应的是发生访问异常的时候所访问的虚拟地址addr

     ​	如果在该地址在某个虚拟内存地址当中找不到，那么说明此地址并非正常虚拟地址空间

   - 一个是错误码error_code，包含三位，分别是：

   ​				 0：是否缺页；1：读或写发生异常；2：用户态或核心态访问

   ​			如果0位和1位有一个为0，那么都被视作非正常缺页，不会处理

2. 处理缺页异常的方法也根据不同情况有所不同（判别页是否使用过的方法见题③）：	

   - 如果该页面从来未被访问调用过，那么需要为其分配一个页面，并将这个页从文件中调出，建立一个页号到块号的映射（page_insert）。然后通过pgdir_alloc_page函数来申请一个物理块，并将内容调入。

   ~~~C
   struct Page *pgdir_alloc_page(pde_t *pgdir, uintptr_t la, uint32_t perm){
       // 分配一个内存块
       struct Page *page = alloc_page();
       if(page != NULL){
           //建立一个页和块的关联
           if(page_insert(pgdir, page, la, perm) != 0){
               free_page(page);
               return NULL;
           }
           if(swap_init_ok){
   			swap_map_swappabble(check_mm_struct, la, page, 0);
               // 分配的page页面所对应的虚拟地址为la
               page->pra_vaddr=la;
               // 把page放入FIFO队列的首位
               assert(page_ref(page) == 1);
           }
       }
       return page;
   } 
   ~~~

   

   - 如果该页面曾经被调用过，那么此时应该到对换区来调出这个页（swap_in)，同时申请物理块page。调用方法是根据offset，从对换区找到并读出相应页面的副本（swapfs_read），最后同样建立页和块的映射（page_insert）

   ~~~C
   int swap_in( struct mm_struct *mm,uintptr_t addr, struct Page**ptr_result){
   	struct Page *result = alloc_page( );
       assert ( result!=NULL);
       // 根据虚拟地址取得页表项
   	pte_t *ptep = get_pte(mm->pgdir, addr,0);
   	int r;
       // 右移8位，将高24位的offset放入result之中
   	if ((r = swapfs_read ( (*ptep) , result) ) != 0)
   		assert (r!=0);
       *ptr_result=result;
   	return 0;
   }
   ~~~

   

3. 将页面调入以后，将其对应的物理块设置为swappable，此处的具体做法是将其放入FIFO队列的队首 （swap_map_swappable） 

   ~~~C
   int swap_map_swappable(struct mm_struct*mm,uintptr_t addr,
                      		struct Page *page,int swap_in){
       // 根据swap_manager的定义，此处也即调用fifo_map_swappable
   	return sm->map_swappable(mm,addr, page，swap_in);
   }
   static int _fifo_map_swappable (struct mm struct *mm, uintptr_t addr, struct Page *page, int swap_in){
       // head 为FIFO链表的表头
   	list_entry_t *head = (list_entry_t*) mm->sm_priv;
       // 获得page的链表项entry
       list_entry t *entry = &(page->pra_page_link);
   	assert(entry != NULL && head != NULL);
       // 将entry放入链表表头之后 
       list add(head, entry);
   	return 0;
   }
   ~~~

   

4. 通过FIFO来实现页面置换，使用swap_out_victim

   ~~~C
   static int _fifo_swap_out_victim 
   (struct mm_struct *mm, struct Page **ptr_page, int in_tick){ 
   	list_entry _t *head=(list_entry_t*) mm->sm_priv;
   		assert( head != NULL);
   	assert(in_tick == 0);
       // 目的是为了淘汰链表队尾的表项,使用循环链表，head的前一项也即链表的尾部
   	list_entry_t *le = head->prev;	assert(head != le) ;
       // 将末尾链表表项转为物理块page
   	struct Page *p = le2page(le, pra_page_link);
       // 删除链表末尾
       list_del(le);
   	assert(p != NULL);	
       // 返回被删去的物理块
       *ptr_page = p;
       return 0;
   }
   ~~~

   

5. 在页面分配alloc_page时使用swap_out，在整体上实现页面置换

   ~~~C
   int swap_out(struct mm_struct *mm，int n, int in_tick){
   	int i;
   	for (i = 0; i != n; ++ i){
   		uintptr_t v;
   		struct Page *page;
   		int r = sm->swap_out_victim( mm,&page, in_tick);
           if (r != 0) break;
   		v=page->pra_vaddr;
   		pte_t *ptep = get_pte(mm->pgdir, v,0);
           /*==================此处和书上不一致==================*/
           // 此处无论源页面是否被修改（脏位是否为1，都会向对换区刷新内容 ）
   		if (swapfs_write( (page->pra_vaddr/PGSIZE+1)<<8,page) != 0){
   			sm->map_swappable ( mm, v, page,0);
   			continue;
           /*==================此处和书上不一致==================*/
   		}else{
   			*ptep = (page->pra_vaddr/PGSIZE+1)<<8;free_page(page);
   		}
   		tlb_invalidate( mm->pgdir, v);
   	}
   	return i;
   }
   
   // 需要分配n个物理地址快
   struct Page* alloc_pages (size_t n){
   	struct Page *page = NULL;
       bool intr_flag;
   	while (1){
   		local_intr_save(intr_flag);
           // 调用alloc_page来分配n个物理块
   		page = pmm_manager->alloc_pages(n);
   		local_intr_restore(intr_flag);
           // page!=NULL表示有n个空物理块，
   		if (page != NULL || n > 1 || swap_init_ok == 0)
               break;
   		extern struct mm_struct *check_mm_struct;
           // 如果没有n个空物理块，那么需要淘汰换出n个页面
           swap_out (check_mm_struct, n, 0);
   	}
       return page;
   }
   
   ~~~

 <div STYLE="page-break-after: always;"></div>

#### 练习2、为了支持页面置换，在数据结构Page中添加了哪些字段，其作用是什么？

```C
struct Page{
    int ref;
    uint32_t flags;
    unsigned int property;
    list_entry_t page_link;
    // ========新增======== //
    list_entry_t pra_page_link;
    uintptr_t pra_vaddr;
    // ========新增======== //
}
```

​	新增了 **pra_page_link** 和 **pra_vaddr** 字段，其中：

​	**pra_page_link** 代表着页面置换算法中的链表项，在此题中是用于组成FIFO的链表队列，如果要替换成其他页面置换算法，例如时钟置换算法，则需要其他类型的字段。

​	**pra_vaddr** 代表了一个虚拟地址，由于使用的是请求式的分页地址，所有的页都是经由请求被置换进内存从而实现FIFO，pra_vaddr表示在访问哪个地址的时候产生了缺页



#### 练习3、请描述页目录项页(PDE)和页表目录项(PTE)的组成部分对uCore实现页面置换算法的潜在用处。

**PDE结构：**

| 地址  |      名称       |                在uCore中的对应以及潜在作用                 |
| :---: | :-------------: | :--------------------------------------------------------: |
| 31:12 |  页表起始地址   |            页表的起始物理地址，用于定位页表位置            |
| 11:9  |      Avail      |                                                            |
|   8   |     Ignored     |                                                            |
|   7   |    Page Size    |                      用于确认页的大小                      |
|   6   |                 |                                                            |
|   5   |     访问位      |           用于确认页表是否被使用。1访问过，0反之           |
|   4   | Cache Disabled  |                                                            |
|   3   |  Write Through  |                                                            |
|   2   | User/Supervisor |   用于确认用户态下是否可以访问对应的物理页。1可用，0反之   |
|   1   |     读写位      |         用于确认对应的物理页是否可写。1可写，0反之         |
|   0   |     存在位      | 用于确认页表项所对应的物理页是否存在于页表内。1存在，0反之 |

<div STYLE="page-break-after: always;"></div>

**PTE结构：**

| 地址  |        名称        |          在uCore中的对应以及潜在作用           |
| :---: | :----------------: | :--------------------------------------------: |
| 31:12 | 物理块地址（块号） | 页号对应物理页的起始物理地址，定位物理块的位置 |
| 11:9  |       Avail        |                                                |
|   8   |       Global       |                                                |
|   7   |         0          |                                                |
|   6   |        脏位        |             用于确认数据是否被修改             |
|   5   |       访问位       |             用于确认页表是否被使用             |
|   4   |   Cache Disabled   |                                                |
|   3   |   Write Through    |                                                |
|   2   |  User/Supervisor   |    用于确认用户态下是否可以访问对应的物理页    |
|   1   |       读写位       |          用于确认对应的物理页是否可写          |
|   0   |       存在位       |  用于确认页表项所对应的物理页是否存在于页表内  |



#### 	请问如何区分未映射的页表表项与被换出页的页表表项？

​		被置换出的页表表项会复用PTE结构

​		使用24位的offset，7位的reserved 以及 1位存在位 来表示一个页表表项，其中offset表示页号+1，也即0号页的offset为1。这样做，使得未被使用的页表表项的offset始终为0，而被置换出去的页表的offset则会保留先前的offset，从而加以区别。



#### 	请描述被换出页的页表表项的组成结构。

​		被置换出去的页表表项最高的24位表示被换出的页位于对换区的位置，剩下的7位和之前保持一致，最后一位存在位被置为0。



<div STYLE="page-break-after: always;"></div>

## 编程作业

#### 	编程实现第二次机会页面置换算法和时钟（Clock）页面置换算法。

#### **第二次页面置换算法**

​	基于FIFO，如果在swap_out的时候，某个即将要被换出的物理块的**访问位为1**，那么将其置零，并且放入队首。不断循环，直到再次**找到访问位为0的页面**。

​	相较于lab给出的FIFO代码，只需要在"_fifo_swap_out_victim"中修改置换方法即可

~~~C
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick){
    list_entry_t *head=(list_entry_t*) mm->sm_priv;
    assert(head != NULL);
    assert(in_tick==0);
    list_entry_t *le = head->prev;
    assert(head!=le);
    while(1){
		struct Page *p = le2page(le, pra_page_link);
		if(p->flags[5] == 1){
            // 如果当前页面被访问过，那么将其访问位清空
            p->flags[5] = 0;

            // 记录head的下一个链表
            list_entry_t head_next = head->next;
            list_entry_t le_prev = le->prev;

            // 所有的next更改完毕
            le->next = head_next;
            head->next = le;
            le_prev->next = head;

            // 所有的prev更改完毕
            head_next->prev = le;
            le->prev = head;
            head->prev = le_prev;

            // 往前进行判断
            le = le_prev;
        }else{
            list_del(le);
            assert(p !=NULL);
            *ptr_page = p;
            break;
        }
	return 0;
}
~~~

<div STYLE="page-break-after: always;"></div>

#### **时钟页面置换算法**

​	基于第二次页面置换算法，将链表**改造成循环链表**，不再从head开始找物理块，而是每次都**记录上次置换的下一个页面**

​	同样基于"_fifo_swap_out_victim"进行修改，将head变为全局变量last，每次结束调用都会保存last所指向的下一个页面

~~~C
// 使用last来保存上一次所查询到的位置，下次继续往后查询
list_entry_t *last = (list_entry_t*) mm->sm_priv;
static int
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick){
    assert(last != NULL);
    assert(in_tick==0);
    while(1){
    	struct Page *p = le2page(last, pra_page_link);
		if(p->flags[5] == 1){
			// 如果当前页面被访问过，那么将其访问位清空
	    	p->flags[5] = 0;
            last = last->next;
		}else{
            list_entry_t le = last;
            last = last->next;
     	    list_del(le);
     	    assert(p != NULL);
     	    *ptr_page = p;
	    	break;
		}
	}
	return 0;
}
~~~


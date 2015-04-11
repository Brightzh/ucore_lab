# 练习一
## 实现first－fit连续内存物理分配算法

参考default_pmm.c中的详细注释。

下面分别介绍first－fit中需要的各个函数的实现。

### default_init
```
static void
default_init(void) {
    list_init(&free_list);
    nr_free = 0;
}
```
实现采用了之前的实现，主要是对空闲页链表进行初始化，设置nr_free变量为0。

### default_init_memmap
```
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;
        set_page_ref(p, 0);
        SetPageProperty(p);

        list_add_before(&free_list, &(p->page_link));
    }
    base->property = n;
    nr_free += n;
}
```
将从base开始的n个页加入空闲页链表。

首先遍历其中的每个页，将它们的flags和property设为0，将ref设置为0。然后将p设置为Property，表示该页空闲。
然后使用函数list_add_before将它们加到空闲页链表中。
最后设置base页的property为n，表示该页开始有长为n的空闲页。
最后将计数器nr_free加上n。


### default_alloc_pages
```
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }

    list_entry_t *le, *nextle;
    le = &free_list;

    struct Page *p, *pp;
    while ((le = list_next(le)) != &free_list){
    	p =  le2page(le, page_link);

    	if (p -> property>= n) {
    			int i= 0;
    			for (i=0; i < n; i++){
    				nextle = list_next(le);
    				pp = le2page(le, page_link);
    				SetPageReserved(pp);
    				ClearPageProperty(pp);
    				list_del(le);
    				le = nextle;
    			}
    			if (p->property > n){
    				(le2page(le, page_link))->property = p->property - n;
    			}
    			ClearPageProperty(p);
    			SetPageReserved(p);
    			nr_free -= n;
    			return p;
    			}
    	}
    return NULL;
}
```

首先如果请求的页面数比现有的空闲页多，那么分配失败，返回NULL。
接下来遍历free_list中的每个项，如果该页当前的property大于等于n，那么根据first fit原则，就将这一段空闲页进行分配。
分配过程中将从首页开始的n个页设置Reserved，取消Property，在链表中删除。
如果property大于n，那么要修改接下来的页的property，将其修改为原property大小和n的差。
最后修改首页p，设置其为Reserved，取消Property。减去空闲页数目，返回p的地址即可。

### default_free_pages
```
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);

    list_entry_t *le = &free_list;
    list_entry_t *bef;

    struct Page* p;
    while ((le = list_next(le)) != &free_list){
    	p = le2page(le, page_link);
    	if (p > base) break;
    }

    for (p = base; p< base+n; p++){
    	list_add_before(le, &(p->page_link));
    }

    base ->flags = 0;
    set_page_ref(base, 0);
    ClearPageReserved(base);		//
    SetPageProperty(base);
    base->property = n;

    p = le2page(le, page_link);
    if (base + n == p){
    	base->property += p->property;
    	p->property = 0;
    }

    bef = list_prev(&(base->page_link));
    p = le2page(bef, page_link);
    if (bef != &free_list && p == base-1){
    		while (bef != &free_list){
    			if (p -> property != 0){
    				p->property += base->property;
    				base->property = 0;
    				break;
    			}
    			bef = list_prev(bef);
    			p = le2page(bef, page_link);
    		}
    }
    nr_free += n;
    return;
}
```

首先遍历链表，找到当前页需要插入的位置。
然后将这n个页插入到当前位置前。
之后考虑是否需要进行合并操作。
如果base＋n＝p，那么需要和后面的页进行合并，那么只需要修改base和p的property的值即可。
如果前面页和base相连，那么还需要向前进行合并，不断向前查找直到property不为0，修改该页和base页的property集合即可。

### first fit改进空间
鉴于循环双向链表的数据结构，我们发现
在合并过程中需要向前不断遍历，花销较大。

我们可以在page中设立一个head域，每一段连续页空间中的页的head值都指向首个页，这样在向前查找时，
使用O(1)的时间就可以找到前面区间的头，而不必进行遍历。

# 练习二
## 寻找虚拟地址对应的页表项

通过参考代码中的注释，实现对于一个给定虚拟地址的页表项的查找。

```
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
	pde_t *pdep = &pgdir[PDX(la)];
	if (!(*pdep & PTE_P)){
		struct Page* page;
		if (!create) return NULL;
		page = alloc_page();
		if (page == NULL) return NULL;
		set_page_ref(page, 1);
		uintptr_t pa = page2pa(page);
		memset(KADDR(pa), 0, PGSIZE);
		*pdep = pa | PTE_U | PTE_W | PTE_P;
	}
	return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
}
```
首先得到pdep，此为虚拟地址对应的页目录表的指针。使用宏PDX可以得到虚拟地址la在页目录中的索引。
如果页表项不存在，那么考察create，如果create为false，那么退出。
如果create为true，那么分配页面。设置页面的ref值，将页面内容清空为0。设置pdep页表项。
最后通过统一的语句查找页表项并返回。
即先从pdep指针中得到PTE的物理地址，然后访问PTE中la中index的位置并返回。

### 请描述页目录项（Page Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处。
查看mmu.h可以得到关于组成内容的宏定义。页目录项和页表项的组成部分如下


－ 31-12 下一级索引地址

－ 11-9 Available for software use

－ 8-7 Must be Zero，其中7为Page Size

－ 6 Dirty

－ 5 Accessed

－ 4 Cache-Disable

－ 3 Write-Through，是否采用写直达

－ 2 User

－ 1 Writeable，是否可写

－ 0 Present，是否存在


位0是存在（Present）标志，用于指明表项对地址转换是否有效。P=1表示有效；P=0表示无效。在页转换过程中，如果说涉及的页目录或页表的表项无效，则会导致一个异常。如果P=0，那么除表示表项无效外，其余位可供程序自由使用。
位1是读/写（Read/Write）标志。如果等于1，表示页面可以被读、写或执行。如果为0，表示页面只读或可执行。当处理器运行在超级用户特权级（级别0、1或2）时，则R/W位不起作用。页目录项中的R/W位对其所映射的所有页面起作用。
位2是用户/超级用户（User/Supervisor）标志。如果为1，那么运行在任何特权级上的程序都可以访问该页面。如果为0，那么页面只能被运行在超级用户特权级（0、1或2）上的程序访问。页目录项中的U/S位对其所映射的所有页面起作用。
位5是已访问（Accessed）标志。当处理器访问页表项映射的页面时，页表表项的这个标志就会被置为1。当处理器访问页目录表项映射的任何页面时，页目录表项的这个标志就会被置为1。处理器只负责设置该标志，操作系统可通过定期地复位该标志来统计页面的使用情况。
位6是页面已被修改（Dirty）标志。当处理器对一个页面执行写操作时，就会设置对应页表表项的D标志。处理器并不会修改页目录项中的D标志。
11-9位该字段保留专供程序使用。

### 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

1. 从CPU收到中断事件后，打断当前程序或任务的执行，根据某种机制跳转到中断服务例程去执行。CPU会根据中断向量查找IDT，找到中断描述符中的段选择子。之后从GDT取得相应的段描述符，这样便得到了中断服务例程的起始地址。然后CPU会根据CPL和DPL确认是否发生了特权级的转换。之后保存现场，开始执行中断服务例程。

2. 发生缺页异常时，中断服务例程会将缺失的页调入到内存中，有些情况下会进行页置换。

3. 每个中断服务例程在有中断处理工作完成后需要通过iret（或iretd）指令恢复被打断的程序的执行。恢复现场保存信息，并完成特权级的转换。回到出现异常的语句继续执行。

# 练习三
## 释放某虚地址所在的页并取消对应二级页表项的映射

观察代码中的注释和提示进行实现。
```
    if (*ptep & PTE_P) {
        struct Page *page = pte2page(*ptep);
        if (page_ref_dec(page) == 0) {
            free_page(page);
        }
        *ptep = 0;
        tlb_invalidate(pgdir, la);
    }
```

如果ptep存在，PRESENCE的位为1。
那么得到当前ptep对应的页，调用free_page进行释放。
删除页表项。
删除页目录项。

### 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？
有对应关系。比如根据pte中的PTE_ADDR，右移12位后即可得到该页表项对应的页在Page全局数组中的index。
即语句pages[PPN(PTE_ADDR(*ptep))]
### 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 鼓励通过编程来具体完成这个问题
修改起始地址，虚拟地址从0xC00000000开始，将其减掉即可。 


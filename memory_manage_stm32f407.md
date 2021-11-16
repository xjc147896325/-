<p>内存管理是操作系统中的重要概念，在有着操作系统的使用环境中十分重要，包括RTOS，其实内存管理不仅在操作系统中很重要，在任何设计内存动态分配的场合都十分重要。</p>
<p>它可以使得用户更方便地使用存储器、提高内存利用率，还可以通过虚拟技术从逻辑上扩充存储器。</p>
<p>内存管理的功能：</p>
<ol>
  <li>内存空间的分配与回收：由操作系统完成主存储器空间的分配和管理，使程序员摆脱存储分配的麻烦，提高编程效率
  </li>
  <li>地址转换：在多道程序环境下，程序中的逻辑地址与内存中的物理地址不可能一致，因此存储管理必须提供地址转换功能，把逻辑地址转换成相应的物理地址
  </li>
  <li>内存空间的扩充：利用虚拟内存技术或自动覆盖技术，从逻辑上扩充内存
  </li>
  <li>4.存储保护：保证各道作业在各自的存储空间内运行，互不干扰
  </li>
</ol>
<p>在使用SD卡时，内存管理机制也十分重要，因为若用二维数组(一个维度表示文件数量，一个维度表示文件名)，会占用很大的空间，不便使用。下图是F407的选型。</p>
<img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/F407_3.png'>
<img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/F407_2.png'>
<img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/F407_1.png'>
<p>能看到RAM只有192K，IC内部共有3个内存块，分别为SRAM1、SRAM2和CCM。</p>
<p>CCM（core coupled memory）（核心耦合存储器）：理论上是最快的，但是它只能被CPU访问，像其他外设（DMA、以太、USB），都无法访问</p>
<img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/memories_f407.png'>
<img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/memories_f407_2.png'>
<p></p>
<h3>分块式内存管理：</h3>
<p>是由<b>内存池</b>和<b>内存管理表</b>两部分组成的，里面有n个内存块，每个内存块对应是内存管理表中的一项，若为0则代表对应块未占用，若非零，里面的数值则表示被连续占用的内存块数。</p>
<img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/memories_f407_3.png'>
<p>分配内存时，先输入参数m表示需要多少内存块，然后从第n项开始，向下查找，直到发现m块连续的空内存，然后将这些空内存的项值都设为m，最终返回空内存块的地址。</p>
<p>以下为正点原子的malloc.c和malloc.h文件</p>
<h4>malloc.c:</h4>
<pre>
#include "malloc.h"	   

//内存池(32字节对齐)
__align(32) u8 mem1base[MEM1_MAX_SIZE];					//内部SRAM内存池
__align(32) u8 mem2base[MEM2_MAX_SIZE] __attribute__((at(0X68000000)));	//外部SRAM内存池，attribute修饰，指向0x68000000首地址
__align(32) u8 mem3base[MEM3_MAX_SIZE] __attribute__((at(0X10000000)));	//内部CCM内存池，attribute修饰，指向0x10000000首地址
//内存管理表
u16 mem1mapbase[MEM1_ALLOC_TABLE_SIZE];							//内部SRAM内存池MAP
u16 mem2mapbase[MEM2_ALLOC_TABLE_SIZE] __attribute__((at(0X68000000+MEM2_MAX_SIZE)));	//外部SRAM内存池MAP
u16 mem3mapbase[MEM3_ALLOC_TABLE_SIZE] __attribute__((at(0X10000000+MEM3_MAX_SIZE)));	//内部CCM内存池MAP
//内存管理参数	   
const u32 memtblsize[SRAMBANK]={MEM1_ALLOC_TABLE_SIZE,MEM2_ALLOC_TABLE_SIZE,MEM3_ALLOC_TABLE_SIZE};	//内存表大小
const u32 memblksize[SRAMBANK]={MEM1_BLOCK_SIZE,MEM2_BLOCK_SIZE,MEM3_BLOCK_SIZE};			//内存分块大小
const u32 memsize[SRAMBANK]={MEM1_MAX_SIZE,MEM2_MAX_SIZE,MEM3_MAX_SIZE};				//内存总大小


//内存管理控制器
struct _m_mallco_dev mallco_dev=
{
	my_mem_init,						//内存初始化
	my_mem_perused,						//内存使用率
	mem1base,mem2base,mem3base,			//内存池
	mem1mapbase,mem2mapbase,mem3mapbase,//内存管理状态表
	0,0,0,  		 				//内存管理未就绪
};

//复制内存
//*des:目的地址
//*src:源地址
//n:需要复制的内存长度(字节为单位)
void mymemcpy(void *des,void *src,u32 n)  
{  
    u8 *xdes=des;
	u8 *xsrc=src; 
    while(n--)*xdes++=*xsrc++;  
}  
//设置内存
//*s:内存首地址
//c :要设置的值
//count:需要设置的内存大小(字节为单位)
void mymemset(void *s,u8 c,u32 count)  
{  
    u8 *xs = s;  
    while(count--)*xs++=c;  
}	   
//内存管理初始化  
//memx:所属内存块
void my_mem_init(u8 memx)  
{  
    mymemset(mallco_dev.memmap[memx], 0,memtblsize[memx]*2);//内存状态表数据清零  
	mymemset(mallco_dev.membase[memx], 0,memsize[memx]);	//内存池所有数据清零  
	mallco_dev.memrdy[memx]=1;				//内存管理初始化OK  
}  
//获取内存使用率
//memx:所属内存块
//返回值:使用率(0~100)
u8 my_mem_perused(u8 memx)  
{  
    u32 used=0;  
    u32 i;  
    for(i=0;i<memtblsize[memx];i++)  
    {  
        if(mallco_dev.memmap[memx][i])used++; 
    } 
    return (used*100)/(memtblsize[memx]);  
}  
//内存分配(内部调用)
//memx:所属内存块
//size:要分配的内存大小(字节)
//返回值:0XFFFFFFFF,代表错误;其他,内存偏移地址 
u32 my_mem_malloc(u8 memx,u32 size)  
{  
    signed long offset=0;  
    u32 nmemb;	//需要的内存块数  
	u32 cmemb=0;//连续空内存块数
    u32 i;  
    if(!mallco_dev.memrdy[memx])mallco_dev.init(memx);//未初始化,先执行初始化 
    if(size==0)return 0XFFFFFFFF;//不需要分配
    nmemb=size/memblksize[memx];  	//获取需要分配的连续内存块数
    if(size%memblksize[memx])nmemb++;  
    for(offset=memtblsize[memx]-1;offset>=0;offset--)//搜索整个内存控制区  
    {     
		if(!mallco_dev.memmap[memx][offset])cmemb++;//连续空内存块数增加
		else cmemb=0;				//连续内存块清零
		if(cmemb==nmemb)			//找到了连续nmemb个空内存块
		{
            for(i=0;i<nmemb;i++)  					//标注内存块非空 
            {  
                mallco_dev.memmap[memx][offset+i]=nmemb;  
            }  
            return (offset*memblksize[memx]);//返回偏移地址  
		}
    }  
    return 0XFFFFFFFF;//未找到符合分配条件的内存块  
}  
//释放内存(内部调用) 
//memx:所属内存块
//offset:内存地址偏移
//返回值:0,释放成功;1,释放失败;  
u8 my_mem_free(u8 memx,u32 offset)  
{  
    int i;  
    if(!mallco_dev.memrdy[memx])//未初始化,先执行初始化
	{
		mallco_dev.init(memx);    
        return 1;//未初始化  
    }  
    if(offset<memsize[memx])//偏移在内存池内. 
    {  
        int index=offset/memblksize[memx];			//偏移所在内存块号码  
        int nmemb=mallco_dev.memmap[memx][index];	//内存块数量
        for(i=0;i<nmemb;i++)  				//内存块清零
        {  
            mallco_dev.memmap[memx][index+i]=0;  
        }  
        return 0;  
    }else return 2;//偏移超区了.  
}  
//释放内存(外部调用) 
//memx:所属内存块
//ptr:内存首地址 
void myfree(u8 memx,void *ptr)  
{  
	u32 offset;   
	if(ptr==NULL)return;//地址为0.  
 	offset=(u32)ptr-(u32)mallco_dev.membase[memx];     
    my_mem_free(memx,offset);	//释放内存      
}  
//分配内存(外部调用)
//memx:所属内存块
//size:内存大小(字节)
//返回值:分配到的内存首地址.
void *mymalloc(u8 memx,u32 size)  
{  
    u32 offset;   
	offset=my_mem_malloc(memx,size);  	   	 	   
    if(offset==0XFFFFFFFF)return NULL;  
    else return (void*)((u32)mallco_dev.membase[memx]+offset);  
}  
//重新分配内存(外部调用)
//memx:所属内存块
//*ptr:旧内存首地址
//size:要分配的内存大小(字节)
//返回值:新分配到的内存首地址.
void *myrealloc(u8 memx,void *ptr,u32 size)  
{  
    u32 offset;    
    offset=my_mem_malloc(memx,size);   	
    if(offset==0XFFFFFFFF)return NULL;     
    else  
    {  									   
	    mymemcpy((void*)((u32)mallco_dev.membase[memx]+offset),ptr,size);	//拷贝旧内存内容到新内存   
        myfree(memx,ptr);  								//释放旧内存
        return (void*)((u32)mallco_dev.membase[memx]+offset);  				//返回新内存首地址
    }  
}
</pre>
<h4>malloc.h</h4>
<pre>
#ifndef __MALLOC_H
#define __MALLOC_H
#include "stm32f4xx.h"

 
#ifndef NULL
#define NULL 0
#endif

//定义三个内存池
#define SRAMIN	 0		//内部内存池
#define SRAMEX   1		//外部内存池
#define SRAMCCM  2		//CCM内存池(此部分SRAM仅仅CPU可以访问!!!)


#define SRAMBANK 	3	//定义支持的SRAM块数.	


//mem1内存参数设定.mem1完全处于内部SRAM里面.
#define MEM1_BLOCK_SIZE			32  	  						//内存块大小为32字节
#define MEM1_MAX_SIZE			100*1024  						//最大管理内存 100K
#define MEM1_ALLOC_TABLE_SIZE	MEM1_MAX_SIZE/MEM1_BLOCK_SIZE 	//内存管理表大小

//mem2内存参数设定.mem2的内存池处于外部SRAM里面
#define MEM2_BLOCK_SIZE			32  	  						//内存块大小为32字节
#define MEM2_MAX_SIZE			960 *1024  						//最大管理内存960K
#define MEM2_ALLOC_TABLE_SIZE	MEM2_MAX_SIZE/MEM2_BLOCK_SIZE 	//内存管理表大小
		 
//mem3内存参数设定.mem3处于CCM,用于管理CCM(特别注意,这部分SRAM,仅CPU可以访问!!)
#define MEM3_BLOCK_SIZE			32  	  						//内存块大小为32字节
#define MEM3_MAX_SIZE			60 *1024  						//最大管理内存60K
#define MEM3_ALLOC_TABLE_SIZE	MEM3_MAX_SIZE/MEM3_BLOCK_SIZE 	//内存管理表大小
		 


//内存管理控制器
struct _m_mallco_dev
{
	void (*init)(u8);					//初始化
	u8 (*perused)(u8);		  	    	//内存使用率
	u8 	*membase[SRAMBANK];				//内存池 管理SRAMBANK个区域的内存
	u16 *memmap[SRAMBANK]; 				//内存管理状态表
	u8  memrdy[SRAMBANK]; 				//内存管理是否就绪
};
extern struct _m_mallco_dev mallco_dev;	 //在mallco.c里面定义

void mymemset(void *s,u8 c,u32 count);	//设置内存
void mymemcpy(void *des,void *src,u32 n);//复制内存     
void my_mem_init(u8 memx);				//内存管理初始化函数(外/内部调用)
u32 my_mem_malloc(u8 memx,u32 size);	//内存分配(内部调用)
u8 my_mem_free(u8 memx,u32 offset);		//内存释放(内部调用)
u8 my_mem_perused(u8 memx);				//获得内存使用率(外/内部调用) 
////////////////////////////////////////////////////////////////////////////////
//用户调用函数
void myfree(u8 memx,void *ptr);  			//内存释放(外部调用)
void *mymalloc(u8 memx,u32 size);			//内存分配(外部调用)
void *myrealloc(u8 memx,void *ptr,u32 size);//重新分配内存(外部调用)
#endif
</pre>
<p>在malloc.h中存在3个宏定义，SRAMIN、SRAMEX和SRAMCCM，SRAMEX为板载外部拓展的SRAM。</p>
<p>下方的SRAMBANK则为内存池数量</p>
<p>三个区间宏定义，mem1、mem2和mem3分别对应SRAMIN、SRAMEX和SRAMCCM，除开内存管理表的大小这个宏定义不可修改外，其他均根据实际使用来定。</p>
<p>这里原博客写的有些问题，<a href='' target='_blank'>参考这里</a>。内存实际大小应该是数组大小*32B。</p>
<pre>
这部分代码，定义了很多关键数据，比如内存块大小的定义：MEM1_BLOCK_SIZE、
MEM2_BLOCK_SIZE 和 MEM3_BLOCK_SIZE，都是 64 字节。内存池总大小，内部 SRAM 内
存池大小为 160K，外部 SDRAM 内存池大小为 28912K，内部 DTCM 内存池大小为 120K。
MEM1_ALLOC_TABLE_SIZE、MEM2_ALLOC_TABLE_SIZE 和 MEM3_ALLOC_TABLE_
SIZE，则分别代表内存池 1、2 和 3 的内存管理表大小。
从这里可以看出，如果内存分块越小，那么内存管理表就越大，当分块为 4 字节 1 个块的
时候，内存管理表就和内存池一样大了（管理表的每项都是 u32 类型）。显然是不合适的，我们
这里取 64 字节，比例为 1:16，内存管理表相对就比较小了。
</pre>
<p>除此之外，有以下四个较为重要的接口函数：my_mem_init、mymalloc、myfree和my_mem_perused。</p>
<p></p>
<p>在malloc.h中定义了内存池、内存管理表和诸多内存管理参数，内存池定义中的align是字节对齐，<a gref='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/memory_manage.md' target='_blank'>参考我的上个md.</a></p>
<p>attribute是修饰，指向其放置首地址，而CCM的首地址即为0x10000000。</p>
<img src='https://github.com/xjc147896325/Cross-hardware-recording/blob/main/memories_f407_4.png'>

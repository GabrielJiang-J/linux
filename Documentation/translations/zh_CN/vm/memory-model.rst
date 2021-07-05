.. SPDX-License-Identifier: GPL-2.0

.. include:: ../disclaimer-zh_CN.rst

.. _physical_memory_model:

:Original: Documentation/vm/index.rst

:译者:

 蒋子奇 Jiang Ziqi <jiangziqi@360.cn>

============
物理内存模型
============

系统中物理内存能够以多种方式进行寻址。最简单的方式是，当物理内存地址
从0开始，跨越一段连续的范围，直到最大地址。当然，物理内存中也可以包含
一些CPU无法访问的小的空洞。此时，在物理内存中完全不同的地址范围内，会
有多段连续的区间。不要忘记还有NUMA，不同的物理内存bank会绑定到不同的
CPU。

Linux将这种多样性抽象为下列三种模型之一：平坦内存模型、非连续内存模型、
稀疏内存模型。每一种体系架构都可以定义其支持的模型，定义默认使用内存
模型，定义是否可以修改默认内存模型。

.. note::
   在写这篇文档时，非连续内存模型已经被认为时废弃了的，虽然它仍然在一些
   体系架构中使用。

所有的内存模型都是跟踪物理页帧的状态，描述物理页帧的数据结构struct page
分布在一个或多个数组中。

不管是什么内存模型，都存在一个一对一映射的关系，即物理页帧号（PFN）和
描述物理页的数据结构`struct page`之间的一一映射。

每一个内存模型都会定义函数 :c:func:`pfn_to_page`和:c:func:`page_to_pfn`
来辅助PFN到`struct page`的映射，以及反向的映射。

平坦内存模型
============

最简单的内存模型是平坦内存模型。平台内存模型适用于连续的或者几乎连续的
non-NUMA物理内存系统。

在平坦内存模型中，有一个全局的`mem_map`数组，用来映射整个物理内存。对
于大部分的体系结构而言，内存空洞在`mem_map`数组中有完整的条目。描述这
些空洞的`struct page`对象永远不会被初始化。

为了分配`mem_map`数组内存空间，体系架构相关的初始化代码需要调用 
:c:func:`free_area_init` 函数。当然，映射数组还不可用，直到调用函数
:c:func:`memblock_free_all` ，这个函数用来处理页分配器的所有内存。

有些体系架构可能会释放掉数组`mem_map`的部分空间，因为这部分空间不再覆
盖实际的物理页。在这种情况下，与体系架构相关的函数 :c:func:`pfn_valid`
实现应该将`mem_map`中的空洞保存在统计信息中。

在平坦内存模型中，PFN和`struct page`之间的转换规则是直接明了的：
`PFN - ARCH_PFN_OFFSET`，表示一个在`mem_map`数组中的索引。

`ARCH_PFN_OFFSET`定义了系统物理内存中从0开始的中第一个页帧号。

非连续内存模型
==============

非连续内存模型认为物理内存是一个`nodes`的集合，类似于Linux NUMA机制。
Linux会为每一个node构造一个独立的内存管理子系统，这些内存管理子系统
使用`struct pglist_data`（或者`pg_data_t`）来表示。`pg_data_t`保存着
`node_mem_map`数组，这个数组用来映射属于node本身的物理页。`node_start_pfn`
字段代表属于node本身的第一个页帧号。

体系架构初始化代码需要调用函数 :c:func:`free_area_init_node`，为系统
中每一个node初始化`pg_data_t`对象和`node_mem_map`。

每一个`node_mem_map`的行为非常类似于平坦内存模型的`mem_map` - node中
每一个物理页帧，在`node_mem_map`数组中都有一个`strcut page`项。当非
连续内存模型使能，`struct page`的`flags`字段的一部分会被编码为拥有该
该页的node编号。

在非连续内存模型中，PFN和`struct page`之间的转换规则变得稍微复杂了一
点点，因为需要确定哪个node拥有这个物理页，还需要确认`struct page`属于
哪一个`pg_data_t`对象。

支持非连续内存模型的体系架构提供函数 :c:func:`pfn_to_nid`用来将PFN转换
成node编号。另一个转换辅助函数 :c:func:`page_to_nid`是通用的，因为它使
用编码在page->flags中的node编号。

一旦取得node编号，PFN就可以索引相应的`node_mem_map`数组来访问`struct page`，
也可以通过用`node_start_pfn`加上`struct page`在`node_mem_map`数组中的偏移，
来获取对应页的PFN。

稀疏内存模型
============

稀疏内存模型在Linux中是非常通用的内存模型，是仅有的可以支持物理内存热插拔、
非易失性存储设备的替代映射和大型系统的存储映射的延迟初始化等多种高级特性的
内存模型。


稀疏内存模型将物理内存表示为内存分段的集合。一个分段使用struct mem_section
描述，struct mem_section含有一个指向struct pages的数组的指针`section_mem_map`。
此外，也存储了一些帮助内存分段理的字段。分段的大小和分段的最大数量分别由
`SECTION_SIZE_BITS`和`MAX_PHYSMEM_BITS`宏描述，这两个宏是由支持稀疏内存模型
体系架构定义的。`MAX_PHYSMEM_BITS`是一个体系架构支持的具体的物理地址的宽度值，
`SECTION_SIZE_BITS`是一个任意的值。

分段数量的最大值是由`NR_MEM_SECTIONS`表示，有下列表达式定义：

.. math::

   NR\_MEM\_SECTIONS = 2 ^ {(MAX\_PHYSMEM\_BITS - SECTION\_SIZE\_BITS)}

`mem_section`对象被组织成一个二维数组`mem_sections`。数组的大小和使用依赖
`CONFIG_SPARSEMEM_EXTREME`和分段数量的最大可能值：

* 当`CONFIG_SPARSEMEM_EXTREME`被禁用时，`mem_sections`数组是一个
  `NR_MEM_SECTIONS`行的静态数组。每一行存储一个`mem_section`对象。
* 当`CONFIG_SPARSEMEM_EXTREME`使能时，`mem_sections`数组是动态分配的。
  每一行存储存储的`mem_setcion`占用的空间尽可能满足PAGE_SIZE，并且计算
  总行数以适应所有的内存分段。

体系架构初始化代码应该调用sparse_init()函数来初始化内存分段和内存映射。

使用稀疏内存模型，有两种方式可以将PFN转换为相关联的`struct page` - 
“classic sparse”和“sparse vmemmap”。转换方式通过宏`CONFIG_SPARSEMEM_VMEMMAP`
定义。

“classic sparse”将内存域的分段编号编码在page-flags中，并且使用PFN的高位
访问分段所映射的页帧。在分段内部，PFN就是页数组的索引。

“sparse vmemmap”使用内存映射来优化pfn_to_page和page_to_pfn操作。全局变量
`struct page *vmemmap`指针指向`struct page`数组。`struct page`在`vmemmap`
中的偏移就是这个页的PFN，PFN是这个`struct page`数组的索引。

为了使用vmemmap，体系架构必须预留一部分虚拟地址，用来映射包含内存映射的
物理页框，并且确保`vmemmap`指针指向这些虚拟地址。另外，体系架构需要实现
函数:c:func:`vmemmap_populate`，用来分配物理内存和为页表创建虚拟地址映射。
如果体系架构对vmemmap映射没有特别的需求，可以使用通用内存管理系统提供的
默认函数:c:func:`vmemmap_populate_basepages`。


虚拟映射的内存映射可以将持久性存储设备的`struct page`对象存储在这些设备上
预先分配的内存中。这些存储使用结构提vmem_altmap表示，并且最终会通过很长的
函数调用链传递到函数vmemmap_populate()中。vmemmap_populate()函数的实现可以
使用`vmem_altmap`和帮助函数:c:func:`vmemmap_alloc_block_buf`在持久性存储设
备中分配内存映射。

ZONE_DEVICE
===========
`ZONE_DEVICE`依赖`SPARSEMEM_VMEMMAP`，为对应物理地址范围的设备驱动提供
`struct page`和`mem_map`服务。`ZONE_DEVICE`的“设备”方面与以下事实有关：
这些地址范围的页面对象不会标记为在线，而且必须对设备(而不仅仅是页面)
进行引用，以保持内存固定以便使用。`ZONE_DEVICE`通过函数:c:func:`devm_memremap_pages`，
对指定的pfn范围执行足够的内存热插拔，以打开服务:c:func`pfn_to_page`,
:c:func:`page_to_pfn`和:c:func:`get_user_pages`服务。由于页的引用计数不会
减少到0，所以页不会被当作空闲内存追踪，并且页的`struct list_head lru`空间
被重定义为对映射内存的设备/驱动的反向映射。

虽然`SPARSEMEM`表示内存为分段的集合，可选地收集到内存块中，`ZONE_DEVICE`
用户需要更小粒度的填充`mem_map`。如果`ZONE_DEVICE`内存从来没有被标记为在线，
那么它也就永远不会被标记为在线，它的内存范围是通过内存块边界上的sysfs内存
热插拔API暴露的。该实现依赖于这种缺乏用户api约束的情况来允许分段大小的内存，
内存热插拔的上半部分通过函数:c:func:`arch_add_memory`指定。对于函数
:c:func:`devm_memremap_pages`，子分段支持使用2MB作为跨平台通用对齐粒度。

`ZONE_DEVICE`的使用者：

* pmem: 持久内存被用来作为直接I/O通过DAX映射。

* hmm: 使用`->page_fault()`和`->page_free()`事件回调函数来拓展`ZONE_DEVICE`,
  能够使设备驱动程序协调与设备内存相关的内存管理事件，特别是GPU内存。参考：
  Documentation/vm/hmm.rst

* p2pdma: 创建`struct page`对象，使PCI/-E拓扑中的对端设备协调它们之间的
  直接dma操作，比如旁路主存储器。

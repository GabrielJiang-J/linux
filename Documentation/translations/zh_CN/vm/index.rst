.. SPDX-License-Identifier: GPL-2.0

.. include:: ../disclaimer-zh_CN.rst

:Original: Documentation/vm/index.rst

:译者:

 蒋子奇 Jiang Ziqi <jiangziqi@360.cn>

=================
Linux内存管理指南
=================

这是关于Linux内存管理（mm）子系统的一系列文档。如果你只是简单的查找关于
内存分配的内容，请参考 :ref:`memory_allocation`。

内存管理特性用户指南
====================

下列的文档提供了关于控制和优化Linux内存管理不同特性的指导。

.. toctree::
   :maxdepth: 1

   swap_numa
   zswap

内核开发者内存管理文档
======================

下列文档从不同层次描述了内存管理子系统的内部原理，从各种笔记和邮件
列表的交互，到相关数据结构和算法的详细描述。

.. toctree::
   :maxdepth: 1

   active_mm
   arch_pgtable_helpers
   balance
   cleancache
   free_page_reporting
   frontswap
   highmem
   hmm
   hwpoison
   hugetlbfs_reserv
   ksm
   memory-model
   mmu_notifier
   numa
   overcommit-accounting
   page_migration
   page_frags
   page_owner
   remap_file_pages
   slub
   split_page_table_lock
   transhuge
   unevictable-lru
   z3fold
   zsmalloc

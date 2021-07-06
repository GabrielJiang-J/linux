.. SPDX-License-Identifier: GPL-2.0

.. include:: ../disclaimer-zh_CN.rst

.. _swap_numa:

:Original: Documentation/vm/swap_numa.rst

:译者:

 蒋子奇 Jiang Ziqi <jiangziqi@360.cn>


==========================
自动绑定交换设备到numa节点
==========================

如果系统有多个交换设备，并且交换设备有节点信息，我们可以利用这些信息来决
定在get_swap_pages()中使用哪个交换设备以获得更好的性能。


如何使用这个特性
================

交换设备具有优先级，并决定其使用的顺序。要使用自动绑定，不需要操作交换设备
的优先级设置。例如，在一个有2个节点的机器上，假设2个交换设备swapA和swapB将
被开启，swapA绑定到节点0，swapB绑定到节点1。通过简单的操作开启交换::

	# swapon /dev/swapA
	# swapon /dev/swapB

节点0将按照swapA然后swapB的顺序使用两个交换设备，节点1将按照swapB然后swapA
的顺序使用两个交换设备。注意，它们开启交换的顺序并不重要。

一个4节点机器上的更复杂的示例。假设要在6个交换设备上开启交换：swapA和swapB
绑定到节点0，swapC绑定到节点1，swapD和swapE绑定到节点2，swapF绑定到节点3。
开启交换的方法与上面相同::

	# swapon /dev/swapA
	# swapon /dev/swapB
	# swapon /dev/swapC
	# swapon /dev/swapD
	# swapon /dev/swapE
	# swapon /dev/swapF

节点0会按照下面的顺序使用它们::

	swapA/swapB -> swapC -> swapD -> swapE -> swapF

swapA和swapB将以轮询方式在其他交换设备之间使用。

节点0会按照下面的顺序使用他们::

	swapC -> swapA -> swapB -> swapD -> swapE -> swapF

节点2会按照下面的顺序使用它们::

	swapD/swapE -> swapA -> swapB -> swapC -> swapF

类似地，swapD和swapE会在其他交换设备之间以轮询模式使用。

节点3会按照下面的顺序使用它们::

	swapF -> swapA -> swapB -> swapC -> swapD -> swapE


实现细节
========

当前代码使用一个基于优先级的列表swap_avail_list来决定使用哪个交换设备，
如果多个交换设备拥有相同的优先级，它们将采用轮询方式。这里的更改将单个
全局swap_avail_list替换为每个numa节点列表，也就是说，对于每个numa节点，
它会看到自己的基于优先级的可用交换设备列表。交换设备的优先级可以在其匹
配节点的swap_avail_list上提升。

当前交换设备的优先级设置为：用户可以设置一个>=0的值，或者系统从-1开始，
然后向下选择一个。swap_avail_list中的优先级是负值，是由于plist从低到高
排序。新策略不会改变优先级>=0情况下的语义，以前从-1开始向下，现在变成从
-2开始向下，-1保留为提升值。因此，如果多个交换设备连接到同一个节点，它
们都将被提升到该节点的plist上的优先级-1，并将在任何其他交换设备之间轮询
使用。

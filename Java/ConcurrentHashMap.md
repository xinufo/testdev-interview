# ConcurrentHashMap

## 概述

ConcurrentHashMap为线程安全的，且在高并发下的表现优于Hashtable，但ConcurrentHashMap在并发读写下为弱一致性，即读取的数据不一定能反应全部的修改操作。ConcurrentHashMap在jdk 1.7时使用分段锁技术保证并发，一个Segment中包含多个Entry，同时多个Segment互不影响，但是jdk 1.8修改了此方案，改用CAS + synchronized来保证并发，同时也加入了红黑树来减少长链表的查询时间。

在jdk 1.8中ConcurrentHashMap和HashMap一样使用 hash & (len - 1)求余数，hash = hash ^ (hash >>> 16)引入hash高位变化，但是ConcurrentHashMap的hash最后会执行hash & 0x7fffffff保证永远是正数（因为负hash有特殊含义）。同样table属性也是延迟初始化的，只有在第一次写入后才正在初始化。

ConcurrentHashMap通过sizeCtl属性来控制table初始化和扩容，sizeCtl == -1 时表示table正在初始化，sizeCtl == -N 时表示正在扩容，table初始化完成后sizeCtl = table.length * 0.75。

### synchronized是重量级锁为什么还使用它进行同步？

> 在jdk 1.6只有对synchronized进行了优化，会以偏向锁->轻量级锁->重量级锁的顺序逐步升级，进而提高了synchronized的性能

## get

ConcurrentHashMap的get操作不加任何锁，直接通过unsafe.getObjectVolatile获取数组里的Node，若Node的hash为负数(ForwardingNode的hash为-1，该对象里扩容时会存储新数组)，说明数组正在扩容，则通过find方法在新数组里查找key；否则遍历后续节点查找

### ConcurrentHashMap中的table属性已经使用了volatile修饰，为什么还要用unsafe.getObjectVolatile获取元素

> 因为volatile只能保证数组引用的可见性，但是无法保证里面元素修改后立刻对其他线程可见

## put

先判断table是否为null，如果是则初始化table，否则根据hash算出索引位置，若位置为空则通过CAS操作将节点写入；若该位置节点hash为-1则表示数组正在扩容，则当前线程也加入进入帮助扩容；若该位置节点hash大于0，则先通过synchronized锁住该节点，然后遍历进行写入，写入完成后检查链表长度是否超过8，若超过则进行扩容或转为红黑树。

## table初始化

先判断sizeCtl是否小于0，若小于0则说明有线程正在对table进行操作，则使用Thread.yield()让出CPU；否则通过CAS将sizeCtl设置-1，初始化table，最后设置sizeCtl = table.length - (table.length >>> 2)（等价于table.length * 0.75但执行更快）

## 扩容

两种情况会触发扩容：

1. 链表长度大于8，但是table长度小于64，此时不是将链表转为红黑树，而是优先将数组扩容以缓解链表过长的问题
2. 数组中元素个数 >= 数组长度 * 0.75

第一个扩容的线程首先通过CAS设置sizeCtl为(resizeStamp(n) << 16) + 2，其中resizeStamp方法为Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1))，因为数组长度为2的整数次幂，所以numberOfLeadingZeros能反映数组的长度，1左移15位是为了使结果一定为负数。之后再每新来一个线程则通过CAS设置sizeCtl + 1。

之后每个线程都会通过CAS操作设置transferIndex以确定自己要处理的元素，transferIndex初始等于数组长度，每个线程默认的处理步长为16，即每新增一个扩容线程都会使transferIndex减16，知道小于等于0为止。之后通过synchronized锁住索引位置的根节点开始迁移，具体逻辑和HashMap类似，通过hash & oldCap == 0将节点分为索引不变和索引变化两类进行迁移。在一个位置 的节点迁移完成后会在原数组的该位置通过CAS放入一个ForwardingNode，以通知访问该位置的线程来帮助扩容。若索引位置为红黑树且迁移后长度小于等于6则会将树转为链表。

### 总结

- 在table初始化时通过CAS设置sizeCtl告诉其他线程正在初始化，使其他线程让出CPU
- 在扩容时通过CAS设置sizeCtl防止线程new出多个新数组，通过CAS设置ForwardingNode通知其他线程来一起参与扩容，通过CAS设置transferIndex防止线程处理迁移相同的数据
- 在扩容期间通过ForwardingNode使get方法有了访问新数组的能力，防止get的结果为null

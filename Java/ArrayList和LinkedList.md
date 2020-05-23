# ArrayList和LinkedList比较

## 实现接口

两者都实现了List接口，但是ArrayList多实现了RandomAccess接口，表明ArrayList支持高效的随机访问，而LinkedList多实现了Deque接口，可用来当作双端队列

## 底层数据结构

- ArrayList底层使用Object数组存储数据，因此使用索引查询数据时时间复杂度为O(1)，但是因为数组是连续，所以在插入删除元素时(直接在尾部操作除外)需要移动数组内元素，时间复杂度为O(n)。同时因为数组长度是固定的，在扩容时需要新申请一个长度为原来1.5倍的新数组，再将原数组中的所有数据全部拷贝到新数组中，时间复杂度为O(n)。综上ArrayList在随机读取时效率高，但修改及扩容效率低，常用于读多写少且长度相对固定的场景。

- LinkedList底层使用双向链表存储数据，因此插入删除时时间复杂度为O(1)，但是按索引读取时需要依次遍历元素去判断，时间复杂度为O(n)，同时链表不需要考虑扩容问题，所以LinkedList修改效率高，随机读取效率低，常用于写多读少的场景。

## 线程安全

两者都不是线程安全的，都采用了fail-fast机制，当并发修改时都回抛出ConcurrentModificationException。若想使用线程安全的List，可使用Collections.synchronizedList(原理是内部类对象加锁)或CopyOnWriteArrayList(原理是在修改时重新复制出一个数组)

> fail-fast是一种错误检测机制，即当一个容器在遍历时其结构被修改则会抛出异常。对应还有fail-safe，与fail-fast相反，fail-safe允许并发修改，但是读取的数据可能并不一定是最新的，java.util.concurrent包下的容器都采用fail-safe机制。

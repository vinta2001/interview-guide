### ConcurrentHashMap如何实现并发安全的？
在jdk1.7时使用分段的概念来实现并发安全。具体来说ConcurrentHashMap把每多个HashEntry放入到一个Segment中，然后对Segment加一个ReentrantLock锁。这里的Segment就与HasMap的概念差不多。Segment默认为16个。不同Segment间能实现写写并发，同一个Segment中能实现读写并发，但是同一个Segment中不能实现写写并发。这样虽然解决了并发安全问题，但是并发度低且不能扩展Segment的数量（一个Segment可以理解为一个并发度）。**jdk1.7解决哈希冲突的方法是使用“再哈希法”，并且jdk1.7的ConcurrentHashMap允许key为空而不允许value为空**。

![alt text](./images/java1.7的并发安全map.png)

而在jdk1.8中使用了CAS+synchronized关键字来解决并发安全问题。与HashMap一样链表也会在长度达到 8 的时候转化为红黑树，这样可以提升大量冲突时候的查询效率；以哈希槽的头结点（链表的头结点或红黑树的 root 结点）为锁，配合自旋 + CAS 避免不必要的锁开销，进一步提升并发性能。**jdk1.8的ConcurrentHashMap不允许key和value为空**。因为无法区分是获取的key为空还是获取失败为空。

具体`put`流程：
- 添加元素时首先会判断容器是否为空（在添加元素时才会初始化）
- 如果容器为空则依赖volatile加CAS来初始化table。
- 如果容器不为空，则利用 CAS 判断计算出的哈希槽的首节点是否为空。
  - 如果首节点为空，则利用 CAS 设置该节点；插入成功就推出循环，否则等待下一轮循环。
  - 如果首节点不为空，并且节点的hash值为-1，则表示有线程正在扩容，那么当前线程就加入扩容队伍帮助扩容。使用CAS和synchronied进行扩容。
  - 如果不为空，也没有扩容。那么使用synchronied插入元素，synchronied锁的是头节点，防止多线程并发修改节点元素。
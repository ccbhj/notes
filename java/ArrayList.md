# ArrayList

1. ### 如何扩容? 

   

   - 扩容时扩容为当前大小的1.5倍

   - arraylist很大时浪费空间, 同时还要进行拷贝, 所以最好在声明时指定大小.
   - 申请新数组后直接用System.arraycopy过去

2. 指定容量:

   - 指定容量后其size仍然不变, 此时如果调用set会报错, 因为大小仍为0, 所以应该调用add

   - 如果不指定容量, 默认为10
   - 如果collection大小为0, 则使用EMPTY_ELEMENTDATA静态变量代替

3. 删除操作:

   使用fastremove用System.arraycopy把删除元素两边复制到新数组(如果删除元素不是最后一个的话), 因此效率比较低, 也就不适合做队列
   
4. ArrayList的遍历和LinkedList遍历性能比较如何？

论遍历ArrayList要比LinkedList快得多，ArrayList遍历最大的优势在于内存的连续性，CPU的内部缓存结构会缓存连续的内存片段，可以大幅降低读取内存的性能开销。

5. ### Vector/ArrayList:

   vector是同步类, 每个方法都用synchronize加锁, 开销较大

6. 




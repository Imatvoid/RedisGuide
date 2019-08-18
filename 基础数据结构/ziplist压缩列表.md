## 场景

压缩列表(ziplist)是列表键和哈希键的底层实现之ー。当一个列表键只包含**少量列表项,并且每个列表项要么就是小整数值，要么就是长度比较短的字符串**，那么 Redis 就会使用压缩列表来做列表键的底层实现.

```shell
redis> RPUSH lst 1 3 5 10086 "hello" "world"  #小整数/小字符串.
(integer)6
k
redis> OBJECT ENCODING lst
"ziplist"
```

另外,当一个哈希健只包含少量键值对，比且每个键值对的键和值要么就是小整数值要么就是长度比较短的字符串,那么 Redis 就会使用压缩列表来做哈希键的底层实现。

```shell
redis>HMSET profile "name" "jack" "age" 28
ok
redis>OBJECT ENCODING profile
"ziplist"/quicklist
```



## 结构

ziplist充分体现了Redis对于存储效率的追求。一个普通的双向链表，链表中每一项都占用独立的一块内存，各项之间用地址指针（或引用）连接起来。这种方式会带来大量的内存碎片，而且地址指针也会占用额外的内存。而ziplist却是将表中每一项存放在前后连续的地址空间内，一个ziplist整体占用一大块内存。它是一个表（list），但其实不是一个链表（linked list）

从宏观上看，ziplist的内存结构如下：

![1565857820970](assets/ziplist压缩列表/1565857820970.png)


![image-20190814122553999](assets/ziplist压缩列表/image-20190814122553999.png)

**结构体**

```c
struct ziplist<T> {修改相邻节点的指针，操作简单又快速。
    int32 zlbytes; // 整个压缩列表占用字节数
    int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength; // 元素个数
    T[] entries; // 元素内容列表，挨个挨个紧凑存储
    int8 zlend; // 标志压缩列表的结束，值恒为 0xFF 255
}
```


压缩列表为了支持双向遍历，所以才会有 `ztail_offset` 这个字段，用来快速定位到最后一个元素，然后倒着遍历。



## entry节点

entry 块随着容纳的元素类型不同，也会有不一样的结构。

```c
struct entry {
    int<var> prevlen; // 前一个 entry 的字节长度
    int<var> encoding; // 元素类型编码
    optional byte[] content; // 元素内容
}
```

​		



## 连锁更新

prevlen

连锁更新在最坏情况下需要对压缩列表执行 `N` 次空间重分配操作， 而每次空间重分配的最坏复杂度为 O(N) ， 所以连锁更新的最坏复杂度为 O(N^2).

要注意的是， 尽管连锁更新的复杂度较高， 但它真正造成性能问题的几率是很低的：

- 首先， 压缩列表里要恰好有多个连续的、长度介于 `250` 字节至 `253` 字节之间的节点， 连锁更新才有可能被引发， 在实际中， 这种情况并不多见；
- 其次， 即使出现连锁更新， 但只要被更新的节点数量不多， 就不会对性能造成任何影响： 比如说， 对三五个节点进行连锁更新是绝对不会影响性能的；

因为以上原因， `ziplistPush` 等命令的平均复杂度仅为 O(N) ， 在实际中， 我们可以放心地使用这些函数， 而不必担心连锁更新会影响压缩列表的性能。







## 相关配置

hash-max-ziplist-entries 512
hash-max-ziplist-value 64

这个配置的意思是说，在如下两个条件之一满足的时候，ziplist会转成dict：

- 当hash中的数据项（即field-value对）的数目超过512的时候，也就是ziplist数据项超过1024的时候（请参考t_hash.c中的`hashTypeSet`函数）。
- 当hash中插入的任意一个value的长度超过了64的时候（请参考t_hash.c中的`hashTypeTryConversion`函数）。



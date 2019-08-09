# 列表对象





## 编码

列表对象的编码可以是 `ziplist` 或者 `linkedlist` 。





### ziplist

`ziplist` 编码的列表对象使用压缩列表作为底层实现， 每个压缩列表节点（entry）保存了一个列表元素。

```shell
redis> RPUSH numbers 1 "three" 5
(integer) 3
```

如果 `numbers` 键的值对象使用的是 `ziplist` 编码， 这个这个值对象将会是

![1565947386261](assets/list (列表对象)/1565947386261.png)



### linkedlist 

如果前面所说的 `numbers` 键创建的列表对象使用的不是 `ziplist` 编码， 而是 `linkedlist` 编码， 那么 `numbers` 键的值对象将是

![1565947423167](assets/list (列表对象)/1565947423167.png)

> `linkedlist` 编码的列表对象在底层的双端链表结构中包含了多个字符串对象， 这种嵌套字符串对象的行为在稍后介绍的哈希对象、集合对象和有序集合对象中都会出现， 字符串对象是 Redis 五种类型的对象中唯一一种会被其他四种类型对象嵌套的对象。

上面途中的第二个节点完整的表示是：

![1565947500898](assets/list (列表对象)/1565947500898.png)

## 编码转换

当列表对象可以同时满足以下两个条件时， 列表对象使用 `ziplist` 编码：

1. 列表对象保存的所有字符串元素的长度都小于 `64` 字节；

2. 列表对象保存的元素数量小于 `512` 个；

不能满足这两个条件的列表对象需要使用 `linkedlist` 编码。

以上两个条件的上限值是可以修改的， 具体请看配置文件中关于 `list-max-ziplist-value` 选项和 `list-max-ziplist-entries` 选项



列表元素大小超过64字节

```shell
# 所有元素的长度都小于 64 字节
redis> RPUSH blah "hello" "world" "again"
(integer) 3

redis> OBJECT ENCODING blah
"ziplist"

# 将一个 65 字节长的元素推入列表对象中
redis> RPUSH blah "wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww"
(integer) 4

# 编码已改变
redis> OBJECT ENCODING blah
"linkedlist"
```





列表元素数量超过512

```shell
# 列表对象包含 512 个元素
redis> EVAL "for i=1,512 do redis.call('RPUSH', KEYS[1], i) end" 1 "integers"
(nil)

redis> LLEN integers
(integer) 512

redis> OBJECT ENCODING integers
"ziplist"

# 再向列表对象推入一个新元素，使得对象保存的元素数量达到 513 个
redis> RPUSH integers 513
(integer) 513

# 编码已改变
redis> OBJECT ENCODING integers
"linkedlist"
```


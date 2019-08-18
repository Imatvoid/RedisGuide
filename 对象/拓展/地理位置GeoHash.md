# GeoHash

> 底层是zset



## GeoHash算法





## 使用

![1566134199455](assets/Untitled/1566134199455.png)



### 添加

geoadd 指令携带集合名称以及多个经纬度名称三元组，注意这里可以加入多个三元组

```shell
127.0.0.1:6379> geoadd company 116.48105 39.996794 juejin
(integer) 1
127.0.0.1:6379> geoadd company 116.514203 39.905409 ireader
(integer) 1
127.0.0.1:6379> geoadd company 116.489033 40.007669 meituan
(integer) 1
127.0.0.1:6379> geoadd company 116.562108 39.787602 jd 116.334255 40.027400 xiaomi
(integer) 2
```

### 计算距离

geodist 指令可以用来计算两个元素之间的距离

```shell
127.0.0.1:6379> geodist company juejin ireader km
"10.5501"
127.0.0.1:6379> geodist company juejin meituan km
"1.3878"
127.0.0.1:6379> geodist company juejin jd km
"24.2739"
127.0.0.1:6379> geodist company juejin xiaomi km
"12.9606"
127.0.0.1:6379> geodist company juejin juejin km
"0.0000"

```

### **获取元素位置**

geopos 指令可以获取集合中任意元素的经纬度坐标，可以一次获取多个。

```shell
127.0.0.1:6379> geopos company juejin
1) 1) "116.48104995489120483"
   2) "39.99679348858259686"
```


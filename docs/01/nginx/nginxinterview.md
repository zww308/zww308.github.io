# 1、nginx常用负载均衡算法
    平滑加权轮询

    IP哈希（IP hash）

    一致性哈希（consistant hash）

    最少连接（least_conn）

# 2、hash与一致性hash区别

**hash存储信息**：比如我们将一张图片存放在后台3个服务器上。我们先将图片的信息进行hash运算，得到的哈希摘要对3取模得到的一定是0，1，2。那么就均匀的分布在这三台服务器上。

问题：如果后台服务器挂掉，或者添加服务器时，此前缓存的数据位置就会发生变化，这样会造成大量的缓存在同一时间失效，造成缓存的雪崩。

分析问题：上述描述的问题时由于HASH算法本身的缘故，这些情况是无法避免的，为解决这些问题，一致性哈希算法就诞生了。

**一致性hash：** 将2^32个点组成的圆环称为hash环。**hash（服务器A的IP地址） %  2^32**，将服务器地址分布在环上，将图片的hash计算后对2^32取模，即使不在服务器上，它也会按照顺时针方向，存储在第一个服务器上。

这样一个服务器宕机后，按这个规则存储在顺时针的第一个服务器上。

好处：使用一致性哈希算法时，服务器的数量如果发生改变，并不是所有缓存都会失效，而是只有部分缓存会失效，环上的其他服务器的缓存仍然能分担整个系统的压力，而不至于所有压力都在同一时间集中到后端服务器上。

问题：hash偏斜，可能服务器打在环上的位置会偏向其他某个地方，我们可以设置虚拟节点来解决。这样虽然没有多余的真正的物理服务器节点，但是我们将现有的物理节点通过虚拟的方法复制出来，便可实现hash相对的均衡。

参考：https://www.zsythink.net/archives/1182/



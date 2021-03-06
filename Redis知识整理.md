#### Redis的复制(Master/Slave)

##### Redis复制原理

slave启动成功连接到master后会发送一个sync命令，Master接到命令启动后台的存盘进程，同时收集所有接收到的用于修改数据集命令，在后台进程执行完毕之后，master将传送整个数据文件到slave,以完成一次完全同步。全量复制：而slave服务在接收到数据库文件数据后，将其存盘并加载到内存中。增量复制：Master继续将新的所有收集到的修改命令依次传给slave,完成同步。但是只要是重新连接master,一次完全同步（全量复制)将被自动执行

##### Redis哨兵模式

能够后台监控主机是否故障，如果故障了根据投票数自动将从库转换为主库。哨兵模式是一种特殊的模式，首先Redis提供了哨兵的命令，哨兵是一个独立的进程，作为进程，它会独立运行。其原理是哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例。

**如果之前的master重启回来，会不会双master冲突？**

​     如果之前的master回来，会成为新master的从机。

![](C:%5CUsers%5CAdministrator%5CDesktop%5C123.jpg)

##### Redis复制的缺点

复制延迟：由于所有的写操作都是先在Master上操作，然后同步更新到Slave上，所以从Master同步到Slave机器有一定的延迟，当系统很繁忙的时候，延迟问题会更加严重，Slave机器数量的增加也会使这个问题更加严重。



#### Redis的持久化

##### RDB（Redis DataBase）

Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到
一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。
整个过程中，主进程是不进行任何IO操作的，这就确保了极高的性能
如果需要进行大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方
式要比AOF方式更加的高效。RDB的缺点是最后一次持久化后的数据可能丢失。

##### AOF（Append Only File）

以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下来(读操作不记录)，
只许追加文件但不可以改写文件，redis启动之初会读取该文件重新构建数据，换言之，redis
重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作

**重写原理**:AOF文件持续增长而过大时，会fork出一条新进程来将文件重写(也是先写临时文件最后再rename)，遍历新进程的内存中数据，每条记录有一条的Set语句。重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件，这点和快照有点类似
**触发机制:**Redis会记录上次重写时的AOF大小，默认配置是当AOF文件大小是上次rewrite后大小的一倍且文件大于64M时触发

##### 如何选择持久化方式

只做缓存：如果你只希望你的数据在服务器运行的时候存在,你也可以不使用任何持久化方式.

RDB持久化方式能够在指定的时间间隔能对你的数据进行快照存储，AOF持久化方式记录每次对服务器写的操作,当服务器重启的时候会重新执行这些。命令来恢复原始的数据,AOF命令以redis协议追加保存每次写的操作到文件末尾.Redis还能对AOF文件进行后台重写,使得AOF文件的体积不至于过大。

同时开启两种持久化方式：RDB的数据不实时，同时使用两者时服务器重启也只会找AOF文件。那要不要只使用AOF呢？
作者建议不要，因为RDB更适合用于备份数据库(AOF在不断变化不好备份)，
快速重启，而且不会有AOF可能潜在的bug，留着作为一个万一的手段。

##### 性能建议

因为RDB文件只用作后备用途，建议只在Slave上持久化RDB文件，而且只要15分钟备份一次就够了，只保留save 900 1这条规则。


如果Enalbe AOF，好处是在最恶劣情况下也只会丢失不超过两秒数据，启动脚本较简单只load自己的AOF文件就可以了。代价一是带来了持续的IO，二是AOF rewrite的最后将rewrite过程中产生的新数据写到新文件造成的阻塞几乎是不可避免的。只要硬盘许可，应该尽量减少AOF rewrite的频率，AOF重写的基础大小默认值64M太小了，可以设到5G以上。默认超过原大小100%大小时重写可以改到适当的数值。

如果不Enable AOF ，仅靠Master-Slave Replication 实现高可用性也可以。能省掉一大笔IO也减少了rewrite时带来的系统波动。代价是如果Master/Slave同时倒掉，会丢失十几分钟的数据，启动脚本也要比较两个Master/Slave中的RDB文件，载入较新的那个。新浪微博就选用了这种架构



#### 过期策略

redis是怎么对设置了过期时间key进行删除的？

定期删除加惰性删除，何为定期删除：就是redis默认每隔100ms会随机抽取某些key进行检查和删除，定期删除可能会导致的问题是很多过期key到时间并没有被删除，这时候就得靠惰性删除了。  何为惰性删除：就是当你获取key的时候，redis会检查这个key是否设置了过期时间且是否过期了？，如果过期则会删除这个key，不会给你返回任何东西。所以设置了过期时间的key并不是到时间了就被删除，而是当你get这个key时候，redis在惰性检查一下。通过定期删除和惰性删除，保证过期时间的key被删除。但是，使用定期删除和惰性删除策略还是会存在如果定期删除漏掉了很多过期key，且也没有及时去查，也就是过期的key既没有走定期删除又没有走惰性删除，这会导致大量过期key堆积在redis内存里，使得redis的内存耗尽，此时redis会启用redis的内存淘汰机制。

#### 内存淘汰机制

noeviction：内存不够，新写入操作报错

allkeys-lru: 内存不够，在键空间，移除最近最少使用的key

allkeys-random:内存不够，在键空间随机删除一个key

volatile-lru：内存不够，在设置过期时间的键空间中删除最近最少用的key

volatile-random:内存不够，在设置过期时间的键空间中随机移除某个key

volatile-ttl:内存不够，在设置过期时间的键空间中，将有更早过期时间的key优先移除



如果redis内存占用过多时，redis会进行内存淘汰

noeviction：当内存不足以容纳新写入数据时，新写入操作会报错，这个一般没人用吧

allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key（这个是最常用的）

allkeys-random：当内存不足以容纳新写入数据时，在键空间中，随机移除某个key，这个一般没人用吧

volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的key（这个一般不太合适）

volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个key

volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的key优先移除

#### 为什么Redis一开始使用单线程

1.I/O多路复用   2.可维护性高    3.基于内存，单线程状态下效率依然高

#### 为什么又引入多线程

我们知道Redis可以使用del命令删除一个元素，如果这个元素非常大，可能占据了几十兆或者是几百兆，那么在短时间内是不能完成的，这样一来就需要多线程的异步支持。

Redis 选择使用单线程模型处理客户端的请求主要还是因为 CPU ，不是 Redis 服务器的瓶颈，所以使用多线程模型带来的性能提升并不能抵消它带来的开发成本和维护成本，系统的性能瓶颈也主要在网络 I/O 操作上；而 Redis 引入多线程操作也是出于性能上的考虑，对于一些大键值对的删除操作，通过多线程非阻塞地释放内存空间也能减少对 Redis 主线程阻塞的时间，提高执行的效率。

**之前用单线程是因为基于内存速度快，而且多路复用有多路复用的作用，也就是足够了，现在引入是因为在某些操作要优化，比如删除操作，因此引入了多线程。**



#### 缓存穿透

用户想要查询一个数据，发现redis中没有，也就是缓存没有命中，于是向数据库查询。发现也没有，于是本次查询失败。当用户很多的时候，缓存都没有命中，于是都去请求了持久层数据库。这会给持久层数据库造成很大的压力，这时候就相当于出现了缓存穿透。

**避免缓存穿透的解决方案**

布隆过滤器是如何解决redis中的缓存穿透呢？首先对所有可能查询的参数以hash形式存储，当用户想要查询的时候，使用布隆过滤器发现不在集合中，就直接丢弃，不再对持久层查询。

![223](C:%5CUsers%5CAdministrator%5CDesktop%5C223.jpg)

当存储层不命中后，即使返回的空对象也将其缓存起来，同时会设置一个过期时间，之后再访问这个数据将会从缓存中获取，保护了后端数据源。

![2334](C:%5CUsers%5CAdministrator%5CDesktop%5C2334.jpg)

但是这种方法会存在两个问题：

如果空值能够被缓存起来，这就意味着缓存需要更多的空间存储更多的键，因为这当中可能会有很多的空值的键；即使对空值设置了过期时间，还是会存在缓存层和存储层的数据会有一段时间窗口的不一致，这对于需要保持一致性的业务会有影响

#### 缓存击穿

缓存击穿，是指一个key非常热点，在不停的扛着大并发，大并发集中对这一个点进行访问，当这个key在失效的瞬间，持续的大并发就穿破缓存，直接请求数据库，就像在一个屏障上凿开了一个洞。



#### 缓存雪崩

缓存雪崩是指，缓存层出现了错误，不能正常工作了。于是所有的请求都会达到存储层，存储层的调用量会暴增，造成存储层也会挂掉的情况

![23445](C:%5CUsers%5CAdministrator%5CDesktop%5C23445.jpg)

解决方案：

**（1）redis高可用**

既然redis有可能挂掉，那我多增设几台redis，这样一台挂掉之后其他的还可以继续工作，其实就是搭建的集群。

**（2）限流降级**

这个解决方案的思想是，在缓存失效后，通过加锁或者队列来控制读数据库写缓存的线程数量。比如对某个key只允许一个线程查询数据和写缓存，其他线程等待。

**（3）数据预热**

数据加热的含义就是在正式部署之前，我先把可能的数据先预先访问一遍，这样部分可能大量访问的数据就会加载到缓存中。在即将发生大并发访问前手动触发加载缓存不同的key，设置不同的过期时间，让缓存失效的时间点尽量均匀。


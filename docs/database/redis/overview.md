# 缓存技术

缓存的本质，系统各级处理速度不匹配，导致利用<font color=red>空间换时间</font>。缓存是提升系统性能的一个简单有效的办法。



## 数据分类和使用频率

数据分类

- 静态数据：一般不变，类似于字典表 

- 准静态数据：变化频率很低，部门结构设置，全国行政区划数据等 

- 中间状态数据：一些计算的可复用中间数据，变量副本，配置中心的本地副本 



这些数据适合于使用缓存的方式访问 

- 热数据：使用频率高
- 读写比较大：读的频率 >> 写的频率

广义上来说，为了加速数据处理，让业务更快的访问**临时存放冗余数据**，都是缓存。狭义上，现在我们一般在分布式系统里把缓存到内存的数据叫做内存缓存。



## 缓存加载时机

- 启动全量加载

  全局有效，使用简单 

- 懒加载

  - 同步使用加载
    - 先看缓存是否有数据，有的话直接返回，没有的话从数据库读取 
    - 读取的数据，先放到内存，然后返回给调用方 
  - 延迟异步加载
    - 从缓存获取数据，不管是否为空直接返回
    - 策略一（异步）：如果为空，则主动发起一个异步加载的线程，负责加载数据 
    - 策略二（解耦）：后台异步线程负责维护缓存的数据，定期或根据条件触发更新



## 缓存的有效性

- 读写比

  对数据的**写操作**导致数据变动，意味着维护成本。N : 1

- 命中率

  命中缓存意味着缓存数据被使用，意味着有价值。90%+



## 缓存使用不当导致的问题

- 系统预热导致启动慢

  试想一下，一个系统启动需要预热半个小时。 导致系统不能做到快速应对故障宕机等问题。 

- 系统内存资源耗尽 

  只加入数据，不能清理旧数据。旧数据处理不及时，或者不能有效识别无用数据。




## 缓存常见问题

- 缓存穿透（恶意，数据源数据不存在）

  大量并发查询不存在的key，导致直接将压力透传到数据库

  解决办法

  1. 缓存空值的key
  2. Bloom过滤或RoaringBitmap判断key是否存在
  3. 完全以缓存为准，使用延迟异步加载策略

- 缓存击穿（偶然，数据源数据存在，但缓存过期或失效）

  某个key失效的时候，正好有大量并发请求访问这个key

  解决办法

  1. key的更新操作添加全局互斥锁（加锁——setnx，成功，load db然后设置缓存；不成功，重试）
  2. 完全以缓存为准，使用延迟异步加载策略

- 缓存雪崩

  当某一时刻发生大规模缓存失效的情况，会有大量的请求直接打到数据库，导致数据库压力过大甚至宕机。

  - 更新策略
  - 随机过期策略

- 数据热点

  - 更新策略在时间上做到比较均匀

  - 使用的热数据尽量分散到不同的机器上
  - 多台机器做主从复制或多副本，实现高可用
  - 实现熔断限流机制，对系统进行负载能力控制



## 缓存读写策略

- Cache Aside（Redis）

  - 读

    从缓存中读，读到直接返回；读不到，查数据库，更新缓存

  - 写

    更新数据库，删除缓存

  **适用于读到写少的场景；缺点：1、首次读缓存肯定不存在，注意热点数据问题；2、频繁写，缓存命中率低，数据库压力**

- Read/Write Through（缓存服务）

  - 读

    从缓存服务中读，读到直接返回；读不到，缓存服务查数据库，更新缓存服务，再返回

  - 写

    更新缓存服务，缓存服务<font color=red>同步</font>更新数据库，成功后返回

  **注意热点数据预热。**

- Write Behind（缓存服务）

  - 读

    从缓存服务中读，读到直接返回；读不到，缓存服务查数据库，更新缓存服务，再返回

  - 写

    更新缓存服务，直接返回，缓存服务<font color=red>异步批量</font>更新数据库

  **考虑没有更新数据库，缓存服务宕机导致缓存数据丢失。写性能高，数据一致性要求低的场景。**




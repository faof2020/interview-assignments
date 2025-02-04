整个系统的设计思路和算法原理如下：
1. 短链接编码采用自增的8位62进制数生成
2. 短链接的生成方法是：每当查询不到长链接时，全局唯一的长整型自增1，然后转化成8位62进制数
3. long->short的映射关系保存在LRU缓存中，可以设置缓存大小
4. 由于2中的机制，失效的长链接在查询时视为第一次出现，因此有可能出现同一个长链接对应多个短链接的问题
5. long->short映射实际在内存中的数据类型是String->long，只保存短链接对应的自增10进制id，查询的时候再转成62进制
6. short->long的映射关系保存在数组中(事实上是ArrayList<String[]>)。为了避免Java数组最大长度的限制，同时为了动态申请内存空间，做出了这样的设计。
7. 根据短链接的8位62进制数，可以直接算出对应数组中的下标，所对应的String就是长链接
8. 每次申请的数组空间和最大短链接个数可以手动设置

系统设计的出发点和假设：
1. 长链接对应多个短链接(出现了重复)，是可以容忍的，而短链接的失效会给用户带来困扰。因此应该把更多的存储空间(内存)分配给short->long的映射
2. 热门的url更有可能出现在最近的请求中，所以采用LRU缓存可以一定程度上避免重复生成短链接
3. 短链接并不需要加密，所以自增id是更简洁，效率更高的方法,同时也避免了碰撞
4. 单机可能存不下62^8个链接映射，因此更好的方法应该是采用2+6位短链接的方式，前面2位从能够保证分布式一致性的服务商获取(比如radis),后6位服务器自己生成，可以通过集群和load balance实现高并发
5. LRU的大小应该取决于长链接的稀疏程度，如果长链接很少重复，那么缓存大小应该尽量小，反之应该加大缓存
6. 保存短链接的数据结构可以根据机器的实际内存大小，调整每次申请的数据大小和总数上限

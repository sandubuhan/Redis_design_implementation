# Redis设计与实现

## 简单动态字符串（String）

+ Redis构建了一种名为简单动态字符串（SDS）的抽象类型，并将SDS用作Redis的默认字符串表示
+ 当Redis需要的不仅是一个字符串字面量，而是一个可以被修改的字符串值时，就会使用SDS来表示字符串值。
+ 比如

~~~shell
redis> RPUSH fruits "apple" "banana"
~~~

+ Redis将在数据库中创建一个新的键值对，其中：
    + 键值对的键是一个字符串对象，底层实现是一个保存了字符串“fruits”的SDS
    + 值是一个列表对象，包含了两个字符串对象。这两个对象分别由两个SDS实现。
+ 除此外，SDS还被用作缓冲区：AOF模块中的AOF缓冲区，以及客户端状态中的输入缓冲区

###  SDS的定义

![image-20220614213514317](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142135641.png)

+ free：表示这个SDS未分配的空间
+ len：表示保存的空间
+ buf：是一个char类型的数组，最后一个字节保存了空字符“\0"
+ 保存空字符的1字节空间不计算在SDS的len属性里，并且为空字符分配额外的1字节空间

### SDS与字符串的区别

+ C语言顺颂的字符串表达方式，并不能满足Redis对字符串在安全性、效率以及功能方面的要求

### 常数复杂度获取字符串长度

+ 因为有len属性， 所以获取一个SDS长度的复杂度仅为O(1)。设置和更新SDS长度的工作是由SDS的API在执行时自动完成的，使用SDS无需进行任何手动修改长度的工作

### 杜绝缓冲区溢出

+ 除了获取字符串长度的复杂度高之外，C 字符串不记录自身长度带来的另一个问题就是容易造成缓冲区溢出。
+ 而当SDS的API需要对SDS进行修改时，API会先检查SDS的空间是否满足修改所需的要求，如果不满足的话，API会自动将SDS的空间扩展至执行修改所需的大小，然后才执行实际的修改操作，所以使用SDS既不需要手动修改SDS的空间大小，也不会出现前面所说的缓冲区溢出

### 减少修改字符串时带来的内存重分配次数

+ 因为不记录自身长度，所以C字符串在增长或者缩短一个C字符串时，程序都要到对保存这个C字符串的数字进行一次内存重分配操作
+ SDS通过未使用空间接触了字符串长度和底层数组长度之间的关联：在SDS中，buf数组的长度不一定就是字符数量加一，数组里面可以包含未使用的字节，而这些字节的数量就由SDS的free属性记录
+ 通过未使用空间，SDS实现了空间预分配和惰性空间释放两种优化策略。
    + 空间预分配：
    + 当SDS的API对一个SDS进行修改，并且需要对SDS进行空间扩展的时候，程序不仅会对SDS分配修改所需要的空间，还会为SDS分配额外的未使用空间
    + 如果对SDS进行修改之后，SDS的长度（len）将小于1MB，那么程序分配和len属性同样大小的未使用空间，这时SDS的len属性的值将和free的值相同
    + 如果对SDS进行修改之后，SDS的长度将大于等于1MB，那么程序会分配1MB的未使用空间。
    + 通过空间预分配策略，Redis可以减少连续执行字符串增长操作所需的内存重分配次数
    + 在扩展SDS空间之前，SDS的API会先检查未使用空间是否足够，如果足够的话，API就会直接使用未使用空间，而无需执行内存重分配
    + 使得内存重分配次数从必定N次变成最多N次
    + 惰性空间释放：
    + 当SDS的API需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性将这批字节的数量记录下来，并等待将来使用
    + 与此同时，SDS也提供了相应的API，让我们可以在有需要的时候，真正的释放SDS未使用的空间，所以不用担心惰性空间释放策略会造成内存浪费

### 二进制安全

+ 因为C字符串中的字符必须符合某种编码（ASCII），并且除了字符串的末位之外，字符串里不能包含空字符，所以C字符串只能保存文本数据，不能保存图片、音视频、压缩文件等二进制数据。
+ 这种二进制数据大多会包含空字符"\0"
+ SDS的API都是二进制安全的：数据在写入时是什么样，被读取时就是什么样

### 兼容部分C字符串函数

### 总结

![image-20220614213536195](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142135655.png)



## 链表（List）

+ Redis使用的C语言并没有内置这种数据结构，所以Redis构建了自己的链表实现
+ 列表建的底层实现之一就是链表。当一个列表建包含了数量比较多的元素，又或者列表中包含的元素都是比较长的字符串时，Redis就会使用链表作为列表键的底层实现
+ 除了链表键之外，发布订阅、慢查询、监视器等功能也用到了链表，Redis服务器本身还是用链表来保存多个客户端的状态信息，以及使用链表来构建客户端输出缓冲区

### 链表和链表节点的实现

+ 链表节点：

![image-20220614001458113](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206140014396.png)

+ 多个listNode通过prev和next指针组成双端链表

+ 链表：

![image-20220614213602068](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142136934.png)

+ 通过list来操作链表
    + 表头指针：head，表尾指针：tail，链表长度计数器：len
    + dup：用于复制链表节点所保存的值
    + free：释放链表节点所保存的值
    + match：对比链表节点所保存的值和另一个输入值是否相等
+ Redis的链表实现的特性：
    + 双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置借点的复杂度都是O(1)
    + 无环：表头节点的prev指针和表尾节点的next指针都指向null，对链表的访问以Null为终点
    + 带表头指针和表尾指针：通过list结构的head和tail，程序获取表头表尾的复杂度为O(1)
    + 带链表长度计数器：程序使用list结构的len属性来对list持有的链表节点进行计数，获取链表节点数量的复杂度为O(1)
    + 多态：链表节点使用void*指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存不同类型的值



## 字典（Hash）

~~~shell
127.0.0.1:6379> hset user name1 hao
(integer) 1
127.0.0.1:6379> hset user email1 hao@163.com
(integer) 1
127.0.0.1:6379> hgetall user
1) "name1"
2) "hao"
3) "email1"
4) "hao@163.com"
127.0.0.1:6379> hget user user
(nil)
127.0.0.1:6379> hget user name1
"hao"
127.0.0.1:6379> hset user name2 xiaohao
(integer) 1
127.0.0.1:6379> hset user email2 xiaohao@163.com
(integer) 1
127.0.0.1:6379> hgetall user
1) "name1"
2) "hao"
3) "email1"
4) "hao@163.com"
5) "name2"
6) "xiaohao"
7) "email2"
8) "xiaohao@163.com"
~~~

+ 又称为符号表、关联数组、映射，是一种用于保存键值对的抽象数据结构
+ 字典中的每个键都是独一无二（key不能重复，value可以重复）
+ C语言没有内置字典，所以Redis构建了自己的字典实现
+ Redis的数据库就是使用字典作为底层实现，对数据库的增删改查操作也是构建在字典的操作之上的

~~~shell
redis> set msg "hello"
ok
~~~

+ 上述命令，，这个键值对就是保存在代表数据库的字典里面的
+ 字典还是哈希键的底层实现之一，当一个哈希键包含的键值对比较多，又或者键值对中的元素都是比较长的字符串时，Redis就会使用字典作为哈希键的底层实现

### 字典的实现

+ Redis的字典使用哈希表作为底层实现，一个哈希表里面可以由多个哈希表节点，每个哈希表节点就保存了字典中的一个键值对

#### 哈希表

![image-20220614214308013](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142143064.png)

+ table属性是一个数组，数组中的每个元素都是一个指向dictEntry结构的指针，每个dictEntry结构保存着一个键值对
+ size属性记录哈希表大小，也就是table数组的大小
+ sizemask的值总是等于size-1，它和哈希值一起决定一个键被放到table数组的哪个索引上面

#### 哈希表节点

+ 哈希表节点使用dictEntry结构表示

![image-20220614214622717](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142146794.png)

+ next属性是指向另一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对连接在一起，解决键冲突（头插法）

![image-20220614214731826](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142147917.png)

####　字典

![image-20220614214801350](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142148449.png)

+ type属性是一个指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型特定函数

![image-20220614214918544](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142149644.png)

+ ht属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表，一般情况下，字典只是用ht[0]哈希表，ht[1]只会在对ht[0]进行rehash时使用
+ rehashid用于记录目前rehash的进度，如果没有rehash，被标记为-1

![image-20220614215101075](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142151177.png)

### 哈希算法

+ 当要将一个新的键值对添加到字典里面，程序需要先根据键值对的键计算出哈希值和索引值，再根据所印制，将包含新键值对的哈希表节点放到哈希表数组的指定索引上面

~~~shell
hash = dict->type->hashFunction(key);
index = hash & dict->ht[x].sizemask;
~~~

+ Redis使用MurmurHash2算法计算键的哈希值

### 解决键冲突

+ 当有两个或以上的键被分配到了哈希表数组的同一个索引上面时，就称为冲突
+ Redis的哈希表使用链地址法解决冲突，每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个车单向链表，被分配到同一个索引上的多个节点可以被连接起来

### rehash

+ 哈希表保存的键值对会随着操作的执行而增多或减少，为了让哈希表的负载因子（[0.75](https://stackoverflow.com/questions/7115445/what-is-the-optimal-capacity-and-load-factor-for-a-fixed-size-hashmap)）维持在一个合理的范围内，当哈希表保存的键值对数量过多或过少，程序会对哈希表的大小进行扩展或者收缩

+ 步骤：
    + 为字典的ht[1]哈希表分配空间
        + 扩展：ht[1]的大小为第一个大于等于ht[0].used*2的"2的n次方幂"
        + 收缩：ht[1]大小为第一个车大于等于ht[0].used的"2的n次方幂"
    + 将保存在ht[0]中的所有键值对rehash到ht[1]上，rehash指重新计算键的哈希值和索引值
    + 当所有的键值对都迁移后，释放ht[0],将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表

### 哈希表的扩展收缩

+ 程序会自动对哈希表扩展：
    + 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令时，并且哈希表的负载因子大于等于1
    + 正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5
+ 公式

~~~shell
# 负载因子 = 哈希表已保存节点数量 / 哈希表大小
load_factor = ht[0].used / ht[0].size
# 已保存节点数量≠桶的数量（数组的大小）
~~~

+ 在执行BGSAVE或者BGREWRITEAOF命令时，Redis需要创建当前服务器进程的子进程，而大多数操作系统都采用写时复制来优化紫禁城的使用效率，所以在子进程存在期间，服务器会提高执行扩展操作所需的负载因子。从而尽可能避免在子进程存在期间进行哈希表扩展操作，可以避免不必要的内存写入操作，最大限度的节约内存
+ 另一方面，当哈希表的负载因子小于0.1时，程序自动开始对哈希表收缩

### 渐进式rehash

+ rehash的动作并不是一次性，而是分多次、渐进式的完成。因为避免庞大的键值对在计算时会对服务器造成影响
+ 在字典中维持索引计数器变量rehashidx，并设置为0，表示rehash工作开始
+ 在rehash期间，每次对字典执行增删改查，程序除了执行指定的操作以外，还会将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]。当rehash 完成后，rehashidx+1
+ 都完成后，rehashidx == -1
+ 渐进式执行期间，新添加到字典的键值对会保存到ht[1]



##  跳跃表 （Zset）

~~~shell
127.0.0.1:6379> zadd myscoreset 100 hao 90 xiaohao
(integer) 2
127.0.0.1:6379> ZRANGE myscoreset 0 -1
1) "xiaohao"
2) "hao"
127.0.0.1:6379> ZSCORE myscoreset hao
"100"
~~~

+ 跳跃表是一种有序数据结构，支持平均O(logN)，最坏O(N)复杂度的节点查找
+ Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员是比较长的字符串时，Redis就会使用跳跃表作为有序集合键的底层实现
+ Redis只在两个地方用到了跳跃表，一个是实现有序集合键，一个是集群节点中用作内部数据结构

### 跳跃表的实现

+ 由zskiplistNode和zskiplist两个结构定义。前者表示跳跃表节点，后者用于保存跳跃表节点的相关信息，比如数量，以及指向表头和表尾节点的指针

![image-20220614234821152](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206142348270.png)

+ zskiplist:
    + level：记录目前跳跃表内，层数最大的节点的层数（表头节点不计算在内）
    + length：记录跳跃表的长度，也就是跳跃表目前包含节点的数量（表头节点不包含）
+ zskiplist:
    + 层level：节点重点用L1、L2等标记节点的各个层，代表第一二层等。每个层都有两个属性：前进指针和跨度。前进指针用于访问位于**表尾方向**的其他节点，跨度是记录了前进指针所指向节点和当前节点的距离。上图，连线上带有数字的箭头就代表前进指针，数字就是跨度。当程序从头到尾遍历时，访问会沿着层的前进指针进行
    + 后退指针bw：指向位于当前节点的前一个节点。在从尾到头遍历时使用
    + 分值score：节点按各自保存的分值从小到大排列
    + 成员对象obj：
+ 表头节点和其他节点的构造是一样的

### 跳跃表节点

1. 层
    + 跳跃表节点的level数组可以包含多个元素，每个元素都包含一个指向其他节点的指针，程序可以通过这些层来加快访问其他节点的速度，一般来说，层的数量越多，访问其他节点的速度就越快
    + 每次创建一个新跳跃表节点，程序都根据幂次定律（越大的数出现的概率越小）随机生成一个介于1到32之间的值作为level的大小，就是层的高度

2. 前进指针
    + 每层都有一个指向表尾方向的前进指针

3. 跨度

    + 层的跨度用于记录两个节点之间的距离：

    + 指向null的所有前进指针的跨度都为0，因为没有联想任何节点

+ 跨度和遍历操作并没有关系。遍历操作只使用前进指针，跨度实际上使用来计算排位的：在查找某个节点的过程中，将沿途访问过的所有层的跨度累计起来，得到的结果就是目标节点在跳跃表中的排位
+ 例子

![image-20220615000956953](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206150009095.png)

+ 虚线标记了查找3.0的O3对象节点，沿途经历的层：1，层的跨度为3。所以目标节点在跳跃表中排位为3

![image-20220615001049877](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206150010983.png)

+ 查找2.0的O2对象，程序经过了两个跨度为1的节点，所以目标节点在跳跃表中的排位为2

4. 后退指针
    1. 用于从尾到头访问节点，每个节点只有一个后退指针，每次必须后退到前一个节点
5. 分值和成员
    + 跳跃表中的所有节点都按照分值从小到大排序
    + 成员对象是一个指针，指向一个字符串对象，这个对象就是SDS
    + 在同一个表中，每个节点保存的成员对象必须为宜，但是多个节点保存的分值却可以相同：**分值相同的节点按照成员对象在字典序中的大小进行排序**

### 跳跃表

+ 虽然靠多个跳跃表节点就可以组成一个跳跃表
+ 但是通过使用一个zskiplist结构来持有这些节点，程序可以更方便的对整个跳跃表进行处理，比如快速访问表头或者表尾，火哥快速获取跳跃表节点的数量



## 整数集合 （Set）

+ 整数集合是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现

~~~shell
redis> SADD numbers  1 3 5 7 9 
(integer) 5
redis> OBJECT ENCODING  numbers
"intset"
~~~

### 整数集合的实现

+ 整数集合（intset）是Redis用于保存整数值的集合抽象数据结构，可以保存类型为int16_t,int32_t,int64_t的整数值，并且保证集合中不出现重复元素

![image-20220616221737753](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206162217930.png)

+ contents数组是整数集合的底层实现：整数集合的每个元素都是contents数组的一个数组项，各个项在数组中按值的大小从小到大有序的排列，并且数组中不包含重复项
+ 虽然intest结构将contents属性声明为int8_t类型的数组，但实际上contents数组并不保存任何int8_t类型的值，contents数组的真正类型取决于encoding属性的值

![image-20220616222024770](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206162220874.png)

+ encoding属性表示整数集合的底层实现为int64_t类型的数组，而数组中保存的都是int64_t类型的整数值
+ length表示包含四个元素
+ contents数组从小到大顺序保存
+ 因为每个集合元素都是int64_t类型的整数值，所以contents数组的大小为64*4=256位
+ 根据整数集合的升级规则，当向一个底层为int16_t数组的证书集合添加一个int64_t类型的整数值时，整数集合已有的所有元素都会被转换成int64_t类型，所以contents数组保存的四个整数值都是int64_t类型的

### 升级

+ 当将一个新元素添加到整数集合里面，并且新元素的类型比整数集合现有所有元素的类型都要长时，整数集合需要先进行升级，然后才能将新元素添加到整数集合里面
+ [三步](https://juejin.cn/post/6844904198019137550)：
    + 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间
    + 将底层数组现有的所有元素都转换成新元素相同的类型，并将类型转换后的元素房知道正确的位上，在放置的过程中，需要继续维持底层数组的有序性质不变
    + 将新元素添加到底层数组里面
+ 想整数集合添加新元素的时间复杂度为O(N)
+ 因为引发升级的新元素的长度总是比整数集合现有所有元素的长度都大，所以这个新元素的值要么大于所有现有元素，要么小于所有现有元素
    + 小于的情况下，新元素会被放置在底层数组的最开头
    + 大于时，会被放在最末尾(索引length-1)

### 升级的好处

+ 提升灵活性
+ 节约内存

### 降级

+ 整数集合不支持降级，一旦对数组进行了升级，编码就会一直保持升级后的状态。



## 压缩列表

+ 列表键和哈希键的底层实现之一。当一个列表建质保函少量列表项，并且没个列表项要么就是小整数值，要么就是长度比较短的字符串，REdis就会使用压缩列表作为列表键的底层实现
+ 当一个哈希键只包含少量键值对，且每个键值对的键和值要么是小整数值或者长度比较短的字符串，Redis会使用压缩列表作为列表键的底层实现

### 压缩列表的构成

+ 压缩是Redis为了节约内存而开发的，是由一系列特殊编码的连续内存块组成的顺序性数据结构。一个压缩列表可以包含任意多个节点，每个节点可以保存一个字节数组或者一个整数值

![image-20220618225109369](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206182251545.png)

+ zlbytes：记录整个压缩列表占用的内存字节数，
+ zltail：记录压缩列表表尾节点距离压缩列表的起始地址有多少个字节，通过此偏移量，程序无须遍历这个压缩列表就可以确定表尾节点地址
+ zllen：记录压缩列表包含的节点数量
+ entryX：压缩列表包含的各个节点
+ zlend：特殊值0xFF，用于标记压缩列表的末端

### 压缩列表节点的构成

+ 每个压缩列表节点可以保存一个字节数组或者一个整数值
+ 都由previous_entry_length,encoding,content三个部分组成

![image-20220618225417431](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206182254529.png)

### previous_entry_length

+ 以字节为单位，记录了压缩列表中前一个节点的长度，此属性的长度可以是1字节或者5字节

+ 如果前一节点的长度小于254字节，那么此属性长度为1字节
+ 如果大于等于254字节，此属性长度为5字节。其中第一个字节会被设置为0xFE，之后的四个字节用于保存前一节点的长度

![image-20220618225635511](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206182256615.png)

+ 图示，属性的值为0xFE00002766，最高位字节0xFE表示这是5字节长的previous_entry_length属性，之后的4字节0x00002766才是前一节点的实际长度（10086）
+ 程序可以通过指针运算，根据当前节点的起始地址计算出前一个节点的起始地址

### encoding

+ 此属性记录了节点的content属性所保存的数据类型及长度
    + 1/2/5字节长，值的最高位为00/01/10的是字节数组编码
    + 1字节长，值的最高位是11开头的是整数编码

### content

+ 节点的content 属性负责保存节点的值

![image-20220618230459753](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206182304885.png)

### 连锁更新

+ 节点的平衡被打破，导致其他节点都需要进行变化。
+ 新增或者删除节点时，可能出发连锁更新。
+ 以删除为例，删除节点的前置节点称为cur，后置节点称为next。cur节点长度超过254，next的“前置节点长度”空间不足以存储254时，需要对next的“前置节点长度”进行扩容，如果next节点扩容后的长度刚好超过254，就会导致next的后置节点也需要对其“前置节点长度”空间扩容，导致连环更新“前置节点长度”空间

![](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206182308354.png)

+ 连锁更新在最坏情况下需要对压缩列表执行N次空间重分配操作，每次空间重分配的最坏复杂度大为O(N),所以连锁更新的最坏复杂度为O(N²)
+ 在实际中，连锁更新出现的可能性很低。ziplistPush、ziplistInsert、ziplistDelete、ziplistDeleteRange四个函数都有可能会引发连锁更新



## 对象

+ Redis并不是直接使用数据结构来实现键值对数据库，而是基于这些数据结构创建了一个对象系统。每种对象都用到了至少一种前面介绍的数据结构
+ Redis的对象系统还是先了基于引用计数技术的内存回收机制，当程序不再使用某个对象的时候，这个对象所占用的内存就会被自动释放。另外Redis还通过引用计数技术实现了对象共享机制，这一机制可以在适当的条件下，通过多个数据库键共享同一个对象来节约内存。Redis的对象带有访问时间记录信息，该信息可以用于计算数据库键的空转时长。

### 对象的类型和编码

+ Redis使用对象来表示数据库中的键和值，每次当我们在Redis的数据库中新创建一个键值对时，我们至少会创建两个对象，一个用作键值对的键（键对象），一个用于键值对的值（值对象）

+ Redis中的每个对象都有一个redisObject结构表示

![image-20220618231617390](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206182316568.png)

### 类型

+ 对于Redis数据库保存的键值对来说，键总是一个字符串对象，而值则可以是字符串对象、列表对象等其中一种
+ 当我们称呼一个数据库键为“字符串键”时，我们指的是“这个数据库所对应的值为字符串对象”
+ 当我们称呼一个键为“列表键”时，我们指“这个数据库所对应的值为列表对象”
+ 当我们对一个数据库键执行TYPE命令时，命令返回的结果为数据库键对应的值对象的类型，而不是键对象的类型

~~~shell
redis > RPUSH numbers 1 35 67 
(integer)5
redis> TYPE numbers
list
# 键为字符串对象，值为列表对象
~~~

### 编码和底层实现

+ 对象的ptr指针指向对象的底层实现数据结构，而这些数据结构由对象的encoding属性决定
+ encoding属性记录了对象所使用的的编码，也就是说这个对象使用了什么数据结构作为对象的底层实现

![image-20220619100035513](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206191000655.png)

|  类型  |     少     |     多     |
| :----: | :--------: | :--------: |
| String | int/embstr |    raw     |
|  List  |  ziplist   | linkedList |
|  Hash  |  ziplist   | hashtable  |
|  Set   |   intset   | hashtable  |
|  ZSet  |  ziplist   |  skiplist  |

+ 通过encoding属性来设定对象所使用的的编码，而不是为特定类型的对象关联一种固定的编码，极大的提升了Redis的灵活性和效率，因为Redis可以成反射不同的使用场景来为对象设置不同的编码，从而优化对象在某一场景下的效率
+ 在列表对象包含的元素较少时，Redis使用压缩列表作为列表对象的底层实现：
    + 因为压缩列表比双端链表更节约内存，并且在元素数量较少时，在内存中已连续块方式保存的压缩列表可以更快的被载入到缓存中
    + 随着列表对象包含的元素越来越多，使用压缩列表来保存云荣盛的优势逐渐消失时，对象就会将底层实现转为功能更强、也更适合保存大量元素的双端链表上

### 字符串对象

+ 字符串对象的编码可以是int、raw、embstr
+ 如果一个字符串对象保存的市政树枝，且可以用long类型来表示，字符串对象就会将整数值宝凑单字符串对象结构的ptr属性里面（将void*转换成long），并将字符串对象的编码设置为int
+ 如果字符串对象保存的是一个字符串值，且长度大于39字节，那么字符串对象将使用SDS来保存这个字符串值，且将对象的编码设置为raw

![image-20220619103043922](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206191030022.png)

+ 如果字符串对象保存的是字符串值，且长度小于等于39字节，就使用embstr编码的方式保存这个字符串值
    + embstr的好处：将创建字符串对象所需的内存分配次数从raw编码的两次降低为一次
    + 释放embstr编码的字符串对象只需要调用一次内存释放函数，而释放raw编码的字符串对象需要调用两次内存释放函数
    + embstr编码的字符串对象的所有数据都保存在一块连续的内存里面，所以这种编码的字符串对象相比起raw编码的字符串对象能更好的利用缓存带来的优势
+ 可以用long double类型标识的浮点数在Redis中也是作为字符串值来保存的，如果要报一个浮点数到字符串对象里面，那么程序会先讲这个浮点数转换成字符串值，然后在保存转换所得的字符串值

#### 编码的转换

+ int、embstr编码的字符串对象在某些条件下，会转换成raw编码
+ 比如对int编码的对象，通过append命令，向一个保存预估水产的字符串对象追加了一个车字符串值，因为追加操作只能对字符串执行，所以程序会现将之前保存用的整数值10086转成字符串“10086”，再追加
+ embstr编码的字符串对象实际上是只读的

 ### 列表对象

+ 列表对象的编码可以是ziplist或者linkedlist
+ ziplist编码的列表对象使用压缩列表作为底层实现，每个压缩列表节点保存了一个列表元素

![image-20220621213356065](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212133230.png)

+ linkedlist编码的列表对象使用双端列表作为底层实现，每个双端链表节点都保存了一个字符串对象，而每个字符串对象都保存了一个列表元素

![image-20220621213600415](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212136514.png)

+ linkedlist编码的列表对象在底层的双端链表结构中包含了多个字符串对象，这种嵌套对象在哈希对象、集合对象和有序集合对象中都会出现，**字符串对象是Redis五种类型的对象中唯一一种会被其他四种对象嵌套的对象**

![image-20220621213832954](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212138072.png)

#### 编码转换

+ 当同时满足两个条件时，列表对象使用ziplist编码
    + 列表对象保存的所有字符串元素的长度都小于64字节
    + 列表对象保存的元素数量小于512个，
+ 不能同时满足这两个条件的列表对象需要使用linkedlist编码

### 哈希对象

+ 编码可以是ziplist或者Hashtable
+ ziplist编码的哈希对象使用压缩列表作为底层实现，每当有新的键值对要加入到哈希对象时，程序会先将保存了键的压缩列表节点推入到压缩列表表尾，然后再将保存了值的压缩列表节点推入到压缩列表表尾

![image-20220621214145807](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212141919.png)

![image-20220621214152375](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212141471.png)

+ 当hashtable编码的哈希对象使用字典作为底层实现，哈希对象中的每个键值对都使用一个字典键值对来保存
    + 字典中的每个键都是一个字符串对象，对象中保存了键值对的键
    + 每个值都是一个字符串对象，对象中保存键值对的值

![image-20220621214313662](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212143772.png)

#### 编码转换

+ 同时满足一下两个条件，哈希对象使用ziplist编码
    + 哈希对象保存的所有键值对的键和值的字符串长度都小于64个字节
    + 键值对数量小于512个

### 集合对象

+ 编码可以是intset或者hashtable
+ intset编码的集合对象使用整数集合作为底层实现，集合对象包含的所有元素都被保存在整数集合里

![image-20220621214555329](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212145430.png)

+ hashtable编码的集合对象使用字典作为底层实现，字典中的每个键都是一个字符串对象，每个字符串对象都包含了一个集合元素，字典的值全为null

![image-20220621214644566](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212146665.png)

#### 编码转换

+ 同时满足两个条件，对象使用intset编码
    + 集合对象包的元素都是整数值
    + 保存的元素数量不超过512个

### 有序集合对象

+ 编码可以是ziplist或者skiplist
+ ziplist编码的有序集合对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员，第二个保存元素的分值
+ 压缩列表内的疾患元素按分值从小到大排序，较小的放在表头

![image-20220621214914069](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212149174.png)

![image-20220621215423760](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212154864.png)

+ skiplist编码的有序集合对象使用zset结构作为底层实现，一个zset结构同时包含一个字典和一个跳跃表
+ zset结构众安的zsl跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素，跳跃表节点的object属性保存了元素的成员，节点的score属性则保存了元素的分值。通过跳跃表，程序可以对有序集合进行范围型操作。
+ zset结构中的dict字典为有序集合创建轮承恩从成员到分值的映射，字典中的每个键值对都保存了一个集合元素，字典的键保存了元素的成员，字典的值保存了元素的分值。通过字典，程序可以用O(1)复杂度查找给定成员的分值
+ 虽然zset结构同时使用跳跃表和字典来保存有序集合元素，但这两种数据结构都会通过指针来共享相同元素的成员和分值，所以同时使用跳跃表和字典来保存集合元素不会产生任何重复成员或者分值，也不会有内存被浪费

<img src="https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212200339.png" alt="image-20220621220057200" style="zoom:50%;" />

<img src="https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212201706.png" alt="image-20220621220136567" style="zoom:67%;" />

#### 编码转换

+ 同时满足两个，使用ziplist编码
    + 有序集合保存的元素数量小于128个
    + 所有元素的长度都小于64字节

### 类型检查与命令多态

+ Redis中的操作键的命令分为两种
    + 对于任何类型的键都执行，DEL、EXPIRE、RENAME等
    + 只能对特定类型的键执行

+ 为了确保只有指定类型的键可以执行某些特定的命令，在执行一个类型特定的命令之前，Redis会先检查输入键的类型是否正确，在决定是否执行
+ Redis还会根据值对象的编码方式，选择正确的命令实现代码来执行命令
+ 比如LLEN命令，除了保证是对列表键执行命令外，还需要根据键的值对象所使用的编码来选择正确的LLEN命令
    + 如果编码是ziplist，就是用ziplistLen函数来返回列表的长度
    + linkedlist情况下，程序使用listLength返回长度
+ 可以认为LLEN命令是多态

### 内存回收

+ Redis在自己的对象系统中构建了一个引用计数技术实现的内存回收机制，通过这一机制，程序可以通过跟踪对象的引用计数信息，在适当的时候自动释放对象并进行内存回收

![image-20220621220942438](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212209575.png)

+ 创建对象时，值会被初始化为1
+ 当对象被一个新程序使用时，+1
+ 不再被一个程序使用时，-1
+ 当值变为0时，对象所占用的内存被释放

### 对象共享

+ 对象的引用计数属性还带有对象共享的作用
+ 当键A创建了一个包含整数值100的字符串对象作为值对象，此时键B也要创建一个同样保存了整数值100的字符串对象作为值对象，那么服务器会让键AB共享同一个字符串对象
+ 需要两步
    + 将数据库键的值指针指向一个现有的值对象
    + 将被共享的值对象的引用计数+1
+ 目前来说，Redis会在初始化服务时，创建一万个字符串对象，包含了从0-9999的所有整数值，便于服务器共享使用‘

+ 这些共享对象不仅只有字符串键可以使用，只要数据结构中嵌套了字符串的对象（linkedlist编码的列表对象，hashtable编码的哈希对象，hashtable边am的集合，zset编码的有序集合）都可以使用

### 对象的空转时长

+ redisObject结构包含的最后一个属性为lru属性，记录了对象最后一次被命令程序访问的时间

![image-20220621221605404](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206212216502.png)

+ OBJECT IDLETIME命令可以打印出给定键的空转时长，通过当前时间减去lru计算得出
+ 如果服务器打开了maxmemory选项，并且服务器用于回收内存的算法为volatile-lru或者allkeys-lru，那么当服务器占用的内存数超过了maxmemory选项所设置的上线值，空转时长较高的部分键会被优先释放，回收内存

## 数据库

+ Redis服务器将所有的数据库都保存在服务器的db数组中，db数组的每个项都是一个redis.h/redisDb结构，每个redisDb结构代表一个数据库

![image-20220626133052297](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206261330420.png)

+ 在初始化服务器时，程序会根据服务器状态的dbnum属性来决定应该创建多少个数据库

![image-20220626133138307](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206261331413.png)

+ 默认16个
+ 使用SELECT命令来切换目标数据库
+ 在服务器内部，客户端状态redisClient结构的db属性记录了客户端当前的目标数据库，这个属性是一个指向redisDb结构的指针

![image-20220626133847104](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206261338185.png)

+ redisClient.db指针指向redisServer.db数组的其中一个元素，而被指向的元素就是客户端的目标数据库
+ 通过修改次指针，指向服务器中的不同数据库，从而实现切换目标数据库的功能---select命令的实现原理

### 数据库键空间

+ 服务器中的每个数据库都有个一个redis.h/redisDb结构表示，其中redisDb结构的dict字典保存了数据库中的所有键值对，我们将这个字典称为键空间

![image-20220626134123658](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206261341756.png)

+ 键空间和用户所见的数据库是直接对应的
    + 键空间的键也就是数据库的键，每个键都是一个字符串对象
    + 值也就是数据库的值，可以是五大对象中任意一种

<img src="https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206261353133.png" alt="image-20220626135329018" style="zoom:50%;" />

+ 因为数据库的键空间是一个字典，所以所有针对数据库的操作，都是通过对键空间字典进行操作
+ 添加一个新建只对都数据库，实际上就是将一个新键值对添加到键空间字典里面，其中键为字符串对象，值就是任意一种类型的Redis对象
+ 删除数据库中的键，就是在键空间里面删除检所对应的键值对对象
+ 更新、取值都是对键空间对应的值进行操作

### 读写键空间的维护操作

+ 当使用Redis命令对数据库进行读写时，服务器不仅会对键空间执行指定的读写操作，还会执行一些额外的维护操作
    + 在读取一个键之后（读写操作都需要对键进行取值），服务器会根据键是否存在来挂吧服务器的键空间命中次数和键空间不命中次数，这两个值在INFO stats命令的keyspace_hits和keyspace_misses中查看
    + 在读取一个键之后，服务器会更新键的LRU（最后一次使用）时间，这个值可以用于计算键的限制时间
    + 如果服务器在读取一个键时发现该键已过期，那么服务器会先删除这个过期键，然后才执行余下的其他操作
    + 如果客户端使用WATCH命令监视了某个键，那么服务器在对被监视的键进行修改职责，会将这个键标记为脏，从而让事务程序注意到这个键已经被修改过
    + 服务器每次修改一个键之后，都会对脏键计数器的值+1
    + 如果服务器开启了数据库通知功能，那么在对键进行下去该之后，服务器将按配置发送相应的数据库通知

### 设置键的生存时间或过期时间

+ 通过EXPIRE命令或者PEXPIRE命令，客户端可以对数据库中键的生存时间进行秒或者毫秒精度的设置
+ SETEX命令只能用于字符串键
+ 过期时间是一个UNIX时间戳，当键的过期时间来临时，服务器就会自动从数据库中删除这个键
+ EPXIRE：设置生存时间为ttl秒；PEXPIRE：设置生存时间为ttl毫秒；EXPIREAT：将key的过期时间设置为指定的秒数时间戳；PEXPIREAT：设置指定毫秒时间戳
+ 这四个命令都是使用PEXPIREAT命令实现的，无论客户端执行以上四个命令中的哪一个，经过转换后，都执行效果都和PEXPIRET命令一样
+ redisDb结构中的expires字典保存了数据库中所有键的过期时间

<img src="https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202206261423788.png" alt="image-20220626142308645" style="zoom:67%;" />

+ PERSIST命令就是PEXPIREAT命令的反向操作：为了在过期字典中查找给定的键，并解除键和值（过期时间）在过期字典中的关联
+ TTL和PTTL是通过计算键的过期时间和当前时间之间的差来实现的

### 过期键删除策略

+ 定时删除：在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来临时，立即执行对键的删除操作
+ 多行删除：放任键过期不管，但是每次从键空间中获取键时，都检查取得的键是否过期，过期就删除，没有就返回
+ 定期删除：每隔一段时间，程序就对数据库进行一次检查，删除里面的过期键。

#### 定时删除

+ 对内存最友好：可以保证过期键会尽可能快的被删除，并释放所占用的内存
+ 对CPU时间不友好，删除这个功能会占用相当一部分CPU时间，将CPU时间用在删除和当前任务无关的过期键上，会对服务器的响应时间和吞吐量造成影响
+ 创建一个定时器需要用到REdis服务器中的时间事件，而当前事件的实现方式---无序链表，查找一个事件的时间复杂度为O(N)---并不能高效的处理大量时间事件

#### 惰性删除

+ 对CPU时间友好，程序只会在取出键时才对键进行过期检查，删除的目标仅限于当前处理的键
+ 对内存不友好：如果一个键过期，又依然保存在数据库中，就会一直占用内存
+ 甚至可以看成是一种内存泄露----无用的垃圾数据占用了大量的内存
+ 策略由db.c/expireIfNeeded函数实现，所有读写数据库的Redis命令在执行之前都会调用expireIfNeeded函数对输入键进行检查
    + 如果过期，就删除，未过期就不动

#### 定期删除

+ 前两种策略的一种整合和折中
+ 策略由redis.c/activeExpireCycle函数实现，在规定的时间内，分多次遍历服务器中的各个数据库，从数据库中的expires字典中随机检查一部分键的过期时间，并删除其中的过期键

#### AOF、RDB和复制功能对过期键的处理

+ 在执行SAVE命令或者BGSAVE命令时，创建一个新的RDB文件，程序会对数据库中的键进行检查，已过期的键不会被保存到新创建的RDB文件中
+ 如果服务器以主服务器模式运行，在载入RDB文件时，程序会对文件中保存的键进行检查，未过期的键会被载入到数据库中，过期的会被忽略。
+ 以从服务器模式运行，在载入RDB文件时，所有的键不论是否过期都会被保存到数据库中，不过因为主从服务器在进行数据同步的时候，从服务器的数据库会被清空，所以过期键载入一般也不会对从服务器造成影响

#### AOF文件写入

+ 服务器以AOF持久化模式运行时，如果数据库中的某个键已经过期，但他还没有被删除，那么AOF文件不会因为这个过期键而产生任何影响
+ 当过期键被惰性删除或者定期删除之后，程序回想AOF文件追加一个DEL命令，来显示地记录该键已被删除

#### AOF重写

+ 在执行AOF重写时，程序会对数据库中的键进行检查，已过期的键不会被保存到重写后的AOF文件中

#### 复制

+ 当服务器运行在复制模式下时，从服务器的过期键删除动作由主服务器控制
    + 主服务器在删除一个过期键之后，会显示的向所有从服务器发送一个DEL命令，告知从服务器删除这个键
    + 从服务器在执行客户端发送的读命令时，即使碰到过期键也不会将过期键删除，而是继续像处理未过期的键一样来处理过期键
    + 从服务器只有在接到主服务器发来的DEL命令时，才会删除

#### 数据库通知

+ 让客户端通过订阅给定的频道或者模式，来获知数据库中键的变化、命令的执行情况

<img src="https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207042240472.png" alt="image-20220704224014299" style="zoom:50%;" />

+ 这一类关注“某个键执行了什么命令”的通知称为键空间通知
+ 还有一类称为键事件通知，关注的是“某个命令被什么键执行了”

![image-20220704224127499](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207042241595.png)



## RDB持久化

+ RDB功能可以将Redis在内存中的数据库状态保存到磁盘里，避免数据意外丢失
+ RDB持久化功能所生成的RDB文件是一个经过压缩的二进制文件，通过改文件可以还原生成RDB文件时的数据库状态

![image-20220704224623044](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207042246135.png)

### RDB文件的创建与载入

+ SAVE和BGSAVE命令可以用于生成RDB文件
+ SAVE命令会阻塞Redis服务进程，直到RDB文件创建完毕之前，服务器不能处理任何命令请求
+ BGSAVE命令会派生出一个子进程，然后子进程来负责创建RDB文件，服务器（父进程）继续处理命令请求
+ RDB文件的载入是在服务器启动时自动执行的，只要检测到RDB文件，就会自动载入

![image-20220704224951802](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207042249884.png)

+ 如果服务器开启了AOF持久化，服务器会优先使用AOF文件来还原数据库状态
+ BGSAVE命令执行期间，Redis仍可以继续处理客户端的命令请求，但是SAVE命令会被拒绝，避免父进程和子进程同时执行两个RDBSave调用；客户端发送的BGREWRITEWAOF命令会被延迟到BGSAVE命令执行完毕之后执行
+ 在载入RDB文件期间，会一直处于阻塞状态

### 自动间隔性保存

+ 用户可以通过SAVE选项设置多个保存条件，只要其中一个被满足，就会执行BGSAVE命令

![image-20220704225227645](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207042252716.png)

+ 服务器程序会根据save选项所设置的保存条件，设置服务器状态RedisServer结构的saveparams属性

<img src="https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207042253977.png" alt="image-20220704225324898" style="zoom:67%;" />

<img src="https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207042253902.png" alt="image-20220704225336823" style="zoom: 67%;" />

+ Redis的服务器周期性操作函数serverCrom默认每隔100ms就会执行一次，该函数用于对正在运行的服务器进行维护，其中一项工作就是检查save选项所设置的保存条件是否已经满足

### dirty计数器和lastsave属性

+ dirty计数器记录距离上一次成功执行save命令或者BGSAVE命令之后，服务器对数据库状态（所有数据库）进行了多少次修改
+ lastsave属性是一个unix时间戳，记录了服务器上一次成功执行save命令或者BGSAVE命令的时间

<img src="https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207042254614.png" alt="image-20220704225458537" style="zoom:67%;" />

### RDB文件结构

![image-20220704225744928](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207042257008.png)

+ 开头是REDIS部分，长度5个字节，保存“REDIS”五个字符，程序可以在载入文件时，通过这五个字符，检查所载入的文件是否是RDB文件
+ db_version长度为4个字节，是一个字符串表示的整数，记录了RDB文件的版本。
+ databases包含着零个或多个数据库，以及各个数据库中的键值对数据
+ EOF常量的长度为1个字节，标志着RDB文件正文内容的结束
+ check_sum是一个8字节长的无符号整数，保存着一个校验和

![image-20220705232042035](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207052320161.png)

#### databases

![image-20220705232114085](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207052321157.png)

+ 每个非空数据库在RDB文件中都可以保存为SELECTDB、db_number、key_value_pairs三个部分

![image-20220705232215226](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207052322316.png)

+ selectdb，当程序读到这个值的时候，知道接下来要读入的是一个数据库号码
+ db_number保存着数据库号码
+ key_value_pairs部分保存了数据库中的所有键值对数据

![image-20220705232333772](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207052323855.png)

#### key_value_pairs

+ RDB文件中的每个key_value_pairs部分都保存了一个或以上数量的键值对，如果键值对带有过期时间的话，那么键值对的过期时间也会被保存在内

![image-20220705232535230](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207052325303.png)

+ TYPE记录了value的类型，程序会根据TYPE的值来决定如何读入和解释value的数据，key和value分别保存了键值对的键对象和值对象
    + key总是一个字符串对象
    + 根据TYPE类型的不对，以及保存内容长度的不同，保存value的结构和长度也会有所不同

![image-20220705232700318](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207052327403.png)

### value的编码

+ RDB文件中的每个value部分都保存了一个值对象，每个值对象的类型都由与之对应的TYPE记录，根据类型的不同，value部分的结构、长度也会有所不同
+ 字符串对象、列表对象、集合对象、哈希表对象、有序集合对象、INTSET编码的集合、ZIPLIST编码的列表、哈希表或者有序集合

#### 分析RDB文件

+ od命令来分析Redis服务器产生的RDB文件，该命令可以用给定的格式转存（dump）并打印输入文件。比如 -c参数可以以ASCII编码的方式打印输入文件，-x参数可以以十六进制的方式打印输入文件

+ 不包含任何键值对的RDB文件

![image-20220706233607283](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207062336430.png)

1. 五个字节的REDIS字符串
2. 四个字节的版本号（db_version)：0006
3. 一个字节的EOF常量：377
4. 八个字节的校验和：334...362 V

+ 包含字符串键的RDB文件

![image-20220706234124338](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207062341426.png)

1. REDIS和版本号
2. 376代表SELECTDB常量，
3. \0代表0号数据库
4. RDB文件包含的内容是 \0 003... H E L L O

+ 包含带有过期时间的字符串键的RDB文件

![image-20220706234738007](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207062347096.png)



## AOF 持久化

+ RDB持久化是保存数据库中的键值对来记录数据库状态不同，AOF持久化是通过保存Redis服务器所执行写命令来记录数据库状态的

![image-20220707212606152](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207072126289.png)

+ RDB持久化保存数据库状态的方法是将所有键的键值对保存到RDB文件中，而AOF持久化保存数据库状态的方法则是将服务器执行的命令保存到AOF文件中
+ 被写入AOF文件的所有命令都是一Redis的命令请求协议格式保存的，因为Redis的命令请求协议是纯文本格式，所以可以直接打开一个AOF文件，观察里面的内容

![image-20220710164114937](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207101641067.png)

### AOF持久化的实现

+ 分为三步：命令追加、文件写入、文件同步

#### 命令追加

+ 当AOF持久化功能处于打开状态时，服务器在执行完一个排序命令之后，会以协议格式将被执行的写命令追加到服务器状态的aof_buf缓冲区的末尾

#### AOF文件的写入与同步

+ Redis的服务器进程就是一个事件循环，这个循环中的文件事件负责接收客户端的命令请求，以及向客户端发送命令回复，而时间事件则负责执行像serverCron函数这样需要定时运行的函数
+ 因为服务器在处理文件事件时可能会执行写命令，使得一些内容被追加到aof_buf缓冲区里，所以在服务器每次结束一个事件循环之前，都会调用flushAppendOnlyFile函数，考虑是否需要将aof_buf缓冲区中的内容写入和保存到AOF文件里面

![image-20220710164858656](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207101648785.png)

+ 为了提高文件的写入效率，在现代操作系统中，当用户调用write函数，将一些数据写入到文件的时候，操作系统通常会将写入数据暂时保存在一个内存缓冲区里，等到缓冲区的空间被填满或者超过指定的时限之后，才将缓冲区中的数据写入到磁盘里面

### AOF文件的载入与数据还原

+ 服务器只需要读入并重新执行一遍AOF文件里面保存的写命令，就可以还原服务器关闭之前的数据库状态
    + 创建一个不带网络连接的伪客户端：因为Redis的命令只能在客户端上下文中执行，而载入AOF文件时所使用的的命令直接来源于AOF文件而不是网络连接，所以服务器使用了一个没有网络连接的微客户端来执行AOF文件保存的写命令，伪客户端执行的效果和带网络连接的客户端执行命令的效果完全一样
    + 从AOF文件中分析并读取出一条写命令
    + 使用微客户端执行被读出的写命令
    + 重复执行2、3步骤，知道AOF文件中所有的写命令都被处理完毕为止

### AOF重写

+ 为了防止AOF文件越来越大，Redis提供了AOF文件重写功能。Redis服务器可以创建一个新AOF文件来替代现有的AOF文件，新旧两个AOF文件所保存的数据库状态相同，但新的AOF文件不会包含任何浪费空间的冗余命令

#### AOF文件重写的实现

+ AOF文件重写并不需要对现有的AOF文件进行任何读取、分析或者写入操作，这个功能是通过读取服务器当前的数据库状态来实现的
+ 首先从数据库中读取键现在的值，然后用一条命令去记录键值对，代替之前记录这个键值对的多条命令
+ 因为aof_rewrite函数生成的新AOF文件只包含还原当前数据库状态所必须的命令，所以新AOF文件不会浪费任何硬盘空间

#### AOF后台重写

+ AOF重写程序aof_rewrite函数可以很好的完成创建一个新AOF文件的任务，但是，因为这个函数会进行大量的写入操作，所以调用次函数的线程将被长时间阻塞。因为Redis服务器使用单个线程来处理命令请求，所以Redis决定将AOF重写程序放到子进程里执行，这样做可以同时达到两个目的
    + 子进程进行AOF重写期间，服务器进程可以继续处理命令请求
    + 子进程带有服务器进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据的安全性
+ 如果当重写期间，又有新的修改数据库的命令。可能会造成数据库状态不一致，为了解决这种问题，Redis服务器设置了一个AOF重写缓冲区，这个缓冲区在服务器创建子进程之后开始使用，当Redis服务器执行完一个些命令之后，他会同时将这个写命令发送给AOF缓冲区和AOF重写缓冲区

![image-20220712210134880](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207122101038.png)

+ 在子进程执行AOF重写期间，服务器进程会执行
    + 执行客户端发来的命令
    + 将执行后的写命令追加到AOF缓冲区
    + 将执行后在的写命令追加到AOF重写缓冲区
+ 当子进程完成AOF重写工作之后，他会想父进程发送一个信号，父进程在接到该信号之后，会调用一个车信号处理函数，并执行
    + 将AOF重写缓冲区中的所有内容写到新AOF文件中，这时新AOF文件所保存的数据库状态将和服务器当前的数据库状态一致
    + 对新的AOF文件进行改命，原子地（atomic）覆盖现有的AOF文件，完成新旧两个AOF文件的替换
+ 在整个AOF后台重写过程中，只有信号处理函数执行时会对服务器进程（父进程）造成阻塞

## 事件

+ 服务器一般需要处理两类事件
    + 文件事件：Redis服务器通过套接字与客户端进行连接，文件事件就是服务器对套接字操作的抽象。服务器与客户端的通信会产生响应的文件事件，而服务器则通过监听并处理这些事件来完成一系列网络通信操作
    + 时间事件：Redis服务器中的一些操作（serverCron）需要在给定的时间点执行

### 文件事件

+ Redis基于Reactor模式开发了自己的网络事件处理器
+ 文件事件处理器使用I/O多路复用程序来同时监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器
+ 当被监听的套接字准备好执行连接应答、读取、写入、关闭等操作时，与操作相对应的文件事件就会产生，这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件
+ 虽然文件事件处理器以单线程方式运行，但通过使用I/O多路复用程序来监听多个套接字，文件事件处理器既实现了高性能的网络通信模型，又可以很好地与Redis服务器中其他同样以单线程方式运行的模块进行对接

#### 文件事件处理器的构成

+ 套接字、I/O多路复用程序，文件事件分派器、事件处理器

![image-20220712211921270](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207122119376.png)

+ 文件事件是对套接字操作的抽象，每当一个套接字准备好执行连接应答、写入、读取等操作时，就会产生一个文件事件，因为一个服务器通常会连接多个套接字，所以多个文件事件有可能会并发的出现
+ I/O多路复用程序负责监听多个套接字，并向文件事件分派器传送那些产生了事件的套接字
+ 尽管多个文件事件可能会并发出现，但I/O多路复用程序总是会将所有产生事件的套接字都放到一个队列里，然后通过这个队列，以有序、同步、每次一个套接字的方式向文件事件分派器传送套接字。

![image-20220712212243465](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207122122556.png)

#### I/O多路复用程序的实现

+ Redis的I/O多路复用程序的所有功能都是通过包装常见的select、epoll、evport和kqueue这些多路复用函数库来实现的，每个函数库在Redis源码中都对应一个单独的文件

![image-20220712212433780](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207122124873.png)

#### 文件事件的处理器

+ Redis为文件事件编写了多个处理器，这些事件处理器分别用于实现不同的网络通信需求
    + 为了对连接服务器的各个客户端进行应答，服务器要为监听套接字关联连接应答处理器
    + 为了接收客户端传来的命令请求，服务器要为客户端套接字关联命令请求处理器
    + 为了向客户端返回命令的执行结果，为客户端套接字关联命令回复处理器
    + 当主从复制时，主从都需要关联特别的复制处理器

### 时间事件

+ 定时事件：一段程序在指定的时间之后执行一次
+ 周期性事件：一段程序每隔指定时间就执行一次
+ 每个时间事件有三个属性
    + id：全局唯一ID，ID号从小到大顺序递增
    + when：毫秒经度的UNIX时间戳，记录了时间事件到达时间
    + timeProc：时间事件处理器，一个函数。
+ 服务器将所有时间事件都放在一个无序链表中，每当时间事件执行器运行时，他就遍历整个链表查找所有已到达的时间事件，调用相应的事件处理器
+ 保存时间事件的链表为无序链表，并不是链表不按照ID排序，而是该链表不按照when属性的大小排序
+ 无序链表并不会影响时间事件处理器的性能

#### serverCron函数

+ 更新服务器的各类统计信息：时间、内存占用、数据库占用等
+ 清理数据库中的过期键值对
+ 关闭和清理连接失效的客户端
+ 尝试进行AOF或者RDB持久化操作
+ 对从服务器进行定期同步或者对集群进行定期同步、连接测试



## 客户端

## 服务端

## 复制

+ 用户执行slaveof或者设置slaveof选项，让一个服务器去复制另一个服务器

 ![image-20220716151150406](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207161511529.png)

+ 进行复制中的主从服务器双方的数据库将保存相同的数据，概念上将这种现象称为“数据库状态一致”

### 旧版复制功能的缺陷

+ 在Redis2.8以前，从服务器对主服务器的辅助分为初次复制和断线后重复制。
+ 对于断线后重复制，旧版复制功能的效率比较低。因为SYNC命令会让从服务器从头开始复制，即使断了几秒。
+ SYNC命令是一个非常耗费资源的操作，每次执行SYNC命令，主从服务器都需要执行
    + 主服务器需要执行BGSAVE命令来生成RDB文件，这个生成操作会耗费主服务器大量的CPU、内存和磁盘资源
    + 主服务器需要将自己生成的RDB文件发送给从服务器，这个操作会耗费主从服务器大量的网络资源
    + 接收到RDB文件的从服务器需要载入主服务器发来的RDB文件，并且在载入期间，从服务器会因为阻塞而没办法处理命令请求

### 新版复制功能

+ 使用PSYNC命令代替SYNC命令来执行复制时的同步操作
+ PSYNC命令具有完整重同步和部分重同步的两种模式

### 复制偏移量

+ 执行复制的双方--主服务器和从服务器会分别维护一个复制偏移量
+ 主服务器每次向从服务器传播N个字节的数据时，就将自己的复制偏移量的值加上N
+ 从服务器每次收到主服务器传播来的N个字节的数据时，就将自己的复制偏移量加上N

![image-20220716154734682](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207161547803.png)

+ 通过对比主从服务器的复制偏移量，就能知道主从是否一致

### 复制积压缓冲区

+ 是由主服务器维护的一个固定长度、先进先出队列，默认大小为1MB

![image-20220716154839490](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207161548590.png)

+ 当从服务器重新脸上主服务器时，从服务器会通过PSYNC命令将自己的复制偏移量offset发送给主服务器，主服务器会根据这个复制偏移量来决定对从服务器的执行何种同步操作
    + 如果offset偏移量之后的数据（偏移量offset+1开始的数据）仍然存在于复制挤压缓冲区里面，那么主服务器将对从服务器执行部分重同步
    + 相反，offset偏移量之后的数据不存在与复制挤压缓冲区，就完整重同步
+ 按需调整复制积压缓冲区的大小
    + 2*second*write_size_per_second

### 服务器运行ID

+ 每个Redis服务器，主从都会有自己的运行ID，在服务启动时自动生成，由40个随机的十六进制字符组成
+ 当从服务器断线并重连上一个车主服务器时，从服务器将项当前连接的主服务器发送之前保存的运行ID

### PSYNC命令的实现

![image-20220716160449493](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207161604617.png)

### 复制的实现

#### 设置主服务器的地址和端口

+ 从服务器首先将客户端给定的主服务器IP地址127.0.0.1和端口6379保存到服务器状态的master属性和masterport属性里面
+ SLAVEOF命令是一个异步命令，在完成masterhost属性和masterport属性的设置工作之后，从服务器将向发送SALVEOF命令的客户端返回OK，表示复制指令已经被接收，而实际的复制工作将在OK返回之后才真正开始执行

#### 建立套接字连接

+ 如果从服务器创建的套接字能成功连接到主武器，那么从服务器将为这个套接字关联一个专门用于处理复制工作的文件事件处理器，这个处理器负责执行后续的复制工作
+ 主服务器在接收从服务器的套接字连接之后，姜维该套接字创建相应的客户端状态，并将从服务器看做是一个连接到主服务器的客户端来对待，这是从服务器同时具有服务器和客户端两个身份

![image-20220716161453392](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207161614536.png)

#### 发送PING命令

 ![image-20220716161522891](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207161615015.png)

#### 身份验证

![image-20220716161553342](https://picgo-machuan.oss-cn-hangzhou.aliyuncs.com/reids/202207161615507.png)

#### 发送端口信息

+ 从服务器向主服务器发送从服务器的监听端口号

#### 同步

#### 命令传播

#### 心跳检测

+ 检测主从服务器的网络连接状态
+ 辅助实现min-slaves选项
+ 检测命令丢失



## Sentinel

+ 哨兵是Redis高可用的解决方案：由一个或多个Sentinel实例组成的Sentinel系统可以监视任意多个主服务器，以及属下的所有从服务器。并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求



# 暂停几天，因为脱单了，外加出差

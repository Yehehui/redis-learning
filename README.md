# redis-learning
[参考文档](http://redisbook.com)

## SDS

### 定义
``` c
struct sdshdr {
    // 记录 buf 数组中已使用字节的数量
    // 等于 SDS 所保存字符串的长度
    int len;
    // 记录 buf 数组中未使用字节的数量
    int free;
    // 字节数组，用于保存字符串
    char buf[];
};
```

### 对比C字符串

- len属性 获取长度复杂度为O(1)
- 动态申请内存空间，避免缓冲区溢出
- 内存预分配、惰性释放，避免频繁重分配
- 二进制安全
- 兼容部分C字符串函数

### 主要API
[SDS API](http://redisbook.com/preview/sds/api.html)

## 链表

### 实现

#### 链表节点实现
``` c
typedef struct listNode {
    // 前置节点
    struct listNode *prev;
    // 后置节点
    struct listNode *next;
    // 节点的值
    void *value;
} listNode;
```

#### 链表实现
``` c
typedef struct list {
    // 表头节点
    listNode *head;
    // 表尾节点
    listNode *tail;
    // 链表所包含的节点数量
    unsigned long len;
    // 节点值复制函数
    void *(*dup)(void *ptr);
    // 节点值释放函数
    void (*free)(void *ptr);
    // 节点值对比函数
    int (*match)(void *ptr, void *key);
} list;
```

### 主要API
[链表和链表节点的 API](http://redisbook.com/preview/adlist/api.html)

## 字典

### 实现

#### 哈希表节点实现

``` c
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    // 指向下个哈希值相同的节点，处理冲突，形成链表(新结点插入表头)
    struct dictEntry *next;
} dictEntry;
```

#### 哈希表实现

``` c
typedef struct dictht {
    // 哈希表数组
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;
    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```

#### 字典实现

``` c
typedef struct dict {
    // 类型特定函数,用于实现多态
    dictType *type;
    // 私有数据，保存了需要传给那些类型特定函数的可选参数
    void *privdata;
    // 哈希表
    dictht ht[2];
    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */
} dict;
```

ht属性是一个包含两个哈希表的数组，一般情况下，字典只使用 ht[0] 哈希表，ht[1] 哈希表只会在对 ht[0] 哈希表进行 rehash 时使用。

#### 字典类型实现

``` c
typedef struct dictType {
    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

#### 普通状态下的字典

![](https://github.com/Yehehui/redis-learning/blob/main/image/%E6%99%AE%E9%80%9A%E7%8A%B6%E6%80%81%E4%B8%8B%E7%9A%84%E5%AD%97%E5%85%B8.png)

### 哈希算法 
当字典被用作数据库的底层实现， 或者哈希键的底层实现时， Redis 使用 [MurmurHash2](https://zh.wikipedia.org/wiki/Murmur%E5%93%88%E5%B8%8C) 算法来计算键的哈希值

``` c
# 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);
# 使用哈希表的 sizemask 属性和哈希值，计算出索引值
# 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]
index = hash & dict->ht[x].sizemask;
```

### rehash

#### 步骤

1. 为字典的 ht[1] 哈希表分配空间
 - 如果执行的是扩展操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used * 2 的 2^n
 - 如果执行的是收缩操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used 的 2^n
2. 将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面： rehash 指的是重新计算键的哈希值和索引值， 然后将键值对放置到 ht[1] 哈希表的指定位置上
3. 当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后 （ht[0] 变为空表）， 释放 ht[0] ， 将 ht[1] 设置为 ht[0] ， 并在 ht[1] 新创建一个空白哈希表， 为下一次 rehash 做准备

#### 扩展与收缩条件

扩展
1. 服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 1 ；
2. 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 5 ；

收缩
哈希表的负载因子小于 0.1 时， 程序自动开始对哈希表执行收缩操作

#### 渐进式rehash

1. 为 ht[1] 分配空间，让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
2. 在字典中维持一个索引计数器变量 rehashidx，并将它的值设置为 0，表示 rehash 工作正式开始。
3. 在 rehash 进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1]，当 rehash 工作完成之后，程序将 rehashidx 属性的值增一。
4. 随着字典操作的不断执行，最终在某个时间点上，ht[0] 的所有键值对都会被 rehash 至 ht[1]，这时程序将 rehashidx 属性的值设为 -1，表示 rehash 操作已完成。

##### 注意事项

1. 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行：程序会先在 ht[0] 里面进行查找，如果没找到的话， 就会继续到 ht[1] 里面进行查找
2. 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 ht[1] 里面， 而 ht[0] 则不再进行任何添加操作： 这一措施保证了 ht[0] 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。

### 字典API
[字典API](http://redisbook.com/preview/dict/api.html)

## 跳跃表
用来实现redis中的有序集合，实现及维护较红黑树及平衡树简单，通过在每个结点维护多个指向其他结点的指针，达到快速访问结点的目的
### 实现

#### 跳跃表结点实现
``` c
typedef struct zskiplistNode {
    // 后退指针
    struct zskiplistNode *backward;
    // 分值
    double score;
    // 成员对象
    robj *obj;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 跨度
        unsigned int span;
    } level[];
} zskiplistNode;
```

#### 跳跃表实现
``` c
typedef struct zskiplist {
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的节点的层数
    int level;
} zskiplist;
```

### 特性

1. 层高为1-32之间的随机数
2. 按score值排序，值相同时按对象在哈希表中的索引排序

### 相关API
[跳跃表API](http://redisbook.com/preview/skiplist/api.html)

## 整数集合
集合键底层实现之一

### 实现

``` c
typedef struct intset {
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组，实际类型根据encoding取值设定
    int8_t contents[];
} intset;
```

#### encoding取值
- INTSET_ENC_INT16 (sizeof(int16_t))
- INTSET_ENC_INT32 (sizeof(int32_t))
- INTSET_ENC_INT64 (sizeof(int64_t))

### 升级

1. 根据新元素的类型， 扩展整数集合底层数组的空间大小， 并为新元素分配空间。
2. 将底层数组现有的所有元素都转换成与新元素相同的类型， 并将类型转换后的元素放置到正确的位上， 而且在放置元素的过程中， 需要继续维持底层数组的有序性质不变。
3. 将新元素添加到底层数组里面。

### 相关API
[整数集合API](http://redisbook.com/preview/intset/api.html)

## 压缩列表
压缩列表是列表键和哈希键的实现之一，为了节约内存而开发

### 构成

#### 压缩列表构成
![](https://github.com/Yehehui/redis-learning/blob/main/image/%E5%8E%8B%E7%BC%A9%E5%88%97%E8%A1%A8%E7%BB%84%E6%88%90%E9%83%A8%E5%88%86.png)

|属性|类型|长度|用途|
|--|--|---|--|
|zlbytes|uint32_t|4 字节|记录整个压缩列表占用的内存字节数：在对压缩列表进行内存重分配， 或者计算 zlend 的位置时使用。|
|zltail|uint32_t|4 字节|记录压缩列表表尾节点距离压缩列表的起始地址有多少字节： 通过这个偏移量，程序无须遍历整个压缩列表就可以确定表尾节点的地址。|
|zllen|uint16_t|2 字节|记录了压缩列表包含的节点数量： 当这个属性的值小于 UINT16_MAX （65535）时， 这个属性的值就是压缩列表包含节点的数量； 当这个值等于 UINT16_MAX 时， 节点的真实数量需要遍历整个压缩列表才能计算得出。|
|entryX|列表节点 |不定  |压缩列表包含的各个节点，节点的长度由节点保存的内容决定。|
|zlend|uint8_t|1 字节|特殊值 0xFF （十进制 255 ），用于标记压缩列表的末端。|

#### 压缩列表结点构成
![](https://github.com/Yehehui/redis-learning/blob/main/image/%E5%8E%8B%E7%BC%A9%E5%88%97%E8%A1%A8%E7%BB%93%E7%82%B9%E7%BB%84%E6%88%90%E9%83%A8%E5%88%86.png)

##### previous_entry_length
根据这个属性及当前结点指针算出前一结点指针，实现表尾到表头的遍历

- 如果前一节点的长度小于 254 字节， 那么 previous_entry_length 属性的长度为 1 字节： 前一节点的长度就保存在这一个字节里面。
- 如果前一节点的长度大于等于 254 字节， 那么 previous_entry_length 属性的长度为 5 字节： 其中属性的第一字节会被设置为 0xFE （十进制值 254）， 而之后的四个字节则用于保存前一节点的长度。

##### encoding
记录content所保存内容的类型及长度

- 一字节、两字节或者五字节长， 值的最高位为 00 、 01 或者 10 的是字节数组编码： 这种编码表示节点的 content 属性保存着字节数组， 数组的长度由编码除去最高两位之后的其他位记录；
- 一字节长， 值的最高位以 11 开头的是整数编码： 这种编码表示节点的 content 属性保存着整数值， 整数值的类型和长度由编码除去最高两位之后的其他位记录；

|编码 |	编码长度 |	content 属性保存的值|
|--|--|--|
|11000000 |	1 字节 |	int16_t 类型的整数。|
|11010000 |	1 字节 |	int32_t 类型的整数。|
|11100000 |	1 字节 |	int64_t 类型的整数。|
|11110000 |	1 字节 |	24 位有符号整数。|
|11111110 |	1 字节 |	8 位有符号整数。|
|1111xxxx |	1 字节 |	使用这一编码的节点没有相应的 content 属性， 因为编码本身的 xxxx 四个位已经保存了一个介于 0 和 12 之间的值， 所以它无须 content 属性。|

##### content
保存结点内容

### 连锁更新
结点的插入或删除，引起后续多个结点内存大小变化，从而导致多次内存重分配

#### 影响
多次内存重分配复杂度很高，但发生概率很小

- 压缩列表里要恰好有多个连续的、长度介于 250 字节至 253 字节之间的节点， 连锁更新才有可能被引发， 在实际中， 这种情况并不多见；
- 即使出现连锁更新， 但只要被更新的节点数量不多， 就不会对性能造成任何影响： 比如说， 对三五个节点进行连锁更新是绝对不会影响性能的；

### 相关API
[压缩列表 API](http://redisbook.com/preview/ziplist/api.html)

## 对象
redis采用对象来表示数据库中的键和值

### 实现、类型和编码
``` c
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向底层实现数据结构的指针
    void *ptr;
    // ...
} robj;
```

#### 类型

|类型常量 |	对象的名称|
|--|--|
|REDIS_STRING| 	字符串对象|
|REDIS_LIST |	列表对象|
|REDIS_HASH |	哈希对象|
|REDIS_SET |	集合对象|
|REDIS_ZSET | 有序集合对象|

#### 编码和底层实现

|编码常量 |	编码所对应的底层数据结构|
|--|--|
|REDIS_ENCODING_INT |	long 类型的整数|
|REDIS_ENCODING_EMBSTR |	embstr 编码的简单动态字符串|
|REDIS_ENCODING_RAW |	简单动态字符串|
|REDIS_ENCODING_HT |	字典|
|REDIS_ENCODING_LINKEDLIST |	双端链表|
|REDIS_ENCODING_ZIPLIST |	压缩列表|
|REDIS_ENCODING_INTSET |	整数集合|
|REDIS_ENCODING_SKIPLIST |	跳跃表和字典|

每种类型的对象都至少使用了两种不同的编码， 下面列出了每种类型的对象可以使用的编码

|类型 |	编码 |	对象|
|--|--|--|
|REDIS_STRING |	REDIS_ENCODING_INT |	使用整数值实现的字符串对象。|
|REDIS_STRING |	REDIS_ENCODING_EMBSTR |	使用 embstr 编码的简单动态字符串实现的字符串对象。|
|REDIS_STRING |	REDIS_ENCODING_RAW |	使用简单动态字符串实现的字符串对象。|
|REDIS_LIST |	REDIS_ENCODING_ZIPLIST |	使用压缩列表实现的列表对象。|
|REDIS_LIST |	REDIS_ENCODING_LINKEDLIST |	使用双端链表实现的列表对象。|
|REDIS_HASH |	REDIS_ENCODING_ZIPLIST |	使用压缩列表实现的哈希对象。|
|REDIS_HASH |	REDIS_ENCODING_HT |	使用字典实现的哈希对象。|
|REDIS_SET |	REDIS_ENCODING_INTSET |	使用整数集合实现的集合对象。|
|REDIS_SET |	REDIS_ENCODING_HT |	使用字典实现的集合对象。|
|REDIS_ZSET |	REDIS_ENCODING_ZIPLIST |	使用压缩列表实现的有序集合对象。|
|REDIS_ZSET |	REDIS_ENCODING_SKIPLIST |	使用跳跃表和字典实现的有序集合对象。|

#### 实例分析

- 在列表对象包含的元素比较少时，Redis 使用压缩列表作为列表对象的底层实现
- 因为压缩列表比双端链表更节约内存，并且在元素数量较少时，在内存中以连续块方式保存的压缩列表比起双端链表可以更快被载入到缓存中
- 随着列表对象包含的元素越来越多，使用压缩列表来保存元素的优势逐渐消失时，对象就会将底层实现从压缩列表转向功能更强、也更适合保存大量元素的双端链表上面

### 字符串对象
字符串对象的编码可以是 int 、 raw 或者 embstr 字符串对象是 Redis 五种类型的对象中唯一一种会被其他四种类型对象嵌套的对象

- 如果一个字符串对象保存的是整数值， 并且这个整数值可以用 long 类型来表示， 那么字符串对象会将整数值保存在字符串对象结构的 ptr 属性里面（将 void* 转换成 long ）， 并将字符串对象的编码设置为 int
- 如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度大于 39 字节， 那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值， 并将对象的编码设置为 raw 。
- 如果字符串对象保存的是一个字符串值， 并且这个字符串值的长度小于等于 39 字节， 那么字符串对象将使用 embstr 编码的方式来保存这个字符串值。

#### raw和embstr

- 都使用redisobject和sdshdr来表示字符串对象
- raw 编码会调用两次内存分配函数来分别创建 redisObject 结构和 sdshdr 结构
- embstr 编码则通过调用一次内存分配函数来分配一块连续的空间， 空间中依次包含 redisObject 和 sdshdr 两个结构

![embstr编码的字符串对象](https://github.com/Yehehui/redis-learning/blob/main/image/embstr%E7%BC%96%E7%A0%81%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%AF%B9%E8%B1%A1.png)
![raw编码的字符串对象](https://github.com/Yehehui/redis-learning/blob/main/image/raw%E7%BC%96%E7%A0%81%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%AF%B9%E8%B1%A1.png)

#### 编码转换
int 编码的字符串对象和 embstr 编码的字符串对象在条件满足的情况下， 会被转换为 raw 编码的字符串对象(不会转化成embstr)

#### 字符串命令实现

|命令 |int 编码的实现方法 |embstr 编码的实现方法 |raw 编码的实现方法|
|--|--|--|--|
|SET |	使用 int 编码保存值。| 	使用 embstr 编码保存值。 |	使用 raw 编码保存值。|
|GET |	拷贝对象所保存的整数值， 将这个拷贝转换成字符串值， 然后向客户端返回这个字符串值。| 	直接向客户端返回字符串值。| 	直接向客户端返回字符串值。|
|APPEND |	将对象转换成 raw 编码， 然后按 raw 编码的方式执行此操作。| 	将对象转换成 raw 编码， 然后按 raw 编码的方式执行此操作。| 	调用 sdscatlen 函数， 将给定字符串追加到现有字符串的末尾。|
|INCRBYFLOAT |	取出整数值并将其转换成 long double 类型的浮点数， 对这个浮点数进行加法计算， 然后将得出的浮点数结果保存起来。| 	取出字符串值并尝试将其转换成 long double 类型的浮点数， 对这个浮点数进行加法计算， 然后将得出的浮点数结果保存起来。 如果字符串值不能被转换成浮点数， 那么向客户端返回一个错误。| 	取出字符串值并尝试将其转换成 long double 类型的浮点数， 对这个浮点数进行加法计算， 然后将得出的浮点数结果保存起来。 如果字符串值不能被转换成浮点数， 那么向客户端返回一个错误。|
|INCRBY |	对整数值进行加法计算， 得出的计算结果会作为整数被保存起来。| 	embstr 编码不能执行此命令， 向客户端返回一个错误。| 	raw 编码不能执行此命令， 向客户端返回一个错误。|
|DECRBY |	对整数值进行减法计算， 得出的计算结果会作为整数被保存起来。| 	embstr 编码不能执行此命令， 向客户端返回一个错误。| 	raw 编码不能执行此命令， 向客户端返回一个错误。|
|STRLEN |	拷贝对象所保存的整数值， 将这个拷贝转换成字符串值， 计算并返回这个字符串值的长度。| 	调用 sdslen 函数， 返回字符串的长度。| 	调用 sdslen 函数， 返回字符串的长度。|
|SETRANGE |	将对象转换成 raw 编码， 然后按 raw 编码的方式执行此命令。 |	将对象转换成 raw 编码， 然后按 raw 编码的方式执行此命令。|	将字符串特定索引上的值设置为给定的字符。|
|GETRANGE |	拷贝对象所保存的整数值， 将这个拷贝转换成字符串值， 然后取出并返回字符串指定索引上的字符。| 	直接取出并返回字符串指定索引上的字符。| 	直接取出并返回字符串指定索引上的字符。|

### 列表对象
列表对象的编码可以是 ziplist 或者 linkedlist

#### 编码转换

- 列表对象保存的所有字符串元素的长度都小于 64 字节
- 列表对象保存的元素数量小于 512 个

满足以上条件时，使用ziplist作为底层实现，否则使用linkedlist
以上两个条件的上限值可通过配置文件中 list-max-ziplist-value 选项和 list-max-ziplist-entries 选项进行修改

#### 列表对象命令实现

|命令 |	ziplist 编码的实现方法 |	linkedlist 编码的实现方法|
|--|--|--|
|LPUSH |	调用 ziplistPush 函数， 将新元素推入到压缩列表的表头。| 	调用 listAddNodeHead 函数， 将新元素推入到双端链表的表头。|
|RPUSH |	调用 ziplistPush 函数， 将新元素推入到压缩列表的表尾。| 	调用 listAddNodeTail 函数， 将新元素推入到双端链表的表尾。|
|LPOP |	调用 ziplistIndex 函数定位压缩列表的表头节点， 在向用户返回节点所保存的元素之后， 调用 ziplistDelete 函数删除表头节点。| 	调用 listFirst 函数定位双端链表的表头节点， 在向用户返回节点所保存的元素之后， 调用 listDelNode 函数删除表头节点。|
|RPOP |	调用 ziplistIndex 函数定位压缩列表的表尾节点， 在向用户返回节点所保存的元素之后， 调用 ziplistDelete 函数删除表尾节点。| 	调用 listLast 函数定位双端链表的表尾节点， 在向用户返回节点所保存的元素之后， 调用 listDelNode 函数删除表尾节点。|
|LINDEX |	调用 ziplistIndex 函数定位压缩列表中的指定节点， 然后返回节点所保存的元素。 |	调用 listIndex 函数定位双端链表中的指定节点， 然后返回节点所保存的元素。|
|LLEN |	调用 ziplistLen 函数返回压缩列表的长度。| 	调用 listLength 函数返回双端链表的长度。|
|LINSERT |	插入新节点到压缩列表的表头或者表尾时， 使用 ziplistPush 函数； 插入新节点到压缩列表的其他位置时， 使用 ziplistInsert 函数。| 	调用 listInsertNode 函数， 将新节点插入到双端链表的指定位置。|
|LREM |	遍历压缩列表节点， 并调用 ziplistDelete 函数删除包含了给定元素的节点。 |	遍历双端链表节点， 并调用 listDelNode 函数删除包含了给定元素的节点。|
|LTRIM |	调用 ziplistDeleteRange 函数， 删除压缩列表中所有不在指定索引范围内的节点。| 	遍历双端链表节点， 并调用 listDelNode 函数删除链表中所有不在指定索引范围内的节点。|
|LSET |	调用 ziplistDelete 函数， 先删除压缩列表指定索引上的现有节点， 然后调用 ziplistInsert 函数， 将一个包含给定元素的新节点插入到相同索引上面。| 	调用 listIndex 函数， 定位到双端链表指定索引上的节点， 然后通过赋值操作更新节点的值。|

### 哈希对象
哈希对象的编码可以是 ziplist 或者 hashtable 

使用ziplist时

- 保存了同一键值对的两个节点总是紧挨在一起， 保存键的节点在前， 保存值的节点在后；
- 先添加到哈希对象中的键值对会被放在压缩列表的表头方向， 而后来添加到哈希对象中的键值对会被放在压缩列表的表尾方向

#### 编码转换

- 哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节
- 哈希对象保存的键值对数量小于 512 个

满足以上条件时，使用ziplist作为底层实现，否则使用hashtable
以上两个条件的上限值可通过配置文件中 hash-max-ziplist-value 选项和 hash-max-ziplist-entries 选项进行修改

#### 哈希对象命令实现

|命令| 	ziplist 编码实现方法| 	hashtable 编码的实现方法|
|--|--|--|
|HSET |	首先调用 ziplistPush 函数， 将键推入到压缩列表的表尾， 然后再次调用 ziplistPush 函数， 将值推入到压缩列表的表尾。 |	调用 dictAdd 函数， 将新节点添加到字典里面。|
|HGET |	首先调用 ziplistFind 函数， 在压缩列表中查找指定键所对应的节点， 然后调用 ziplistNext 函数， 将指针移动到键节点旁边的值节点， 最后返回值节点。| 	调用 dictFind 函数， 在字典中查找给定键， 然后调用 dictGetVal 函数， 返回该键所对应的值。|
|HEXISTS |	调用 ziplistFind 函数， 在压缩列表中查找指定键所对应的节点， 如果找到的话说明键值对存在， 没找到的话就说明键值对不存在。 |	调用 dictFind 函数， 在字典中查找给定键， 如果找到的话说明键值对存在， 没找到的话就说明键值对不存在。|
|HDEL |	调用 ziplistFind 函数， 在压缩列表中查找指定键所对应的节点， 然后将相应的键节点、 以及键节点旁边的值节点都删除掉。 |	调用 dictDelete 函数， 将指定键所对应的键值对从字典中删除掉。|
|HLEN |	调用 ziplistLen 函数， 取得压缩列表包含节点的总数量， 将这个数量除以 2 ， 得出的结果就是压缩列表保存的键值对的数量。 |	调用 dictSize 函数， 返回字典包含的键值对数量， 这个数量就是哈希对象包含的键值对数量。|
|HGETALL |	遍历整个压缩列表， 用 ziplistGet 函数返回所有键和值（都是节点）。 |	遍历整个字典， 用 dictGetKey 函数返回字典的键， 用 dictGetVal 函数返回字典的值。|

### 集合对象
集合对象的编码可以是 intset 或者 hashtable

#### 示例
![](https://github.com/Yehehui/redis-learning/blob/main/image/intset%E7%BC%96%E7%A0%81%E7%9A%84%E9%9B%86%E5%90%88%E5%AF%B9%E8%B1%A1.png)

集合对象使用字典作为底层实现时， 字典的每个键都是一个字符串对象， 每个字符串对象包含了一个集合元素， 而字典的值则全部被设置为 NULL

![](https://github.com/Yehehui/redis-learning/blob/main/image/hashtable%E7%BC%96%E7%A0%81%E7%9A%84%E9%9B%86%E5%90%88%E5%AF%B9%E8%B1%A1.png)

#### 编码转换

- 集合对象保存的所有元素都是整数值
- 集合对象保存的元素数量不超过 512 个

满足以上条件时，使用intset作为底层实现，否则使用hashtable
第两个条件的上限值可通过配置文件中 set-max-intset-entries 选项进行修改

#### 集合对象命令实现

|命令 |	intset 编码的实现方法 |	hashtable 编码的实现方法|
|--|--|--|
|SADD |	调用 intsetAdd 函数， 将所有新元素添加到整数集合里面。 |	调用 dictAdd ， 以新元素为键， NULL 为值， 将键值对添加到字典里面。|
|SCARD |	调用 intsetLen 函数， 返回整数集合所包含的元素数量， 这个数量就是集合对象所包含的元素数量。| 	调用 dictSize 函数， 返回字典所包含的键值对数量， 这个数量就是集合对象所包含的元素数量。|
|SISMEMBER |	调用 intsetFind 函数， 在整数集合中查找给定的元素， 如果找到了说明元素存在于集合， 没找到则说明元素不存在于集合。 |	调用 dictFind 函数， 在字典的键中查找给定的元素， 如果找到了说明元素存在于集合， 没找到则说明元素不存在于集合。|
|SMEMBERS |	遍历整个整数集合， 使用 intsetGet 函数返回集合元素。 |	遍历整个字典， 使用 dictGetKey 函数返回字典的键作为集合元素。|
|SRANDMEMBER |	调用 intsetRandom 函数， 从整数集合中随机返回一个元素。 |	调用 dictGetRandomKey 函数， 从字典中随机返回一个字典键。|
|SPOP |	调用 intsetRandom 函数， 从整数集合中随机取出一个元素， 在将这个随机元素返回给客户端之后， 调用 intsetRemove 函数， 将随机元素从整数集合中删除掉。 |	调用 dictGetRandomKey 函数， 从字典中随机取出一个字典键， 在将这个随机字典键的值返回给客户端之后， 调用 dictDelete 函数， 从字典中删除随机字典键所对应的键值对。|
|SREM |	调用 intsetRemove 函数， 从整数集合中删除所有给定的元素。 |	调用 dictDelete 函数， 从字典中删除所有键为给定元素的键值对。|

### 有序对象集合
有序集合的编码可以是 ziplist 或者 skiplist

#### 底层实现

##### 使用ziplist编码

- 每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员（member），而第二个元素则保存元素的分值（score）
- 压缩列表内的集合元素按分值从小到大进行排序，分值较小的元素被放置在靠近表头的方向，而分值较大的元素则被放置在靠近表尾的方向

##### 使用skiplist编码

使用zset作为底层实现
``` c
typedef struct zset {
    zskiplist *zsl;
    dict *dict;
} zset;
```

- zsl 跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素：跳跃表节点的 object 属性保存了元素的成员，而跳跃表节点的 score 属性则保存了元素的分值。通过这个跳跃表， 程序可以快速对有序集合进行范围型操作
- dict 字典为有序集合创建了一个从成员到分值的映射，字典中的每个键值对都保存了一个集合元素：字典的键保存了元素的成员，而字典的值则保存了元素的分值。通过这个字典，程序可以用 O(1) 复杂度查找给定成员的分值
- 两种数据结构都会通过指针来共享相同元素的成员和分值，所以同时使用跳跃表和字典来保存集合元素不会产生任何重复成员或者分值，也不会因此而浪费额外的内存

![](https://github.com/Yehehui/redis-learning/blob/main/image/skiplist%E7%BC%96%E7%A0%81%E7%9A%84%E6%9C%89%E5%BA%8F%E9%9B%86%E5%90%88%E5%AF%B9%E8%B1%A1.png)

#### 编码转换

- 有序集合保存的元素数量小于 128 个
- 有序集合保存的所有元素成员的长度都小于 64 字节

满足以上条件时，使用ziplist作为底层实现，否则使用zset
以上两个条件的上限值可通过配置文件中 zset-max-ziplist-entries 选项和 zset-max-ziplist-value 选项进行修改

#### 有序集合对象命令实现

|命令 |	ziplist 编码的实现方法 |	zset 编码的实现方法|
|--|--|--|
|ZADD |	调用 ziplistInsert 函数， 将成员和分值作为两个节点分别插入到压缩列表。 |	先调用 zslInsert 函数， 将新元素添加到跳跃表， 然后调用 dictAdd 函数， 将新元素关联到字典。|
|ZCARD |	调用 ziplistLen 函数， 获得压缩列表包含节点的数量， 将这个数量除以 2 得出集合元素的数量。 |	访问跳跃表数据结构的 length 属性， 直接返回集合元素的数量。|
|ZCOUNT |	遍历压缩列表， 统计分值在给定范围内的节点的数量。 |	遍历跳跃表， 统计分值在给定范围内的节点的数量。|
|ZRANGE |	从表头向表尾遍历压缩列表， 返回给定索引范围内的所有元素。 |	从表头向表尾遍历跳跃表， 返回给定索引范围内的所有元素。|
|ZREVRANGE |	从表尾向表头遍历压缩列表， 返回给定索引范围内的所有元素。 |	从表尾向表头遍历跳跃表， 返回给定索引范围内的所有元素。|
|ZRANK |	从表头向表尾遍历压缩列表， 查找给定的成员， 沿途记录经过节点的数量， 当找到给定成员之后， 途经节点的数量就是该成员所对应元素的排名。 |	从表头向表尾遍历跳跃表， 查找给定的成员， 沿途记录经过节点的数量， 当找到给定成员之后， 途经节点的数量就是该成员所对应元素的排名。|
|ZREVRANK |	从表尾向表头遍历压缩列表， 查找给定的成员， 沿途记录经过节点的数量， 当找到给定成员之后， 途经节点的数量就是该成员所对应元素的排名。 |	从表尾向表头遍历跳跃表， 查找给定的成员， 沿途记录经过节点的数量， 当找到给定成员之后， 途经节点的数量就是该成员所对应元素的排名。|
|ZREM |	遍历压缩列表， 删除所有包含给定成员的节点， 以及被删除成员节点旁边的分值节点。 |	遍历跳跃表， 删除所有包含了给定成员的跳跃表节点。 并在字典中解除被删除元素的成员和分值的关联。|
|ZSCORE |	遍历压缩列表， 查找包含了给定成员的节点， 然后取出成员节点旁边的分值节点保存的元素分值。| 	直接从字典中取出给定成员的分值。|

### 类型检查与命令多态

#### 命令类型

1. 可对任意类型的键操作，如DEL 命令、 EXPIRE 命令、RENAME 命令、TYPE 命令、OBJECT 命令等
2. 只能对特定类型的键进行操作，如:
   - SET 、 GET 、 APPEND 、 STRLEN 等命令只能对字符串键执行；
   - HDEL 、 HSET 、 HGET 、 HLEN 等命令只能对哈希键执行；
   - RPUSH 、 LPOP 、 LINSERT 、 LLEN 等命令只能对列表键执行；
   - SADD 、 SPOP 、 SINTER 、 SCARD 等命令只能对集合键执行；
   - ZADD 、 ZCARD 、 ZRANK 、 ZSCORE 等命令只能对有序集合键执行；

#### 类型检查实现
类型特定命令所进行的类型检查是通过 redisObject 结构的 type 属性来实现的

- 在执行一个类型特定命令之前，服务器会先检查输入数据库键的值对象是否为执行命令所需的类型，即查询值对象redisObject结构type属性，如果是的话， 服务器就对键执行指定的命令
- 否则， 服务器将拒绝执行命令，并向客户端返回一个类型错误

#### 多态命令实现
执行特定类型命令时，根据encoding进行不同的操作，如:

LLEN命令:
- 如果列表对象的编码为 ziplist，那么说明列表对象的实现为压缩列表，程序将使用 ziplistLen 函数来返回列表的长度
- 如果列表对象的编码为 linkedlist，那么说明列表对象的实现为双端链表，程序将使用 listLength 函数来返回双端链表的长度

DEL等命令为基于类型的多态，LLEN等命令为基于编码的多态

### 内存回收
redis使用引用计数系统进行内存回收

#### 实现
``` c
typedef struct redisObject {
    // ...
    // 引用计数
    int refcount;
    // ...
} robj;
```

- 在创建一个新对象时， 引用计数的值会被初始化为 1 ；
- 当对象被一个新程序使用时， 它的引用计数值会被增一；
- 当对象不再被一个程序使用时， 它的引用计数值会被减一；
- 当对象的引用计数值变为 0 时， 对象所占用的内存会被释放。

#### 引用计数相关API

|函数 |	作用|
|--|--|
|incrRefCount |	将对象的引用计数值增一。|
|decrRefCount |	将对象的引用计数值减一， 当对象的引用计数值等于 0 时， 释放对象。|
|resetRefCount |	将对象的引用计数值设置为 0 ， 但并不释放对象， 这个函数通常在需要重新设置对象的引用计数值时使用。|

### 对象共享
redis的引用计数系统不止用于内存回收，也用于对象共享

当我们创建了一个值为100的键A后，再创建一个值为100的键B，键B共享键A的redisObject

![](https://github.com/Yehehui/redis-learning/blob/main/image/%E5%85%B1%E4%BA%AB%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%AF%B9%E8%B1%A1.png)

 Redis 会在初始化服务器时， 创建一万个字符串对象， 这些对象包含了从 0 到 9999 的所有整数值， 当服务器需要用到值为 0 到 9999 的字符串对象时， 服务器就会使用这些共享对象， 而不是新创建对象。数量可以通过修改 redis.h/REDIS_SHARED_INTEGERS 常量来修改
 
共享对象不单单只有字符串键可以使用，那些在数据结构中嵌套了字符串对象的对象（linkedlist 编码的列表对象、hashtable 编码的哈希对象、hashtable 编码的集合对象、以及 zset 编码的有序集合对象）都可以使用这些共享对象

### 对象空转时长
``` c
typedef struct redisObject {
    // ...
    // 记录对象最后一次被访问的时间
    unsigned lru:22;
    // ...
} robj;
```

#### 作用

- OBJECT IDLETIME 命令可以打印出给定键的空转时长，这一空转时长就是通过将当前时间减去键的值对象的 lru 时间计算得出的(这个命令不会影响对象的lru属性)
- 如果服务器打开了 maxmemory 选项，并且服务器用于回收内存的算法为 volatile-lru 或者 allkeys-lru，那么当服务器占用的内存数超过了 maxmemory 选项所设置的上限值时，空转时长较高的那部分键会优先被服务器释放，从而回收内存

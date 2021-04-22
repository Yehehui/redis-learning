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
![](https://github.com/Yehehui/redis-learning/blob/main/image/%E5%8E%8B%E7%BC%A9%E5%88%97%E8%A1%A8%E7%BB%84%E6%88%90%E9%83%A8%E5%88%86.png)

|属性|类型|长度|用途|
|--|--|---|--|
|zlbytes|uint32_t|4 字节|记录整个压缩列表占用的内存字节数：在对压缩列表进行内存重分配， 或者计算 zlend 的位置时使用。|
|zltail|uint32_t|4 字节|记录压缩列表表尾节点距离压缩列表的起始地址有多少字节： 通过这个偏移量，程序无须遍历整个压缩列表就可以确定表尾节点的地址。|
|zllen|uint16_t|2 字节|记录了压缩列表包含的节点数量： 当这个属性的值小于 UINT16_MAX （65535）时， 这个属性的值就是压缩列表包含节点的数量； 当这个值等于 UINT16_MAX 时， 节点的真实数量需要遍历整个压缩列表才能计算得出。|
|entryX|列表节点 |不定  |压缩列表包含的各个节点，节点的长度由节点保存的内容决定。|
|zlend|uint8_t|1 字节|特殊值 0xFF （十进制 255 ），用于标记压缩列表的末端。|

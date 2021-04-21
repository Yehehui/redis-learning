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
[SDS API]（http://redisbook.com/preview/sds/api.html）

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
[链表和链表节点的 API]（http://redisbook.com/preview/adlist/api.html）

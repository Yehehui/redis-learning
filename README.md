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



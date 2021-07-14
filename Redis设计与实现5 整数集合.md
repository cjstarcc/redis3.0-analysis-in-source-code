# 整数集合

整数集合是集合键的底层实现之一 ，当一个集合只包含整数值元素，并且这个自己的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。



## 整数集合的实现



```c
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```


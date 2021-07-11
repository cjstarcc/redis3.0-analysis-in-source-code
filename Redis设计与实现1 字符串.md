[toc]

# Redis设计与实现  

## 简单动态字符串

### SDS的定义

```c
struct sdshdr {
    
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```

`free`属性的值为0，表示这个SDS没有分配任何未使用空间。

`len`属性值为5，表示这个SDS保存了一个五字符串。

`buf`属性值是一个char类型的数组。这里的值，表示存储的C类型的字符串。



#### SDS初始化代码

```c
/*
 * 根据给定的初始化字符串 init 和字符串长度 initlen
 * 创建一个新的 sds
 *
 * 参数
 *  init ：初始化字符串指针
 *  initlen ：初始化字符串的长度
 *
 * 返回值
 *  sds ：创建成功返回 sdshdr 相对应的 sds
 *        创建失败返回 NULL
 *
 * 复杂度
 *  T = O(N)
 */
sds sdsnewlen(const void *init, size_t initlen) {

    struct sdshdr *sh;

    // 根据是否有初始化内容，选择适当的内存分配方式
    // T = O(N)
    if (init) {
        // zmalloc 不初始化所分配的内存
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    } else {
        // zcalloc 将分配的内存全部初始化为 0
        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
    }

    // 内存分配失败，返回
    if (sh == NULL) return NULL;

    // 设置初始化长度
    sh->len = initlen;
    // 新 sds 不预留任何空间
    sh->free = 0;
    // 如果有指定初始化内容，将它们复制到 sdshdr 的 buf 中
    // T = O(N)
    if (initlen && init)
        memcpy(sh->buf, init, initlen);
    // 以 \0 结尾
    sh->buf[initlen] = '\0';

    // 返回 buf 部分，而不是整个 sdshdr
    return (char*)sh->buf;
}
```

这里注意一下返回的是`SDS->buf` 而不是完整的`SDS`结构体,如果想得到完整的`SDS`结构体怎么办呢？通过使用这行代码

```c
struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)))
```

 详情请看[struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)))讲解](https://blog.csdn.net/u014303647/article/details/88740109)

#### SDS复制代码

```c
/*
 * 复制给定 sds 的副本
 *
 * 返回值
 *  sds ：创建成功返回输入 sds 的副本
 *        创建失败返回 NULL
 *
 * 复杂度
 *  T = O(N)
 */

sds sdsdup(const sds s) {
    return sdsnewlen(s, sdslen(s));
}
/*
 * 返回 sds 实际保存的字符串的长度
 *
 * T = O(1)
 */
static inline size_t sdslen(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->len;
}
```

这段代码如果掌握了[之前的知识](####SDS初始化代码)，那么就知道`const sds s` 只是`sds->buf`，因此可以直接用memcpy拷贝。

### SDS与C字符串的区别

#### 1.常熟复杂度获取字符串的长度

在C字符串中，我们获取字符串的长度是通过`strlen`这个库函数生成的，每次获取长度都要调用一次，并且需要遍历整个字符串，复杂度为O(N)，但是`SDS`是通过结构体中的`len `属性获取的，直接打印`len`值就能获取长度复杂度仅为O(1)。

#### 2.杜绝缓冲区溢出

C字符串在执行`strcat`函数时候，如果没有为dest分配猪狗多的内存，就容易造成缓冲区溢出。

但是`SDS`的空间分配策略完全杜绝了发生缓冲区溢出的可能性：当`API`需要对`SDS`发生修改时，会先检查`SDS`的空间是否满足修改所需的要求，如果不满足的话，`API`会自动将`SDS`的空间扩展至执行修改所需的大小，然后才执行实际的修改操作。

#### 3.减少修改字符串带来的内存重分配次数

由于C字符串不计算自己的长度，所以每增长或者缩短一个C字符串，程序都要对保存这个C字符串数组进行一次内存重新分配的操作：

1.如果程序执行的是增长字符串的操作，比如拼接操作，那么在执行这个操作之前，程序需要先通过内存重新分配来扩展底层的数组空间大小，假如忘了这一步就会产生缓存溢出。

2.如果程序执行的是缩短字符串的操作，比如说截断操作，那么在执行这个操作之后，程序需要通过内存重新分配来释放字符串不再使用的那部分空间，如果忘了这一步就会产生内存泄漏。

如果是一般程序中，每一次修改都需要执行内存重新分配是可以接受的，但是redis经常用于速度要求严苛，数据被频繁修改的场合，如果每次修改字符串的长度都需要执行一次内存重新分配的话，对性能的影响很大。

所以，为了避免这种情况，`SDS` 用了空间预分配和惰性释放空间的两种策略。

1.如果对`SDS`进行修改之后，`SDS`的长度如果小于`1MB`那么对于程序的分配和`len`属性同样大小的未使用空间（即`free`的空间）。假如`SDS`的`len`变成了`13字节`，那么总大小等于 `13字节 +13字节 +1字节 = 27字节`

2.如果对`SDS`进行修改之后，`SDS`的长度如果大于`1MB`,那么程序将会分配`1MB`的未使用空间。假如`SDS`的`len`变成了`30MB`，那么总大小等于 `30MB + 1MB = 31MB`

```c
/*
 * 最大预分配长度
 */
#define SDS_MAX_PREALLOC (1024*1024)
/*
 * 将另一个 sds 追加到一个 sds 的末尾
 * 
 * 返回值
 *  sds ：追加成功返回新 sds ，失败返回 NULL
 *
 * 复杂度
 *  T = O(N)
 */
sds sdscatsds(sds s, const sds t) {
    return sdscatlen(s, t, sdslen(t));
}
/*
 * 将长度为 len 的字符串 t 追加到 sds 的字符串末尾
 *
 * 返回值
 *  sds ：追加成功返回新 sds ，失败返回 NULL
 *
 * 复杂度
 *  T = O(N)
 */
sds sdscatlen(sds s, const void *t, size_t len) {
    
    struct sdshdr *sh;
    
    // 原有字符串长度
    size_t curlen = sdslen(s);

    // 扩展 sds 空间
    // T = O(N)
    s = sdsMakeRoomFor(s,len);

    // 内存不足？直接返回
    if (s == NULL) return NULL;

    // 复制 t 中的内容到字符串后部
    // T = O(N)
    sh = (void*) (s-(sizeof(struct sdshdr)));
    memcpy(s+curlen, t, len);

    // 更新属性
    sh->len = curlen+len;
    sh->free = sh->free-len;

    // 添加新结尾符号
    s[curlen+len] = '\0';

    // 返回新 sds
    return s;
}

/*
 * 对 sds 中 buf 的长度进行扩展，确保在函数执行之后，
 * buf 至少会有 addlen + 1 长度的空余空间
 * （额外的 1 字节是为 \0 准备的）
 *
 * 返回值
 *  sds ：扩展成功返回扩展后的 sds
 *        扩展失败返回 NULL
 *
 * 复杂度
 *  T = O(N)
 */
sds sdsMakeRoomFor(sds s, size_t addlen) {

    struct sdshdr *sh, *newsh;

    // 获取 s 目前的空余空间长度
    size_t free = sdsavail(s);

    size_t len, newlen;

    // s 目前的空余空间已经足够，无须再进行扩展，直接返回
    if (free >= addlen) return s;

    // 获取 s 目前已占用空间的长度
    len = sdslen(s);
    sh = (void*) (s-(sizeof(struct sdshdr)));

    // s 最少需要的长度
    newlen = (len+addlen);

    // 根据新长度，为 s 分配新空间所需的大小
    if (newlen < SDS_MAX_PREALLOC)
        // 如果新长度小于 SDS_MAX_PREALLOC 
        // 那么为它分配两倍于所需长度的空间
        newlen *= 2;
    else
        // 否则，分配长度为目前长度加上 SDS_MAX_PREALLOC
        newlen += SDS_MAX_PREALLOC;
    // T = O(N)
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);

    // 内存不足，分配失败，返回
    if (newsh == NULL) return NULL;

    // 更新 sds 的空余长度
    newsh->free = newlen - len;

    // 返回 sds
    return newsh->buf;
}
/*
 * 返回 sds 可用空间的长度
 *
 * T = O(1)
 */
static inline size_t sdsavail(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr)));
    return sh->free;
}
```

3.惰性释放空间用于优化`SDS`缩短的操作：当`SDS`的`API`需要缩短`SDS`保存的字符串时，程序并不立即使用重新分配来回收缩短后多出来的字节，而是使用`free`属性将这些字节数量记录起来，并等待将来使用。

```c
/*
 * 在不释放 SDS 的字符串空间的情况下，
 * 重置 SDS 所保存的字符串为空字符串。
 *
 * 复杂度
 *  T = O(1)
 */
void sdsclear(sds s) {

    // 取出 sdshdr
    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));

    // 重新计算属性
    sh->free += sh->len;
    sh->len = 0;

    // 将结束符放到最前面（相当于惰性地删除 buf 中的内容）
    sh->buf[0] = '\0';
}
/*
 * 对 sds 左右两端进行修剪，清除其中 cset 指定的所有字符
 *
 * 比如 sdsstrim(xxyyabcyyxy, "xy") 将返回 "abc"
 *
 * 复杂性：
 *  T = O(M*N)，M 为 SDS 长度， N 为 cset 长度。
 */
sds sdstrim(sds s, const char *cset) {
    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
    char *start, *end, *sp, *ep;
    size_t len;

    // 设置和记录指针
    sp = start = s;
    ep = end = s+sdslen(s)-1;

    // 修剪, T = O(N^2)
    while(sp <= end && strchr(cset, *sp)) sp++;
    while(ep > start && strchr(cset, *ep)) ep--;

    // 计算 trim 完毕之后剩余的字符串长度
    len = (sp > ep) ? 0 : ((ep-sp)+1);
    
    // 如果有需要，前移字符串内容
    // T = O(N)
    if (sh->buf != sp) memmove(sh->buf, sp, len);

    // 添加终结符
    sh->buf[len] = '\0';

    // 更新属性
    sh->free = sh->free+(sh->len-len);
    sh->len = len;

    // 返回修剪后的 sds
    return s;
}
```

#### 4.二进制安全

由于C字符串中的字符必须符合某种编码（比如ASCII），并且除了字符串的末尾之外，字符串里面也不能包含空字符，否则最先被程序读入的空字符将被认为是字符串结尾，所以C字符串只能保存文本数据，而不能保存图片，音频，视屏，压缩文件这样的二进制数据。

`SDS`的`API`都是二进制安全的，都会以二进制来处理`SDS`存放在数组里的数据，程序不会对其中的数据做任何限制，过滤和假设，数据在之前写入是怎么样的，读出来就是怎么样的。



#### 5.兼容部分字符串函数

`SDS`他们遵循C字符串以空字符结尾的管理：这些API总会将SDS博爱村的数据的末尾设置为空字符，并且，总会在位buf数组分配空间是多分配一个字节来容纳这个空字符。因此他们兼容部分C字符串函数



## 引用

> 《redis设计与实现》
>
> https://blog.csdn.net/u014303647/article/details/88740109
>
> [带有注释的redis-3.0源码](https://github.com/huangz1990/redis-3.0-annotated)


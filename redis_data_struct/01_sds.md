# redis 底层数据结构(1)——sds

从 redis 内部实现的角度来说，redis 提供了 sds, dict, skiplist, ziplist 和 quicklist 数据结构，初步计划分为以下系列文章对 redis 数据结构进行归纳总结：
* redis 底层数据结构(1)——sds
* redis 底层数据结构(2)——dict
* redis 底层数据结构(3)——skiplist
* redis 底层数据结构(4)——ziplist
* redis 底层数据结构(5)——quicklist
* redis 底层数据结构(6)——intset

从使用者的角度来说，redis 提供了 string（字符串），list（列表），hash（哈希），set（集合）和 sorted set（有序集合）五种数据类型，这些将在 redis 对象系列文章进行归纳学习。

Redis 暴露给用户的字符串数据类型（string），在其内部使用 SDS 数据结构来存储字符串变量的。本文基于 redis-6.0.5 整理，完成时间：2020.06.13。

## 1 sds 的数据结构
存储字符串变量时，Redis 没有使用 C 语言的字符串表示方法——以空字符'\0'结尾的字符数组，而是定义了 SDS（Simple Dynamic String，简单动态字符串）的抽象类型，并将 SDS 作为 Redis 的默认字符串表示。

当存储的是一个可以被修改的字符串值时，Redis 就会使用 SDS 来表示字符串值，在 Redis 的数据库里，包含字符串值的键值对在底层都是由 SDS 实现的，同时采用预分配冗余空间的方式来减少内存的频繁分配。

以下分析基于 redis 的存储的值是字符串类型，当存储的是整型数据时，数据结构不再是 sds 了，需单独分析。

【源码】
```
// redis-6.0.5/src/sds.h
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

// 以下 5 个常量表示 5 种 sds struct 类型，即 flags 的有效取值范围
// sds struct 类型: sdshdr5, sdshdr8, sdshdr16, sdshdr32, sdshdr64
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```
【解析】
* 1 计算机及 c 语言基础：
    * msb: Most Significant Bit, 最高有效位；lsb: Least Significant Bit, 最低有效位
    * 使用 __attribute__ ((__packed__)) 声明结构体的作用：让编译器以紧凑模式来分配内存，如果不声明该属性，编译器可能会为结构体成员做内存对齐优化，在其中填充空字符。这样就不能保证 结构体成员在内存上是紧凑相邻的，也不能通过 buf 向低地址偏移一个字节来获取 flags 字段的值
    * char buf[]：一个没有指定长度的字符数组，这是 c 语言定义字符数组的一种特殊写法，称为 flexible array member，只能定义在一个结构体的最后一个成员。buf 起标记作用，表示在 flags 字段后面就是一个字符数组，程序在为 sdshdr 分配内存时，它并不占用内存空间。
    * sizeof(struct sdshdr5) == 1; sizeof(struct sdshdr8) == 3; sizeof(struct sdshdr16) == 5; sizeof(struct sdshdr32) == 9;  sizeof(struct sdshdr64) == 17; 
* 2 sdshdr5 的成员分析：
    * flags：3 lsb of type, and 5 msb of string length，低 3 位用于表示 sds struct 的类型，即使用哪种 sdshdr，5 位高有效位用于记录字符串的长度。（数字 5 很重要，后续字会以 2^5 作为字符串长度的分界线，从而决定选择哪种 sdshdr）
    * buf: 存储字符串内容的字符数组，其长度等于 alloc+1。一般情况下，buf 字符数组的最大容量大于字符串的真实长度，未使用的字符数组空间一般用'\0'字符填充，这样可实现在不重新分配内存的前提下实现字符串扩展。为了兼容 C 字符串，在真正字符串之后，还有一个 NULL 结束符（即 ASCII 码为 0 的 '\0' 字符），所以 buf 的长度比最大容量多一个字节。而多出来的这个字节空间正是为了在字符串长度达到最大容量时，仍有足够的空间存放'\0'字符。
* 3 sdshdr8, sdshdr16, sdshdr32, sdshdr64 的成员分析：
    * len：表示字符串的实际长度，不包含 C 字符串结尾的 '\0' 字符
    * alloc：字符串数组 buf 的最大容量，不包含 len, alloc, flag 和 buf 的结束符(即 '\0')
    * flags：占用 1 个字节的内存空间，低三位表示 sdshdr 的类型，另外 5 个比特位暂未使用
    * buf：与 sdshdr5 的 buf 一致

由上述分析可知，不同长度的字符串可以使用不同的类型的 sdshdr，短字符串使用较小的 struct，从而节省内存。

备注：“sdshdr5 is never used, we just access the flags byte directly. ”查看 redis 源码发现 sdshdr5 是有用的，所以个人觉得这句注释不对。

## 2 图解 sds
C 语言同一个结构体里的成员变量，在内存上的地址是连续的。以下讲解字符串 "Redis" 分别使用 sdshdr8 和 sdshdr16 结构体的存储示意图。

### 2.1 sdshdr8
首先分析 sdshdr8 的结构体成员：
* len 和 alloc 都是 uint8_t 类型（typedef unsigned char uint8_t;），也就是 len 和 alloc 都是占用 1 个字节的内存空间
* flags 是 unsigned char 类型，占用 1 个字节的内存空间。由 #define SDS_TYPE_8  1 可知，此时 flags 的值为 1
* 假设存储字符串为"Redis"，由于字符串 "Redis" 的大小为 5，所以 len 为 5；再假设 alloc 为 9，所以 buf 字符数组的大小为 10。

由上述分析可得"Redis"在内存的存储如下图所示：

![sdshdr8.png](https://github.com/wxdyhj/awesome-redis/blob/master/redis_data_struct/png/sdshdr8.png)


以下代码用于验证上述内存示例地址的合理性
```
#include <stdio.h>

typedef unsigned char uint8_t;

struct sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

int main() {
    struct sdshdr8 sds;
    printf("%p\n%p\n%p\n%p\n", &sds, &sds.len, &sds.alloc, &sds.flags);
    return 0;
}

/* 运行环境：macOS 10.14.6 Intel Core i5
输出内容：
0x7ffeec3a9898
0x7ffeec3a9898
0x7ffeec3a9899
0x7ffeec3a989a
*/
```

### 2.2 sdshdr16
首先分析 sdshdr16 的结构体成员：
* len 和 alloc 都是 uint16_t 类型（typedef unsigned short int uint16_t;），也就是 len 和 alloc 都是占用 2 个字节的内存空间。
* flags 是 unsigned char 类型，占用 1 个字节的内存空间，由 #define SDS_TYPE_16 2 可知，此时 flags 的值为 2。

假设存储字符串为"Redis"，由于字符串 "Redis" 的大小为 5，所以 len 为 5；再假设 alloc 为 11，所以 buf 字符数组的大小为 12。由上述分析可得"Redis"在内存的存储如下图所示：

![sdshdr16.png](https://github.com/wxdyhj/awesome-redis/blob/master/redis_data_struct/png/sdshdr16.png)

以下代码用于验证上述内存示例地址的合理性。
```
#include <stdio.h>

typedef unsigned short int uint16_t;

struct sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

int main() {
    struct sdshdr16 sds;
    printf("%p\n%p\n%p\n%p\n", &sds, &sds.len, &sds.alloc, &sds.flags);
    return 0;
}

/* 运行环境：macOS 10.14.6 Intel Core i5
输出内容：
0x7ffee662d890
0x7ffee662d890
0x7ffee662d892
0x7ffee662d894
*/
```
sdshdr32 和 sdshdr64 的存储方式与上述类似，不再重复分析；sdshdr5 的存储方式比较特殊，字符串的长度存储在 flags 字段的低 5 位有效位中，由于没有 alloc 字段，所以无法实现内存预分配的功能。

小结：SDS 字符串的 header 部分（由 len, alloc 和 flags 组成）其实是隐藏在真正字符串的前面，也就是内存低地址方向，好处：减少内存碎片，提高存储效率。虽然 sds 有多个结构体，但是都可以同 char* 来表示，当 SDS 存储的是可打印字符串，此时可以直接将它赋值给 C 函数，比如使用 printf() 函数打印等。

## 3 sds 的主要函数
### 3.1 创建字符串
【函数原型】

sds sdsnew(const char *init);

【功能】

创建新的 sds 字符串。

【源码】
```
// redis-6.0.5/src/sds.h
#define SDS_TYPE_BITS 3

sds sdsnew(const char *init);

// redis-6.0.5/src/sds.c
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32) // ll 代表 long long, 所以 1ll 表示 long long 类型的数字 1
        return SDS_TYPE_32;
    return SDS_TYPE_64;
#else
    return SDS_TYPE_32;
#endif
}

sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen); // 判断使用哪种 sdshdr 结构体
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type); // 计算 sdshdr header 占用多少字节的内存
    unsigned char *fp; /* flags pointer. */

    // 只有一次内存分配，包含：header，数据，额外一个空字节（用于兼容 C 字符串）
    sh = s_malloc(hdrlen+initlen+1);
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    s = (char*)sh+hdrlen; // 指针偏移 hdrlen 字节，此时 s 指向有效字符数组的起始位置
    fp = ((unsigned char*)s)-1; // fp 指针向低地址偏移 1 字节，此时 fp 指向 flags 成员
    switch(type) {
        case SDS_TYPE_5: { // 将字符串的长度存储在 flags 成员的高 5 位有效位中
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen; // 实现获取字符串长度的时间复杂度为 O(1) 
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}

/* Create a new sds string starting from a null terminated C string. */
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}
```
【解析】
* 字符串长度范围 [0, 2^5)：sdshdr 的类型为 SDS_TYPE_5，即使用 sdshdr5，由 sdsnewlen 函数可知，当字符串长度为 0 时，会使用 sdshdr8；
* 字符串长度范围 [2^5, 2^8)：sdshdr 的类型为 SDS_TYPE_8，即使用 sdshdr8；
* 字符串长度范围 [2^8, 2^16)：sdshdr 的类型为 SDS_TYPE_16，即使用 sdshdr16；
* 字符串长度范围 [2^16, 2^32)：sdshdr 的类型为 SDS_TYPE_32，即使用 sdshdr32；
* 字符串长度范围 [2^32, 2^64)：sdshdr 的类型为 SDS_TYPE_64，即使用 sdshdr64。
* Empty strings are usually created in order to append. Use type 8 since type 5 is not good at this. 创建长度为 0 的空字符串时，不使用 sdshdr5，而是使用 sdshdr8，这是因为 sdshdr8 方便实现字符串追加操作，而 sdshdr5 并不适合，会重新分配内存；
* s[initlen] = '\0'; 初始化的 SDS 字符串，数据后会追加一个空字符'\0'，目的是为了兼容 C 字符串

### 3.2 删除字符串
【函数原型】

void sdsfree(sds s);

【功能】

释放内存，将内存归还给系统。

【源码】
```
// redis-6.0.5/src/sds.h
#define SDS_TYPE_MASK 7 // 二进制表示 00000111

void sdsfree(sds s);

// redis-6.0.5/src/sds.c
// sdsHdrSize 函数返回不同 sdshdr 类型的 header 部分的长度
static inline int sdsHdrSize(char type) {
    switch(type&SDS_TYPE_MASK) { // type 与 7 进行按位与运算，最终只保留低 3 位有效位(sdshdr5 的字符串长度存在 type 的高 5 位有效位中)
        case SDS_TYPE_5:
            return sizeof(struct sdshdr5); // sdshdr5 header 的长度 1
        case SDS_TYPE_8:
            return sizeof(struct sdshdr8); // sdshdr8 header 的长度 3
        case SDS_TYPE_16:
            return sizeof(struct sdshdr16); // sdshdr16 header 的长度 5
        case SDS_TYPE_32:
            return sizeof(struct sdshdr32); // sdshdr32 header 的长度 9 
        case SDS_TYPE_64:
            return sizeof(struct sdshdr64); // sdshdr64 header 的长度 17
    }
    return 0;
}

/* Free an sds string. No operation is performed if 's' is NULL. */
void sdsfree(sds s) {
    if (s == NULL) return;
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```
【解析】
* s[-1] 通过字符串内存向低地址偏移一个字节，获取 flags 的值;
* sdsHdrSize(s[-1])) 计算得出不同 sdshdr 的 header 所占的字节数；
* (char*)s-sdsHdrSize(s[-1]) 将 s 指针向低地址偏移 header 的长度, 找到分配内存的起始地址；
通过调用 s_free 函数释放内存。

### 3.3 连接字符串
【函数原型】

sds sdscat(sds s, const char *t);

【功能】

sdscat 将 t 指针指向的任意二进制数据追加到 sds 字符串 s 的后面。

【源码】
```
// redis-6.0.5/src/sds.h
sds sdscatlen(sds s, const void *t, size_t len);
sds sdscat(sds s, const char *t);

#define SDS_TYPE_MASK 7 // 二进制表示为 00000111
// 将 s 指针向低地址偏移 header 大小的字节数，可得 sds 字符串 s 的起始指针位置
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));

#define SDS_MAX_PREALLOC (1024*1024) // 单位: byte，所以 sds 字符串最大预分配内存大小为 1M 字节

// sdsavail 获取 sds 字符串 s 的可用内存空间，单位: byte(字节)
static inline size_t sdsavail(const sds s) {
    // s 指向 sds 字符串的 buf, 即真正字符串的内存地址
    unsigned char flags = s[-1]; // s[-1] 往 s 指针的低地址方向偏移一个字节, 即指向 flags 字段
    switch(flags&SDS_TYPE_MASK) { // 将 flags 变量的高 5 位有效位置为 0
        case SDS_TYPE_5: { // sdshdr5 无预申请内存空间，所以直接返回 0，这也说明 sdshdr5 不适合字符串追加操作
            return 0;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            return sh->alloc - sh->len; // 字符数组 buf 的最大长度减去已使用字节数，可得字符数组的可用空间
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            return sh->alloc - sh->len;
        }
    }
    return 0;
}

// 设置 sds 字符串 s 的 len 字段为 newlen
static inline void sdssetlen(sds s, size_t newlen) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            {
                unsigned char *fp = ((unsigned char*)s)-1;
                *fp = SDS_TYPE_5 | (newlen << SDS_TYPE_BITS);
            }
            break;
        case SDS_TYPE_8:
            SDS_HDR(8,s)->len = newlen;
            break;
        case SDS_TYPE_16:
            SDS_HDR(16,s)->len = newlen;
            break;
        case SDS_TYPE_32:
            SDS_HDR(32,s)->len = newlen;
            break;
        case SDS_TYPE_64:
            SDS_HDR(64,s)->len = newlen;
            break;
    }
}

// 获取 sds 字符串 s 的有效字符串长度，若存入 redis 的字符串为"hello"，则长度为 5
static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}

// 设置 sds 字符串 s 的 alloc 字段的值为 newlen，即设置字符数组的最大容量
static inline void sdssetalloc(sds s, size_t newlen) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            /* Nothing to do, this type has no total allocation info. */
            break;
        case SDS_TYPE_8:
            SDS_HDR(8,s)->alloc = newlen;
            break;
        case SDS_TYPE_16:
            SDS_HDR(16,s)->alloc = newlen;
            break;
        case SDS_TYPE_32:
            SDS_HDR(32,s)->alloc = newlen;
            break;
        case SDS_TYPE_64:
            SDS_HDR(64,s)->alloc = newlen;
            break;
    }
}

// redis-6.0.5/src/sds.c
/* Enlarge the free space at the end of the sds string so that the caller
 * is sure that after calling this function can overwrite up to addlen
 * bytes after the end of the string, plus one more byte for nul term.
 *
 * Note: this does not change the *length* of the sds string as returned
 * by sdslen(), but only the free buffer space we have. */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    /* Return ASAP if there is enough space left. */
    if (avail >= addlen) return s; // 可用空间够用，返回原有 sds 字符串 s

    // 需要重新分配内存
    len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype); // sh 指针指向 sds 字符串 s 的 header 的起始位置
    newlen = (len+addlen); // 新的长度等于原有字符串长度加上待追加的字符串长度
    if (newlen < SDS_MAX_PREALLOC) // 如果新的字符串长度小于 1M 字节
        newlen *= 2; // 新的长度翻倍，即冗余空间为 newlen 字节
    else
        newlen += SDS_MAX_PREALLOC; // 新的长度再增加 SDS_MAX_PREALLOC，即冗余空间为 1M 字节

    type = sdsReqType(newlen); // 重新选择 sdshdr 类型

    /* Don't use type 5: the user is appending to the string and type 5 is
     * not able to remember empty space, so sdsMakeRoomFor() must be called
     * at every appending operation. */
    if (type == SDS_TYPE_5) type = SDS_TYPE_8; // 若是 sdshdr5 则使用 sdshdr8，减少重新分配内存的次数

    hdrlen = sdsHdrSize(type);
    if (oldtype==type) { // 新的 sdshdr 类型与原有的相同，即 hedaer 未发生变化
        // s_realloc 尝试在原有内存地址上重新分配空间，如果原来的内存为止有足够的空间完成重新分配，
        // 则返回的新地址与传入的旧地址相同；反之，该函数将分配新的内存块，并完成数据迁移。
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else { // 新的 sdshdr 类型与原有的不同，即 hedaer 发生变化
        // 整个字符串空间（包括 header）都需要重新分配，并拷贝。
        /* Since the header size changes, need to move the string forward,
         * and can't use realloc */
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1); // 将原有的字符数组拷贝到新的字符数组中
        s_free(sh); // 释放原有 sds 的内存空间
        s = (char*)newsh+hdrlen; // 让 s 指针指向新的 sds 的内存起始位置
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, newlen);
    return s;
}

/* Append the specified binary-safe string pointed by 't' of 'len' bytes to the
 * end of the specified sds string 's'.
 *
 * After the call, the passed sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call. */
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);

    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    memcpy(s+curlen, t, len);
    sdssetlen(s, curlen+len);
    s[curlen+len] = '\0';
    return s;
}

/* Append the specified null termianted C string to the sds string 's'.
 *
 * After the call, the passed sds string is no longer valid and all the
 * references must be substituted with the new pointer returned by the call. */
sds sdscat(sds s, const char *t) {
    return sdscatlen(s, t, strlen(t));
}
```
【解析】
* redis 的 append 命令，在 redis 内部由 sdscat 函数完成;
* 通过 sdsMakeRoomFor 函数保证 sds 字符串 s 有足够的空间来保存追加的字符串，可能会重新分配内存也可能不会，取决于追加的字符串长度和 sds buf 数组的剩余空间大小。

## 4 sds 与 c 字符串的区别
* 获取字符串长度的时间复杂度：SDS 只需O(1)，而 C 字符串需要 O(n)
* 可动态扩展内存，SDS 标识的字符串内容可以修改，也可以追加。申请的内存一般大于字符串的大小，可减少修改字符串时带来的内存重分配次数
* SDS 是二进制安全(binary safe)的，能存储任意二进制数据，而不仅仅是可打印字符。而 C 字符串以 '\0' 字符标识字符串结束，无法存 '\0' 字符
* SDS 与 C 字符串类型兼容，且兼容部分 C 字符串函数

## 5 参考资料
* 1 [redis-6.0.5 源码](http://download.redis.io/releases/redis-6.0.5.tar.gz)
* 2 [Redis内部数据结构详解(2)——sds](http://zhangtielei.com/posts/blog-redis-sds.html)

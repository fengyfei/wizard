# Dynamic String

## 基础数据结构

```C
typedef char *sds;

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
```

## 方法

- sdsHdrSize

```C
static inline int sdsHdrSize(char type) {
    switch(type&SDS_TYPE_MASK) { // 掩码屏蔽掉长度信息
        case SDS_TYPE_5:
            return sizeof(struct sdshdr5);
        case SDS_TYPE_8:
            return sizeof(struct sdshdr8);
        case SDS_TYPE_16:
            return sizeof(struct sdshdr16);
        case SDS_TYPE_32:
            return sizeof(struct sdshdr32);
        case SDS_TYPE_64:
            return sizeof(struct sdshdr64);
    }
    return 0;
}
```

- sdsReqType

根据字符串长度，选择合适的 sdshdr 类型

```C
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
#endif
    return SDS_TYPE_64;
}
```

- sdsnewlen

```C
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);
    // 切换类型
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;

    // 获取 header 长度
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp;

    // 分配空间，加一是为了保存 '\0'
    sh = s_malloc(hdrlen+initlen+1);
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        // 全部清零，空 string
        memset(sh, 0, hdrlen+initlen+1);

    // 内存不足
    if (sh == NULL) return NULL;

    // 跳过 header
    s = (char*)sh+hdrlen;

    // 指向 sdshdr5/8/16/32/64 的 flags
    fp = ((unsigned char*)s)-1;
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS); // 高 5 位表示长度，低 3 位表示类型
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
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

    // 复制初始内容
    if (initlen && init)
        memcpy(s, init, initlen);

    // 添加字符串结尾
    s[initlen] = '\0';
    return s;
}
```

- sdsfree

```C
void sdsfree(sds s) {
    // NULL 不能 free
    if (s == NULL) return;

    // s[-1] 指向 sdshdr5/8/16/32/64 的 flags
    // s - sdsHdrSize(s[-1]) 指向要回收内存起始位置
    s_free((char*)s-sdsHdrSize(s[-1]));
}
```

- sdsMakeRoomFor

```C
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    // 剩余空间
    size_t avail = sdsavail(s);
    size_t len, newlen;
    // 获取原 sds 类型，字符串增长时，可能会引起 header 类型的变更
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    // 空间足够，直接返回
    if (avail >= addlen) return s;

    // 空间不够，获取当前长度
    len = sdslen(s);

    // 获取当前起始指针
    sh = (char*)s-sdsHdrSize(oldtype);

    // 预留新 string 长度
    newlen = (len+addlen);
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    // 获取新 header 类型
    type = sdsReqType(newlen);

    // 至少使用 SDS_TYPE_8
    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    hdrlen = sdsHdrSize(type);

    // header 类型无变化
    if (oldtype==type) {
        // realloc 即可
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        // 分配新 header
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;

        // 拷贝字符串
        memcpy((char*)newsh+hdrlen, s, len+1);

        // 释放原有内存
        s_free(sh);
        s = (char*)newsh+hdrlen;

        // 设定新类型
        s[-1] = type;

        // 设定长度
        sdssetlen(s, len);
    }

    // 设定 alloc，此时 alloc 与 len 不一定相等
    sdssetalloc(s, newlen);
    return s;
}
```

- sdsRemoveFreeSpace

```C
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen, oldhdrlen = sdsHdrSize(oldtype);
    size_t len = sdslen(s);
    sh = (char*)s-oldhdrlen;

    // 获取新 header
    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);

    // 根据新、旧类型是否相同，做对应处理
    if (oldtype==type || type > SDS_TYPE_8) {
        newsh = s_realloc(sh, oldhdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+oldhdrlen;
    } else {
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }

    // 此时 len == alloc
    sdssetalloc(s, len);
    return s;
}
```

- sdsIncrLen: 调整 sds 长度

- sdsgrowzero

```C
sds sdsgrowzero(sds s, size_t len) {
    size_t curlen = sdslen(s);

    // 直接返回，不能将当前使用的字符串清零
    if (len <= curlen) return s;

    // 预留空间
    s = sdsMakeRoomFor(s,len-curlen);
    if (s == NULL) return NULL;

    // 将需要的最小空间清零，含最后的 '\0' 结尾符
    memset(s+curlen,0,(len-curlen+1));
    sdssetlen(s, len);
    return s;
}
```

## References

- [Common Type Attributes](https://gcc.gnu.org/onlinedocs/gcc-7.3.0/gcc/Common-Type-Attributes.html#Common-Type-Attributes)

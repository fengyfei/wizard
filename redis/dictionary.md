# Dictionary

## 基础数据结构

- dictEntry

存储 key/value。key 可为任意值，value 也可为任意值。注意此处定义中 union 的使用，是 union 的经典使用方法。

结合 dictType 中 hashFunction，next 域的作用应该是冲突解决。由于是链表方式，因此，如果 hash 实现不当，可能存在 Hash-flooding DoS Reloaded 攻击风险。

```C
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

- dictType

通过行为定义数据结构，类似于 Go 语言的 interface。

```C
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

- dictht

```C
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```

- dict

```C
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

## 宏定义

```C
// 传入 dict, key, 返回 hash key
#define dictHashKey(d, key) (d)->type->hashFunction(key)

// 传入 dictEntry, 返回 dictEntry 的 key 值
#define dictGetKey(he) ((he)->key)

// 传入 dictEntry, 返回 val
#define dictGetVal(he) ((he)->v.val)

// 传入 dictEntry, 返回 s64
#define dictGetSignedIntegerVal(he) ((he)->v.s64)

// 传入 dictEntry, 返回 u64
#define dictGetUnsignedIntegerVal(he) ((he)->v.u64)

// 传入 dictEntry, 返回 d
#define dictGetDoubleVal(he) ((he)->v.d)

// 传入 dict，返回可用空间
#define dictSlots(d) ((d)->ht[0].size+(d)->ht[1].size)

// 传入 dict，返回已使用的空间
#define dictSize(d) ((d)->ht[0].used+(d)->ht[1].used)

// 传入 dict，返回 dict 是否在 rehashing
#define dictIsRehashing(d) ((d)->rehashidx != -1)
```

## 方法

- dictCreate

```C
dict *dictCreate(dictType *type, void *privDataPtr)
{
	  // 分配 dict 结构基本内存空间
    dict *d = zmalloc(sizeof(*d));

    // 初始化 dict
    _dictInit(d,type,privDataPtr);
    return d;
}
```

- \_dictInit

```C
int _dictInit(dict *d, dictType *type, void *privDataPtr)
{
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}
```

## References

- [Hash-flooding DoS Reloaded](http://emboss.github.io/blog/2012/12/14/breaking-murmur-hash-flooding-dos-reloaded/)

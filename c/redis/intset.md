# Int Set

## 数据结构

- intset

同类型、有序、无重复元素

```C
typedef struct intset {
	  // 保存的数据类型编码 16/32/64 bits
    uint32_t encoding;

    // 元素数量
    uint32_t length;

    // 底层存储
    int8_t contents[];
} intset;
```

- encoding

```C
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```

## 方法

- \_intsetValueEncoding

```C
// 传入参数注意需要使用最长类型
static uint8_t _intsetValueEncoding(int64_t v) {
    if (v < INT32_MIN || v > INT32_MAX)
        return INTSET_ENC_INT64;
    else if (v < INT16_MIN || v > INT16_MAX)
        return INTSET_ENC_INT32;
    else
        return INTSET_ENC_INT16;
}
```

- \_intsetGetEncoded

```C
// 返回值需要 64-bits
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    int64_t v64;
    int32_t v32;
    int16_t v16;

    if (enc == INTSET_ENC_INT64) {
        // 获取 64-bits
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64));
        memrev64ifbe(&v64);
        return v64;
    } else if (enc == INTSET_ENC_INT32) {
        // 获取 32-bits
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32));
        memrev32ifbe(&v32);
        return v32;
    } else {
        // 获取 16-bits
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16));
        memrev16ifbe(&v16);
        return v16;
    }
}
```

- \_intsetSet

```C
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        // 存储 64-bits
        ((int64_t*)is->contents)[pos] = value;
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        // 存储 32-bits
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        // 存储 16-bits
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
```

- intsetResize

```C
static intset *intsetResize(intset *is, uint32_t len) {
    // 计算需要分配内存空间
    uint32_t size = len*intrev32ifbe(is->encoding);

    // 分配内存，并保留原有内容
    is = zrealloc(is,sizeof(intset)+size);
    return is;
}
```

- intsetNew

```C
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));

    // 初始使用 16-bits
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);

    // 重置长度
    is->length = 0;
    return is;
}
```

- intsetSearch

```C
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;

    // 空集合
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        // intset 是递增的，因此，判断 value 是否大于最大值；
        // 或是否小于最小值；如果成立，那么 value 不存在
        if (value > _intsetGet(is,intrev32ifbe(is->length)-1)) {
            if (pos) *pos = intrev32ifbe(is->length);
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }

    // 二分查找
    while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```

- intsetUpgradeAndAdd

```C
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    // 获取当前编码及新 value 使用的编码
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);

    // 获取当前长度
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    // 设置新编码方式，并重新分配空间
    // 此处并没有判断新编码方式占用更多内存空间，因此，需要调用者保证
    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    // 从后向前复制现有元素
    // 当 prepend == 0 时，循环并不是多余的，因为需要扩容存储空间
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    // 将元素放入合适的位置，此时，将自然有序
    // 因为，在扩容时，value 必然大于旧 encoding 方式的最大值，或小于其最小值
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);

    // 刷新长度
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

- intsetMoveTail

```C
static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    void *src, *dst;
    // 计算需要移动的元素数量
    uint32_t bytes = intrev32ifbe(is->length)-from;
    uint32_t encoding = intrev32ifbe(is->encoding);

    // 计算源地址、目标地址、需要拷贝的字节数
    if (encoding == INTSET_ENC_INT64) {
        src = (int64_t*)is->contents+from;
        dst = (int64_t*)is->contents+to;
        bytes *= sizeof(int64_t);
    } else if (encoding == INTSET_ENC_INT32) {
        src = (int32_t*)is->contents+from;
        dst = (int32_t*)is->contents+to;
        bytes *= sizeof(int32_t);
    } else {
        src = (int16_t*)is->contents+from;
        dst = (int16_t*)is->contents+to;
        bytes *= sizeof(int16_t);
    }

    // 移动内存
    memmove(dst,src,bytes);
}
```

- intsetAdd

```C
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;

    // success 置为成功状态
    if (success) *success = 1;

    // 是否需要升级
    if (valenc > intrev32ifbe(is->encoding)) {
        return intsetUpgradeAndAdd(is,value);
    } else {
        // 检查值是否存在
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        // 扩容
        is = intsetResize(is,intrev32ifbe(is->length)+1);

        // pos 没有越界，需要向后移动一个位置
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    // pos 处可以存入新内容
    _intsetSet(is,pos,value);

    // 刷新长度
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

- intsetRemove

```C
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;

    // 只有 encoding 匹配时，才能查找
    // 如果存在，才能删除
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);

        // delete 不会失败
        if (success) *success = 1;

        // 需要向前移动一个位置
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);

        // 调整空间
        is = intsetResize(is,len-1);

        // 刷新长度
        is->length = intrev32ifbe(len-1);
    }
    return is;
}
```

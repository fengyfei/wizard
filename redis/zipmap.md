# Zip Map

## Zip Map Layout

```text
 uint8_t                   uint8_t
+-------+-----+-----+-----+------+-------+-------------------+------+
| zmlen | len | key | len | free | value |        ...        | 0xFF |
+-------+-----+-----+-----+------+-------+-------------------+------+
    |      |                  |
    |      |                  +----> 由于修改操作，造成的剩余空间
    |      |              +---------+
    |      +-----+------->| 0 ~ 253 | (1 Byte)
    |            |        +---------+
    |            |
    |            |        +------+-------------------------------+
    |            +------->| 0xFE | 4 bytes length in host endian | (5 Bytes)
    |                     +------+-------------------------------+
    |
    |
    +----+----> < 254，直接表示
         |
         +----> >= 254，遍历获取 zip map 长度
```

## 实现

- zipmapNew

```C
unsigned char *zipmapNew(void) {
    unsigned char *zm = zmalloc(2); // zmlen + zmend

    zm[0] = 0;
    zm[1] = ZIPMAP_END;
    return zm;
}
```

- zipmapDecodeLength

```C
static unsigned int zipmapDecodeLength(unsigned char *p) {
	  // 获取 uint8_t
    unsigned int len = *p;

    // < 254，直接返回长度
    if (len < ZIPMAP_BIGLEN) return len;

    // 读取 p 后四字节内容
    memcpy(&len,p+1,sizeof(unsigned int));

    // 转换字节序
    memrev32ifbe(&len);
    return len;
}
```

- zipmapEncodeLength

```C
static unsigned int zipmapEncodeLength(unsigned char *p, unsigned int len) {
    if (p == NULL) {
    	  // 仅计算，不存储
        return ZIPMAP_LEN_BYTES(len);
    } else {
        if (len < ZIPMAP_BIGLEN) {
            p[0] = len;

            // 返回长度所占字节数
            return 1;
        } else {
            p[0] = ZIPMAP_BIGLEN;
            memcpy(p+1,&len,sizeof(len));
            memrev32ifbe(p+1);
            return 1+sizeof(len);
        }
    }
}
```

- zipmapRequiredLength

计算 key/value 需要的字节总长

```C
static unsigned long zipmapRequiredLength(unsigned int klen, unsigned int vlen) {
    unsigned int l;

    // key，value 实际长度加 len + free + len 各 1 字节
    l = klen+vlen+3;

    // klen >= 254, 长度加 4
    if (klen >= ZIPMAP_BIGLEN) l += 4;

    // vlen >= 254, 长度加 4
    if (vlen >= ZIPMAP_BIGLEN) l += 4;
    return l;
}
```

- zipmapRawKeyLength

```C
static unsigned int zipmapRawKeyLength(unsigned char *p) {
	  // key 长度
    unsigned int l = zipmapDecodeLength(p);

    // key 长度占字节数 + key 内容长度
    return zipmapEncodeLength(NULL,l) + l;
}
```

- zipmapRawValueLength

```C
static unsigned int zipmapRawValueLength(unsigned char *p) {
	  // 获取 value 内容长度
    unsigned int l = zipmapDecodeLength(p);
    unsigned int used;

    // used 为 value 长度所占字节数
    used = zipmapEncodeLength(NULL,l);

    // 追加 free(1 Byte)，free 长度，value 内容实际长度
    used += p[used] + 1 + l;
    return used;
}
```

- zipmapLookupRaw

```C
static unsigned char *zipmapLookupRaw(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned int *totlen) {
    // p 跳过 zmlen
    unsigned char *p = zm+1, *k = NULL;
    unsigned int l,llen;

    // 遍历 zip map
    while(*p != ZIPMAP_END) {
        unsigned char free;

        // 获取长度
        l = zipmapDecodeLength(p);

        // 获取长度所占字节数
        llen = zipmapEncodeLength(NULL,l);

        // key 不为空
        // k 为空，说明没有找到匹配的内容
        // l == klen 是快速匹配分支
        // memcmp 匹配实际内容
        if (key != NULL && k == NULL && l == klen && !memcmp(p+llen,key,l)) {
            if (totlen != NULL) {
            	  // 如果同时请求 zip map 长度，仅记录找到的 k 值
                k = p;
            } else {
            	  // 不关心总长度，直接返回位置
                return p;
            }
        }

        // 跳过 key
        p += llen+l;

        // 计算 value 长度
        l = zipmapDecodeLength(p);

        // 跳过 value 长度所占字节
        p += zipmapEncodeLength(NULL,l);

        // 获取 free 长度
        free = p[0];

        // 跳过 free: +1
        // 跳过 value 内容: l
        // 跳过 free 内容: free
        p += l+1+free;
    }

    // 计算总长度
    if (totlen != NULL) *totlen = (unsigned int)(p-zm)+1;

    return k;
}
```

- zipmapResize

```C
static inline unsigned char *zipmapResize(unsigned char *zm, unsigned int len) {
	  // 直接分配新的总长度
    zm = zrealloc(zm, len);

    // 最后一字节填充 0xFF
    zm[len-1] = ZIPMAP_END;
    return zm;
}
```

- zipmapSet

```C
unsigned char *zipmapSet(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned char *val, unsigned int vlen, int *update) {
    unsigned int zmlen, offset;

    // reqlen 计算 key/value 需要的实际空间
    unsigned int freelen, reqlen = zipmapRequiredLength(klen,vlen);
    unsigned int empty, vempty;
    unsigned char *p;

    // freelen 设置为总长
    freelen = reqlen;

    // 初始化 update
    if (update) *update = 0;

    // 查找 key 是否存在，并获取 zip map 总长度
    p = zipmapLookupRaw(zm,key,klen,&zmlen);

    if (p == NULL) {
        // key 不存在，扩容
        zm = zipmapResize(zm, zmlen+reqlen);

        // 跳至 key/value 实际存储位置
        p = zm+zmlen-1;

        // 更新 zip map 总长度
        zmlen = zmlen+reqlen;

        // key/value 对小于 254
        if (zm[0] < ZIPMAP_BIGLEN) zm[0]++;
    } else {
        // key 存在，标记 update 为 true，说明此次操作为更新
        if (update) *update = 1;

        // 获取当前 key/value 占有空间长度
        freelen = zipmapRawEntryLength(p);

        // 当期空间不足够容纳新 value 值
        if (freelen < reqlen) {
            // 记录当前位置
            offset = p-zm;

            // 分配空间
            zm = zipmapResize(zm, zmlen-freelen+reqlen);

            // 跳转至存储位置
            p = zm+offset;

            // 将原有内容移动至 kev/value 所占长度后，预留出足够空间
            memmove(p+reqlen, p+freelen, zmlen-(offset+freelen+1));

            // 更新 zip map 总长
            zmlen = zmlen-freelen+reqlen;

            // 记录需要的长度
            freelen = reqlen;
        }
    }

    // 计算空闲空间
    empty = freelen-reqlen;

    // 剩余空间超过预定大小
    if (empty >= ZIPMAP_VALUE_MAX_FREE) {
        // 计算前序内容长度
        offset = p-zm;

        // 移动元素，缩减空间
        memmove(p+reqlen, p+freelen, zmlen-(offset+freelen+1));

        // 计算去掉 empty 后需要的长度
        zmlen -= empty;

        // 重构 zip map
        zm = zipmapResize(zm, zmlen);

        // 跳转至 key/value 存储位置
        p = zm+offset;

        // empty 置 0
        vempty = 0;
    } else {
        vempty = empty;
    }

    // 写入 key 长度
    p += zipmapEncodeLength(p,klen);

    // 写入 key 内容
    memcpy(p,key,klen);
    p += klen;

    // 写入 value 长度
    p += zipmapEncodeLength(p,vlen);

    // 写入 free
    *p++ = vempty;

    // 写入 value 内容
    memcpy(p,val,vlen);
    return zm;
}
```

- zipmapDel

```C
unsigned char *zipmapDel(unsigned char *zm, unsigned char *key, unsigned int klen, int *deleted) {
    unsigned int zmlen, freelen;

    // 查找 key 所在位置
    unsigned char *p = zipmapLookupRaw(zm,key,klen,&zmlen);

    // key 存在
    if (p) {
    	  // 计算 key/value 占用字节数
        freelen = zipmapRawEntryLength(p);

        // 移动后续内容
        memmove(p, p+freelen, zmlen-((p-zm)+freelen+1));

        // 重构 zip map
        zm = zipmapResize(zm, zmlen-freelen);

        // entry 数量减一
        if (zm[0] < ZIPMAP_BIGLEN) zm[0]--;

        if (deleted) *deleted = 1;
    } else {
        if (deleted) *deleted = 0;
    }
    return zm;
}
```

- zipmapNext

```C
unsigned char *zipmapNext(unsigned char *zm, unsigned char **key, unsigned int *klen, unsigned char **value, unsigned int *vlen) {
	  // 已经到结尾
    if (zm[0] == ZIPMAP_END) return NULL;

    // key 不为空，记录
    if (key) {
        *key = zm;
        *klen = zipmapDecodeLength(zm);
        *key += ZIPMAP_LEN_BYTES(*klen);
    }

    // 跳过 key
    zm += zipmapRawKeyLength(zm);

    // value 不为空，记录
    if (value) {
    	  // 跳过 free
        *value = zm+1;
        *vlen = zipmapDecodeLength(zm);

        // 跳过 value 长度所占字节数
        *value += ZIPMAP_LEN_BYTES(*vlen);
    }

    // 跳过 value
    zm += zipmapRawValueLength(zm);
    return zm;
}
```

- zipmapLen

```C
unsigned int zipmapLen(unsigned char *zm) {
    unsigned int len = 0;

    if (zm[0] < ZIPMAP_BIGLEN) {
    	  // 可直接返回
        len = zm[0];
    } else {
        unsigned char *p = zipmapRewind(zm);
        // 遍历
        while((p = zipmapNext(p,NULL,NULL,NULL,NULL)) != NULL) len++;

        // 如果可回写，写入
        if (len < ZIPMAP_BIGLEN) zm[0] = len;
    }
    return len;
}
```

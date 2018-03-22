# Skip List

## 介绍

```text
+---+    +---+    +---+    +---+    +---+    +---+    +---+
| H |--->| 0 |--->| 1 |--->| 2 |--->| 3 |--->| 4 |--->| T |
+---+    +---+    +---+    +---+    +---+    +---+    +---+
```

先看普通链表，对于查找，时间复杂度为 O(n)。对于数组的二分查找，更是无法使用。如果我们把每一个中间位置通过链表记录下来，那么二分查找自然可以使用了。

```text
+---+                               +---+                               +---+
|   |------------------------------>| 3 |------------------------------>| T |
+---+             +---+             +---+             +---+             +---+
|   |------------>| 1 |------------>| 3 |------------>| 5 |------------>| T |
+---+    +---+    +---+    +---+    +---+    +---+    +---+    +---+    +---+
| H |--->| 0 |--->| 1 |--->| 2 |--->| 3 |--->| 4 |--->| 5 |--->| 6 |--->| T |
+---+    +---+    +---+    +---+    +---+    +---+    +---+    +---+    +---+
```

从上图可以看出，由于保留了中间位置，因此，可以使用二分查找。

看一下 Redis 中定义

```C
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned int span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    // 头、尾节点
    struct zskiplistNode *header, *tail;

    // 节点数量，不含头、尾节点
    unsigned long length;

    // 最大层级
    int level;
} zskiplist;
```

## 实现

- zslCreateNode

创建 node

```C
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    // 注意空间的计算
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}
```

- zslCreate

创建空的 skip list

```C
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));

    // 空表，层级为 1
    zsl->level = 1;
    zsl->length = 0;

    // header 按照最大允许的层级创建
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL);

    // 每一层均无指向
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;
    return zsl;
}
```

- zslFree

```C
void zslFree(zskiplist *zsl) {
    // 指向最低一级，表现为普通链表
    zskiplistNode *node = zsl->header->level[0].forward, *next;

    // 释放头节点
    zfree(zsl->header);
    while(node) {
        // 保存下一节点
        next = node->level[0].forward;

        // 释放当前节点
        zslFreeNode(node);
        node = next;
    }

    // 释放 skip list
    zfree(zsl);
}
```

- zslInsert

```C
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));

    // 指向列表头
    x = zsl->header;

    // 从高 level 向低 level，每一层独立观察
    for (i = zsl->level-1; i >= 0; i--) {
        // rank 从 0 开始记录跨越长度
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];

        // 当前 level 有下一个节点
        // 下一节点 score < 当前 score
        // 或者下一节点内容 < 查找内容
        // 结论：找到从当前 level 观察，现有链表中小于新增元素的元素数量
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            // 累加跨越元素数量
            rank[i] += x->level[i].span;

            // 指向当前 level 的下一条记录
            x = x->level[i].forward;
        }

        // 指向不小于新增元素的第一个元素
        update[i] = x;
    }

    // 在此，假设元素不存在；因为在新增元素调用前，需要通过 hash 确认元素是否已在链表中
    // 生成随机 level 值
    level = zslRandomLevel();

    // 如果随机 level 值大于当前链表最大 level 值
    if (level > zsl->level) {
        // 多余 node 均指向当前链表头
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            // 多余 level 的 span 均跨越当前链表长度
            update[i]->level[i].span = zsl->length;
        }

        // 刷新当前链表最大层级
        zsl->level = level;
    }

    // 创建新增节点
    x = zslCreateNode(level,score,ele);

    // 构建链表层
    for (i = 0; i < level; i++) {
        // 新增元素插入当前层
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        // rank [0] 记录了最大跨越
        // rank [i] 记录了当前层跨越
        // 由于 x 介于 update[i] 与下一个元素间，因此调整跨越为下值
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);

        // 原 update 跨越调整为 (rank[0] - rank[i]) + 1
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

    // 设置 backward
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    // 总长度加一
    zsl->length++;
    return x;
}
```

- zslDelete

从链表中移除元素

```C
int zslDelete(zskiplist *zsl, double score, sds ele, zskiplistNode **node) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;

    // 从高层向低层观察
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }

        // 记录每层首个不小于要删除元素首个节点
        update[i] = x;
    }
    
    // 最低层第一个不小于比待删除元素
    x = x->level[0].forward;

    // 如果第一个不相等，那么后续一定比带删除元素大
    if (x && score == x->score && sdscmp(x->ele,ele) == 0) {
        zslDeleteNode(zsl, x, update);
        if (!node)
            zslFreeNode(x);
        else
            *node = x;
        return 1;
    }
    return 0;
}
```

- zslDeleteNode

删除节点，并更新链表

```C
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;

    // 从每一层独立观察
    for (i = 0; i < zsl->level; i++) {
        // 当前层下一条记录为待删除元素
        if (update[i]->level[i].forward == x) {
            // 当前层记录跨度为 x 跨度减一，因为 x 被删除
            update[i]->level[i].span += x->level[i].span - 1;

            // 指向 x 的下一条记录
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            // 仅将跨度减一
            update[i]->level[i].span -= 1;
        }
    }

    // x 不是最后一条记录
    if (x->level[0].forward) {
        // 下一条记录的 backward 刷新
        x->level[0].forward->backward = x->backward;
    } else {
        // 刷新链表尾
        zsl->tail = x->backward;
    }

    // 计算链表新的最大层数
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;

    // 节点数量减一
    zsl->length--;
}
```

## Reference

- [Skip List](https://en.wikipedia.org/wiki/Skip_list)

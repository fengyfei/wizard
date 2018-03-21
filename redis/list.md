# 基础数据结构

## list - 通用双向链表结构

```C
// 通用 list 节点
typedef struct listNode {
    // 双向链表
    struct listNode *prev;
    struct listNode *next;
    // void* 代表可放入任意类型，通过强制类型转换，可变为目标类型
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
     // 遍历方向
    int direction;
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    // list 操作，C 中面向对象的方式
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

## References

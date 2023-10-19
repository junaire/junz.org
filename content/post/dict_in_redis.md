---
title: "由Redis学习数据结构--字典"
author: "Jun"
date: 2021-05-08T21:21:22+08:00
---

## 字典的定义

字典，是一种保存键值对（key value pair）的抽象数据结构。

字典中的每一个键都是独一无二的，我们可以通过键查找，更新，删除与之关联的值。

Redis中用C语言实现了字典，其本质即为[哈希表（Hash table）](https://en.wikipedia.org/wiki/Hash_table)的一层wrapper。

### 哈希表

Redis中的字典具有很高的查询速度，这主要得益于其基于哈希表的底层实现。

在字典中，假如其键为`key`，它的值(value)储则存在`f(key)`上，故我们可以很快地通过`f()`找到需要操作的值，而不需要遍历。

其中`f()`函数为散列函数，其本身很大程度上决定着字典的效率，我们将在下文简单介绍Redis所使用的散列函数。

#### 哈希冲突

在某些时候，不同的键会产生相同的值，我们称为哈希冲突。此时我们就需要对冲突进行解决。

在Redis中，我们使用了链地址法（separatechaining）来解决键冲突。具体为每个哈希表节点都有一个next指针，多个节点以此形成一个单向链表，以解决键冲突的问题。

#### 载荷因子

散列表的载荷因子定义为： 

**α  = 填入表中的元素个数 / 散列表的长度**

α 是散列表装满程度的标志因子。由于表长是定值， α与“填入表中的元素个数”成正比，所以，α越大，表明填入表中的元素越多，产生冲突的可能性就越大；反之，α越小，标明填入表中的元素越少，产生冲突的可能性就越小。实际上，散列表的平均查找长度是载荷因子 α的函数，只是不同处理冲突的方法有不同的函数。 

载荷因子一定程度上决定着哈希表的效率。为了使Redis的负载因子维持在一个合理的范围，当哈希表中保存的键值对过多或过少时，我们就会对哈希表进行相应的扩展或收缩。这个过程称为rehashing。

## 相关结构

### 字典
其中`ht`为哈希表数组，第一个用来储存数据，第二个用来`rehashing`
```
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;

```

### 操作字典的一些方法

此结构体保存了一些函数指针，它主要是为了多态。

```
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
    int (*expandAllowed)(size_t moreMem, double usedRatio);
} dictType;

```

### 哈希表

其中`table`是一个数组，保存着键值对的指针。

```
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```

### 键值对 

其中next指针主要是为了解决哈希冲突的问题。

```
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

## 操作函数

### 新建一个字典

```
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    dict *d = zmalloc(sizeof(*d));

    _dictInit(d,type,privDataPtr);
    return d;
}

//初始化字典
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}

//初始化哈希表
static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```

### 向字典中添加键值对

1. 判断是否在rehashing，如果是则停止。 
2. 新建一个键值对，并将其加在哈希表最前面。
3. 重置键。
4. 重置值。

```
int dictAdd(dict *d, void *key, void *val)
{
    dictEntry *entry = dictAddRaw(d,key,NULL); 

    if (!entry) return DICT_ERR;
    dictSetVal(d, entry, val);
    return DICT_OK;
}

dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* Get the index of the new element, or -1 if
     * the element already exists. */
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    //如果在rehashing就使用第二个哈希表。
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];
    ht->table[index] = entry;
    ht->used++;

    //重置字典的键
    dictSetKey(d, entry, key);
    return entry;
}

```

具体向字典中加入键和值的为两个宏。

如果字典本身提供了操作的方法，优先使用字典提供的。

```
#define dictSetVal(d, entry, _val_) do { \
    if ((d)->type->valDup) \
        (entry)->v.val = (d)->type->valDup((d)->privdata, _val_); \
    else \
        (entry)->v.val = (_val_); \
} while(0)

#define dictSetKey(d, entry, _key_) do { \
    if ((d)->type->keyDup) \
        (entry)->key = (d)->type->keyDup((d)->privdata, _key_); \
    else \
        (entry)->key = (_key_); \
} while(0)

```

### 删除字典中的某个键值对

```
int dictDelete(dict *ht, const void *key) {
    return dictGenericDelete(ht,key,0) ? DICT_OK : DICT_ERR;
}

static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    uint64_t h, idx;
    dictEntry *he, *prevHe;
    int table;

    if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;

    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);

    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask; //位与运算，找到索引。
        he = d->ht[table].table[idx]; //找到键值对
        prevHe = NULL;
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                /* Unlink the element from the list */
                if (prevHe)
                    prevHe->next = he->next;
                else
                    d->ht[table].table[idx] = he->next;
                if (!nofree) {
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                    zfree(he);
                }
                d->ht[table].used--;
                return he;
            }
            prevHe = he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    return NULL; /* not found */
}
```

### 查找值

```
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    uint64_t h, idx, table;

    if (dictSize(d) == 0) return NULL; /* dict is empty */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    h = dictHashKey(d, key);
    for (table = 0; table <= 1; table++) {
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}

```

## rehashing

随着操作的进行，为了使得荷载因子维持在一个合理的范围，我们需要对哈希表进行相应的扩张或缩小。

1. 如果执行的是扩张操作，那么ht[1]，即第二个哈希表的大小为2，否则为
2. 将保存在ht[0]中所有的键值对rehashing到第二个哈希表上
3. 将原来的ht[1]设为ht[0]，释放原来的ht[0]，并准备一个新的哈希表，设为ht[1]


```
static long _dictKeyIndex(dict *d, const void *key, uint64_t hash, dictEntry **existing)
{
    unsigned long idx, table;
    dictEntry *he;
    if (existing) *existing = NULL;

    /* Expand the hash table if needed */
    if (_dictExpandIfNeeded(d) == DICT_ERR)
        return -1;
    for (table = 0; table <= 1; table++) {
        idx = hash & d->ht[table].sizemask;
        /* Search if this slot does not already contain the given key */
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (existing) *existing = he;
                return -1;
            }
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    return idx;
}
```

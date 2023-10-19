---
title: "由Redis学习数据结构--链表"
author: "Jun"
date: 2021-04-14T21:26:42+08:00
categories: ["数据结构"]
tags: ["C语言","redis","链表"]
---

## 简介
Redis是一个优秀的非关系型数据库，常常在高性能分布式系统中用于储存缓冲队列。它本身是用C语言写的，其中实现了许多基本数据类型，如安全的字符串（SDS），链表，字典等。虽然它是一个数据库，但也非常适合用于C语言基础和数据结构的学习。下面是我在Redis中关于链表实现的笔记。

## 链表概述

简单来说，链表是一种数据结构，类似于数组。

在详细解释链表是什么之前，我们先简单回顾下数组。数组是一种内存连续分布的结构，我们分配一整片内存，并将其人工分割为一小块一小块，来作为数组中的每一个元素。这意味着数组在我们分配后的大小便确定了，不能改变。我们也不能随意在数组中插入一个元素。而当我们需要访问数组中的元素时，我们实际上是通过首元素的基地址加上偏移量访问的。考虑下面这个例子：

```
int a[5] = {1,2,3,4,5};
int b = a[2];
int c = *(a + 2 * sizeof(int));
```
在上面这个例子中，b和c的值应该是一样的。所以我们可以知道数组元素访问的时间复杂度应该是`O(1)`，也就是不管多大的数组，它元素访问的速度应该都是一样的。

而链表的出现就很好的解决了数组的一些不足。链表本身由一个一个单独的节点组成，然后像一条铁链一样连起来。**它本身内存是不连续分配的，而不是在初始化时就确定的。所以我们可以随意在两个节点之间插入元素，也可以不断拓展链表的长度。但是，为了找到某一特定的元素，我们必须从头开始遍历链表，也就是说它的访问速度为`O(n)`。**

## Redis中的实现

### 链表的节点

在redis中，链表的节点是一个由`struct`声明的一个数据结构（listNode），其中储存着3个对象：两个指针，分别指向前后节点，一个`void*`指针，用于指向数据值。

```
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

```

### 链表本身

在redis中，用于表示链表的数据结构自然就是list啦，它仍然也是由`struct`声明的。在它内部有这么几个对象：链表的第一个节点（head），链表的最后一个节点（tail），链表中节点的数目，还有三个函数指针。

```
typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

### 操作链表的函数
下面的所有函数均来自于redis的源码，具体位置在`redis/src/adlist.c`中。

其中`zmalloc`和`zfree`是作者实现的`malloc`和`free`版本，我们可以把它直接看作`malloc`和`free`。

#### 创建链表
创建空链表的过程也很简单，具体就是先申请一块和`list`一样大小的内存，如果申请失败返回了`NULL`，就返回`NULL`。然后再把链表中的各个对象置为空。

```c
list *listCreate(void)
{
    struct list *list;

    if ((list = zmalloc(sizeof(*list))) == NULL)
        return NULL;
    list->head = list->tail = NULL;
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    return list;
}
```

#### 清空链表
注意，这里的清空链表不是销毁了整个链表，而只是清空了链表中的所有元素，也就是listNode。
我们遍历了整个链表，并依次将各个节点free掉，最后将链表的首元素指针和尾元素指针指向NULL，并将链表的长度设为0。

```c
void listEmpty(list *list)
{
    unsigned long len;
    listNode *current, *next;

    current = list->head;
    len = list->len;
    while(len--) {
        next = current->next;
        if (list->free) list->free(current->value);
        zfree(current);
        current = next;
    }
    list->head = list->tail = NULL;
    list->len = 0;
}
```

#### 添加元素到链表头部

1. 首先创建一个节点，并将我们的值赋给它
2. 如果我们的链表的长度是0，那么我们就让链表的首元素指针和尾元素指针都指向它，并让该节点的前一个元素和后一个元素指向NULL
3. 如果我们的链表长度不为0，那么我们就让该节点的后一个元素指向本来链表的头元素，让本来链表的头元素指向该节点，最后让链表的头元素指针指向该节点
4. 将链表的长度加1，并返回出去

```c
list *listAddNodeHead(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {
        node->prev = NULL;
        node->next = list->head;
        list->head->prev = node;
        list->head = node;
    }
    list->len++;
    return list;
}
```

#### 添加元素到链表尾部

1. 首先创建一个节点，并将我们的值赋给它
2. 如果我们的链表的长度是0，那么我们就让链表的首元素指针和尾元素指针都指向它，并让该节点的前一个元素和后一个元素指向NULL
3. 如果我们的链表长度不为0，那么我们就让该节点的前一个元素指向本来链表的尾元素，让本来链表的尾元素指向该节点，最后让链表的尾元素指针指向该节点
4. 将链表的长度加1，并返回出去

```c
list *listAddNodeTail(list *list, void *value)
{
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (list->len == 0) {
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {
        node->prev = list->tail;
        node->next = NULL;
        list->tail->next = node;
        list->tail = node;
    }
    list->len++;
    return list;
}
```

#### 插入元素到链表中

1. 首先创建一个节点，并将我们的值赋给它
2. 如果after的值不为0，那么向后插入。将该节点的前元素指针指向给定元素，并让该节点后元素指针指向给定元素的后一个元素。
3. 如果给定元素是链表的最后一个元素，那么让链表的尾元素指针指向该节点。 
4. 如果after的值为0，那么向前插入。将该节点的后元素指针指向给定元素，并让该节点前元素指针指向给定元素的前一个元素。
5. 如果给定元素是链表的第一个元素，那么让链表的头元素指针指向该节点。 
6. 将链表的长度加1，并返回出去

```c
list *listInsertNode(list *list, listNode *old_node, void *value, int after) {
    listNode *node;

    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    node->value = value;
    if (after) {
        node->prev = old_node;
        node->next = old_node->next;
        if (list->tail == old_node) {
            list->tail = node;
        }
    } else {
        node->next = old_node;
        node->prev = old_node->prev;
        if (list->head == old_node) {
            list->head = node;
        }
    }
    if (node->prev != NULL) {
        node->prev->next = node;
    }
    if (node->next != NULL) {
        node->next->prev = node;
    }
    list->len++;
    return list;
}
```

#### 删除链表中的元素

1. 先判断需要删除节点是否为头元素，如果不是，就让前一个元素的后元素指针指向给定元素的后一个元素。

2. 如果是的话，就让链表的头元素指针指向给定元素的后一个元素。

3. 再判断需要删除节点是否为尾元素，如果不是，就让后元素的前指针指向给定元素的前一个元素。

4. 如果是的话，就让链表的尾指针指向给定元素的前一个元素。

5. 释放需要删除节点。

6. 将链表的长度减一。
```c
void listDelNode(list *list, listNode *node)
{
    if (node->prev)
        node->prev->next = node->next;
    else
        list->head = node->next;
    if (node->next)
        node->next->prev = node->prev;
    else
        list->tail = node->prev;
    if (list->free) list->free(node->value);
    zfree(node);
    list->len--;
}
```

#### 按索引查找指定元素

1. 如果索引是负数，即从后往前查找。
2. 如果索引是正数，即从前往后查找。

```c
listNode *listIndex(list *list, long index) {
    listNode *n;

    if (index < 0) {
        index = (-index)-1;
        n = list->tail;
        while(index-- && n) n = n->prev;
    } else {
        n = list->head;
        while(index-- && n) n = n->next;
    }
    return n;
}
```

#### 合并两个链表

将第二个链表o合并到第一个链表l上。

1. 如果o的长度为0，即空链表，则直接返回。
2. 将l链表的尾元素指针指向o链表的头元素上。
3. 如果l链表不为空，就把l链表的为元素和o链表的头元素连接在一起。
4. 如果l链表为空，就l链表的头元素指针指向o链表的头元素。
5. 将l链表的尾元素指针指向o链表的最后一个元素，并重新设置l链表的长度。
6. 置空o链表。
```c
void listJoin(list *l, list *o) {
    if (o->len == 0) return;

    o->head->prev = l->tail;

    if (l->tail)
        l->tail->next = o->head;
    else
        l->head = o->head;

    l->tail = o->tail;
    l->len += o->len;

    /* Setup other as an empty list. */
    o->head = o->tail = NULL;
    o->len = 0;
}

```

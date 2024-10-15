---
title: 关于C语言链表，队列的实现
date: 2024-10-16 00:58:39
categories: 
 - 计算机底层
tags: 
 - Linux内核
 - C
 - 数据结构
---

## 前言

最近在读***Linux内核设计与实现***的时候，觉得linux内核里的链表实现很有意思，于是就
突发奇想，能不能利用C的结构体特性来实现一个**通用链表**，然后用其实现一个**通用队列**

## 通用链表

在Linux内核里中，并不是将结构体写在链表中，或是将结构体实现为链表，而是将链表写在结构
体中，这样做的好处是不需要再实现一个特定的结构，仅需要通过操作函数对链表进行操作即可。

假设我们有一个需要链表存储的结构体，例如：

```c
struct fox{
    int age;
    char *name;    
}
```

使用我们将他存入结构体中，可以这样做
```c
struct fox{
    int age;
    char *name;
    struct fox *next;
    struct fox *prev;
}
```
或是
```c
struct list_node{
    struct list_node *next;
    struct list_node *prev;
    void *data;
}
struct list{
    struct list_node head;
    struct list_node tail;
}
//再将fox指针存入list_node.data中
```
但这样做的缺点是，我们需要为每个结构体都实现链表的操作函数，这样会使得代码量大大增加，
并且很麻烦（ ~~其实是我比较懒~~

> **tips** ：C语言中对于结构体的访问，其实就是结构体的内存地址加上`偏移量`实现的  
> **tips** ：`偏移量`可以通过`offsetof()`宏来获取，例如：`offsetof(struct fox, name)`  
> **tips** ：`offsetof()`宏的第一个参数是结构体类型，第二个参数是结构体成员的名称  
> **tips** ：`offsetof()`宏的返回值是结构体成员的偏移量，单位是字节, 定义在stddef.h头文件中

在C语言中，以下的两个结构体的内存布局是相同的：
```c
struct list_node{
    void *next;
    void *prev;
}

/* Structure 1 */
struct fox1{
    struct list_node node;
    int age;
    char *name;
}

/* Structure 2 */
struct fox2{
    void *next;
    void *prev;
    int age;
    char *name;
}
```
于是，我们发现fox2中的`next`和`prev`的偏移量和`list_node`中的`next`和`prev`的偏移量是相同的
而fox1和fox2的内存布局是相同的，所以我们可以使用`list_node`指针来访问`fox1`和`fox2`的`next`和`prev`成员

于是就有了通用链表的实现：
```c
/* list.h */
typedef struct list_node{
    struct list_node *next;
    struct list_node *prev;
}list_node_t;

typedef struct list{
    list_node_t *head;
    list_node_t *tail;
}list_t;

#define list_init(list) ({
    (list)->head = NULL;
    (list)->tail = NULL;
})

#ifdef __GNUC__
static inline void list_alloc(list_t **list){
    *list = (list_t *)malloc(sizeof(list_t));
    if(*list == NULL)
        return;
    list_init(*list);
}
#else
void list_alloc(list_t **list);
#endif

//list is a pointer,
//node is also a pointer, but it can be a
//struct with list_node as its first member.
#define list_add_head(list, node) ({
    if((list)->head == NULL){
        (list)->head = (list_node_t *)(node);
        (list)->tail = (list_node_t *)(node);
        (list_node_t *)(node)->next = NULL;
        (list_node_t *)(node)->prev = NULL;
    }else{
        (list_node_t *)(node)->next = (list)->head;
        (list_node_t *)((list_node_t *)(node)->next)->prev = (list_node_t *)(node);
        (list)->head = (list_node_t *)(node);
        (list_node_t *)(node)->prev = NULL;
    }
})

#define list_add_tail(list, node) ({
    if((list)->tail == NULL){
        (list)->head = (list_node_t *)(node);
        (list)->tail = (list_node_t *)(node);
        (list_node_t *)(node)->next = NULL;
        (list_node_t *)(node)->prev = NULL;
    }else{
        (list_node_t *)(node)->prev = (list)->tail;
        (list_node_t *)((list_node_t *)(node)->prev)->next = (list_node_t *)(node);
        (list)->tail = (list_node_t *)(node);
        (list_node_t *)(node)->next = NULL;
    }
})

#define list_del(list, node) ({
    (list_node_t *)((list_node_t *)node->prev)->next = (list_node_t *)node->next;
    (list_node_t *)((list_node_t *)node->next)->prev = (list_node_t *)node->prev;
    if((list)->head == (list_node_t *)(node)){
        (list)->head = (list_node_t *)(node)->next;
    }
    if((list)->tail == (list_node_t *)(node)){
        (list)->tail = (list_node_t *)(node)->prev;
    }
})

//pos is a pointer
#define list_for_each_h(list, pos) ({
    for((list_node_t *)(pos) = (list)->head;
        (list_node_t *)(pos) != NULL;
        (list_node_t *)(pos) = (list_node_t *)pos->next)
})

#define list_for_each_t(list, pos) ({
    for((list_node_t *)(pos) = (list)->tail;
        (list_node_t *)(pos) != NULL;
        (list_node_t *)(pos) = (list_node_t *)pos->prev)
})

#define list_entry(ptr, type) ({
    (type *)(ptr)
})

#ifdef __GNUC__
stadic inline void list_free(list_t **list){
    list_node_t *pos;
    list_for_each_h(*list, pos){
        list_del(*list, pos);
    }
    free(*list);
    *list = NULL;
}
#else
void list_free(list_t **list);
#endif

/* list.c */

void list_alloc(list_t **list){
    *list = (list_t *)malloc(sizeof(list_t));
    if(*list == NULL)
        return;
    list_init(*list);
}

void list_free(list_t **list){
    free(*list);
    *list = NULL;
}

/* example.c */

#include "list.h"

typedef struct fox{
    list_node_t node;
    int age;
    char *name;
}fox_t;

int main(){
    list_t *list;
    list_alloc(&list);
    if(list == NULL){
        //error handling
    }
    fox_t *fox1 = (fox_t *)malloc(sizeof(fox_t));
    fox_t *fox2 = (fox_t *)malloc(sizeof(fox_t));
    fox_t *fox3 = (fox_t *)malloc(sizeof(fox_t));
    fox_t *pos;
    list_add_head(list, fox1);
    list_add_head(list, fox2);
    list_add_tail(list, fox3);
    list_for_each_h(list, pos){
        printf("%d %s\n", fox1->age, fox1->name);
    }
    list_for_each_t(list, pos){
        printf("%d %s\n", fox1->age, fox1->name);
    }
    list_del(list, fox2);
    list_free(&list);
    return 0;
}
```
这样，我们就实现了一个通用链表，并且可以对其进行一些
基础的操作，比如添加、删除、遍历等。当然，也可可以添加
一些其他的操作，比如合并，查找等等。


## 通用队列

至于通用队列，其实是和通用链表差不多，就算是留给你的课后作业吧  
(~~绝对不是应为我懒得写~~)  
(~~绝对不是~~)  
(~~！！！~~)

## 总结

通过这种方式使用链表和队列，可以大大减少代码量，降低维护成本。
当然，通用链表和队列的实现还有很多细节需要考虑，比如内存分配、内存释放、内存泄漏等问题，
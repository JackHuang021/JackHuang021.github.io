---
title: Linux内核链表结构
tags:
  - Linux
  - List
categories:
  - Linux
abbrlink: 4896cd7d
date: 2022-10-24 10:06:42
---


+ Linux内核中，对于数据管理，提供了2种类型的双向链表，一种是使用list_head结构体构成的双向环形链表。

<!-- more -->
#### list_head链表
+ *list_head*结构体定义在*include/linux/types.h*中，定义如下：
    ```c
    struct list_head {
        struct list_head *next, *prev;
    };
    ```
    *list_head*组成的双向链表，仅包含两个成员，*next*和*prev*指针，分别指向下一个和前一个*list_head*
    
+ *list_head*的简单应用
    ```c
    struct my_list {
        void *item;
        struct list_head list1;
        struct list_head list2; 
    };
    ```

+ *list_head*初始化分为静态初始化和动态初始化
    + 静态初始化
        ```c
        #define LIST_HEAD_INIT(name) { &(name), &(name) }
        #define LIST_HEAD(name) struct list_head name = LIST_HEAD_INIT(name)

        /* 例如 */
        LIST_HEAD(my_list);
        /* 展开即为 */
        struct list_head mylist = { &(mylist), &(mylist) };
        ```

    + 动态初始化
        ```c
        #define LIST_HEAD_INIT(name) { &(name), &(name) }
        /* 或者 */
        struct inline void INIT_LIST_HEAD(struct list_head *list)
        {
            list->next = list
            list->prev = list;
        }
        /* 例如 */
        struct list_head mylist = LIST_HEAD_INIT(mylist);
        struct list_head mylist2;
        INIT_LIST_HEAD(&mylist2);
        ```

+ 插入节点
    ```c
    static inline void __list_add(struct list_head *new,
                      struct list_head *prev,
                      struct list_head *next)
    {
        next->prev = new;
        new->next = next;
        new->prev = prev;
        prev->next = new;
    }

    /* new插入到head后 */
    static inline void list_add(struct list_head *new,
                      struct list_head *head)
    {
        __list_add(new, head, head->next);
    }

    /* new插入到head之前 */
    static inline void list_add_tail(struct list_head *new,
                      struct list_node *head)
    {
        __list_add(new, head->prev, head);
    }
    ```

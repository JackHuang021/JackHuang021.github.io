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

Linux内核中，对于数据管理，提供了2种类型的双向链表，一种是使用list_head结构体构成的双向环形链表。
![](https://raw.githubusercontent.com/JackHuang021/images/master/20221103103152.png)

<!-- more -->
#### list_head链表
`list_head`结构体定义在`include/linux/types.h`中，定义如下：
```c
struct list_head {
    struct list_head *next, *prev;
};
```
`list_head`组成的双向链表，仅包含两个成员，`next`和`prev`指针，分别指向下一个和前一个`list_head`

`list_head`一般不是单独使用的，一般用来嵌入到其他结构体中，知道`list_head`指针时就可以通过`include/linux/list.h`中提供的`list_entry`宏来获取它父结构的地址，其中调用了`container_of`宏，该宏定义在`include/linux/kernel`
```c
/* list_head 使用示例 */
struct my_list {
    void *item;
    struct list_head list;
};

/* list_entry */
/**
* list_entry - get the struct for this entry
* @ptr:	the &struct list_head pointer.
* @type:	the type of the struct this is embedded in.
* @member:	the name of the list_head within the struct.
*/
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)

/**
* container_of - cast a member of a structure out to the containing structure
* @ptr:	the pointer to the member.
* @type:	the type of the container struct this is embedded in.
* @member:	the name of the member within the struct.
*
*/
#define container_of(ptr, type, member) ({ \
    void *__mptr = (void *)(ptr); \
    ((type *)(__mptr - offsetof(type, member))); })

/* offsetof */
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```
`offsetof`获取结构体成员在结构体中地址的偏移量  
`container_of`的作用是通过结构体的成员地址获取结构体变量的地址，container_of一共
需要传入三个参数，`ptr`指针地址，`type`结构体类型，`member`结构体成员名称，具体
的做法就是通过成员的指针地址，减去成员在结构体中偏移的地址

#### `list_head`链表操作
+ 从`list_head`中获取对象结构体
    ```c

    /**
     * @ptr:	the list head to take the element from.
     * @type:	the type of the struct this is embedded in.
     * @member:	the name of the list_head within the struct.
     */

    // 实际就是使用container_of通过list_head地址来获取原结构体的地址
    #define list_entry(ptr, type, member) \
        container_of(ptr, type, member)

    // 获取链表上的第一个节点，这里ptr传入的默认应该是链表头节点，且传入的这个ptr不能为空指针
    #define list_first_enrty(ptr, type, member) \
        container_of((ptr)->next, type, member)

    // 获取链表上的最后一个节点，通过双向链表的prev指针来获取
    #define list_last_entry(ptr, type, member) \
        container_of((ptr)->prev, type, member)

    // 判断当前给入的链表是否为空链表，不为空表返回下一个节点对应的结构体地址
    #define list_first_enrty_or_null(ptr, type, member) ({ \
        struct list_head *head__ = (ptr); \
        struct list_head *pos__ = READ_ONCE(head__->next); \
        pos__ != head__ ? list_entry(pos__, type, member) : NULL; \
    })

    /**
    * @pos:     含list_head结构体对象的指针
    * @member:  list_head在这个结构体中的成员名
    */
    // 通过结构体对象的指针来获取链表中下一个节点的结构体对象地址
    #define list_next_entry(pos, member) \
        list_entry((pos)->member.next, typeof(*(pos)), member)

    


    ```
+ `list_head`初始化，`list_head`初始化分为静态初始化和动态初始化
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
          WRITE_ONCE(list->next, list);
          list->prev = list;
      }
      /* 例如 */
      struct list_head mylist = LIST_HEAD_INIT(mylist);
      
      struct list_head mylist2;
      INIT_LIST_HEAD(&mylist2);
      ```

+ 从`list_head`中取得目标结构体
    ```
    

    ```

+ `list_head`增加节点
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

+ 遍历链表
    + 类型安全的遍历整个链表
    ```c
    /**
    * list_for_each_entry_safe - iterate over list of given type safe against removal of list entry
    * @pos:	    the type * to use as a loop cursor.
    * @n:		another type * to use as temporary storage
    * @head:	the head for your list.
    * @member:	the name of the list_head within the struct.
    */
    #define list_for_each_entry_safe(pos, n, head, member)			\
        for (pos = list_first_entry(head, typeof(*pos), member),	\
            n = list_next_entry(pos, member);			\
            !list_entry_is_head(pos, head, member); 			\
            pos = n, n = list_next_entry(n, member))
    ```
    `platform`总线初始化的时候`platform_bus_init()`会调用`early_platform_clean()`来清除`early_platform_device_list`链表，代码如下：
    ```c

    static __initdata LIST_HEAD(early_platform_device_list);

    /**
     * early_platform_cleanup - clean up early platform code
     */
    void __init early_platform_cleanup(void)
    {
        struct platform_device *pd, *pd2;

        /* clean up the devres list used to chain devices */
        list_for_each_entry_safe(pd, pd2, &early_platform_device_list,
                     dev.devres_head) {
            list_del(&pd->dev.devres_head);
            memset(&pd->dev.devres_head, 0, sizeof(pd->dev.devres_head));
        }
    }

    // 展开即为
    void __init early_platform_cleanup(void)
    {
        struct platform_device *pd, pd2;

        for (pd = list_first_entry(&early_platform_device_list, 
                            typeof(*pd), dev.devres_head), 
             pd2 = list_next_entry(pd, dev.devres_head);
             !list_entry_is_head(pd, &early_platform_device_list, 
                                dev.devres_head);
             pd = pd2, pd2 = list_next_entry(pd2, dev.devres_head))
        {
            list_del(&pd->dev.devres_head);
            memset(&pd->dev.devres_head, 0, sizeof(pd->dev.devres_head));
        }
    }
    ```

    



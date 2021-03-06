---
layout: post
title:  "链表"
date:   2015-10-22 00:00:00
categories: Data-Structures
---

链表Linked list是一种常见的基础数据结构,是一种线性表,但是并不会按线性的顺序存储数据,而是在每一个节点里存到下一个节点的指针(Pointer)。由于不必须按顺序存储链表,在插入的时候可以达到O(1)的复杂度,比另一种线性表顺序表快得多,但是查找一个节点或者访问特定编号的节点则需要O(n)的时间,而顺序表相应的时间复杂度分别是O(logn)和O(1)。
链表允许插入和移除表上任意位置上的节点,但是不允许随机存取。链表有很多种不同的类型:单向链表,双向链表以及循环链表。

* 单链表
![single](https://upload.wikimedia.org/wikipedia/commons/thumb/6/6d/Singly-linked-list.svg/408px-Singly-linked-list.svg.png)

* 双向链表
![double](https://upload.wikimedia.org/wikipedia/commons/thumb/5/5e/Doubly-linked-list.svg/610px-Doubly-linked-list.svg.png)

* 循环链表
![circular](https://upload.wikimedia.org/wikipedia/commons/thumb/d/df/Circularly-linked-list.svg/350px-Circularly-linked-list.svg.png)

<!--more-->


#操作实现

[Chrome中的完整实现](https://code.google.com/p/chromium/codesearch#chromium/src/base/containers/linked_list.h&q=linked_list&sq=package:chromium&type=cs&l=5)

{% highlight c %}
// Questions and Answers:
//
// Q. Should I use std::list or base::LinkedList?
//
// A. The main reason to use base::LinkedList over std::list is
//    performance. If you don't care about the performance differences
//    then use an STL container, as it makes for better code readability.
//
//    Comparing the performance of base::LinkedList<T> to std::list<T*>:
//
//    * Erasing an element of type T* from base::LinkedList<T> is
//      an O(1) operation. Whereas for std::list<T*> it is O(n).
//      That is because with std::list<T*> you must obtain an
//      iterator to the T* element before you can call erase(iterator).
//
//    * Insertion operations with base::LinkedList<T> never require
//      heap allocations.
//
// Q. How does base::LinkedList implementation differ from std::list?
//
// A. Doubly-linked lists are made up of nodes that contain "next" and
//    "previous" pointers that reference other nodes in the list.
//
//    With base::LinkedList<T>, the type being inserted already reserves
//    space for the "next" and "previous" pointers (base::LinkNode<T>*).
//    Whereas with std::list<T> the type can be anything, so the implementation
//    needs to glue on the "next" and "previous" pointers using
//    some internal node type.
{% endhighlight %}


以下代码摘自Linux内核2.6.21.5源码(部分)展示了链表的另一种实现思路未采用ANSI C标准采用GNU C标准遵从GPL版权许可。

{% highlight c %}
struct list_head {
        struct list_head *next, *prev;
};

#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
        struct list_head name = LIST_HEAD_INIT(name)

static inline void INIT_LIST_HEAD(struct list_head *list)
{
        list->next = list;
        list->prev = list;
}

static inline void __list_add(struct list_head *new,
                              struct list_head *prev,
                              struct list_head *next)
{
        next->prev = new;
        new->next = next;
        new->prev = prev;
        prev->next = new;
}

static inline void list_add(struct list_head *new, struct list_head *head)
{
        __list_add(new, head, head->next);
}

static inline void __list_del(struct list_head * prev, struct list_head * next)
{
        next->prev = prev;
        prev->next = next;
}


static inline void list_del(struct list_head *entry)
{
        __list_del(entry->prev, entry->next);
        entry->next = NULL;
        entry->prev = NULL;
}

#define __list_for_each(pos, head) \
        for (pos = (head)->next; pos != (head); pos = pos->next)

#define list_for_each_entry(pos, head, member)                          \
        for (pos = list_entry((head)->next, typeof(*pos), member);      \
             prefetch(pos->member.next), &pos->member != (head);        \
             pos = list_entry(pos->member.next, typeof(*pos), member))
{% endhighlight %}

## linus的实现

Torvalds大婶在slashdot上回答一些编程爱好者的提问,其中一个人问他什么样的代码是他所喜好的,大婶表述了自己一些观点之后举了一个指针的例子,让我们见识了什么才是core low-level kind of coding。

At the opposite end of the spectrum, I actually wish more people understood the really core low-level kind of coding. Not big, complex stuff like the lockless name lookup, but simply good use of pointers-to-pointers etc. For example, I've seen too many people who delete a singly-linked list entry by keeping track of the "prev" entry, and then to delete the entry, doing something like

在这段话的最后,我实际上希望更多的人了解什么是真正的核心底层代码。这并不像无锁文件名查询注：可能是git源码里的设计那样庞大、复杂,只是仅仅像诸如使用二级指针那样简单的技术。例如我见过很多人在删除一个单项链表的时候维护了一个"prev"表项然后删除当前表项,就像这样

{% highlight c %}
if (prev)
prev->next = entry->next;
else
list_head = entry->next;
{% endhighlight %}

and whenever I see code like that, I just go "This person doesn't understand pointers". And it's sadly quite common.

当我看到这样的代码时,我就会想“这个人不了解指针”。令人难过的是这太普遍了。

People who understand pointers just use a "pointer to the entry pointer", and initialize that with the address of the list_head. And then as they traverse the list, they can remove the entry without using any conditionals, by just doing a "*pp = entry->next".

了解指针的人会使用链表头的地址来初始化一个“指向表项指针的指针”。当遍历链表的时候,可以不用任何条件判断(注:指prev是否为链表头)就能移除某个表项只要写"*pp = entry->next"。

So there's lots of pride in doing the small details right. It may not be big and important code, but I do like seeing code where people really thought about the details, and clearly also were thinking about the compiler being able to generate efficient code (rather than hoping that the compiler is so smart that it can make efficient code *despite* the state of the original source code).

纠正细节是令人自豪的事。也许这段代码并非庞大而且重,要但我喜欢注重代码细节的人,以及那些清楚地了解如何编译出有效代码的人(而不是寄望于聪明的编译器来产生有效代码即使是那些原始的汇编代码)。

Torvalds举了一个单向链表的例子,但给出的代码太短了,有个爱好者阅读了这段话并给出了一个比较完整的代码。他的话我就不翻译了下面给出代码说明。

{% highlight c %}

// This person doesn’t understand pointers
typedef struct node
{
    struct node * next;
    ....
} node;

typedef bool (* remove_fn)(node const * v);

// Remove all nodes from the supplied list for which the 
// supplied remove function returns true.
// Returns the new head of the list.
node * remove_if(node * head, remove_fn rm)
{
    for (node * prev = NULL, * curr = head; curr != NULL; )
    {
        node * const next = curr->next;
        if (rm(curr))
        {
            if (prev)
                prev->next = next;
            else
                head = next;
            free(curr);
        }
        else
            prev = curr;
        curr = next;
    }
    return head;
}
{% endhighlight %}


{% highlight c %}
//Two star programming
void remove_if(node ** head, remove_fn rm)
{
    for (node** curr = head; *curr; )
    {
        node * entry = *curr;
        if (rm(entry))
        {
            *curr = entry->next;
            free(entry);
        }
        else
            curr = &entry->next;
    }
}

{% endhighlight %}

#参考

* [Wikipedia linked list](https://en.wikipedia.org/wiki/Linked_list)
* [two start programming](http://wordaligned.org/articles/two-star-programming)
* [Torvalds大婶：很多人不了解如何写核心底层代码](http://www.oldlinux.org/oldlinux/viewthread.php?tid=14575&extra=page%3D1)

---
layout: post
title:  "How Linux Kernel implements generic linked list: part 1"
date:   2020-11-20 22:12:12 +0000
categories: linked list, Linux, Linux kernel linked list
---

`Circular Doubly  Linked List` <- - -> is one of the most widely used `data structure` in the Linux system implementation. It holds importance in the scheduling of the processes(RunQueue),  the buffers cache, device driver implementation, and so on.

This is a 3 part series. 
- In the 1st part, we will take a look at building blocks of `circular doubly  linked list` in the Linux kernel.
- The 2nd part will focus on implementing the basic list routines and implement the complete routine.
- The 3rd part will focus on why generic list implementation is inevitably important in the process scheduling context, hence it would help in appreciating how such implementation help in optimising the process scheduling.

## Assumptions and notes
- Understands the basics of linked list data structure.
- For the intent of the learning, the majority of the `C` code snippets in this post is not following the Linux code style guide.
- Good grasp on the `C` pointers.

## Why generic linked list?
The `linked list` is quite a common data structure used through out the Linux implementation. Let's take a look at the simple search result in the [Linux kernel source code](https://github.com/torvalds/linux/search?q=INIT_LIST_HEAD) which initialize the `linked list`. Over the period, Linux community has implemented unified and generic API's hence the code duplication can be reduced.
Overall `linked list` has a finite set of routines such as `initialize`, `add_entry`, `remove_entry` and more in the Linux context. Hence it makes sense to generalize the `linked list` implementation while the data type it holds varies.

## What's the fundamental problem with the concrete type approach?
The `Node` structure; it's tightly bind `data` and `links` fields together.

Example: Let's consider the `Node` structure of rudimentary `linked list` which holds the process list of the type `task_t`.
For a simplicity, I've dropped the other details from the real `task_t` structure.

```c
struct task_t {
    //data properties
    int pid;

    //link properties
    task_t *prev;
    task_t *next;
}
```
Now `prev` and `next` are `pointers to the structure of concrete type task_t`. Hence it could not point to the data of any other type.

Hence in order to make the list implementation generic, link pointers needs to be independent of data type it points to.

## Solution: Separate the links from the structure implementation
```c
struct list_head {
    struct list_head *prev;
    struct list_head *next;
};
```
Hence `linked list` could be formed in the following way. Its generic enough now.

![basic list head](/assets/list_part1_1.PNG)

## Problem: What's the use of such structure and how to associate the data types?

What if a structure embed `list_head` as a field of `type struct`? In the case of `task_t` it would look like:
```c
struct task_t {
    int pid;

    struct list_head tasks;
}
```

![task_t list illustration and memory layout](/assets/list_part1_2.PNG)

Then despite this how to get the handle of the data type `task_t`?

## Solution: Calculate the offset of `list_head` to trace back the base address of the struct

The entry in the list of `task_t` is a structure. Hence memory allocated to fields of type `task_t` is contiguous. 
Then, how about calculating the memory `offset` of the `tasks`( or `tasks.next` or `tasks.prev`) and tracing back the `base address` of the `task_t` data in which  `tasks` field `contains`?

The following example demonstrates the calculating offset of any field relative to its structure.
```c
/// offset.c

#include <stdio.h>
#include <stdlib.h>

#define OFFSET_OF(T, x) (unsigned long long int) ((&(((T*)0) -> x))) //the macro calculate the offset of field x with respect to struct T

struct list_head {
    struct list_head *prev, *next;
};

struct task_t {
    int pid;
    struct list_head tasks;
};

int main() {
    unsigned long int off_pid, off_tasks;

    off_pid = OFFSET_OF(struct task_t, pid);
    off_tasks = OFFSET_OF(struct task_t, tasks);
    
    printf("task_t.pid offset    %li\n", off_pid);
    printf("task_t.tasks offset  %li\n", off_tasks);

    return 0;
}
```
```bash
$ gcc -o offset offset.c
$ ./offset              
  task_t.pid offset    0
  task_t.tasks offset  8
```

The following example illustrate how address are assigned to the field.

```c
//address.c

#include <stdio.h>
#include <stdlib.h>

struct list_head {
    struct list_head *prev, *next;
};

struct task_t {
    int pid;
    struct list_head tasks;
};

int main() {
    struct task_t t1;

    printf("%lu size pid\n", sizeof(t1.pid));
    printf("%lu size prev\n", sizeof(t1.tasks.prev));
    printf("%lu size next\n", sizeof(t1.tasks.next));

    printf("%p address of pid\n", &t1.pid);
    printf("%p address of prev\n", &t1.tasks.prev); //also same as t1.task
    printf("%p address of next\n", &t1.tasks.next);

    return 0;
}
```
```bash
$ gcc -o address address.c
$ ./address
  4 size pid
  8 size prev
  8 size next
  140732870370816 address of pid
  140732870370824 address of prev
  140732870370832 address of next
```
Note: that `pid` field has the size 4 bytes however the next address start after 8 bytes. It's because the compiler in this case adds the padding of 4 bytes, so the data can be accessed in less number of instructions(internal memory access optimization).

Now let's, get the handle to `task_t` data using one macro called `container_of` by tracing the base address type `task_t`. 
The following program illustrates how `CONTAINER_OF` and `OFFSET_OF` macros are used together.

```C
// generic_list_node.c

#include <stdio.h>
#include <stdlib.h>

#define OFFSET_OF(T, x) (unsigned long long int) ((&(((T*)0) -> x)))
#define CONTAINER_OF(x, T, name) (T*)((((char*)(x)) - OFFSET_OF(T,name)))

struct list_head {
    struct list_head *prev, *next;
};

struct task_t {
    int pid;
    struct list_head tasks;
};

int main() {   
    //~initialize list 
    struct list_head tasks_list; 

    struct task_t t1;
    t1.pid = 1274;

    //~insert task_t entry
    t1.tasks.next = &tasks_list;
    t1.tasks.prev = &tasks_list;
    tasks_list.next = &t1.tasks;
    tasks_list.prev = &t1.tasks;
    
    struct task_t *task = CONTAINER_OF(tasks_list.next, struct task_t, tasks);
    printf("original task address: %p,  retrieved task address: %p\n", &t1, task);
    
    printf("pid %d\n", task -> pid);

    return 0;
}
```
```bash
$ gcc -o node_access generic_list_node.c
$ ./node_access              
  original task address: 0x7ffee7dfc5f0, retrieved task address: 0x7ffee7dfc5f0
  pid 1274
```
Once the `list` is build on this ground, `tasks list` would look like as per the following diagram.
![task_t list illustration](/assets/list_part1_3.PNG)

## Putting this together

This is how the base of `circular doubly linked list` is built in the Linux Kernel. Please note that this is still a rudimentary implementation of the Kernel list.
On can refer the actual implementation of the `container_of` macro [kernel.h](https://github.com/torvalds/linux/blob/master/include/linux/kernel.h#L692) and `offsetof` macro [kernel.h](https://elixir.bootlin.com/linux/latest/source/tools/include/linux/kernel.h#L23)

In the next part, let's use this base then implement basic list routines. 
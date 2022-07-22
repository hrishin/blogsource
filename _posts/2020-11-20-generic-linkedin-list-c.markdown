---
layout: post
title:  "How Linux Kernel implements generic linked list: part 1"
date:   2020-11-20 22:12:12 +0000
categories: linked list, Linux, Linux kernel linked list
---

Circular Doubly  Linked List <- - -> is one of the most widely used data structure in the Linux system implementation. It holds importance in the scheduling of the processes(RunQueue), the buffers cache, device driver implementation, and so on.

This is a 3 part series. 
- In the 1st part, we will take a look at building blocks of circular doubly  linked list in the Linux kernel.
- The 2nd part will focus on implementing essential list routines.
- The 3rd part will focus on why such a generic list implementation is important in the process management context, hence it would help in appreciating optimizing process scheduling.

## Assumptions and notes for the readers
- Understanding the basics of linked list data structure.
- Good grasp on the `C` pointers.
- For the intent of the learning, the majority of the `C` code snippets in this post is not following the Linux code style guide.

## Why generic linked list?
The linked list is quite a common data structure used through out the Linux implementation. Let's take a look at the simple search result in the [Linux kernel source code](https://github.com/torvalds/linux/search?q=INIT_LIST_HEAD) which initialize the linked list.
Overall linked list has a finite set of routines such as `initialize`, `add_entry`,`remove_entry` and more in the Linux's context. Hence it makes sense to generalize the linked list implementation such that data type it holds can vary.

## What's the fundamental problem with the concrete type approach?
For example, let's consider a `Node` structure of rudimentary linked list which holds the process list of the type `task_t`.
For a simplicity, we've dropped the other details from the real `task_t` structure.
```c
struct task_t {
    //data properties
    int pid;

    //link properties
    task_t *prev;
    task_t *next;
}
```
The `task_t`(Node) structure, it tightly binds `data`(pid) and `links`(prev, next) fields together. Now `prev` and `next` are pointers to the structure of concrete type `task_t`. Hence it could not point to the data of any type other than `task_t`.
Hence in order to make the list implementation generic to hold any data type, `links` pointers needs to be independent of data type it points to. It means is should be able to point any addresses of any data type or `struct` entity.
One option is to use a pointer to the  type void `*void`, however the tradeoff with approach is loosing the type safety. Is there any other way?

## Solution: Separate the links from the structure implementation
```c
struct list_head {
    struct list_head *prev;
    struct list_head *next;
};
```
Lets think of `links` pointers role, its pointing to the `next` or `previous` node in list.
So what if we construct the links as a single struct which can point to `next` and `previous` links only? Hence the linked list could be formed as per the following diagram way. It's generic enough now to be part of any node or struct.

![basic list head](/assets/list_part1_1.PNG)

## Problem: What's the use of such structure and how to associate such links and data types?

What if a node struct embed `list_head` as a field? In the case of `task_t` it would look like:
```c
struct task_t {
    int pid;

    struct list_head tasks;
}
```

![task_t list illustration and memory layout](/assets/list_part1_2.PNG)

Then despite this how to get the handle of the data type `task_t` from list node field `tasks`(links)? 

## Solution: Calculate the offset of `tasks` to trace back the base address of the struct `task_t` instance 

The entry or node in the linked list of `task_t` is a structure. Hence memory allocated to fields of type `task_t` is contiguous which is assured. 
Then, how about tracing back the base address of the `task_t` data(instance) by calculating the relative memory offset of the `tasks`( or `tasks.prev` or `tasks.next` which is 8 or 16 bytes on 64 bit processor respectively) to the base address of the `task_t` data(instance) in which `tasks` field contains?

which means 
```
base_address of previous (type_t) = address(link.prev) - 8 bytes
base_address of next (type_t) = address(link.next) - 16 bytes
```

The following example demonstrates calculating the byte offset of any field relative to its structure using a macro `OFFSET_OF`. This macro accepts two arguments, the `structure type T` and `field name` belonging the structure. 
```c
/// offset.c

#include <stdio.h>

//calculates the byte offset of the field x with respect to its position in the struct T
#define OFFSET_OF(T, x) (unsigned long long int) (&(((T*)0) -> x))

struct list_head {
    struct list_head *prev; 
    struct list_head *next;
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
#include <stdint.h>


struct list_head {
    struct list_head *prev;
    struct list_head *next;
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

    printf("%lu address of pid\n", (uintptr_t)&t1.pid); //also same as t1
    printf("%lu address of prev\n", (uintptr_t)&t1.tasks.prev);
    printf("%lu address of next\n", (uintptr_t)&t1.tasks.next);

    return 0;
}
```
```bash
$ gcc -o address address.c
$ ./address               
4 size pid
8 size prev
8 size next
140720804185888 address of pid
140720804185896 address of prev
140720804185904 address of next

```
Note: that `pid` field has the size 4 bytes however the next address start after 8 bytes. It's because the compiler in this case adds the padding of 4 bytes, so the next data `prev` values can be accessed in a single instruction(internal memory access optimization). This data organization in the memory is known `byte alignment`. This alignment may vary based on 32 or 64 bit processor architecture.

Let's decompose `OFFSET_OF` macro and understand how it get the offset(location) of filed with `structure`.
```
#define OFFSET_OF(T, x) (unsigned long long int) ((&(((T*)0) -> x)))
...
...
off_tasks = OFFSET_OF(struct task_t, tasks);
...
```
Upon pre-processing the `OFFSET_OF(struct task_t, tasks)` macro, it gets expand into the following form.
```
off_pid = (unsigned long long int) (&((struct task_t*)0) -> pid);
```

The `&((struct task_t*)0) -> pid` casts zero value to pointer `struct task_t*` and gets the address of the `pid` field. Then cast its address to 8 bytes unsigned data. This is ~ of placing the struct instance at memory zero and then finding the offset of the field from 0th address in terms of the number of `bytes`.
In this case compiler don't dereference the `pid` field.



Now the next step is to get the handle to `task_t` data using another macro called `CONTAINER_OF` by tracing the base address type `task_t`.
The following program illustrates how `CONTAINER_OF` and `OFFSET_OF` macros are used together.

```c
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

This is how the base of `circular doubly linked list` is built in the Linux Kernel. Please note that this is still a rudimentary implementation of the Kernel's linked list.
One can refer the actual implementation of the `container_of` macro [container_of.h](https://github.com/torvalds/linux/blob/4b0986a3613c92f4ec1bdc7f60ec66fea135991f/include/linux/container_of.h#L17) and `offsetof` macro [kernel.h](https://elixir.bootlin.com/linux/latest/source/tools/include/linux/kernel.h#L23)

In the next part, let's use this base then implement essential list routines. 
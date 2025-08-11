---
layout: post
title: "On READ_ONCE and Memory Barriers in the Linux Kernel"
date: 2025-06-22
categories: systems
---

I've been reading *Understanding the Linux Kernel* by Daniel P. Bovet and Marco Cesati, specifically looking at how memory management works in the kernel. I came across this macro in a deep dive into the **buddy system allocator**: 

```c
#define list_first_entry_or_null(ptr, type, member) ({ \
	struct list_head *head__ = (ptr); \
	struct list_head *pos__ = READ_ONCE(head__->next); \
	pos__ != head__ ? list_entry(pos__, type, member) : NULL; \
})
```

At its surface, this is a simple macro that checks if a linked list is empty, and returns a pointer to its first entry if not. If we look a little closer, the read of the list entry is wrapped with the `READ_ONCE` macro, which expands to 

```c
#define READ_ONCE(x) (*(volatile typeof(x) *)&(x))
``` 

so we cast the variable to a `volatile` and then read that. This is an example of a *compiler barrier*, preventing the compiler from reordering instructions and doing optimisations that we don't want. This got me interested to learn more, and so this post highlights my research into memory barriers in Linux, both compiler and CPU barriers.

# Compiler Barriers


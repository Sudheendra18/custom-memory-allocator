# Table of Contents
&nbsp;[Introduction](#introduction)  <br/> 
&nbsp;[Build instructions](#build-instructions)  <br/> 
&nbsp;[What's wrong with Malloc?](#whats-wrong-with-malloc)  <br/> 
&nbsp;[Custom allocators](#custom-allocators)  <br/> 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Linear Allocator](#linear-allocator)  <br/> 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Stack Allocator](#stack-allocator)  <br/> 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Pool Allocator](#pool-allocator)  <br/> 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Free list Allocator](#free-list-allocator)  <br/> 
&nbsp;[Benchmarks](#benchmarks)  <br/> 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Time complexity](#time-complexity)  <br/> 
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Space complexity](#space-complexity)  <br/> 
&nbsp;[Summary](#summary)  <br/> 
&nbsp;[Last thoughts](#last-thoughts)  <br/> 
&nbsp;[Future work](#future-work)  <br/> 

# Introduction
Dynamic memory allocation on the heap via standard `malloc` and `free` is flexible but carries performance overhead. This project implements custom C++ memory allocators that manage pre-allocated memory arenas to achieve faster, more predictable allocations. The goal is to understand how common allocators work, what trade-offs they offer, and compare their performance.

# Build instructions

```bash
git clone https://github.com/Sudheendra18/custom-memory-allocator.git
cmake -S . -B build 
cmake --build build
```

# What's wrong with Malloc?
* **General purpose:** Designed to handle any allocation size (from 1 byte to gigabytes), leading to bookkeeping overhead and fragmentation.
* **Slow syscalls:** When expanding the heap, `malloc` transitions between user and kernel mode (`brk`/`mmap`), which is slow and non-deterministic.

# Custom allocators
Because programs often have predictable allocation patterns, custom allocators achieve superior performance by pre-allocating large memory arenas:
* **Fewer Mallocs:** Pre-allocate once to eliminate runtime system calls.
* **Specialized Data Structures:** Use lightweight tracking structures (offsets, intrusive linked lists) tailored to specific workloads.
* **Domain Constraints:** Imposing constraints (e.g., fixed sizes, LIFO order) enables fast $O(1)$ operations.

## Linear allocator
The simplest allocator. Maintains an offset pointer at the beginning of the memory arena and increments it sequentially for each allocation.

### Data structure
Requires only an offset pointer to track the current allocation boundary. No per-allocation metadata or headers are needed.

![Data structure of a Linear Allocator](docs/images/linear1.png)

_Complexity: **O(1)**_

### Allocate
Simply moves the offset pointer forward by the requested allocation size.

![Allocating memory in a Linear Allocator](docs/images/linear2.png)

_Complexity: **O(1)**_

### Free
Individual allocations cannot be freed. The entire memory arena is reset all at once in $O(1)$.

---

## Stack allocator
A LIFO (Last-In, First-Out) allocator. Similar to the Linear Allocator, but allows allocations to be rolled back in reverse order.

### Data structure
Maintains an offset pointer tracking the top of the stack. Each allocation is preceded by a small header containing block size and padding metadata.

![Data structure of a Stack Allocator](docs/images/stack1.png)

_Complexity: **O(N)** space overhead for headers_

### Allocate
Advances the offset pointer forward and stores an allocation header right before the returned memory block.

![Allocating memory in a Stack Allocator](docs/images/stack2.png)

_Complexity: **O(1)**_

### Free
LIFO rollback. Reads the block header and restores the offset pointer back to the previous allocation boundary.

![Freeing memory in a Stack Allocator](docs/images/stack3.png)

_Complexity: **O(1)**_

---

## Pool allocator
Splits a memory arena into equal, fixed-size chunks. It offers fast allocations and deallocations with zero internal fragmentation for uniform objects.

![Splitting scheme in a Pool Allocator](docs/images/pool1.png)

### Data structure
Tracks free blocks using a singly linked list. To eliminate extra memory overhead, this list is **intrusive**—the `NEXT` pointers are stored directly inside the unused memory chunks themselves.

![Linked List used in a Pool Allocator](docs/images/pool2.png)
![In memory Linked List used in a Pool Allocator](docs/images/pool3.png)

_Constraint: Chunk size must be at least as large as a pointer (`sizeof(void*)`)._  
_Complexity: **O(1)**_

### Allocate
Pops the first free block from the head of the linked list.

![Allocation in a Pool Allocator](docs/images/pool4.png)
![Random State of a Linked List in a Pool Allocator](docs/images/pool5.png)

_Complexity: **O(1)**_

### Free
Pushes the freed block back onto the head of the linked list.

_Complexity: **O(1)**_

---

## Free list allocator
A general-purpose allocator that imposes no restrictions on allocation order, deallocation order, or block sizes.

### Linked list data structure
Maintains an address-sorted linked list of free memory blocks and their sizes. Each allocated block is prefixed with an allocation header to track size and alignment.

![Data structure in a Free list Allocator](docs/images/freelist_seq1.png)

_Complexity: **O(M)** space overhead where M is the number of allocated blocks._

### Linked list Allocate
Searches the free list using **First-Fit** (first block that fits) or **Best-Fit** (smallest block that fits, minimizing fragmentation). The selected block is split, returning the requested size and keeping the remainder in the free list.

![Allocating in a Free list Allocator](docs/images/freelist_seq2.png)

_Complexity: **O(N)** where N is the number of free blocks._

### Linked list Free
Reads the block header, inserts the block back into the address-sorted free list, and merges adjacent contiguous free blocks (**Coalescence**) in $O(1)$ to prevent fragmentation.

![Freeing in a Free list Allocator](docs/images/freelist_seq3.png)

_Complexity: **O(N)** search + **O(1)** merge._

### Red black tree data structure
Using a Red-Black Tree keyed by size reduces search complexity from $O(N)$ to $O(\log N)$ while enabling fast Best-Fit allocation. An auxiliary doubly linked list maintains address order for $O(1)$ coalescence.

---

# Benchmarks
Benchmarked with varying operation counts using a fixed block size of 4,096 bytes.

## Time complexity
* **Malloc:** Slowest allocator ($O(N)$) due to kernel transitions and general-purpose bookkeeping.
* **Free List:** ~3x faster than malloc while remaining general-purpose ($O(N)$).
* **Pool, Stack, Linear:** Exhibit near-instantaneous runtime. (Initial slopes reflect one-time `Init()` arena setup).

![Time complexity of different allocators](docs/images/operations_over_time.png)

Excluding the one-time `Init()` setup clearly demonstrates true constant $O(1)$ runtime for Linear, Stack, and Pool allocators:

![Time complexity of different allocators without Init](docs/images/operations_over_time_no_init.png)

## Space complexity
All allocators scale linearly ($O(N)$) with total requested memory, as metadata headers represent a negligible fraction of large allocations.

![Space complexity of different allocators](docs/images/operations_over_space.png)

---

# Summary
* **Linear Allocator:** Best when all data has a common lifetime and can be freed together (e.g., per-frame game data).
* **Stack Allocator:** Best for scoped or nested allocations freed in reverse LIFO order.
* **Pool Allocator:** Best for high-frequency objects of uniform size (e.g., game entities, particles).
* **Free List Allocator:** General-purpose allocator for arbitrary sizes and lifetimes where `malloc` is too slow.

# Last thoughts
* Avoid dynamic memory allocations during performance-critical loops whenever possible.
* Specific-purpose allocators (**Linear, Stack, Pool**) drastically outperform general-purpose ones (**Free List, Malloc**).
* Choose the most constrained allocator that fits your data access patterns.

# Future work
* Implement 8-byte alignment optimization to eliminate headers.
* Implement a Red-Black Tree Free List allocator ($O(\log N)$).
* Implement Buddy and Slab allocators.
* Benchmark cache misses and spatial locality.

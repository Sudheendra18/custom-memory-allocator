# Custom Memory Allocators in C++

Fast, deterministic custom memory allocators in C++ designed to replace general-purpose `malloc`/`free`.

---

## Quick Start

```bash
git clone https://github.com/Sudheendra18/custom-memory-allocator.git
cd custom-memory-allocator
cmake -S . -B build && cmake --build build
```

---

## Why Custom Allocators?

- **Zero Syscall Overhead:** Pre-allocates a large arena once upfront; no runtime kernel transitions (`brk`/`mmap`).
- **High Cache Locality:** Allocations are kept contiguous in memory.
- **Constant Time:** Enforcing domain constraints enables true $O(1)$ allocation and deallocation.

---

## Comparison Matrix

| Allocator | Alloc | Free | Space | Best Use Case | Constraint |
| :--- | :---: | :---: | :---: | :--- | :--- |
| **Linear** | $O(1)$ | $O(1)$ (bulk) | $O(1)$ | Per-frame scratchpads | No individual deallocation |
| **Stack** | $O(1)$ | $O(1)$ | $O(N)$ headers | Scoped / nested allocations | Strict LIFO order |
| **Pool** | $O(1)$ | $O(1)$ | $O(1)$ (intrusive) | Game entities, particles | Fixed block size only |
| **Free List** | $O(N)$ | $O(N)$ | $O(M)$ headers | General-purpose dynamic memory | Search overhead & fragmentation |

---

## Allocators

### 1. Linear Allocator
Advances an offset pointer forward sequentially. All memory is freed at once.

![Linear Allocator Data Structure](docs/images/linear1.png)
![Linear Allocator Allocation](docs/images/linear2.png)

- **Allocation:** $O(1)$ (bump pointer)
- **Free:** $O(1)$ (clears entire buffer)

---

### 2. Stack Allocator
LIFO allocator that rolls back allocations in reverse order using per-block headers.

![Stack Allocator Data Structure](docs/images/stack1.png)
![Stack Allocator Allocation](docs/images/stack2.png)
![Stack Allocator Free](docs/images/stack3.png)

- **Allocation:** $O(1)$
- **Free:** $O(1)$ (LIFO rollback)

---

### 3. Pool Allocator
Splits memory into equal fixed-size chunks using an intrusive linked list embedded directly inside free blocks (zero metadata overhead).

![Pool Splitting Scheme](docs/images/pool1.png)
![Pool Linked List Concept](docs/images/pool2.png)
![Intrusive Free List](docs/images/pool3.png)
![Pool Allocation](docs/images/pool4.png)
![Pool Interleaved State](docs/images/pool5.png)

- **Allocation:** $O(1)$ (pop free list head)
- **Free:** $O(1)$ (push to free list head)
- **Constraint:** Chunk size $\ge$ pointer size (8 bytes).

---

### 4. Free List Allocator
General-purpose allocator supporting arbitrary sizes and lifetimes with adjacent block coalescence.

![Free List Data Structure](docs/images/freelist_seq1.png)
![Free List Allocation](docs/images/freelist_seq2.png)
![Free List Free and Coalescence](docs/images/freelist_seq3.png)

- **Allocation:** $O(N)$ (First-Fit / Best-Fit search)
- **Free:** $O(N)$ (sorted insert) + $O(1)$ (coalescence)

---

## Benchmarks

Benchmarked with 4,096-byte blocks up to 1,000,000 operations.

### Time Complexity
- **Linear, Stack, Pool:** Constant $O(1)$ runtime.
- **Free List:** ~3.5x faster than `malloc`.
- **Malloc:** Slowest due to system call and general bookkeeping overhead.

![Time Complexity with Init](docs/images/operations_over_time.png)
![Time Complexity without Init](docs/images/operations_over_time_no_init.png)

### Space Complexity
All allocators scale linearly $O(N)$ with total requested memory.

![Space Complexity](docs/images/operations_over_space.png)

---

## When to Use What?

- **Linear:** Data discarded all at once (game frame, temporary level loader).
- **Stack:** Scoped/nested memory (recursive calls, tree walks).
- **Pool:** Many objects of the same size (entities, bullets, network packets).
- **Free List:** Unpredictable lifetimes and sizes where `malloc` is too slow.

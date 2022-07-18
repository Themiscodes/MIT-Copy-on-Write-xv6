# Copy-on-Write Fork for xv6

My implementation of copy-on-write fork in the xv6 kernel. This project is a lab assigment for the MIT course CS-081: Operating System Engineering of Fall 2021.

### The Problem

- The `fork()` system call in xv6 copies all of the parent process's user-space memory into the child. If the parent is large, copying can take a long time. Worse, the work is often largely wasted; for example, a `fork()` followed by `exec()` in the child will cause the child to discard the copied memory, probably without ever using it. On the other hand, if both parent and child use a page, and one or both writes it, a copy is truly needed.

### The Solution

- The goal of copy-on-write (COW) `fork()` is to defer allocating and copying physical memory pages for the child until the copies are actually needed, if ever.

- COW `fork()` creates just a pagetable for the child, with PTEs for user memory pointing to the parent's physical pages. COW `fork()` marks all the user PTEs in both parent and child as not writable. When either process tries to write one of these COW pages, the CPU will force a page fault. 

- The kernel page-fault handler detects this case, allocates a page of physical memory for the faulting process, copies the original page into the new page, and modifies the relevant PTE in the faulting process to refer to the new page, this time with the PTE marked writeable. When the page fault handler returns, the user process will be able to write its copy of the page.

## Implementation

- For each physical page, I decided to keep in a reference_counters array (protected with a spinlock) the number of user page tables that refer to that page. This way I set a page's reference count to one when `kalloc()` allocates it, I increment when fork causes a child to share the page, and decrement a page's count each time any process drops the page from its page table. `kfree()` places a page back on the free list if its reference counter drops to zero. This is because a physical page may be referred to by multiple processes' page tables and should be freed only when the last reference disappears.

- I initially modified `uvmcopy()` to map the parent's physical pages into the child, instead of allocating new pages. I set the PTE_W of both child and parent to 0, while making the COW flag to 1. This is my way to record, for each PTE, whether it is a COW mapping, by using the RSW (reserved for software) bits in the RISC-V PTE for this.

- I adapted the `copyout()` to handle page errors, that is, when it encounters a user page that has a copy on write mapping. Initially, I have set error handling for the fact that the virtual address is within the limits of the system. Then with `walk()` I get the pte from this address and check that it is not 0. I call kalloc to allocate a page and copy with `memmove()` the elements of the old physical page to it. The last step is to call `kfree()` to delete the old physical page or reduce its counter accordingly.
 
- Finally, I modified `usertrap()` to recognize page faults the same way. The error number is 15 in `r_scause()`. When a page-fault occurs on a COW page, I allocate a new page with `kalloc()`, copy the old page to the new page, and install the new page in the PTE with PTE_W set. Finally, to restart the command that caused the pagefault, I simply set the epc to the `r_sepc()`. If it was a COW page, it means that I do not need to create a new page, but it is enough to simply change its flags to writeable.

## Compilation & Execution

The RISC-V versions of a couple of different tools are required: QEMU 5.1+, GDB 8.3+, GCC, and Binutils. Then in the directory [xv6-project-2021](xv6-project-2021/):

- To compile and run xv6: 
```
$ make qemu
```

- To run all the tests: 
```
$ cowtest
$ usertests
```

You can find further information on the xv6 operating system in the [xv6 book](https://pdos.csail.mit.edu/6.828/2021/xv6/book-riscv-rev2.pdf).
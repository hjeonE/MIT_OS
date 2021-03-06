# Chapter 3 Page Tables
## Table of Contents
## Paging Hardware
### I. PTE
1. xv6 runs on `Sv39 RISC-V`, means only the bottom 39 bits of a 64-bit virtual address are used. Top 25 bits are not used.
2. xv6 page table is an array of 2^27 (27 comes from 39 - 12, 12 is the offset) `PTE`s (page table entries). Each PTE contains a 44-bit physical page number (`PPN`) and some flags. Which makes a 56-bit physical address. (2^56 bytes = 2^26 GB = 67108864 GB)
3. the offset 12 bits comes from page size 2^12 = 4096 bytes
### II. Three level structure
1. in sv39, page table is stored in physical memory as a three-level tree. The root is a 4096B page-table page that contains 512 PTEs (one is 8B).
2. Paging hardware raises a page-fault exception when an address is not present.
3. Three-level structure allows a page table to omit entire table pages when large ranges of virtual address have no mappings.
4. PTE flag bits: `PTE_V` - validity, `PTE_R` - read, `PTE_W` - write, `PTE_X` - execute, `PTE_U` - whether it's allowed in user mode code.
### III. Physical Address and Virtual address
1. each CPU keeps the physical address of root page-table into the `satp` register. Different CPU can run different processes.
2. physical memory refers to storage cells in DRAM. Instructions can only use virtual address.
## Kernel Address Space
### I. Layout
1. QEMU simulates a computer includes RAM starting at physical address 0x80000000(Kernel Base) and continuing through at least 0x86400000, which xv6 calls `PHYSTOP`.
### II. Direct Mapping
1. Kernel is located at same address in both virtual and physical address. Direct mapping simplifies the kernel code.
2. a couple of kernel virtual addresses that aren't direct-mapped: a. the trampoline page(mapped twice, once physical and once direct). b. kernel stack pages, each process has its own kernel stack.
### III. Mapping
1. kernel maps the pages for the trampoline and the kernel text with the permissions `PTE_R` and `PTE_X`
2. kernel maps the other pages with the permissions `PTE_R` and `PTE_W`.
3. mappings for guard pages are invalid.
## Creating an address space
### I. data structures and functions
1. central data structure is `pagetable_t` which is really a pointer to a RISC-V root page-table page. It can refer to both `kernel page table` and one of the `process page tables`.
2. central functions are a. `walk`, which finds the PTE for a virtual address. b. `mappages`, which installs PTEs for new mappings.
3. functions starts with `kvm` manipulate the kernel page table. `uvm` manipulate a user page table. other are used for both.
4. `copyout` and `copyin` copy data to and from user vm provided as system call arguments. They are in `vm.c` because they need to translate address to find the corresponding physical memory.
### II. setup procedures
1. `main` calss `kvminit` to create kernel's page table (physical memory).
2. `kvminit` allocates a page of physical memory to hold the root page-table page. 
3. Then it calls `kvmmap` to install the translations that the kernel needs.
4. `kvmmap` calls `mappages` to install mappings to a page table for a range of virtual address
5. for each virtual address to be mapped, `mappages` calls `walk` to find the address of the `PTE` for that address. It then initialized the `PTE` and mark it with flags.
6. `walk` mimics the RISCV paging hardware as it looks up the `PTE` for a virtual address. If `alloc` is set, `walk` allocates a new page-table page and puts its physical address in the PTE. It returns the PTE in the lowest layer in the tree.
Note: above code depends on physical memory being direct-mapped into kernel virtual address space.
### III. `kvminithart`
1. `main` calls `kvminithart` to install the kernel page table. It writes the physical address of the root page-table page into register `satp`. 
2. After this, CPU will translate address using the kernel page table. Since the kernel uses an identity mapping, the now virtual address of the next instruction will map to the right physical memory address.
### IV. `procinit`
1. `main` calls `procinit` to allocate a kernel stack for each process. It maps each stack at the virtual address generated by `KSTACK`, which leaves room for the invalid stack-guard page.
2. `kvmmap` adds the mapping PTEs to the kernel page table, and the call to `kvminithart` reloads the kernel page table into `satp` so that the hardwares knows about the PTEs.
### V. `Translation Look-aside Buffer (TLB)`
1. Each RISC-V CPU caches page table entries in a TLB
2. when xv6 changes a page table, it must tell the CPU to invalidate corresponding cached TLB entries. If it didn't, then at some point later the TLB might use an old cached mapping, pointing to a physical page that in the meantime has been allocated to another process.
3. RISC-V has an instruction `sfence.vma` that flushes the current CPU's TLB. 
4. xv6 executes `sfence.vma` in `kvminithart` after reloading the `sapt` register, and in the `trampoline` code that switches to a user page table before returning to user space.
## Physical memory allocation
xv6 uses the physical memory between the end of the kernel and `PHYSTOP` for run-time allocation. It allocates and frees whole 4096B pages at a time.
## Code: Physical Memory Allocator
### I. free list
1. allocator resides in `kalloc.c` with data structure `free list` of physical memory pages available. Each element is a `struct run`. 
2. allocator stores each free page's `run` structure in the free page itself, since there is nothing else stored there.
3. free list is protected by a `spin lock`. The list and the lock are wrapped in a struct to make clear that the lock protects the fields in the struct.
### II. `kinit`
1. `main` calls `kinit` to initialize the allocator. `kinit` initializes the `free list` to hold every page between the end of the kernel and `PHYSTOP`.
2. xv6 didn't determine how much physical memory is available, instead it assumes that the machine has 128MB of RAM.
3. `kinit` calls `freerange` to add memory to the free list via per-page calls to `kfree`. 
4. a PTE can only refer to a physical address that is multiple of 4096B, so `freerange` uses `PGROUNDUP` to align it.
### III. types - int and pointer
allocator sometimes treats addresses as a. int to perform arithmetic on them (traversing all pages in `freerange`). b. pointers to read and write memory.
### IV. `kfree`
1. `kfree` begins by setting every byte in the memory being freed to 1 to make it garbage to break the code reading it later faster.
2. then `kfree` prepends the page to the free list. It casts `pa` to a pointer to `struct run`, records the old start of the free list in `r->next` and sets the free list equal to `r.kalloc` removes and returns the first element in the free list.
## Process address Space
### I. range
process's user memory starts at virtual address 0 and can grow up to `MAXVA`, allowing a process to address in principle 256G of memory.
### II. grow memory
1. when a process asks for more `user memory`, xv6 first uses `kalloc` to allocate physical pages.
2. Then adds PTEs to the process's page table that points to the new pa. xv6 sets the `PTE_W`, `PTE_X`, `PTE_R`, `PTE_U` and `PTE_V` flags in these PTEs. xv6 leaves `PTE_V` clear in unused PTEs.
### III. examples of use of page tables.
1. each process has private user memory.
2. each process sees its memory contiguous addresses starting at 0, while the physical memory can be non-contiguous.
3. the kernel maps a page with trampoline code at the top of user address space. Thus a single page of physical memory shows at in all address spaces.
### IV. user stack
grows from high to low, guard page is lower.
## Code: sbrk
### I. implementation
1. `sbrk` is the system call for a process to shrink or grow its memory.
2. it's implemented by the function `growproc`, which calls `uvmalloc` or `uvmdealloc`, depending on whether `n` is pos or neg.
3. `uvmalloc` allocates physical memory with `kalloc` and adds PTEs to the user page table with `mappages`.
4. `uvmdealloc` calls `uvmunmap` which uses `walk` to find PTEs and `kfree` to free the physical memory they refer to.
### II. usage of process's page table
1. tell the hardware how to map user va.
2. as the only record of thich physical memory pages are allocated to that process.
## Code: exec
### I. overview
1. `exec` is the system call that creates the user part of an address space.
2. it opens `path` using `namei`, then reads the ELF header `struct elfhdr`, followed by a sequence of program section headers, `struct proghdr`, each describes a section of the application that must be loaded into memory.
3. xv6 programs have only one program section header, but other sytems might have separate sections for instructions and data.
### II. Procedures
1. quick check the magic number or `ELF_MAGIC`.
2. allocates a new page table with no user mappings with `proc_pagetable` allocates memory for each ELF segment with `uvmalloc` and loads each segment into memory with `loadseg`.
3. `loadseg` uses `walkaddr` to find the physical address of the allocated memory and `readi` to read from the file.
4. the program section header's `filesz` may be less than the `memsz`, indicating that the gap between should be filled with 0s (for C global variables) rather than read from the file.
5. `exec` allocates and initializes the user stack for just one page.
6. place the argument strings to the top of the stack one at the time. Recording the pointers to them in `ustack`. It places a null pointer at the end of what will be the `argv` list. First three entries in `ustack` are the fake return program counter, `argc` and `argv` pointer.
7. `exec` places an inaccessible page just below the stack page. If arguments too large, `copyout` will notice that and return -1.
### III. exception
1. if `exec` detects an error like an invalid program segment, it jumps to the label `bad`, frees the new image and returns -1.
2. `exec` must wait to free the old image until it's sure the system call will succeed.
3. if the old image is gone, the system call cannot return -1 to it.
4. the only error cases in `exec` happen during the creation of the image. Once the image is complete, `exec` can commit to the new page table and free the old one.
### IV. security checks for ELF
1. `exec` leads bytes from the ELF file into memory at addresses specified by the ELF file. Users or processes can place whatever addresses into ELF file. Thus `exec` is risky.
2. xv6 performs a number of checks, for example `ph.vaddr + ph.memsz < ph.vaddr` checks for whether the sum overflows a 64 bit integer.
## Real World
### I. paging
Most OS make far more sophisticated use of paging than xv6 by combining paging and page-fault exceptions.
### II. direct mapping
real world kernel address not necessary to start from 0x8000000, it may be allocate dynamically.
### III. protection
RISC-V supports protections at the level of physical addresses, but xv6 doesn't.
### IV. super pages.
1. On machines with lots of memory, it might make sense to use RISC-V's support for "super pages".
2. xv6 kernel doesn't have `malloc-like` cllocator that can provide memory for small objects. It prevents the kernel from using sophisticated data structures that would require dynamic allocation.
3. A more elaborate kernel would likely allocate many different sizes of small blocks, rather than (as in xv6) just 4MB blocks.

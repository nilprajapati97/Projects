Page tables are hierarchical data structures stored in main memory that map virtual pages to physical pages. Each entry in a page table contains not just address translation information, but also permission bits (read, write, execute), presence flags, and architecture-specific metadata.


The kernel maintains these page tables through a sophisticated memory management subsystem. In Linux, the core data structures include the memory descriptor (mm_struct), virtual memory areas (vm_area_struct), and the actual page tables themselves. The kernel provides architecture-independent interfaces that hide the differences between various MMU designs.

When a process requests memory through mmap() or brk(), the kernel typically doesn't immediately populate page tables. Instead, it creates a VMA (Virtual Memory Area) describing the mapping and defers actual page table creation until the process accesses the memory. This lazy allocation approach saves memory and improves performance.

Here's a simplified example of what happens during memory allocation:
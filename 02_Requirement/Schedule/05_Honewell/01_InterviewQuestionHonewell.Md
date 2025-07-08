Interview Question Honewell
=====================================

01. Write a program to reverse word in string.
02. Find no of 1's in number
03. Convert Little Enedian to Big.
04. Write program to check 16th bit is set or not. If Set clear 15th bits.
04. Write macro which complement 3rd bit in given number
05. What is device driver.
06. Explain TCP and UDP And explain handshaking in HDP and TCP.
07. Explain All api and flow how server coneect with clients.
08. In server how we can get data from perticular 5 client. And its fast data comming.
09. How we can write senario which can expect only one client connection.
10. Write a code to add node in Single Link List.
11. How we can send bulk (memory) data from user to kernel and they can acesss both.
12. How you will debug I2C driver wrt interrupt and Hw.
13. Write code for finding loop in linklist.
======================================================================================================

Nice list — full walkthrough from scratch for each item. I’ll assume C / Linux user-space & kernel contexts where relevant. I’ll number answers to match yours.

---

# 01. Reverse words in a string

Goal: reverse word order in-place (e.g. `"hello world here"` → `"here world hello"`). Approach: reverse entire string, then reverse each word.

```c
#include <stdio.h>
#include <string.h>
#include <ctype.h>

static void reverse_range(char *s, int i, int j) {
    while (i < j) {
        char t = s[i]; s[i] = s[j]; s[j] = t;
        i++; j--;
    }
}

void reverse_words(char *s) {
    int n = strlen(s);
    // 1) reverse whole string
    reverse_range(s, 0, n-1);

    // 2) reverse each word
    int i = 0;
    while (i < n) {
        while (i < n && isspace((unsigned char)s[i])) i++;    // skip spaces
        int start = i;
        while (i < n && !isspace((unsigned char)s[i])) i++;
        int end = i - 1;
        if (start <= end) reverse_range(s, start, end);
    }
}

int main(void) {
    char s[] = "  hello   world  here ";
    printf("orig: '%s'\n", s);
    reverse_words(s);
    printf("reversed: '%s'\n", s);
    return 0;
}
```

Notes:

* Preserves spacing at ends and compresses internal spaces? Above preserves original spacing except word order; if you want single spaces, you can build a new buffer.

---

# 02. Count number of `1` bits in an integer

Several ways: loop+shift, Kernighan’s algorithm (faster), builtin popcount.

Kernighan’s algorithm:

```c
unsigned int count_ones(unsigned int x) {
    unsigned int cnt = 0;
    while (x) {
        x &= (x - 1); // clears lowest-set bit
        cnt++;
    }
    return cnt;
}
```

GCC builtin (fast & uses CPU instruction if available):

```c
unsigned int cnt = __builtin_popcount(x);      // for 32-bit
unsigned long long cnt64 = __builtin_popcountll(x);
```

---

# 03. Convert Little Endian to Big Endian

If you have a 32-bit value and want to swap byte order:

```c
#include <stdint.h>

uint32_t swap32(uint32_t x) {
    return ((x & 0x000000FFU) << 24) |
           ((x & 0x0000FF00U) << 8)  |
           ((x & 0x00FF0000U) >> 8)  |
           ((x & 0xFF000000U) >> 24);
}
```

Portable helpers:

```c
#include <arpa/inet.h> // htonl, ntohl
// On little-endian system, htonl converts host -> network (network is big-endian).
uint32_t big = htonl(host_value);
```

For 16-bit use `htons` / `ntohs`.

---

# 04. Check 16th bit set; if set clear 15th bit

Interpretation: bits numbered starting at 1 for LSB (so 1→LSB, 16→bit index 15). If bit16 is set, clear bit15.

```c
#include <stdint.h>
#include <stdio.h>

uint32_t modify_bits(uint32_t v) {
    const uint32_t BIT16 = (1U << 15); // 16th
    const uint32_t BIT15 = (1U << 14); // 15th
    if (v & BIT16) {
        v &= ~BIT15; // clear 15th
    }
    return v;
}

int main() {
    uint32_t x = 0x8000; // example with 16th bit set
    printf("before 0x%08x\n", x);
    x = modify_bits(x);
    printf("after  0x%08x\n", x);
}
```

(If you count bits from 0, adjust masks accordingly — clarify convention in real code.)

---

# 04 (again). Macro to complement (toggle) 3rd bit

Assume 3rd bit means bit number 3 (LSB=1). Macro to toggle bit:

```c
#define TOGGLE_BIT3(x) ((x) ^ (1U << 2))  /* toggles 3rd bit (bit index 2) */

/* Usage:
   unsigned v = TOGGLE_BIT3(v);
*/
```

Explain: `(1U<<2)` is mask for bit 3 if counting from 1. Use parentheses to be safe.

---

# 05. What is a device driver (from scratch)

Short, practical explanation:

* A device driver is kernel software that provides a uniform interface for user-space and kernel subsystems to interact with hardware.
* Responsibilities:

  * Initialize the device and hardware resources (MMIO, IRQs, DMA).
  * Provide operations (read/write/ioctl) to user-space or other kernel components.
  * Handle interrupts and DMA completion.
  * Expose device capabilities (via sysfs, /dev nodes, netdev, etc).
* Types:

  * Character drivers (byte stream, `/dev/*`).
  * Block drivers (storage).
  * Network drivers (register with netstack).
  * Bus drivers / controller drivers (I2C master, SPI master).
* Key kernel interfaces:

  * `file_operations` for character/block devices.
  * `probe`/`remove` for bus/device model (PCI/USB/platform).
  * `net_device_ops` for network drivers.
* Typical flow: probe → allocate resources → register device node / netdev → implement open/read/write/ioctl/mmap/irq handler → on removal release resources.

---

# 06. Explain TCP and UDP and handshaking in UDP and TCP

TCP vs UDP:

* **UDP (User Datagram Protocol)**:

  * Connectionless, unreliable, no ordering guarantee.
  * Minimal header, low overhead.
  * Use when low-latency or application-layer reliability is preferred (e.g., DNS, VoIP, video streaming).
  * No handshake: sender just sends datagrams.

* **TCP (Transmission Control Protocol)**:

  * Connection-oriented, reliable, ordered, flow-control (windowing), congestion control.
  * Guarantees delivery via sequence numbers, ACKs, retransmissions.
  * Use for streams that require reliability (HTTP, SSH, file transfer).

**TCP 3-way handshake** (establish connection):

1. Client → Server : `SYN (seq = x)`
2. Server → Client : `SYN+ACK (seq = y, ack = x+1)`
3. Client → Server : `ACK (ack = y+1)`
   After this, connection established.

**TCP teardown**: 4-way FIN/ACK handshake (FIN from one side, ACK, then FIN other side, ACK).

**UDP handshake**: none. Some higher-level protocols implement their own handshake (e.g., QUIC runs over UDP and does handshake at application layer).

If you meant "HDP" or "HTTP" handshake — HTTP runs over TCP typically and uses TCP handshake to establish connection; HTTP itself is stateless request/response.

---

# 07. Explain all APIs and flow how server connects with clients (sockets)

Classic BSD sockets server flow (IPv4 TCP single-threaded):

1. `socket()` — create socket descriptor.
2. `bind()` — bind address (IP/port).
3. `listen(backlog)` — mark socket passive to accept connections.
4. `accept()` — accept incoming connection; returns new fd per client.
5. `recv()/send()` or `read()/write()` — I/O on client fd.
6. `close()` — close client fd. Optionally `shutdown()`.

Client side:

1. `socket()`
2. `connect(server_addr)`
3. `send()/recv()`, `close()`.

Important APIs for scalable servers:

* `select()` — portable, limited fd count, O(n).
* `poll()` — better than select, O(n).
* `epoll` (Linux) — scalable, O(1) for many fds (use `epoll_create1`, `epoll_ctl`, `epoll_wait`).
* `kqueue` (BSD/macOS).
* `accept4()` to set non-blocking and close-on-exec atomically.

Example server skeleton (single-threaded, accept multiple clients via `select`):

```c
/* short pseudo-skeleton (error checks omitted for brevity) */
int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
bind(listen_fd, ...);
listen(listen_fd, 128);
fd_set master, rfds;
FD_ZERO(&master);
FD_SET(listen_fd, &master);
int maxfd = listen_fd;

while (1) {
    rfds = master;
    select(maxfd+1, &rfds, NULL, NULL, NULL);
    for (int fd=0; fd<=maxfd; ++fd) {
        if (FD_ISSET(fd, &rfds)) {
            if (fd == listen_fd) {
                int cfd = accept(listen_fd, ...);
                FD_SET(cfd, &master);
                if (cfd > maxfd) maxfd = cfd;
            } else {
                // recv/handle client data
            }
        }
    }
}
```

---

# 08. In server how to get data from a particular 5th client and handle fast incoming data

Two concerns: addressing a particular client; handling high-rate data.

1. **Identifying the 5th client**:

   * Track accepted client fds in an array or map. The "5th client" can be the 5th entry in your connection list (order of acceptance).
   * Maintain a structure:

     ```c
     #define MAX_CLIENTS 1024
     int clients[MAX_CLIENTS];
     int client_count = 0;
     // on accept:
     clients[client_count++] = accepted_fd;
     // 5th client -> clients[4] (check bounds)
     ```
   * If you need stable identification even after others disconnect, assign a client ID at accept time and store mapping `client_id -> fd`.

2. **High-throughput / fast incoming data**:

   * Use non-blocking sockets and an I/O multiplexing mechanism (`epoll` on Linux) with edge-triggered mode + large read loops to drain socket.
   * Use per-client circular/ring buffer to store data quickly (avoid per-byte processing in the event loop).
   * Use `recvmmsg()`/`sendmmsg()` for batching UDP messages (Linux).
   * Use asynchronous I/O (aio/io\_uring) for very high throughput (modern Linux).
   * Avoid copying: for user-space protocol processing, use zero-copy where possible (splice(), sendfile() for files).
   * Use multiple threads or a thread-per-core model: accept connections on one thread, distribute fds to worker threads (lock-free queues).

Example using epoll (concept):

```c
// On incoming event:
if (event & EPOLLIN) {
    while (1) {
        ssize_t r = read(fd, buf, sizeof(buf));
        if (r > 0) push_to_client_buffer(fd, buf, r);
        else if (r == 0) /* closed */ break;
        else if (errno == EAGAIN) break;
        else /* error */ break;
    }
}
```

If you only want data from the 5th client:

* Look up its fd `clients[4]`. If you use `epoll`, you will still receive events for all fds; just ignore other fds or drop data if unnecessary. Better: remove other fds from interest set temporarily if you truly want to only process one client.

---

# 09. Scenario expecting only one client connection

Simple: server accepts one client and either stops listening or never `accept()` again.

Two approaches:

* `listen(listen_fd, 1)` and after first `accept`, close `listen_fd`.
* Or single-threaded loop that does `accept()` once, handle client, then exit.

Example:

```c
int listen_fd = socket(...);
bind(listen_fd, ...);
listen(listen_fd, 1);        // backlog 1
int client_fd = accept(listen_fd, ...);
close(listen_fd);            // stop accepting further
// handle client_fd
```

---

# 10. Add node in singly linked list (C)

Insert at head, tail, or after a node. Example functions:

```c
#include <stdlib.h>
#include <stdio.h>

typedef struct Node {
    int data;
    struct Node *next;
} Node;

Node* add_at_head(Node *head, int value) {
    Node *n = malloc(sizeof(*n));
    n->data = value;
    n->next = head;
    return n;
}

Node* add_at_tail(Node *head, int value) {
    Node *n = malloc(sizeof(*n));
    n->data = value;
    n->next = NULL;
    if (!head) return n;
    Node *p = head;
    while (p->next) p = p->next;
    p->next = n;
    return head;
}

void print_list(Node *head) {
    for (Node *p = head; p; p = p->next) printf("%d -> ", p->data);
    puts("NULL");
}
```

---

# 11. Send bulk (memory) data from user to kernel and both can access

Common methods:

1. **write()/read()**

   * User writes buffer to device node `/dev/mydev`. Kernel driver implements `.write` and uses `copy_from_user()` to pull data into kernel buffer.
   * Simple but copies user→kernel.

2. **ioctl**

   * For control and small transfers; not ideal for huge buffers.

3. **mmap**

   * Set up `mmap()` on device to map kernel buffer into user-space address space (shared memory). Kernel implements `mmap` file op and uses `remap_pfn_range()` or `vm_insert_page()` to map physical pages to user space.
   * Zero-copy between user and kernel (depending on implementation). Useful when both need shared access.

4. **splice()/vmsplice()**

   * Can move data between file descriptors in kernel without copying to user-space.

5. **DMA / scatter-gather** with `dma_buf` / `dma-buf` framework for sharing buffers across drivers/subsystems.

**Example sketch: mmap approach (high-level)**

* Kernel driver:

  * Allocate contiguous/shared pages (e.g., `dma_alloc_coherent()` or `get_user_pages()` for mapping).
  * Implement `struct vm_operations_struct` and `mmap` file op:

    ```c
    static int mydev_mmap(struct file *filp, struct vm_area_struct *vma) {
        unsigned long pfn = virt_to_phys(kernel_buffer) >> PAGE_SHIFT;
        return remap_pfn_range(vma, vma->vm_start, pfn, vma->vm_end - vma->vm_start, vma->vm_page_prot);
    }
    ```
* User space:

  * `fd = open("/dev/mydev", O_RDWR);`
  * `buf = mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);`
  * Now both sides access the same memory (careful with cache-coherency on DMA).

**Important:**

* Synchronization: use mutexes, futexes, or memory barriers to coordinate access.
* Cache management: if kernel uses DMA, ensure CPU cache is flushed/invalidated as needed.
* Security: validate sizes, permissions.

---

# 12. Debugging I2C driver wrt interrupts and hardware

Checklist and steps:

1. **Reproduce and isolate**:

   * Does the device respond at basic level? Use `i2cdetect` / `i2cget` / `i2cset` (userspace i2ctools) to probe the bus (if accessible).
2. **Check kernel logs**:

   * `dmesg`/`journalctl -k` for probe errors, IRQ registration errors, `printk()` outputs.
3. **Enable dynamic debug / verbose logging**:

   * Add `dev_dbg()`/`pr_debug()` and enable debug via `echo 'module driver +p' > /sys/kernel/debug/dynamic_debug/control` or set `#define DEBUG` appropriately.
4. **Verify hardware connections**:

   * SDA/SCL lines, pull-ups, correct voltage levels.
5. **Use a logic analyzer / Saleae**:

   * Capture SCL/SDA sequences to ensure correct start/stop, ACK/NACK, timings, repeated start sequences.
6. **Check IRQ flow**:

   * Verify IRQ number mapping, `request_irq()` success.
   * Add IRQ entry prints; ensure ISR fires when expected; check `irqreturn_t` and `wake_up` semantics.
7. **Check controller registers**:

   * Read controller MMIO registers to see status flags, error bits.
8. **Test with known-good code**:

   * Use Linux `i2c-core` adapter APIs or a simple master transfer tests to isolate whether issue is adapter or client driver.
9. **Concurrency and locking**:

   * Confirm proper locking for ISR and thread context; avoid sleeping in atomic context.
10. **Use `strace`/`perf`**:

    * To capture timing and syscalls for userspace testing.
11. **Simulate errors**:

    * Force NACKs or clock stretching to see driver response.
12. **Instrument with tracer**:

    * Use ftrace / tracepoints to view function call timing.

A good debug loop: reproduce → capture bus traces → add kernel logs → confirm register values → fix.

---

# 13. Detect loop in linked list (Floyd’s cycle detection)

Floyd’s Tortoise and Hare algorithm — O(n) time, O(1) memory.

```c
int has_cycle(Node *head) {
    Node *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return 1; // cycle detected
    }
    return 0; // no cycle
}

/* To find the start node of the cycle:
   After detecting cycle (meeting point), set slow=head, then
   advance slow and fast by one until they meet again; that node is cycle start.
*/
Node* find_cycle_start(Node *head) {
    Node *slow = head, *fast = head;
    // detect
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) break;
    }
    if (!fast || !fast->next) return NULL; // no cycle
    slow = head;
    while (slow != fast) {
        slow = slow->next;
        fast = fast->next;
    }
    return slow; // start of cycle
}
```

---

## Quick summary / tips

* Always document bit-numbering conventions in bit-manipulation code.
* For high-throughput servers prefer `epoll`/`io_uring` and per-client ring buffers.
* For kernel-user shared memory use `mmap()` for minimal copies; ensure sync and cache management.
* For I2C debugging, logic analyzers + kernel logs + register dumps are your best friends.
* Use standard, tested building blocks (Kernighan popcount, Floyd cycle detection).

---

If you want, I can:

* Provide a complete working TCP server example using `epoll` and handling the “5th client” case.
* Show a kernel `mmap` example (full kernel module + user program) for shared memory (longer, but I can paste it).
* Create unit tests for linked-list functions.

Which of these would you like next?

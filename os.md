<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Thread](#thread)
  - [Process vs Thread](#process-vs-thread)
  - [Kernal Thread vs User Thread](#kernal-thread-vs-user-thread)
    - [Kernel thread:](#kernel-thread)
    - [User thread:](#user-thread)
  - [Race Condition](#race-condition)
  - [Lock](#lock)
    - [Spinlock](#spinlock)
    - [Mutex](#mutex)
    - [Semarphore](#semarphore)
    - [Conditional variable](#conditional-variable)
      - [Why need to be w/ Mutex](#why-need-to-be-w-mutex)
    - [Read/Write Lock](#readwrite-lock)
  - [Common Issues in Multi-Threads](#common-issues-in-multi-threads)
    - [Deadlock](#deadlock)
      - [Prevention](#prevention)
    - [Priority Inversion](#priority-inversion)
      - [Solution: Priority inheritance](#solution-priority-inheritance)
- [Memory Management](#memory-management)
  - [Linux Memory Management](#linux-memory-management)
    - [Virtual memory](#virtual-memory)
    - [Buddy System: Page frame management](#buddy-system-page-frame-management)
      - [Buddy system algorithm](#buddy-system-algorithm)
    - [Slab: Page frame management](#slab-page-frame-management)
      - [Slab Algorithm](#slab-algorithm)
  - [OtherCommon Memory Management Schemes](#othercommon-memory-management-schemes)
    - [Pre-allocation](#pre-allocation)
    - [Reference counter](#reference-counter)
    - [COW(Copy on write)](#cowcopy-on-write)
- [RTOS](#rtos)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Thread

http://cs.mtu.edu/~shene/NSF-3/e-Book/FUNDAMENTALS/threads.html
In computer science, a thread of execution is the smallest sequence of programmed instructions that can be managed independently by a scheduler,

## Process vs Thread

1. Processes are not very efficient

	* Processes carry considerably more state information than threads, 

		Each process has its own PCB and OS resources. whereas multiple threads within a process share process state as well as memory and other resources

	* Typically high overhead for each process: e.g., 1.7 KB per task_struct on Linux!
	* Creating a new process is often very expensive

2. Processes don't (directly) share memory

	* Each process has its own address space
	* Parallel and concurrent programs often want to directly manipulate the same memory
		
		Note: Many OS's provide some form of inter-process shared memory, like UNIX ```shmget()``` and ```shmat()``` system call. Still, this requires more programmer work and does not address the efficiency issues.

## Kernal Thread vs User Thread


### Kernel thread:

* Pro: OS knows about all the threads in a process
	Can assign different scheduling priorities to each one,Kernel can context switch between multiple threads in one process

* Con: Thread operations require calling the kernel,Creating, destroying, or context switching require system calls

### User thread:
* Pro: Thread operations are very fast

	Typically 10-100x faster than going through the kernel. Thread state is very small,Just CPU state and stack, no additional overhead

* Con: If one thread blocks, it stalls the entire process

	E.g., If one thread waits for file I/O, all threads in process have to wait . Can't use multiple CPUs!

* Con: OS may not make good decisions. Could schedule a process with only idle threads Could deschedule a process with a thread holding a lock

## Race Condition

Race conditions arise in software when separate processes or threads of execution depend on some shared state. Operations upon shared states are critical sections that must be mutually exclusive in order to avoid harmful collision between processes or threads that share those states.

In concurrent programming a critical section is a piece of code that accesses a shared resource (data structure or device) that must not be concurrently accessed by more than one thread of execution.

## Lock

### Spinlock 
Spinlock perform a busy wait - i.e. it keeps running loop:
```C
while (try_acquire_resource ());
	//Do sth
release();
```
It performs very lightweight locking/unlocking but if the locking thread will be preempted by other which will try to access the same resouce the second one will simply try to acquitre resource untill it run out of it CPU quanta.


### Mutex

If the thread try to acquire blocked resource it will be suspended till it will be available for it. Locking/unlocking is much more heavy but the waiting is 'free' and 'fair'.

```C
if (!try_lock()) {
    add_to_waiting_queue ();
    wait();
}
...
process *p = get_next_process_from_waiting_queue ();
p->wakeUp ();   

```



### Semarphore

Semaphore is a lock that is allowed to be used multiple (known from initialization) number of times - for example 3 threads are allowed to simultaneously hold the resource but no more. It is used for example in producer/consumer problem or in general in queues:

semaphore is somewhat like an integer variable, but is special in that its operations (increment and decrement) are guaranteed to be atomic . In real implementation, atomic can be easily achieved by using mutex. 

In pthreads, the implementation of sempahores is pretty simple. Here it is

```C
typedef struct
{
        pthread_mutex_t lock;
        pthread_cond_t wait;
        int value;
} sema;

void pthread_sema_init(sema *s, int count)
{
        s->value = count;
        pthread_cond_init(&(s->wait),NULL);
        pthread_mutex_init(&(s->lock),NULL);
        return;
}

void pthread_sema_P(sema *s)
{
        pthread_mutex_lock(&(s->lock));
        s->value--;
        if(s->value < 0) {
                pthread_cond_wait(&(s->wait),&(s->lock));
        }
        pthread_mutex_unlock(&(s->lock));
        return;
}

void pthread_sema_V(sema *s)
{

        pthread_mutex_lock(&(s->lock));
        s->value++;
        if(s->value <= 0) {
                pthread_cond_signal(&(s->wait));
        }
        pthread_mutex_unlock(&(s->lock));
}

```

### Conditional variable

* Condition variables provide yet another way for threads to synchronize. While mutexes implement synchronization by controlling thread access to data, condition variables allow threads to synchronize based upon the actual value of data. 

* Without condition variables, the programmer would need to have threads continually polling (possibly in a critical section) spin wait, to check if the condition is met. This can be very resource consuming since the thread would be continuously busy in this activity. A condition variable is a way to achieve the same goal without polling. 

* A condition variable is always used in conjunction with a mutex lock.

#### Why need to be w/ Mutex

Here is the typical way to use a condition variable:

```
// The reader(s)
lock(some_mutex);
if(protected_by_mutex_var != desired_value)
    some_condition.wait(some_mutex);
unlock(some_mutex);

// The writer
lock(some_mutex);
protected_by_mutex_var = desired_value;
unlock(some_mutex);
some_condition.notify_all();
```

A simple atomic dose not work

```C
// Thread 1                                            
if(protected_by_mutex_var != desired_value) -> true 

// Thread 2                                   		
atomic_set(protected_by_mutex_var, 
desired_value); 
some_condition.notify_all();

// Thread 1  
some_condition.notify_all(); 

```





### Read/Write Lock

https://www.ibm.com/support/knowledgecenter/ssw_aix_72/generalprogramming/using_readwrite_locks.html

```CPP
class rw_lock_t {

    int NoOfReaders;
    int NoOfWriters, NoOfWritersWaiting;
    pthread_mutex_t class_mutex;
    pthread_cond_t  reader_gate;
    pthread_cond_t  writer_gate;

public:

    rw_lock_t()
    : NoOfReaders(0), NoOfWriters(0), NoOfWritersWating(0),
      class_mutex(PTHREAD_MUTEX_INITIALIZER),
      reader_gate(PTHREAD_COND_INITIALIZER),
      writer_gate(PTHREAD_COND_INITIALIZER)
    {}
    ~rw_lock_t()
    {
        pthread_mutex_destroy(&class_mutex);
        pthread_cond_destroy(&reader_gate);
        pthread_cond_destroy(&writer_gate);
    }
    void r_lock()
    {
        pthread_mutex_lock(&class_mutex);
        //while(NoOfWriters>0 || NoOfWritersWaiting>0) //Writer Preference
        while(NoOfWriters>0)
        {
            pthread_cond_wait(&reader_gate, &class_mutex);
        }
        NoOfReaders++;        
        pthread_mutex_unlock(&class_mutex);
    }
    void w_lock()
    {
        pthread_mutex_lock(&class_mutex);
        NoOfWritersWaiting++;
        while(NoOfReaders>0 || NoOfWriters>0)
        {
            pthread_cond_wait(&writer_gate, &class_mutex);
        }
        NoOfWritersWaiting--; NoOfWriters++;
        pthread_mutex_unlock(&class_mutex);
    }
    void r_unlock()
    {
        pthread_mutex_lock(&class_mutex);
        NoOfReaders--;
        if(NoOfReaders==0 && NoOfWritersWaiting>0)
            pthread_cond_signal(&writer_gate);
        pthread_mutex_unlock(&class_mutex);
    }
    void w_unlock()
    {
        pthread_mutex_lock(&class_mutex);
        NoOfWriters--;
        if(NoOfWritersWaiting>0)
            pthread_cond_signal(&writer_gate);
        //else //Writer Preference - don't signal readers unless no writers
        pthread_cond_broadcast(&reader_gate);
        pthread_mutex_unlock(&class_mutex);
    }
};

```



## Common Issues in Multi-Threads

### Deadlock
A deadlock is a situation in which two or more competing actions are each waiting for the other to finish, and thus neither ever does. Conditions: all needs to be met
1.  Mutual exclusion: at least one resource is not sharable
2.  Hold and wait: A process is currently holding at least one resource and requesting additional resources which are being held by other processes.
3.  No preemption 
4.  Circle wait

#### Prevention

https://en.wikipedia.org/wiki/Deadlock_prevention_algorithms


### Priority Inversion

Scenario:

1. L runs and acquires X 
2. Then H tries to access X while L has it, because of semaphore, H sleeps. 
3. M arrives, pre-empts L and runs. In effect, H & M were two processes waiting to run but M ran because H was waiting on lock and couldn't run.
4. M finishes, H can't enter because L has the lock, so L runs.
5. L finishes, relinquishes the lock. Now H gets the lock and executes.
H had the highest priority but ran after the lower priority processes had run. This is Priority Inversion. 

![priority_inversion](https://github.com/zhangruiskyline/system/blob/main/images/priority_inversion.png)



#### Solution: Priority inheritance

When a low priority process has is running and has a lock, and if a high priority process tries to acquire the lock, the priority of the low priority process is raised to the priority of the high priority process. That is, if L is running and has the lock, when H tries to acquire it, L's priority will be raised to that of the H for the duration L holds the lock. This way, M can't pre-emp it. After, L finishes, H runs and acquires the lock. After H is done, M runs preserving the priority ordering. 







# Memory Management

## Linux Memory Management

http://duartes.org/gustavo/blog/post/how-the-kernel-manages-your-memory

Linux processes are implemented in the kernel as instances of [task_struct](http://lxr.linux.no/linux+v2.6.28.1/include/linux/sched.h#L1075), the process descriptor. The [mm](http://lxr.linux.no/linux+v2.6.28.1/include/linux/sched.h#L1129) field in task_struct points to the memory descriptor, [mm_struct](http://lxr.linux.no/linux+v2.6.28.1/include/linux/mm_types.h#L173), which is an executive summary of a program’s memory. It stores the start and end of memory segments as shown above, the number of physical memory pages used by the process (rss stands for Resident Set Size), the amount of virtual address space used, and other tidbits. Within the memory descriptor we also find the two work horses for managing program memory: the set of virtual memory areas and the page tables. Gonzo’s memory areas are shown below:

![linux_memory_1](https://github.com/zhangruiskyline/system/blob/main/images/linux_memory_1.png)

Each virtual memory area (VMA) is a contiguous range of virtual addresses; these areas never overlap. An instance of [vm_area_struct](http://lxr.linux.no/linux+v2.6.28.1/include/linux/mm_types.h#L99) fully describes a memory area, including its start and end addresses, flags to determine access rights and behaviors, and the vm_file field to specify which file is being mapped by the area, if any. A VMA that does not map a file is anonymous. Each memory segment above (e.g., heap, stack) corresponds to a single VMA, with the exception of the memory mapping segment. This is not a requirement, though it is usual in x86 machines. VMAs do not care which segment they are in.
A program’s VMAs are stored in its memory descriptor both as a linked list in the [mmap](http://lxr.linux.no/linux+v2.6.28.1/include/linux/mm_types.h#L174) field, ordered by starting virtual address, and as a red-black tree rooted at the [mm_rb](http://lxr.linux.no/linux+v2.6.28.1/include/linux/mm_types.h#L175) field. The red-black tree allows the kernel to search quickly for the memory area covering a given virtual address. When you read file /proc/pid_of_process/maps, the kernel is simply going through the linked list of VMAs for the process and [printing each one](http://lxr.linux.no/linux+v2.6.28.1/fs/proc/task_mmu.c#L201).

![linux_memory_2](https://github.com/zhangruiskyline/system/blob/main/images/linux_memory_2.png)

### Virtual memory

The 4GB virtual address space is divided into pages. x86 processors in 32-bit mode support page sizes of 4KB, 2MB, and 4MB. Both Linux and Windows map the user portion of the virtual address space using 4KB pages. Bytes 0-4095 fall in page 0, bytes 4096-8191 fall in page 1, and so on. The size of a VMA must be a multiple of page size. Here’s 3GB of user space in 4KB pages:

![linux_vm_1](https://github.com/zhangruiskyline/system/blob/main/images/linux_vm_1.png)

The processor consults page tables to translate a virtual address into a physical memory address. Each process has its own set of page tables; whenever a process switch occurs, page tables for user space are switched as well. Linux stores a pointer to a process’ page tables in the pgd field of the memory descriptor. To each virtual page there corresponds one page table entry (PTE) in the page tables, which in regular x86 paging is a simple 4-byte record shown below:

![linux_vm_2](https://github.com/zhangruiskyline/system/blob/main/images/linux_vm_2.png)

Linux has functions to read and set each flag in a PTE. Bit P tells the processor whether the virtual page is present in physical memory. If clear (equal to 0), accessing the page triggers a page fault. Keep in mind that when this bit is zero, the kernel can do whatever it pleases with the remaining fields. The R/W flag stands for read/write; if clear, the page is read-only. Flag U/S stands for user/supervisor; if clear, then the page can only be accessed by the kernel. These flags are used to implement the read-only memory and protected kernel space we saw before.

Bits D and A are for dirty and accessed. A dirty page has had a write, while an accessed page has had a write or read. Both flags are sticky: the processor only sets them, they must be cleared by the kernel. Finally, the PTE stores the starting physical address that corresponds to this page, aligned to 4KB. This naive-looking field is the source of some pain, for it limits addressable physical memory to 4 GB. The other PTE fields are for another day, as is Physical Address Extension.

A virtual page is the unit of memory protection because all of its bytes share the U/S and R/W flags. However, the same physical memory could be mapped by different pages, possibly with different protection flags. Notice that execute permissions are nowhere to be seen in the PTE. This is why classic x86 paging allows code on the stack to be executed, making it easier to exploit stack buffer overflows (it’s still possible to exploit non-executable stacks using return-to-libc and other techniques). This lack of a PTE no-execute flag illustrates a broader fact: permission flags in a VMA may or may not translate cleanly into hardware protection. The kernel does what it can, but ultimately the architecture limits what is possible.

Virtual memory doesn’t store anything, it simply maps a program’s address space onto the underlying physical memory, which is accessed by the processor as a large block called the physical address space. While memory operations on the bus are somewhat involved, we can ignore that here and assume that physical addresses range from zero to the top of available memory in one-byte increments. This physical address space is broken down by the kernel into page frames. The processor doesn’t know or care about frames, yet they are crucial to the kernel because the page frame is the unit of physical memory management. Both Linux and Windows use 4KB page frames in 32-bit mode; here is an example of a machine with 2GB of RAM:

![linux_vm_3](https://github.com/zhangruiskyline/system/blob/main/images/linux_vm_3.png)


In Linux each page frame is tracked by a [descriptor](http://lxr.linux.no/linux+v2.6.28/include/linux/mm_types.h#L32) and several [flags](http://lxr.linux.no/linux+v2.6.28/include/linux/page-flags.h#L14). Together these descriptors track the entire physical memory in the computer; the precise state of each page frame is always known. Physical memory is managed with the [buddy memory allocation technique](https://en.wikipedia.org/wiki/Buddy_memory_allocation), hence a page frame is free if it’s available for allocation via the buddy system. An allocated page frame might be anonymous, holding program data, or it might be in the page cache, holding data stored in a file or block device. There are other exotic page frame uses, but leave them alone for now. Windows has an analogous Page Frame Number (PFN) database to track physical memory.
Let’s put together virtual memory areas, page table entries and page frames to understand how this all works. Below is an example of a user heap:

Let’s put together virtual memory areas, page table entries and page frames to understand how this all works. Below is an example of a user heap:

![linux_vm_4](https://github.com/zhangruiskyline/system/blob/main/images/linux_vm_4.png)

A VMA is like a contract between your program and the kernel. You ask for something to be done (memory allocated, a file mapped, etc.), the kernel says “sure”, and it creates or updates the appropriate VMA. But it does not actually honor the request right away, it waits until a page fault happens to do real work. The kernel is a lazy, deceitful sack of scum; this is the fundamental principle of virtual memory. It applies in most situations, some familiar and some surprising, but the rule is that VMAs record what has been agreed upon, while PTEs reflect what has actually been done by the lazy kernel. These two data structures together manage a program’s memory; both play a role in resolving page faults, freeing memory, swapping memory out, and so on. Let’s take the simple case of memory allocation:

![linux_vm_5](https://github.com/zhangruiskyline/system/blob/main/images/linux_vm_5.png)

When the program asks for more memory via the _brk()_ system call, the kernel simply [updates](http://lxr.linux.no/linux+v2.6.28.1/mm/mmap.c#L2050) the heap VMA and calls it good. No page frames are actually allocated at this point and the new pages are not present in physical memory. Once the program tries to access the pages, the processor page faults and [do_page_fault()](http://lxr.linux.no/linux+v2.6.28/arch/x86/mm/fault.c#L583) is called. It [searches](http://lxr.linux.no/linux+v2.6.28/arch/x86/mm/fault.c#L692) for the VMA covering the faulted virtual address using [find_vma()](http://lxr.linux.no/linux+v2.6.28/mm/mmap.c#L1466). If found, the permissions on the VMA are also checked against the attempted access (read or write). If there’s no suitable VMA, no contract covers the attempted memory access and the process is punished by Segmentation Fault.

When a VMA is [found](http://lxr.linux.no/linux+v2.6.28/arch/x86/mm/fault.c#L711) the kernel must [handle](http://lxr.linux.no/linux+v2.6.28/mm/memory.c#L2653) the fault by looking at the PTE contents and the type of VMA. In our case, the PTE shows the page is [not present](http://lxr.linux.no/linux+v2.6.28/mm/memory.c#L2674). In fact, our PTE is completely blank (all zeros), which in Linux means the virtual page has never been mapped. Since this is an anonymous VMA, we have a purely RAM affair that must be handled by [do_anonymous_page()](http://lxr.linux.no/linux+v2.6.28/mm/memory.c#L2681), which allocates a page frame and makes a PTE to map the faulted virtual page onto the freshly allocated frame.

Things could have been different. The PTE for a swapped out page, for example, has 0 in the Present flag but is not blank. Instead, it stores the swap location holding the page contents, which must be read from disk and loaded into a page frame by [do_swap_page()](http://lxr.linux.no/linux+v2.6.28/mm/memory.c#L2280) in what is called a major fault.

### Buddy System: Page frame management

In theory, paging eliminates the need for contiguous memory allocation, but…
– Some operations like DMA ignores paging circuitry and accesses the address bus directly while transferring data
* As an aside, some DMA can only write into certain addresses – Contiguous page frame allocation leaves kernel paging tables unchanged, preserving TLB and reducing effective access time
* As a result, Linux implements a mechanism for allocating contiguous page frames
So how does it deal with external fragmentation?

#### Buddy system algorithm

1. All page frames are grouped into 10 lists of blocks that contain groups of 1, 2, 4, 8, 16, 32, 64, 128, 256, and 512 contiguous page frames, respectively
    * The address of the first page frame of a block is a multiple of the group size, for example, a 16 frame block is a multiple of 16 × 212
    * The algorithm for allocating, for example, a block of 128 contiguous page frames
    * First checks for a free block in the 128 list
    * If no free block, it then looks in the 256 list for a free block
    * If it finds a block, the kernel allocates 128 of the 256 page frames and puts the remaining 128 into the 128 list
    * If no block it looks at the next larger list, allocating it and dividing the block similarly
    * If no block can be allocated an error is reported

2. When a block is released, the kernel attempts to merge together pairs of free buddy blocks of size b into a single block of size 2b
    * Two blocks are considered buddies if
        * Both have the same size
        * They are located in contiguous physical addresses
        * The physical address of the first page from of the first block is a multiple of 2b × 212
    * The merging is iterative
        Linux makes use of two different buddy systems, one for page frames suitable for DMA (i.e., addresses less than 16MB) and then all other page frames

3. Each buddy system relies on
    * The page frame descriptor array mem_map
    * An array of ten free_area_struct, one element for each group size; each free_area_struct contains a doubly linked circular list of blocks of the respective size
    * Ten bitmaps, one for each group size, to keep track of the blocks it allocates


### Slab: Page frame management

The buddy algorithm is fine for dealing with relatively large memory requests, but it how does the kernel satisfy its needs for small memory areas?
    - In other words, the kernel must deal with internal fragmentation
        - Linux 2.2 introduced the slab allocator for dealing with small memory area allocation
    - View memory areas as objects with data and methods (i.e.,constructors and destructors)
    - The slab allocator does not discard objects, but caches them(so next allocation of same object directly get cached memory)
    - Kernel functions tend to request objects of the same type repeatedly, such as process descriptors, file descriptors, etc.

#### Slab Algorithm

* Groups objects into caches
    - A set of specific caches is created for kernel operations
    - Each cache is a “store” of objects of the same type (for example, a file pointer is allocated from the filp slab allocator). Look in /proc/slabinfo for run-time slab statistics

* Slab caches contain zero or more slabs, where a slab is one or more contiguous pages frames from the buddy system

    - Objects are allocated using ```kmem_cache_alloc(cachep)```, where cachep points to the cache from which the object must be obtained
    - Objects are released using ```kmem_cache_free(cachep, objp)```
    - A group of general caches exist whose objects are geometrically distributed sizes ranging from 32 to 131072 bytes – To obtain objects from these general caches, use ```kmalloc(size, flags)```
    - To release objects from these general caches, use ```kfree(objp)```

![slab](https://github.com/zhangruiskyline/system/blob/main/images/slab.png)



## OtherCommon Memory Management Schemes

### Pre-allocation
Just as in dynamic array memory allocation: The ordinary steps will be re allocate a bigger memory, then copy the data from old cache to new memory, then free old memory.
So the solution is to pre allocate a bigger memory space. Then use this for memory allocation until it is used up(most of them pre allocate 1.5 times). It is mostly used containers with buffer. Such as vector and string

### Reference counter

In OO system, the cooperation among objects is very complicated.  Reference counter is a technique of storing the number of references, pointers, or handles to a resource such as an object, block of memory, disk space or other resource. If some object is referenced, then counter is increased. If object is not referenced anymore, then decrease it by 1. When the counter equals to 0, then no other object refers this one, then it could be destroyed.
It is widely used in garbage collection.

### COW(Copy on write) 

It is commonly used in OS to create child process(like fork). When child process is created, it will inherit the memory space from parents child. Copying the whole memory will be wasteful, especially when most child process will only read the inherited data, not modify them. 
COW means the copy at first is only reference counter increase where child and parents process share the same memory space. Only when writing, this memory is copied.


# RTOS


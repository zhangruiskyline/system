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

Linux processes are implemented in the kernel as instances of task_struct, the process descriptor. The mm field in task_struct points to the memory descriptor, mm_struct, which is an executive summary of a program’s memory. It stores the start and end of memory segments as shown above, the number of physical memory pages used by the process (rss stands for Resident Set Size), the amount of virtual address space used, and other tidbits. Within the memory descriptor we also find the two work horses for managing program memory: the set of virtual memory areas and the page tables. Gonzo’s memory areas are shown below:

![linux_memory_1](https://github.com/zhangruiskyline/system/blob/main/images/linux_memory_1.png)

Each virtual memory area (VMA) is a contiguous range of virtual addresses; these areas never overlap. An instance of vm_area_struct fully describes a memory area, including its start and end addresses, flags to determine access rights and behaviors, and the vm_file field to specify which file is being mapped by the area, if any. A VMA that does not map a file is anonymous. Each memory segment above (e.g., heap, stack) corresponds to a single VMA, with the exception of the memory mapping segment. This is not a requirement, though it is usual in x86 machines. VMAs do not care which segment they are in.
A program’s VMAs are stored in its memory descriptor both as a linked list in the mmap field, ordered by starting virtual address, and as a red-black tree rooted at the mm_rb field. The red-black tree allows the kernel to search quickly for the memory area covering a given virtual address. When you read file /proc/pid_of_process/maps, the kernel is simply going through the linked list of VMAs for the process and printing each one.

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


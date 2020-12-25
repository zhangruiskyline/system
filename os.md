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
- [Linux Memory Management](#linux-memory-management)
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
| Thread 1                                            | Thread 2
  if(protected_by_mutex_var != 
desired_value) -> true 
                                          		atomic_set(protected_by_mutex_var, 
                                             	desired_value); 
					       						some_condition.notify_all();

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











# Linux Memory Management

# RTOS


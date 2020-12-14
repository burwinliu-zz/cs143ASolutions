 Answers may be found here: https://www.ics.uci.edu/~aburtsev/143A/2017winter/final-answ/paper.pdf

# Fall 2018
# Question 1
> Topic: File system

Xv6 lays out the file system on disk as follows:

![Table Image](../img/W2017Final_Aqs.png)


Block 1 contains the super block. Blocks 2 through 31 contain the log header and the log.
Blocks 32 through 57 contain inodes. Block 58 contains the bitmap of free blocks. Blocks 59
through the end of the disk contain data blocks.
Ben modifies the function bwrite in bio.c to print the block number of each block written.
Ben boots xv6 with a fresh fs.img and types in the command ln cat cat2. This command creates
a symbolic link cat2 to file cat. This command produces the following trace:
<pre>
$ ln cat cat2
write 3
write 4
write 2
write 32
write 59
write 2
$
</pre>
## Part A
### Problem
(a) (5 points) Why is block 2 written twice?
### Solution
The Logging System in xv6 is setup in a manner to allow recovery from failure. So therefore, on writes to logs, one does not simply input their information into the logging section. So, follow the commit message -- 
  
First, write_log will alter the log by getting info to the log. (write 3, 4)  
Next, it writes the head with write_head, the true commit and the modification of block 2 as it is the header. (write 2)  
Then, it writes to disk with install_trans() into correct data locations (write 32, 59)  
Finally, write_head again to erase the transactions and writing to block 2 once again. (write 2)  

## Part B
### Problem
(b) (5 points) Briefly explain what block 32 contains in the above trace. Why is it written?

### Solution
It was written to as block 32 is for inodes. Because we create a symbolic link to this inode, we must alter and increment the inode reference count on disk

## Part C
### Problem
(c) (5 points) What does block 3 contain?

### Solution
Block 3 contains the log blocks which act as a fail safe and interim place to write blocks before they are commited to memory, which holds the copy of block 32 at this time

## Part D
### Problem
(d) (5 points) If writes to 32 and 59 are reordered like below, will it violate correctness of the
file system, explain why?

<pre>
$ ln cat cat2
write 3
write 4
write 2
write 59
write 32
write 2
$
</pre>
### Solution
No it would not. Due to the nature of this logging mechanism, either the entire function goes through once we get to this section (both write 59 and 32) or none go through -- since write 2 has succeeded once we reached this stage, we know that these two writes are atmoic for all intents and purposes, and therefore, reordering it will not violage the correctness.

# Question 2
> Topic: Memory Management

## Part A
### Problem
(a) (5 points) Explain organization of the xv6 memory allocator.
### Solution
In xv6, the memory allocator is a pointer in kernel to a linked list of "free" pages. Therefore, whenver memory is requested, if there is a need for new pages, we retrieve from the head here. The memory is then linked from some virtual address and the newly passed page

##  Part B
(5 points) Why do you think xv6 does not have buddy or slab allocators? Under what
conditions you would have to add these allocators to the xv6 kernel?
### Solution

Xv6 doesn’t implement buddy and slab allocators, since it doesn’t really allocate any
memory for kernel data structures. I.e., all data structures used by the kernel are statically
preallocated as arrays with max number of elemements, e.g.,
struct proc proc[NPROC];
Xv6 allocates only pages of memory for user-level processes, kernel stacks, and page tables. If xv6 allocated kernel data structures dynamically, it would require a hierarchy of
allocators, malloc on top of slab, on top of buddy to serve allocations of variable size.

# Question 3

> Topic: Synchronization
## Part A
### Problem
(10 points) The code below is the xv6’s sleep() function. Remember the whole idea of
passing a lock inside sleep() is to make sure it is released before the process goes to
sleep (otherwise it will never be woken up). However, it looks like that if the lock passed
inside sleep is ptable.lock (i.e., lk == &ptable.lock) the lock remains acquired and is never
released. But xv6 does call sleep with ptable.lock as an argument and it works, can you
explain why?
<pre>
2806 // Atomically release lock and sleep on chan.
2807 // Reacquires lock when awakened.
2808 void
2809 sleep(void *chan, struct spinlock *lk)
2810 {
2811    if(proc == 0)
2812    panic("sleep");
2813
2814    if(lk == 0)
2815        panic("sleep without lk");
2816
2817    // Must acquire ptable.lock in order to 2818 // change p>state and
2818    // change p>state and then call sched.
2819    // Once we hold ptable.lock, we can be
2820    // guaranteed that we wont miss any wakeup
2821    // (wakeup runs with ptable.lock locked),
2822    // so its okay to release lk.
2823    if(lk != &ptable.lock){
2824        acquire(&ptable.lock);
2825        release(lk);
2826    }
2827
2828    // Go to sleep.
2829    proc>chan = chan;
2830    proc>state = SLEEPING;
2831    sched();
2832
2833    // Tidy up.
2834    proc>chan = 0;
2835
2836    // Reacquire original lock.
2837    if(lk != &ptable.lock){
2838        release(&ptable.lock);
2839        acquire(lk);
2840    }
2841 }

</pre>

### Solution

This works without issue as a pre-req for any sched call is the held ptable.lock, so that it knows no other process will interfere, and therefore, will switch, then free and setup another process. So therefore, if the ptable.lock is passed, theres no aquires or releases due to 2823 and 2837s conditions, and therefore, will just swap process without issue, and behave as if it was passed any lk in the eyes of sched


## Part B
### Problem
(b) (10 points) Alyssa runs xv6 on a machine with 8 processors and 8 processes. Each process
calls uptime() (3738) system call continuously, reading the number of ticks passed since
boot. Alyssa measures the number of uptime() system calls per second and notices that 8
processes achieve the same total throughput as 1 process, even though each process runs
on a different processor. Why is the throughput of 8 processes the same as that of 1
process?

### Solution



Uptime() acquires the tickslock lock. Hence, all 8 processes are serialized, i.e., one is
reading the ticks variable, and 7 others are waiting for the lock. Of course, reading ticks
counter takes only 1 cycle, and Alyssa will see some speed up, but not a lot. The process
that wins the lock, reads the counter, exits to user-space and returns back to the kernel
(around 400 cycles) to wait on the lock again (on average waiting for 7*time-to-acquireand-release-the-lock cycles, which depending on the architecture can take 240-620 cycles).

(My understanding: Due to the use of a cross process coordination and sleep locks, regardless of the number of processors, the number of ticks are uniformally regulated, and so therefore, the throughput is still held as if it were running on a single processor due to the serialized execution of the process)

# Question 4

> Topic: Scheduling
## Part A
### Problem
(10 points) You would like to extend xv6 with priority based scheduler, i.e., each process
has a priority, and processes with a higher priority are scheduled first. Write the code for
your implementation below (which xv6 functions need to be changed?)
Implement the Linux O(1) scheduler.

### Solution
<pre>
struct proc* sched_next() {
    int i;
    do {
        // Loop through the priority array starting from the
        // highest priority
        for (i = MAX_PRIORITY - 1; i--; i >= 0) {
            if(current->prio[i]->head ! = 0) {
                p = current->prio->[i]->head;
                current[i]->head = current->prio[i]->head->next;
                break;
            }
        }
        if(p != 0)
            return p;
        // No processes in the current queue
        // flip current and expried queues
        tmp = current;
        current = expired;
        expired = tmp;
        // we believe there is always one non-sleeping process
        // now we’ll definitely find something
    } while (1);
    return 0;
}
// Put the process in expired queue
void sched_put(struct proc *p) {
    p->next = expired->prio[p->prio]->head;
    expired->prio[p->prio]->head = p;
    return;
}
scheduler(void)
{
    struct proc *p;
    for(;;) {
        sti();
        Principles of Operating Systems Final - Page 8 of 12 03/22/2017
        acquire(&ptable.lock);
        // get the next process to run
        p = sched_next();
        proc = p;
        switchuvm(p);
        p->state = RUNNING;
        swtch(&cpu->scheduler, p->context);
        switchkvm();
        proc = 0;
        // put the process back
        sched_put(p);
        release(&ptable.lock);
    }
}

</pre>

## Part B
### Problem
 (10 points) Now you would like to extend your priority scheduler with support for interactive tasks, e.g., a task that spends a lot of time waiting, should run first (i.e., receive
priority boost). Provide code that handles waiting tasks and implements priority boost
(again, just change related xv6 functions).

<pre>
// Put the process in expired queue but now with a boost
void sched_put(struct proc *p) {
    p->next = expired->prio[p->prio + p->boost]->head;
    expired->prio[p->prio + p->boost]->head = p;
    return;
}
// Naive boost function
int boost(int wait_time){
    if (wait_time > 10)
        return 5;
        return 0;
    }
wakeup1(void *chan)
{
    struct proc *p;
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++)
        if(p->state == SLEEPING && p->chan == chan) {
            p->state = RUNNABLE;
        // compute the boost based on how long the process was waiting
        p->boost = boost(sys_uptime() - p->start_wait_ticks);
        sched_put(p);
    }
}
// Modify sleep to account for wait time
void
sleep(void *chan, struct spinlock *lk) {
    ...
    proc->state = SLEEPING;
    proc->start_wait_ticks = sys_uptime();
    sched();
    ...
}
</pre>

# Question 4
> Topic: Page Tables

Xv6 uses 4MB page table during boot. It is defined as:
<pre>
1406 // The boot page table used in entry.S and entryother.S.
1407 // Page directories (and page tables) must start on page boundaries,
1408 // hence the __aligned__ attribute.
1409 // PTE_PS in a page directory entry enables 4Mbyte pages.
1410
1411 __attribute__((__aligned__(PGSIZE)))
1412 pde_t entrypgdir[NPDENTRIES] = {
1413    // Map VAs [0, 4MB) to PAs [0, 4MB)
1414    [0] = (0) | PTE_P | PTE_W | PTE_PS,
1415    // Map VAs [KERNBASE, KERNBASE+4MB) to PAs [0, 4MB)
1416    [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
1417 };
</pre>
## Part A
### Problem
(5 points) What virtual addresses (and to what physical addresses) does this page table
map?
### Solution
// Map VAs [0, 4MB) to PAs [0, 4MB)  
// Map VAs [2GBs, 2GBs+4MB) to PAs [0, 4MB)  

## Part B
### Problem
 (10 points) Imagine now that 4MB pages are not available, and you have to use regular
4KB pages. How do you need to change the definition of entrypgdir for xv6 to work
correctly (provide code and short explanation).

### Solution
<pre>
#define NPDENTRIES 1024

__attribute__((__aligned__(PGSIZE)))
pde_t entrypgdir[NPDENTRIES] = {
   // Map VAs [0, 4MB) to PAs [0, 4MB)
   [0] = V2P(entrypgtable) | PTE_P | PTE_W | PTE_PS,
   // Map VAs [KERNBASE, KERNBASE+4MB) to PAs [0, 4MB)
   [KERNBASE>>PDXSHIFT] = V2P(entrypgtable) | PTE_P | PTE_W | PTE_PS,
};

pde_t entrypgtable[NPDENTRIES] = {
    0x000000 | PTE_P | PTE_W,
    0x000001 | PTE_P | PTE_W,
    ... (for all values of NPDENTRIES)
    0x3ff000 | PTE_P | PTE_W
}
</pre>
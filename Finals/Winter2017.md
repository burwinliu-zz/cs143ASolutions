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
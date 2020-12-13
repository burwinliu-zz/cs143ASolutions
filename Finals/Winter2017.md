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
It was written to as it block 32 as a new inode for the new file, which cat2 will now correspond to. It is written to properly initiate the new file.

## Part C
### Problem
(c) (5 points) What does block 3 contain?

### Solution
Block 3 contains the log blocks which act as a fail safe and interim place to write blocks before they are commited to memory

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

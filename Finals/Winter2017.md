# Fall 2018
# Question 1
> Topic: File system
## Part A
### Problem
Xv6 lays out the file system on disk as follows:

![Table Image](../W2017Final_Aqs.png)


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

(a) (5 points) Why is block 2 written twice?
The Logging System in xv6 is setup in a manner to allow recovery from failure. So therefore, on writes to logs, one does not simply input their information into the logging section. So, follow the commit message -- 
<pre>
First, write_log will alter the log by getting info to the log. (write 3, 4)
Next, it writes the head with write_head, the true commit and the modification of block 2 as it is the header. (write 2)
Then, it writes to disk with install_trans() into correct data locations (write 32, 59)
Finally, write_head again to erase the transactions and writing to block 2 once again. Therefore, block 2 was written to twice. (write 2)






Topic : [[Operating Systems]], [[Book : OS from Scratch]]


# The Boot Process

When a Computer is powered on, there is no operating system. The first program to execute during this *pre-OS* phase is called BIOS - Basic Input Output System

BIOS is a very simple software with not many functionality. Even a File System is a luxury
BIOS can only handle simple input, output function.


## BIOS and Boot Block 
One of the primary functions of BIOS is to load the actual Boot Block.
However BIOS has no functionality like reading or writing from a files.
So to load the BIOS we have to load the content of the BIOS to the CPU. This can be done by loading the contents directly from the disk. The actual physical hard disk is generally partitioned into block and sectors. The content of the blocks can be either data or actual boot block, to differentiate/identify the boot block, the boot block is ended with the **hex values** `0xaa55`

##### **How a Boot Block Looks Like :**
![[Pasted image 20241030185807.png]]

**Note here the ending of the boot block - "55 aa"** - This is called little endian notation. This differs between processor. Little Endian is used by x86 processor
```
Endian Notation - When reading multi-byte values. Which byte is at FIRST position and SECOND position is denoted by Endian Notation. 

Little Endian Notation - less significant bytes proceed more significant bytes.
```

Initial three bytes are defined to perform endless jumps this is so that the CPU doesn't execute some data or programs like erasing the hard drives memory.

*Word - maximum processing size of a processor. For a 16bit CPU , word = 16bits*

Topic : [[Operating Systems]], [[Book : OS from Scratch]], [[Boot Sector]], [[Assembly]]

## How to run our "Operating System" ?
**Using `Qemu` - CPU Simulation Program**

## How to convert Assembly to Binary Files (Hexadecimal) ?
**Using `nasm`**
Example;
```
$nasm boot_sect.asm -f bin -o boot_sect.bin
```
- `-f bin` : Output raw Machine code without any "linking" or "routing"
- `-o` : Output Path file name


# 1. Simple Boot Sector Program
---
```x86asm
;
; Simple Boot Sector Program that Loads Forever
;

loop:
    jmp loop    ; Unconditional Infinite Jump

times 510-($-$$) db 0

; times - Run an instrunction 'n' times
; $ - current address of this intruction
; $$ - Starting address of the "code" block
; db 0 - `define byte` 0 , will spam 0 at the address

; Suppose `current address is 5016`
;         `Starting Block addr is 5000`
;

; $-$$ = 5016 - 5000 = 16

; We know we have to reach at the second last position to place our boot sector ending instruction/"magic_number"
; so we Calculate How many times we have spam 0's
; **Note**: Boot sector has to be 512bytes
; so 510 - ($ - $$) = 510 - (5016 - 5000) = 510 - 16 = 494

dw 0xaa55 ; place 0xaa55H a 16bit/2Byte instruction
```

# 2. 16 Bit Real Mode
---
**For the sake of backwards compatibility CPUs run in 16bit mode, this means that even 16bit OS can work like normal in a 32bit or 64bit CPU.**
**So when running 32bit or 64bit OS, the OS has to manually specify the CPU to switch to a 32bit or 64bit processing.**



# 3. Writing the First "Hello World !!"
---
To print something onto a screen we will have to interact with the display.
Each display will have its own way of communicating and so on.
All this can get very complicated at this pre-os stage just perform some simple tasks.
So to make our lives much simpler and better we can use the BIOS.

When we start our computer, the BIOS already interacts with the hardware devices to check if all is working as intended. In doing this, in our case it also prints a simple "Everything is working message". 

So for our use case, we can perform our printing of - "Hello World !!" by either
1. Accessing and Interacting with the BIOS code memory
2. Tell BIOS to print this message for us

Option 1, will get very complex and messy and can even cause some unintended errors and crash.

So our best case can be by instructing BIOS to print our message.
This can be done via:

## Interrupts
---
#### What are Interrupts ?
**Interrupts is a mechanism baked into the CPU itself that will inform the CPU to halt whatever its doing and perform the provided instruction. This interrupts have an interrupt level that signifies the interrupts urgency. CPU can increase the interrupt level, meaning it will only consider interrupts above a certain importance level, this is useful when CPU is performing some critical task.**

#### Types of interrupts:
1. Hardware Interrupts:
	- Network I/O
2. Software Interrupts
	- Print this message	

**Each interrupt is represented by a unique number that is an index to the interrupt vector table. This table is initially set upped by the BIOS at the start of the memory (Physically - 0x0) that contains an index pointing to a *interrupt service routines* (ISRs).**
**ISRs are a set of instructions to be executed when the following interrupt is called.**
**The interrupt is formed using the higher nibble of `ax` register with `int`.**
1. `int` -> Type of interrupt 
	- Eg: 
	1. `0x10 - screen related ISR`
	2. `0x13 - Disk Related ISR`
2. `ah` (higher nibble of register a) -> ?"Specific Operation"
### How to call an interrupt
```x86-asm
mov ah , 0x0e ; int 10/ ah = 0eh -> scrolling teletype BIOS routine 
mov al , ’H’ 
int 0x10
```

This calls an interrupt to print the character "H" on the screen.


## Printing the actual "Hello World !!"
---
**How to Print:**
```x86-asm
mov ah, 0x0e ; Set the interrupt to call Scrolling Teletype BIOS Routine
mov al, 'H'  ; Set al as H, as the Routine Prints the charac from al 
int 0x10     ; Call the Interrupt - of Display Type
```


Final "Hello World!!" Code:
```x86-asm
;
; Resources
;          - OS From Scratch by by Nick Blundell [Book]
;

;
; Simple Boot Sector Program that Loads Forever and Prints Hello World!!
;

mov ah, 0x0e ; Set the interrupt to call Scrolling Teletype BIOS Routine
mov al, 'H'  ; Set al as H, as the Routine Prints the charac from al 
int 0x10     ; Call the Interrupt - of Display Type
mov al, 'E'
int 0x10
mov al, 'L'
int 0x10
mov al, 'L'
int 0x10
mov al, 'O'
int 0x10

mov al, ' '
int 0x10

mov al, 'W'
int 0x10
mov al, 'O'
int 0x10
mov al, 'R'
int 0x10
mov al, 'L'
int 0x10
mov al, 'D'
int 0x10

mov al, '!'
int 0x10
mov al, '!'
int 0x10

loop:
    jmp loop    ; Unconditional Infinite Jump

times 510-($-$$) db 0

dw 0xaa55 ; place 0xaa55H a 16bit/2Byte instruction
```

Output:
![[Pasted image 20241031171045.png]]
![[Pasted image 20241031172300.png]]

**The Unconditional Infinite Jump Loop:**
```x86-asm
loop:
    jmp loop    ; Unconditional Infinite Jump
```

**Can also be written as:**
```x86-asm
jmp $ ; Jump at current Address -> Resulting to infinite loop
```


# 4. A more Advanced "Hello World !?"
---
### Where is our boot-sector stored ?
**The first ever code to execute is our BIOS. Once the BIOS has initialized its routines which include verifying all hardware devices, initializing the ISR tables etc. it is then responsible to execute(more like point the CPU) the boot-sector.**
**Now the question arises that where is the boot-sector stored at? This question is necessary to answer because if we point the CPU at the wrong place in the memory, than it could execute/overwrite our existing data. Even a simple assumption like 0x0, will be problematic as we learned that the BIOS stores/initializes values like the ISR table. As it turns out the boot sector is stored at a specific address:** 
- `0x7c00` 
**here the BIOS knows that we will not be overwriting any crucial memory.**

**This is how the memory is typically laid out:**
![[Pasted image 20241031175711.png]]



### Learning about Labels and Offset
**Labels are a convenient way for us *programmers* to point to a memory location. This memory location can than be called using the same label name.**
Example:
```x86-asm
mov ah,0x0e
mov al, data_label
int 0x10

data_label;
	db 'X'
```
Here we are "trying" to store the value X in the Lower Nibble of register a.

#### Wait Why didn't it Print 'X' ?
**The reason why the above code block not print X, is because we are storing the address of the label rather than the content itself into reg al.**
**To store the content of the address rather than the address itself is by using *Direct Addressing*.**

**Direct Addressing:** When using direct addressing the CPU will store the contents of the memory at the provided address.
**How to (in x86):**
```
mov ax, [1234H]
```
**Using [ ] Brackets will inform the CPU to store the value, rather than the address.**

**When we use Labels, these labels are then converted into offset address by assembler, so by doing this we can print 'X'**:
```x86-asm
mov ah,0x0e
mov al, [data_label]
int 0x10

data_label;
	db 'X'
```
[Note : In the Textbook this is wrongly referred to as indirect addressing]

**Than What is Indirect Addressing ?:**
Indirect Addressing is the same as Direct Addressing, with the difference being that the address of the memory is stored inside of a register
```
mov bx, 0x1234
mov ax, [bx]
; ax = 0x5000H

; Assume In Memory
1234 = 0x5000H
```

### Who will 'Print X' ?
In the following code example who will print X ?
```x86-asm
; 
; A simple boot sector program that demonstrates addressing. 
; 
mov ah , 0 x0e ; int 10/ah = 0eh -> scrolling teletype BIOS routine 

; First attempt 
mov al , the_secret 
int 0x10 ; Does this print an X? 

; Second attempt 
mov al , [the_secret] 
int 0x10 ; Does this print an X? 

; Third attempt 
mov bx , the_secret 
add bx , 0 x7c00 
mov al , [bx] int 0 x10 ; Does this print an X? 

; Fourth attempt
mov al , [0x7c1e] 
int 0 x10 ; Does this print an X? 

jmp $ ; Jump forever. 

the_secret :
	db "X " 

; Padding and magic BIOS number. 
times 510 - ($ - $$) db 0 
dw 0xaa55
```

**In this code, only the 3rd and the 4th Attempt print X. Why?**
- **1st Attempt:** Stores, and tries to print the address offset of the label `the_secret`.
- **2nd Attempt:** Labels are converted into address offset by the assemblers. However the offset is calculated from 0. But as we saw previously our boot block is stored from 0x7c00. So what we are printing is some value stored in ISR(they are stored from 0x0)
- **3rd Attempt**: We consider that the assembler calculates the offset from 0x0, so we add the extra `0x7c00`
- **4th Attempt:** We are directly accessing the address of the X (this is not always feasible as we don't always know the hardcoded address of our "content")

### So will we have to manually add the offset every time?
**No, we can use `[org addre]` at the starting of the code. To specify the starting address of our code. The offsets will be calculated accordingly.
Example:**
```x86-asm
[org 0x7c00]
mov ah, 0x0e
mov al, [the_secret]
int 0x10

the_secret:
	db 'X'
times 510 - ($ - $$) db 0
dw 0xaa55
```


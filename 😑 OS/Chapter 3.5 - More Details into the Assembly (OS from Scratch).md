Topic : [[Operating Systems]], [[Book : OS from Scratch]], [[Boot Sector]], [[Assembly]]

***This is a continuation of chapter 3, from the OS from Scratch Book, [Page- 20]**


### How to define Strings ?
Up until know we have been declaring strings like this:
```x86-asm
my_string:
	db 'Booting OS'
```
However the problem arises when reading the string, it is impossible for us to know when the string is over.
So we generally attach a *null-terminating* - 0 at the end, for us to know when reading that the string is over
```
my_string:
	db 'Booting OS', 0
```


# Using Stack
---
**Stack is a way to store temporary data, the stack can store only 16byte during 16bit real mode.
Stack has 2 operations - `PUSH` and `POP`. Stack is maintained by 2 Special Registers:**
1. bp - Base pointer. Start of the stack
2. sp - Stack Pointer. Last stored values index
The stack grows *downwards*, so the value of bp is kept far away from any crucial memory.
[Stack is stored from 0x7FFF0000 to 0x80000000. According to ChatGPT not fact checked]
**Syntax:**
```x86-asm
push data_here
pop register_name
```

# How to JUMP 🐇
---
1. `CMP RegName Value` - Compare the value with the reg. Use `JE`, `JC` ...
### - >Jumps
![[Pasted image 20241101181129.png]]
Use these jumps after the `CMP` instruction
### Calling Functions Using JMPs
![[Pasted image 20241101181328.png]]
![[Pasted image 20241101181338.png]]
Return back by using labels and jump to them.

### Calling Functions Using Calls and RET
In x86 the current executing instruction is pointed by **Instruction Pointer**(ip). However in x86 we are not allowed directly interact with the ip.
**Call & Ret**: Are an instruction set that allows us to return to the point where the function was called. By pushing the callers address to stack. RETurn than pops the address back to ip.
![[Pasted image 20241205063712.png]]


### Reserve Current Register States
When a function is called, it might contain instructions and implicitly modifies the values of registers. This creates unexpected outcomes. To prevent this, it is "polite" to push all the register values to the stack. The CPU fortunately provides an instruction to do that.
- **PUSHA, POPA** (Push/Pop All Registers)
![[Pasted image 20241205064119.png]]


# Please INCLUDE me
---
It is redundant to rewrite the same code... So NASM provides allows you to **Include** external files using;
```asm
%include "external_file.nasm"
```
This will replace the contents of the code from the external file to the calling file.
Now the code can be used like normal.
Eg.
![[Pasted image 20241205064522.png]]



# Create a Better Hello World
---
Use the above call, ret and include to convert the printing into a separate function.

1. **boot_sector.asm**:
```x86-asm
; Better Hello World v1

[org 0x7c00] ; Set Code address from 0x7c00, so labels address are calculated from here

mov bx, HELLO_WORLD
call print_string

mov bx, LESSGOO
call print_string

loop:
    jmp loop    ; Unconditional Infinite Jump

%include "print/print_string.asm"

; Data
HELLO_WORLD:
    db 'HELLO WORLD!! ', 0
    
LESSGOO:
    db 'LESSGOO', 0

times 510-($-$$) db 0
```

2. print/print_string.asm
```x86-asm
; Print Chars pointed by the bx register
; Print Until string not null terminated ~ `0`

; STEPS:
;   1. Save all registers values into the STACK
;   2. Set Higher Byte of ax reg to 0x0e [Set Print type (BIOS Teletype Output)]
;   3. Check if Value of BX is 0 by placing the value in al and cmp it
;       4. If so Restore the register values and return back to caller
;   5. else call the interrupt for print
;   6. Increament the BX pointer
;   7. Jump to 3.
  
print_string:
    pusha           ; Save all reg vals to stack
    mov ah, 0x0e    ; Set Print type
print_loop:
	; Check if (value of bx) = 0
    mov al, [bx]    ; Indirect addressing. Value of register pointed by bx reg stored in al
    cmp al, 0      
    je print_done
    int 0x10        ; Call interrupt [print msg]
    inc bx          ; Inc to next char
    jmp print_loop
    
print_done:
    popa            ; Get back reg vals
    ret             ; return to caller
```

# Printing HEX
Printing hex required extra formatting of the data.
The objective was to print the hex value of a register.
Suppose register dx has value 0xAB19, our job is to create a subroutine to print it.

```x86-asm
MOV dx, 0xAB19
call print_hex
	...
```
Expected Output:
```
ab19
```

To print, we will be using interrupt just like above, now to print anything using the INT 0x10 we need to have the data in ASCII Value.

So our job is to modify the raw hex to ascii.
Below is a table containing the hex values of our decimal digits and their binary codes.
![[Pasted image 20241206211454.png]]

As we can see that 0 - 9 is 30 - 39 and a - f is 61 - 66 in ascii
So for 0 - 9 we simply need to add/unmask 0x30H using the OR operation
```x86-asm
or bx, 0x0030       ; Add 3H for ascii num(0d - 9d: 30H-39H)
```

For a - f, However we need to modify to 1 - 6.
Here we can use a concept from the HEX to BCD converter.
To Convert any HEX number to BCD we Added the HEX Value with 6.
So If Suppose we ADD 6 with A, we get 10
![[Pasted image 20241206212257.png]]

Same with
	B + 6 = 11
	C + 6 = 12
	D + 6 = 13

With this concept we can ADD 7, for the offset of 1, as A is 61 and Mask the extra Carry
And add/unmask 0x60 using OR
```x86-asm
add bx, 0x0007      ; Add 6 to convert number to BCD + 1 offset as A = 61
and bx, 0x000F      ; Mask Any Carry Generated
or bx, 0x0060       ; Add 6H for ascii num(Ah - Fd: 61H-66H)
```

Now that the conversion part is completed we need to focus on the shifting and looping the 16bit value to access print, the HEX Value first and the Last one at last.
Some Key Instruction to Note of:
1. SHR -> shifts the content of the register(8/16/32) to **RIGHT** by `n` number of bits specified in the cl(lower order of the "Counter" register)
	- **Operands**: shr cl, r/m(8 | 16 | 32)
	- **Example**: 
```x86-asm
mov cl, 12          ; Shift bx by 12 digits. bx = 0x2000 -> 0x0002
shr bx, cl
```


2. SHL -> shifts the content of the register(8/16/32) to **LEFT** by `n` number of bits specified in the cl(lower order of the "Counter" register) or Immediate 8-bit Data
	- **Operands**: shl cl/imm8, r/m(8 | 16 | 32)
	- **Example**: 
```x86-asm
mov cl, 12          ; Shift bx by 12 digits. bx = 0x0002 -> 0x2000
shl bx, cl
```

3. DEC -> Decrement any register by 1
	- **Operands**: dec r/m(8 | 16 | 32)
	- **Example**:
```x86-asm
mov ch, 0x04        ; set counter to 4 
dec ch              ; ch = 3
```

**Final Print Hex Code:**
print/print_hex.asm:
```x86-asm
print_hex:
    pusha               ; Save Register Values
    mov ah, 0x0e        ; Set to Print type
    mov ch, 0x04        ; set counter to 4 for loop
print_hex_loop:  
; Access HEX by masking
    mov bx, dx          ; Preserve Value of dx
    and bx, 0xF000      ; Mask All Except First Number (1111 0000 0000 000)

; Check if Number >= A
    cmp bx, 0xA000      ; Check if number >= A
    jnc format_for_alpha  

; Modify to Set HEX Format 3_ where _ is the number
    mov cl, 12          ; Shift bx by 12 digits. bx = 0x2000 -> 0x0002
    shr bx, cl
    or bx, 0x0030       ; Add 3H for ascii num(0d - 9d: 30H-39H)
    jmp print_hex_val
 
format_for_alpha:
    mov cl, 12          ; Shift bx by 13 digits. bx = 0x2000 -> 0x0002
    shr bx, cl
    add bx, 0x0007      ; Add 6 to convert number to BCD + 1 offset as A = 61
    and bx, 0x000F      ; Mask Any Carry Generated
    or bx, 0x0060       ; Add 6H for ascii num(Ah - Fd: 61H-66H)
  
print_hex_val:
    mov al, bl          ; Move data to al
    int 0x10            ; Set int to print
    mov cl, 4           ; Set CL to 4 (Shift Register bx by 4 bits to left)  
                        ; This allows access of each number

; Update Loop Counter and Shift Values
    mov cl, 4
    shl dx, cl
    dec ch
    cmp ch, 0
    jne print_hex_loop          

; Return Back Once Loop Done
    popa
    ret
```


# 3.6 Reading 
---
## 3.6.1 Reading from Memory
Our typical register size is of 16bits, this allows us to access a total memory size of 2^16 = 64KB.
This is not a lot, to expand our memory access range we use segment registers.
### Segment Registers
Segment Registers are Special Registers that add an additional offset to your registers.
So our actual absolute address = (segment_address * 16) + register
Using this, we extend our memory to 1MB (0xffff * 16 + 0xffff)
**Note: Any HEX * 16 = left shift. (E.g. 0x42 * 16 =  0x420)**
There are a total of 4 Segment Registers:
1. **CS: Code Segment** register is a register used calculating absolute code address using the instruction pointer(IP). Code Segment Register cannot be directly modified. Instructions that do modify are ljmp, lcals, lref etc...
2. **DS: Data Segment** register is used implicitly when using immediate address mode(including labels). In instructions like `mov bx, [0x0919]` the absolute address is calculated using the DS register. You can directly modify the ds register
3. **SS: Stack Segment** register, similar to DS but for stack
4. **ES: General Purpose** Segment. Needs to be specified in the instruction when used. E.g. `mov al, [es:0x0919]`

**Note: You cannot using immediate data to change the value of ds, you will have to use register mode.**
**E.g.**
```x86-asm
mov bx , 0 x7c0 ; Can ’t set ds directly , so set bx 
mov ds , bx     ; then copy bx to ds. 
mov al , [ the_secret ]
```


## 3.6.2 Reading from the Disk Drives
So a Hard Disk looks like this:
![[Pasted image 20241207215048.png]]

The Hard disk is generally divide into 3 components:
1. Cylinder/Track : Describes the distance to move the head from outer edge.
2. Head: Head describes the platter number
3. Sector: Circular track that is divided into 512 bytes
This can be used similarly for floppy disks too, but I am not too sure.
### Using BIOS to Read the Disk
Just like with displaying characters, we can take the help of the BIOS by using interrupts.
This interrupt requires some key register initialization.
1. ah = 0x02 : Type of interrupt (BIOS read sector function)
2. al: Number of Sectors to read (base: 1)
3. ch: Select Cylinder Number (base: 0)
4. dh: Select Head (base: 0)
5. cl: Start Sector Number (base: 1??)
6. int = 0x13: BIOS Interrupt - BIOS Read

#### Code E.g. (From Text Book):
```x86-asm
; load sectors to ES:BX from drive DL
disk_load : 
	mov ah , 0x02 ; BIOS read sector function 
	mov al , dh   ; Read DH number of sectors 
	mov ch , 0x00 ; Select cylinder 0 
	mov dh , 0x00 ; Select head 0 
	mov dl , 0x80 ; Select Device [0x00:Floppy drive, 0x80:First hard disk, 0x81:Second hard disk, and so on.]
	mov cl , 0x02 ; Start reading from second sector ( i.e. ; after the boot sector ) 
	int 0x13     ; BIOS interrupt 
```
**Note: when boot sector is loaded the dl is set by the BIOS with the device selected**
#### Result
- The data is than stored at address ES:BX.
	**Note: ES is a segment register, so Absolute Address = ES * 16 + BX**
- If there is an error, the **Carry Flag(CF)** = 1
- AL = number of segments actually read
	**Note: if AL != < no of segments inputted > there is an error**
**NOTE:**
**If there is an error, the error code will be displayed in the `AH` Register**
![[Pasted image 20241208215343.png]]



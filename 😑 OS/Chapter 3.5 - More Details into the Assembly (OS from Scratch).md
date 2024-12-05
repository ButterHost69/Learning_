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
---
title: Hack The Box Bombs Landed CTF Writeup
author: balpars
pubDatetime: 2024-02-23T08:03:01.372Z
slug: htb-bombs-landed-ctf-writeup
featured: true
draft: false
tags:
  - ctf
  - reverse_engineering
ogImage: "@assets/images/ogImage_htb_ctf_1.png"
description: Writeup for the Hack The Box Bombs landed ctf
canonicalURL: https://medium.com/@_Alp/hack-the-box-bombs-landed-writeup-5f123888bf3f
---

# Hack The Box Bombs Landed Writeup

## _start:

![1_checksec.png](@assets/images/1_checksec.png)

Using checksec we see that PIE is not enabled, this will make it easier for us to use a disassembler with a debugger simultenously, since the address offsets will be the same. We have a 32-bit binary.

Global Offset Table is corrupted to which will make it harder to debug.

![Ekran Alıntısı.PNG](@assets/images/Ekran_Alnts.png)

Running the program, we don’t get any hints whatsoever. Ltrace doesn’t work, which indicates the file is statically linked, but we will see that this is not the case.

![Untitled](@assets/images/Untitled.png)

There are some interesting strings. We see ptrace, which is used by debuggers to attach to a process, we may need bypass any checks used for anti-debugging. For this binary I didn’t need to do anything.

## Reversing

![reversing.PNG](@assets/images/reversing.png)

1. We see that the program checks if it had less than 4 command arguments and comparing two variables that it had setup. First argument for main function in C is the argument count.
2. We first see a call to mmap, which returns a pointer to the starting address of the memory region it allocates. I renamed the variable to memory_address, suddenly the code becomes significantly more readable.
3. We then see memset to fill the buffer that mmap allocat with  `0xc3` which is the x86 assembly language opcode for the `RET` instruction (according to a certain AI 😃)
4. Following our memory_address variable, we see that some data is being written to address it points to.
5. Then we see a function call to our memory_address, which indicates that the code above unpacked a function to the memory region that our memory_address points to.

There’s nothing more we can do here, because we do not know what is in the memory region. We need to use a debugger to continue.


Before leaving Ghidra, we should also look at other functions, one function should get out attention. The strncmp function.

![strncp.PNG](@assets/images/strncp.png)

![bp.PNG](@assets/images/bp.png)

We see that this is far from the original strncmp function in C. There’s a while loop that looks interesting, like it’s building a string, probaby the password. We will note the offset for the while loop’s condition to inspect it closer in debugger.

We do not see any call to this function yet. Indicating that call to this function will be resolved dynamically if it’s going to be called. 


## Debugging

Some breakpoints worth putting initially are:

![2bp.PNG](@assets/images/2bp.png)

- first argument for the if statement.
- second argument for the same if statement to bypass it.
- the strncmp’s while loop. which might give us the password.

We might also need to put breakpoint and step-into the function call to our memory_address if the initial breakpoints aren’t enough to get the password.

![call.PNG](@assets/images/call.png)

![bpss.PNG](@assets/images/bpss.png)

Of course no function names.

![Ekran Alıntısı5.PNG](@assets/images/Ekran_Alnts5.png)

We need to set ZF to bypass first check which is JA. Similarly flipping the sign flag for the second check which is JG, is enough to bypass these checks. Some resources to learn more about gdb and conditonal jumps are below. 

[https://cheatography.com/superkojiman/cheat-sheets/gdb-pwndbg/](https://cheatography.com/superkojiman/cheat-sheets/gdb-pwndbg/)

[https://www.philadelphia.edu.jo/academics/qhamarsheh/uploads/Lecture 18 Conditional Jumps Instructions.pdf](https://www.philadelphia.edu.jo/academics/qhamarsheh/uploads/Lecture%2018%20Conditional%20Jumps%20Instructions.pdf)

The program asks for a password. After entering something we land on our third breakpoint as we guessed. Continuing and pressing enter repeatedly, we see that our password is being built step by step in the stack.

![final.PNG](@assets/images/final.png)

1. Our third breakpoint is hit.
2. A string is built in the stack.

Finally we test the password we found. We need to run the binary with more than 4 arguments to bypass first check. We could have left it at 4, since the first argument in C main function is the program’s name.

![finall.PNG](@assets/images/finall.png)

We win…

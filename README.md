# Wonderland CTF - Reverse Engineering Write-ups

Welcome to my repository containing solutions and write-ups for the **Wonderland CTF** reverse engineering challenges.
---
## The Core Architecture: The Stage Router

Instead of moving from one level to another linearly, the program uses a central routing function (`sub_401E00`). It takes the level number from the user and directs the execution using a **Jump Table (Switch-Case)** mechanism.

### How the Switch Works in Assembly

When you select a level, the binary runs the following assembly code to jump to the right function:

```assembly
mov     ecx, dword ptr [ebp+var_4]   ; Load the stage number (index)
jmp     ds:jpt_401E13[ecx*4]         ; Jump using the table
```

* **The Index Check**: The program first checks if the number is valid (under the maximum number of cases) using a cmp instruction.

* **The Pointer Calculation**: Every address in memory takes exactly 4 bytes. The code multiplies our input index by 4 (ecx*4) to find the exact offset in the jump table (jpt_401E13).

* **The Jump**: The jmp instruction takes the function address from that offset and jumps directly to the selected level.

I split the analysis into two separate files based on the primary reverse engineering methodology used:

[Static Analysis Write-up (Levels 2 & 3)](static_analysis_writeup.md)

[Dynamic Analysis Write-up (Levels 4 & 5)](dynamic_analysis_writeup.md)

# Wonderland CTF - Reverse Engineering Write-ups

Welcome to my repository containing solutions and write-ups for the **Wonderland CTF** reverse engineering challenges.
---
## The Core Architecture: The Global Game Loop

At the heart of the binary is the main execution engine located inside the `_main` function. **This function runs continuously in a loop** (`loc_401D26`) to keep the game alive, handle user input, and manage stage progression rules.

### State Management & The Level Unlocking Check

Inside the `_main` loop, before letting you access a level, the program runs a lock mechanism to make sure players can't just skip ahead to later levels without finishing the previous ones first:

```assembly
mov     eax, dword ptr [ebp+var_4]   ; Load the level number the user typed
cmp     eax, dword ptr [ebp+var_8]   ; Compare it with the highest unlocked level
jle     short loc_401D87             ; Go ahead if it's valid (less than or equal)
```

* **Tracking Progress:** At startup, the program calls sub_401080 to check your current save state and saves the highest level you cleared inside var_8.

* **The Input Check:** The program uses sub_401EB0 (acting like scanf) with a "%d" format to read your choice into var_4. Then, it immediately compares it using cmp eax, [ebp+var_8].

* **The Lock:** If you type a level number that is higher than what you actually unlocked, the check fails. The program branches to a block that prints "You haven't unlocked this level yet." using the printing function sub_401DC0.

### The Central Switch Router (sub_401E00)

If you pass the unlocking check, the program moves to block loc_401D87 and calls sub_401E00. This is the central router function that contains the Jump Table (Switch-Case) mechanism:

```assembly
mov     ecx, dword ptr [ebp+var_4]   ; Load the validated level number
jmp     ds:jpt_401E13[ecx*4]         ; Jump directly using the table offset
```

* **The Pointer Calculation**: Every address in memory takes exactly 4 bytes, so the code multiplies our input index by 4 (ecx*4) to find the exact offset in the jump table (jpt_401E13).

* **The Jump**: The jmp instruction takes the function address from that offset and jumps directly to the selected level.

I split the analysis into two separate files based on the primary reverse engineering methodology used:

[Static Analysis Write-up (Levels 2 & 3)](static_analysis_writeup.md)

[Dynamic Analysis Write-up (Levels 4 & 5)](dynamic_analysis_writeup.md)

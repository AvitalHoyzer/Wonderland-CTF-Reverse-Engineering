# Write-Up: Wonderland 
Level 4 
---

## 1. Introduction & Initial Analysis
After bypassing Level 3, the application updates its state and provides the following prompt:
> *"Wait... I have something on the tip of my tongue! (Enter the correct number)"*

The program stops and waits for a single user input and reads this input using the format specifier `"%du"`, which indicates that the application expects an unsigned decimal integer.

---

## 2. Assembly Code Analysis & The Memory Vulnerability
Inside the level processing block (`loc_4015B5`), the following assembly sequence handles the core logic before calling the string validation routine:


<img width="353" height="483" alt="image" src="https://github.com/user-attachments/assets/92c53264-13eb-445c-9c5f-a65c3c8256ce" />

### Step-by-Step Breakdown:
* **Dynamic Buffer Initialization:** The program manually builds the target string "Good luck!!" (Little Endian) inside a local stack buffer ([ebp+Str]) by pushing raw hexadecimal DWORD chunks into consecutive memory positions.

```assembly
mov     dword ptr [ebp+edx+Str], 63756C20h   ; " cul"
...
mov     dword ptr [ebp+eax+Str], 21216Bh     ; "k!!"
...
mov     dword ptr [ebp+edx+Str], 646F6F47h   ; "Good"
```

* **The Type Confusion Bug:** The big mistake here is how the program handles our input (Str1). The code takes the raw number we typed, moves it directly into the edx register, and pushes it straight into the stack as the first argument for strncmp.

* **The Pointer Requirement:** Because strncmp expects a memory address pointing to text, and NOT a regular number, our input must be a valid memory address that actually holds the words "Good luck!!". If we type a random number, the function tries to read from an invalid address and crashes.

## 3. The Anti-Cheat Trap
When we look at how "Good luck!!" is built dynamically inside the stack, we notice that we cannot simply type that dynamic stack address. The developer put an intentional anti-cheat check at loc_401618:

<img width="220" height="136" alt="image" src="https://github.com/user-attachments/assets/01ec222a-3ca0-4c88-a1b0-4a592749a3e6" />

### How the Security Check Works:
The line cmp [ebp+Str1], eax checks if the address we typed is exactly the same as the internal stack buffer address.

If the addresses match, the jnz (Jump if Not Zero) branch fails, and the program drops into a trap that says: "Cheater.. Try again another way."

## 4. The Bypass Solution 
To pass the check and beat the anti-cheat without modifying registers or changing flags, the solution requires finding another place in memory where the exact text `"Good luck!!"` is stored.

It can be noticed that when the level is cleared successfully, the program prints a specific victory message. Looking up this text inside the data section (`.data`) reveals its exact location:

<img width="791" height="199" alt="image" src="https://github.com/user-attachments/assets/2494d022-ccfa-4bab-8878-1acebdaa1860" />

<img width="1129" height="75" alt="image" src="https://github.com/user-attachments/assets/6162f021-3b2a-4656-a2b7-5c2a35650cc1" />

Even though this string starts with "Yeah! ", a small byte offset can be added to point the input directly to the start of the word "Good":

* **Base Address:** 00404738 (Points to the letter 'Y')

* **Byte Offset:** +6 bytes (Skipping 'Y', 'e', 'a', 'h', '!', and the space ' ')

* **Target Address:** 0040473E

By giving the program this address, strncmp reads `"Good luck!!"` from a safe, permanent memory section, doesn't crash, and returns 0 (Success). At the same time, because the address 0040473E is completely different from the dynamic stack address, the Anti-Cheat check sees that they are not equal, the jnz jump triggers perfectly, and the level is cleared.

## 5. Finding the Final Number & Testing
We convert our hex target address 40473E into a regular decimal number that we can type into the program prompt: `4212542`

Now, we just run the program normally (without any debugger) and type the number 4212542

The program accepts it, passes both checks, and unlocks Level 5!

<img width="1141" height="371" alt="image" src="https://github.com/user-attachments/assets/e3423f0a-9d3f-4583-8e66-cda8bf6b02af" />


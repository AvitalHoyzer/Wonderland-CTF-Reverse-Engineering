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


Level 5 
---

## 1. Introduction & File Input Discovery
After bypassing Level 4, the application updates its state and provides the following prompt:
> *"You may enter, but can you find the Queen's palace?..."*

When looking at the assembly code, the program does not ask for a regular number this time. Instead, it uses a WinAPI function called `CreateFileA` to read an external input.

### How the File Check Works:
* **The Input is a Filename:** The program takes the string typed in the console and uses it directly as a filename.
* **The File Must Exist:** The code sets the file opening setting to `OPEN_EXISTING` (`3`). This means the file must already exist on the computer. If the file is missing, the function fails, and the program prints `"Wrong!"` before asking for input again.
* **The Solution Setup:** To satisfy this initial check, I created a simple text file named `path.txt` manually inside the program folder. Typing `path.txt` into the prompt allowed the program to find the file successfully and move to the maze validation function.

---

## 2. The Internal Maze Logic (The Movement Code)
After the program successfully opens the file, it reads its content using `ReadFile` and sends the text to a function at `sub_401770`. 

This function acts exactly like a **Maze Game**. It reads the text file character by character. Every number from `'1'` to `'4'` updates the player position inside the memory array:

* **Input `'1'`:** Moves the player **Right** (Adds 1 to the position).
* **Input `'2'`:** Moves the player **Down** (Adds 16 to the position).
* **Input `'3'`:** Moves the player **Left** (Subtracts 1 from the position).
* **Input `'4'`:** Moves the player **Up** (Subtracts 16 from the position).

### Why does 16 represent Up and Down?
The row width is exactly **16 bytes** (`10h` in hex). 
* To move **Down** to the exact same column in the next row, the program must skip a whole row ahead in memory, which requires adding `16` to the current index (`add position, 10h`).
* To move **Up**, the program must go back exactly one full row in memory, which requires subtracting `16` from the current index (`sub position, 10h`).

* **The Starting Point:** Before reading our file, the program uses `strchr` to scan the map for the character **`O`**. It automatically places the player on this letter as the starting position (index 0).
* 
### The Boundary Rules:
Every time a movement character is processed, the program validates the new position against the map:
* If the player steps on a period character (`.`), the path is clean, and the loop keeps running.
* If the player steps on a hashtag character (`#`), a wall is hit. The program stops immediately and prints `"You are lost!"`.

* **The Win Condition:** After processing all the characters from the file, the program checks the final position. If the player is standing exactly on the character **`X`**, the level is cleared successfully.
  
---

## 3. Finding the Maze Map in Memory
To solve the maze, the actual layout had to be analyzed. The hardcoded map characters are stored inside the static data section (`.data`) at address `00404020`.

Inside the maze function, the code constantly checks the player's position against the map. By looking at the pointer named `Str` used during this validation step, I saw that it points directly to the global variable `aO` at address `00404020`. This confirmed exactly where the maze map is stored.

<img width="1156" height="176" alt="image" src="https://github.com/user-attachments/assets/5f8476ca-c69c-4cf3-8fb7-22e3123810db" />

Initially, the standard IDA text view showed the map as a messy, broken text string, making it impossible to see the path. To fix this, I switched to the Hex View. Since each row in the maze is exactly 16 bytes wide, the Hex View automatically aligned the characters into a perfect, readable grid.

<img width="822" height="147" alt="image" src="https://github.com/user-attachments/assets/503b82cc-5c61-4a95-9bae-7dfe86ace9e7" />

Now the entire layout was clear:

* The starting point O is located at address 00404033 (Row 404030, Column 3).

* The winning target X is located at address 0040406C (Row 404060, Column C).

---

## 4. Calculating the Winning Path
Using the Hex View grid map, the dots (.) were tracked from O to X step by step, completely avoiding all the walls (#):

Start at O, Move Down 3 times (222), Right 3 times (111), Up 2 times (44), Right 2 times (11) and Up 1 time (4), Right 2 times (11), Down 3 times (222), Right 3 times (111) to step directly onto the X marker!

Combining all these direction commands together produces the final exact sequence:
```
2221114411411222111
```
## 5. Testing & Success
The final path sequence `2221114411411222111` was written into the path.txt file and saved.

Running the program normally, choosing Level 5, and passing the path.txt file satisfied all conditions. The program completed the loop, verified that the final position matched the exit marker X, and successfully unlocked Level 6!

<img width="1123" height="316" alt="image" src="https://github.com/user-attachments/assets/7378f9d8-b1a2-479a-a0e9-2cac85396f49" />


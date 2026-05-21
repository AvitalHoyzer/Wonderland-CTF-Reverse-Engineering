# Write-Up: Wonderland 
Level 2 
---
## 1. Introduction & Initial Analysis
After passing the first level, the program prints the following message on the screen:
> *"You know what? That was too easy. \*Now\* tell me the second password."*

The application stops and waits for user input, which is saved inside a local memory buffer called `Buffer`.

---
## 2. Assembly Code Analysis & Reverse Engineering
Inside the main processing block (`loc_401380`), the following assembly sequence handles changing the input string:

```assembly
mov     eax, [ebp+var_4]
mov     ecx, dword ptr [ebp+eax+Buffer]
xor     ecx, 41524241h
mov     edx, [ebp+var_4]
mov     dword ptr [ebp+edx+Buffer], ecx
jmp     short loc_4013A5
```
### Step-by-Step Breakdown:

* **Index Management:**  The local variable `var_4` acts as the loop index counter (starting at 0) and is loaded into the `eax` register.

* **Reading Data in Chunks (DWORD):** The instruction `mov ecx, dword ptr [ebp+eax+Buffer]` reads 4 bytes at the same time (a DWORD) from the input buffer into `ecx`. This is different from Level 1, where the code processed only one byte at a time.

* **The XOR Operation:** The instruction `xor ecx, 41524241h` performs a bitwise XOR between the 4 bytes of the input and the hardcoded hexadecimal constant `0x41524241`.

* **Writing Back to Memory:** The code takes the modified result from `ecx` and writes it back to the exact same position in `Buffer`, overwriting the original input.

### Analyzing the Loop Jump (`loc_4013A5`):
Immediately following the mutation, control branches to the loop modifier block:
```
loc_4013A5:
mov     ecx, [ebp+var_4]
add     ecx, 4
mov     [ebp+var_4], ecx
```

The instruction `add ecx, 4` shows that the loop index steps up by 4 bytes every turn. This matches perfectly with the fact that the previous block processes 4 bytes (a DWORD) at a time.

## 3. Extracting the Key & Byte Order (Little Endian)
To write a script that recovers the password, the hex constant `0x41524241` needs to be mapped to its actual characters (ASCII). Because the x86 architecture saves data in memory using **Little-Endian** format, the bytes are read in reverse order (from end to start):

**Byte 1** (Lowest Address): `41h`→ The character 'A' in ASCII.

**Byte 2**: `42h`→ The character 'B' in ASCII.

**Byte 3**: `52h`→ The character 'R' in ASCII.

**Byte 4** (Highest Address): `41h` → The character 'A' in ASCII.

This means the encryption key applied to every 4-byte segment of the input is the string: `"ABRA"`.

## 4. The Validation Check & Target Ciphertext
After the loop finishes, the program uses the `strncmp` function to compare the encrypted `Buffer` against a hardcoded string embedded in the code:

```
"into the rabbit hole"
```

If the comparison returns 0 (indicating an identical match), the program routes directly to a success block, printing: `"Correct! you may enter.."`.

## 5. Automated Solution Script (Python)
Since the bitwise XOR cipher is its own mathematical inverse (
A⊕B=C⟹C⊕B=A
), running the target ciphertext `"into the rabbit hole"` through the exact same chunk-based XOR routine using the key `"ABRA"` turns the ciphertext back into the required password.

The following simple Python script automates this process:
```
def doXor():
    str = "into the rabbit hole"
    key = "ABRA"
    result = ""
    
    # Iterate through the ciphertext in strides of 4 bytes
    for i in range(0, len(str), 4):
        block = str[i:i+4]
        
        # XOR each character of the block against the parallel key character
        for j in range(len(block)):
            result += chr(ord(block[j]) ^ ord(key[j]))
            
    return result

if __name__ == "__main__":
    print(doXor())
```
### Script Output & Found Password:
Running the script generates the following input string:
```
(, &.a6:$a03##+&a)->$
```
<img width="864" height="376" alt="צילום מסך 2026-05-21 161626" src="https://github.com/user-attachments/assets/32f9f570-02a4-4ba7-be30-116fefb0bd3d" />

Entering this password bypasses the check and lets us enter Level 3!

<img width="1162" height="260" alt="צילום מסך 2026-05-21 161610" src="https://github.com/user-attachments/assets/c2167d52-2d52-4adf-867d-cc313f1afd41" />


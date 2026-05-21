# Write-Up: Wonderland 
Level 2 
---
## 1. Introduction & Initial Analysis
Upon passing the initial level, the execution flow triggers the following terminal challenge:
> *"You know what? That was too easy. \*Now\* tell me the second password."*

The binary halts to await user input, which is subsequently stored inside a local stack buffer designated as `Buffer`.

---
## 2. Assembly Code Analysis & Reverse Engineering
Within the core processing block (`loc_401380`), the following assembly sequence is responsible for mutating the input string:

```assembly
mov     eax, [ebp+var_4]
mov     ecx, dword ptr [ebp+eax+Buffer]
xor     ecx, 41524241h
mov     edx, [ebp+var_4]
mov     dword ptr [ebp+edx+Buffer], ecx
jmp     short loc_4013A5
```
### Deconstruction of the Routine:
* **Index Tracking:**  The local variable var_4 acts as the loop index offset (initialized to 0) and is loaded into the eax register.

* **DWORD-Sized Chunk Processing:** The instruction mov ecx, dword ptr [ebp+eax+Buffer] reads 4 bytes simultaneously (a double-word / DWORD) from the input buffer into ecx. This marks a structural escalation from Level 1, which processed input on a single-byte scale.

* **The XOR Cipher:** The instruction xor ecx, 41524241h executes a bitwise Exclusive-OR operation between the extracted 4-byte chunk and the hardcoded hexadecimal constant 0x41524241.

* **Memory Write-Back:** The mutated 4-byte block in ecx is written back into its original offset within Buffer, overriding the raw input dword-for-dword.

### Analyzing the Loop Stride (loc_4013A5):
Immediately following the mutation, control branches to the loop modifier block:
```
loc_4013A5:
mov     ecx, [ebp+var_4]
add     ecx, 4
mov     [ebp+var_4], ecx
```

The explicit instruction add ecx, 4 confirms that the loop index increments by 4 bytes per iteration. This structural alignment completely synchronizes with the 4-byte DWORD mutations examined in the prior block.

## 3. Cryptographic Key Extraction & Endianness Context
To build a functional decryption routine, the hex constant 0x41524241 must be mapped to its raw ASCII representation. Because the x86 architecture natively handles data layout using Little-Endian formatting, bytes are evaluated from the least significant byte (LSB) to the most significant byte (MSB):

**Byte 1** (Lowest Address): 41h 
→
 Maps to character 'A' in ASCII.

**Byte 2**: 42h 
→
 Maps to character 'B' in ASCII.

**Byte 3**: 52h 
→
 Maps to character 'R' in ASCII.

**Byte 4** (Highest Address): 41h 
→
 Maps to character 'A' in ASCII.

Consequently, the operational encryption key applied cyclically across every 4-byte segment of the input sequence translates to the static string: "ABRA".

## 4. Validation Routine & Target Ciphertext
After terminating the parsing loop, the routine utilizes strncmp to validate the processed Buffer against a hardcoded target anchor string embedded within the binary:

```
"into the rabbit hole"
```

If the comparison returns 0 (indicating an identical match), the program routes directly to a success block, printing: "Correct! you may enter..".

## 5. Automated Solution Script (Python)
Since the bitwise XOR cipher is its own mathematical inverse (
A⊕B=C⟹C⊕B=A
), running the target ciphertext "into the rabbit hole" through the exact same chunk-based XOR routine using the key "ABRA" inverts the ciphertext back into the required cleartext password.

The following Python script automates this reverse transformation:
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
### Execution Output & Verified Password:
Running the automated script produces the following required input string:

```
(, &.a6:$a03##+&a)->$
```
<img width="864" height="376" alt="צילום מסך 2026-05-21 161626" src="https://github.com/user-attachments/assets/32f9f570-02a4-4ba7-be30-116fefb0bd3d" />

Submitting this passphrase successfully bypasses the second verification check!

<img width="1162" height="260" alt="צילום מסך 2026-05-21 161610" src="https://github.com/user-attachments/assets/c2167d52-2d52-4adf-867d-cc313f1afd41" />


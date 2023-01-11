# SMarT1
## Crypto (442 points) by sera

> I made a small, efficient block cipher. It's small because I can fit it on the page and it's efficient because it only uses the minimal amount of rounds. I didn't even try it but I'm sure it works. Here's some output with the key. Can you get the flag back out?

## Attachments:

- [chal.py](assets/smart/chal.py)
- [output1.txt](assets/smart/output1.txt)

## Solution

Love the premise here of combining cryptography and reversing! The first part of this challenge mainly comprises reversing. Let's first take a look at `chal.py`:

```python
from pwn import xor

# I don't know how to make a good substitution box so I'll refer to AES. This way I'm not actually rolling my own crypto
SBOX = [99, 124, 119, 123, 242, 107, 111, 197, 48, 1, 103, 43, 254, 215, 171, 118, 202, 130, 201, 125, 250, 89, 71, 240, 173, 212, 162, 175, 156, 164, 114, 192, 183, 253, 147, 38, 54, 63, 247, 204, 52, 165, 229, 241, 113, 216, 49, 21, 4, 199, 35, 195, 24, 150, 5, 154, 7, 18, 128, 226, 235, 39, 178, 117, 9, 131, 44, 26, 27, 110, 90, 160, 82, 59, 214, 179, 41, 227, 47, 132, 83, 209, 0, 237, 32, 252, 177, 91, 106, 203, 190, 57, 74, 76, 88, 207, 208, 239, 170, 251, 67, 77, 51, 133, 69, 249, 2, 127, 80, 60, 159, 168, 81, 163, 64, 143, 146, 157, 56, 245, 188, 182, 218, 33, 16, 255, 243, 210, 205, 12, 19, 236, 95, 151, 68, 23, 196, 167, 126, 61, 100, 93, 25, 115, 96, 129, 79, 220, 34, 42, 144, 136, 70, 238, 184, 20, 222, 94, 11, 219, 224, 50, 58, 10, 73, 6, 36, 92, 194, 211, 172, 98, 145, 149, 228, 121, 231, 200, 55, 109, 141, 213, 78, 169, 108, 86, 244, 234, 101, 122, 174, 8, 186, 120, 37, 46, 28, 166, 180, 198, 232, 221, 116, 31, 75, 189, 139, 138, 112, 62, 181, 102, 72, 3, 246, 14, 97, 53, 87, 185, 134, 193, 29, 158, 225, 248, 152, 17, 105, 217, 142, 148, 155, 30, 135, 233, 206, 85, 40, 223, 140, 161, 137, 13, 191, 230, 66, 104, 65, 153, 45, 15, 176, 84, 187, 22]

TRANSPOSE = [[3, 1, 4, 5, 6, 7, 0, 2],
 [1, 5, 7, 3, 0, 6, 2, 4],
 [2, 7, 5, 4, 0, 6, 1, 3],
 [2, 0, 1, 6, 4, 3, 5, 7],
 [6, 5, 0, 3, 2, 4, 1, 7],
 [2, 0, 6, 1, 5, 7, 4, 3],
 [1, 6, 2, 5, 0, 7, 4, 3],
 [4, 5, 6, 1, 2, 3, 7, 0]]

RR = [4, 2, 0, 6, 9, 3, 5, 7]
def rr(c, n):
    n = n % 8
    return ((c << (8 - n)) | (c >> n)) & 0xff

import secrets
ROUNDS = 2
MASK = secrets.token_bytes(8)
KEYLEN = 4 + ROUNDS * 4
def encrypt(block, key):
    assert len(block) == 8
    assert len(key) == KEYLEN
    block = bytearray(block)

    for r in range(ROUNDS):
        block = bytearray(xor(block, key[r*4:(r+2)*4]))
        for i in range(8):
            block[i] = SBOX[block[i]]
            block[i] = rr(block[i], RR[i])

        temp = bytearray(8)
        for i in range(8):
            for j in range(8):
                temp[j] |= ((block[i] >> TRANSPOSE[i][j]) & 1) << i

        block = temp

        block = xor(block, MASK)
    return block

def ecb(pt, key):
    if len(pt) % 8 != 0:
        pt = pt.ljust(len(pt) + (8 - len(pt) % 8), b"\x00")

    out = b""
    for i in range(0, len(pt), 8):
        out += encrypt(pt[i:i+8], key)
    return out

key = secrets.token_bytes(KEYLEN)
FLAG = b"irisctf{redacted}"
print(f"MASK: {MASK.hex()}")
print(f"key: {key.hex()}")
import json
pairs = []
for i in range(8):
    pt = secrets.token_bytes(8)
    pairs.append([pt.hex(), encrypt(pt, key).hex()])
print(f"some test pairs: {json.dumps(pairs)}")
print(f"flag: {ecb(FLAG, key).hex()}")
```

Let's take a look at what `encrypt` is doing here. It performs 2 rounds of encryption. Inside each round, we perform the following:

- XOR the block with a portion of the key (either [0:8] or [4:12])
- Map each byte of the block to the the element at that index in `SBOX`; then perform a rightwards bit rotation by `RR[i]` bits.
- Re-map the bits in the order specified by the `TRANSPOSE` function.

As we're given the key in part 1 of this challenge, all we have to do is write up the decryption algorithm.

```python
def decrypt(block, key):
    assert len(block) == 8
    assert len(key) == KEYLEN
    block = bytearray(block)

    for r in range(ROUNDS):
        temp = xor(block, MASK)
        origblock = bytearray(8)
        for i in range(8):
            for j in range(8):
                origblock[i] |= ((temp[j] >> i) & 1) << TRANSPOSE[i][j]

        for i in range(8):
            origblock[i] = rl(origblock[i], RR[i])
            origblock[i] = [j for j in range(len(SBOX)) if SBOX[j] == origblock[i]][0]
        
        pt = bytearray(xor(origblock, key[(ROUNDS - 1 - r)*4:((ROUNDS - 1 - r)+2)*4]))
        block = pt
    
    return block

def ecb_dec(ct, key):
    out = b""
    for i in range(0, len(ct), 8):
        out += decrypt(ct[i:i+8], key)
    return out
```

From here, all we have to do is run the decryption!

```python
MASK = bytes.fromhex('3d5e286c30e3af35')
key = bytes.fromhex('bc62c0b71ac3ebb55c01ca09')
print(ecb_dec(bytes.fromhex('efb6d7f1a2ddefdd04567cedb6d2a6c5fa8b96ad26f92fb1b0b55ad6a13838c6'), key))
```

`b'irisctf{ok_at_least_it_works}\x00\x00\x00'`

Be sure to check out the [SMarT2 writeup](./smart2.md) for the conclusion to this series :)

# SMarT2
## Crypto (487 points) by sera

SMarT1, but without the key!

## Attachments

- [chal.py](assets/smart/chal.py) (from SMarT1)
- [output2.txt](assets/smart/output2.txt)

## Solution

(If you haven't read the [SMarT1 writeup](./smart1.md), I would highly recommend checking that out first! I'll be going through this assuming the understanding we gained in SMarT1)

## The Cryptosystem

As a quick refresher: the cryptosystem in SMarT1 is a two-round cipher, where each round consists of:
- XORing with a portion of the key (bytes 0:8 on the first round and 4:12 on the second)
- passing the bytes through a S-box and a bit rotation
- transposing each group of 64 bits

## The Attack

I've never attempted an attack on a multi-round cipher like this before irisCTF, so I just tried what I knew, and eventually made it work. Due to the transposition step, we don't map bytes to bytes, so we can't just brute force each byte individually. However, one thing that stood out to me was how the key was used. If I was writing this cryptosystem, I would use a 16-byte key (8B for each round). However, this system uses a **12-byte** key, and reuses the middle 4 bytes for both rounds. However, they are XORed with different portions of the message, so there shouldn't be any way to utilize this, right?

This is where the transposition step comes in. Let's first take a look at how this step works:

`block` before transposition: (`a_b` refers to bit `b` of byte `a`)

```
0_7 0_6 0_5 0_4 0_3 0_2 0_1 0_0
1_7 1_6 1_5 1_4 1_3 1_2 1_1 1_0
2_7 2_6 2_5 2_4 2_3 2_2 2_1 2_0
3_7 3_6 3_5 3_4 3_3 3_2 3_1 3_0
4_7 4_6 4_5 4_4 4_3 4_2 4_1 4_0
5_7 5_6 5_5 5_4 5_3 5_2 5_1 5_0
6_7 6_6 6_5 6_4 6_3 6_2 6_1 6_0
7_7 7_6 7_5 7_4 7_3 7_2 7_1 7_0
```

We first start by creating a new empty bytearray `temp` of 8 elements. Then, on iteration `i=0`, we loop through each iteration, and utilize a bitwise OR to move bit `0_3` (`TRANSPOSE[0][0] = 3`) into the new bit `0_0`, bit `0_1` (`TRANSPOSE[0][1] = 1`) into the new bit `1_0`, bit `0_4` (`TRANSPOSE[0][2] = 4`) into the new bit `2_0`, and so on. Thus, after this step, we have:

```
--- --- --- --- --- --- --- 0_3
--- --- --- --- --- --- --- 0_1
--- --- --- --- --- --- --- 0_4
--- --- --- --- --- --- --- 0_5
--- --- --- --- --- --- --- 0_6
--- --- --- --- --- --- --- 0_7
--- --- --- --- --- --- --- 0_0
--- --- --- --- --- --- --- 0_2
```

If we continue this, we end up with:

```
7_4 6_1 5_2 4_6 3_2 2_2 1_1 0_3
7_5 6_6 5_0 4_5 3_0 2_7 1_5 0_1
7_6 6_2 5_6 4_0 3_1 2_5 1_7 0_4
7_1 6_5 5_1 4_3 3_6 2_4 1_3 0_5
7_2 6_0 5_5 4_2 3_4 2_0 1_0 0_6
7_3 6_7 5_7 4_4 3_3 2_6 1_6 0_7
7_7 6_4 5_4 4_1 3_5 2_1 1_2 0_0
7_0 6_3 5_3 4_7 3_7 2_3 1_4 0_2
```

However, one important thing is that the order of the bits eventually doesn't matter for the attack! Let's see why.

Let's consider the message "01234567" being encrypted with the key "ABCDEFGHIJKL". Our goal is to exploit the fact that `EFGH` is used twice during encryption. Let's take a look at what is affected:

**Round 1:** 
- XOR "4567" with "EFGH".
- Run this through the S-box (mapping them to different numbers and thus changing all four of those bytes)
- Perform the transposition.
- XOR the _first_ four bytes with "EFGH"

After this point, the key is no longer used, and we are able to use part of the `SMarT1` solution to reverse the encryption back to that point. Now, let's consider what would happen if we brute forced a single byte of the key (for sake of example, let's brute force the `E` position in the key above)

- If the byte is correct (`E`):
  - `4` is XORed with the correct key value `E` to get the "correct" input to the S-box.
  - The S-box maps the "correct" input to the "correct" output byte at index 4.
  - The transposition maps the byte at index 4 to the 4th bit of each output byte.
  - The second round XORs the output byte at index 0 with the key.

Thus, we can cross-check if the 2^4 bit of the 0th byte of "correct" output (which we guarantee can be known due to no key being used in the latter half of the second round) matches the 2^4 bit of the 0th byte of output when we attempt to encrypt the plaintext using the key. However, there's a problem: what if the S-box _happens_to map to an integer such that the bit ends up correct? Since we have 8 plaintext-ciphertext pairs, the likelihood of all 8 pairs having a match is low, and we can at least use this to reduce our search space.

From here, we can implement the previous algorithm.

```python
def encrypt_1round_justkey(block, key, r):
    assert len(block) == 8
    assert len(key) == KEYLEN
    block = bytearray(block)

    block = bytearray(xor(block, key[r*4:(r+2)*4]))

    return block

def encrypt_1round_nonkey(block):
    assert len(block) == 8
    block = bytearray(block)

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

def decrypt_1round_justkey(block, key, r):
    assert len(block) == 8
    assert len(key) == KEYLEN
    block = bytearray(block)

    block = bytearray(xor(block, key[r*4:(r+2)*4]))

    return block

def decrypt_minus_key(block):
    assert len(block) == 8
    block = bytearray(block)

    temp = xor(block, MASK)
    origblock = bytearray(8)
    for i in range(8):
        for j in range(8):
            origblock[i] |= ((temp[j] >> i) & 1) << TRANSPOSE[i][j]

    for i in range(8):
        origblock[i] = rl(origblock[i], RR[i])
        origblock[i] = [j for j in range(len(SBOX)) if SBOX[j] == origblock[i]][0]
    
    return origblock

def pair_to_key4(pts, cts):
    # BF middle 4 bytes of the key
    # test example: start with byte 4 of the key
    key = bytearray(12)
    valids = []
    for idx in range(4, 8):
        valid = []
        for k in range(256):
            allzero = True
            # print(f'{k:x}')
            key[idx] = k
            for pt, ct in zip(pts, cts):
                postkey_0 = encrypt_1round_justkey(pt, key, 0)
                postround0 = encrypt_1round_nonkey(postkey_0)
                trykey_1 = encrypt_1round_justkey(postround0, key, 1)
                postkey_1 = decrypt_minus_key(ct)
                # print(f'{(trykey_1[idx - 4] ^ postkey_1[idx - 4]):08b}')
                xor = (trykey_1[idx - 4] ^ postkey_1[idx - 4]) & (1 << idx)
                if xor != 0:
                    allzero = False
            if allzero:
                valid.append(f'{k:x}')
        valids.append(valid)
    return valids
```

## Getting the Rest of the Key

From here, we've successfully gotten four of the twelve key bytes. How can we find the rest?

Since we know what `key[4:8]` is, we can start by brute forcing the first four bytes. Since we are able to get the first four bytes of the block after the first round, we can brute force utilizing a similar method to compare the "correct" bits with the bits we get. In this case, we want to look at the 0th bit of byte 0, the 1st bit of byte 1, the 2nd bit of byte 2, and the 3rd bit of byte 3:

```python
def key4_to_key0(pts, cts, key):
    # since we know key[4:8], we can BF key[0], key[1], key[2], key[3] individually
    key = bytearray(key)
    valids = []
    for idx in range(0, 4):
        valid = []
        for k in range(256):
            allzero = True
            key[idx] = k
            for pt, ct in zip(pts, cts):
                postkey_1 = decrypt_minus_key(ct)
                postround0 = decrypt_1round_justkey(postkey_1, key, 1)
                trykey_0 = encrypt_1round_justkey(pt, key, 0)
                tryround0 = encrypt_1round_nonkey(trykey_0)
                diff = xor(tryround0, postround0)[idx] & (1 << idx)
                if diff != 0:
                    allzero = False
            if allzero:
                valid.append(f'{k:x}')
        valids.append(valid)
    return valids
```

Finally, since we know the first 8 bytes of the key, brute forcing the last four is feasible:

```python
def key04_to_key8(pts, cts, key):
    # since we know key[0:8], we can BF key[9], ...
    key = bytearray(key)
    hexes = set()
    for pt, ct in zip(pts, cts):
        postkey_0 = encrypt_1round_justkey(pt, key, 0)
        postround0 = encrypt_1round_nonkey(postkey_0)
        postkey_1 = decrypt_minus_key(ct)
        diff = xor(postround0, postkey_1)[4:8]
        hexes.add(diff.hex())
    if len(hexes) == 1:
        print([k for k in hexes][0])
        return [k for k in hexes][0]
    else:
        return None
```

Now, all that is left is to brute force, with a much reduced search space!

```python
import sys
import itertools

key4s = pair_to_key4([bytes.fromhex(p[0]) for p in pairs], [bytes.fromhex(p[1]) for p in pairs])
for k4 in itertools.product(*key4s):
    key4 = bytes.fromhex('00000000' + ''.join(k4) + '00000000')
    key0s = key4_to_key0([bytes.fromhex(p[0]) for p in pairs], [bytes.fromhex(p[1]) for p in pairs], key4)
    for k0 in itertools.product(*key0s):
        key04 = bytes.fromhex(''.join(k0) + ''.join(k4) + '00000000')
        key8 = key04_to_key8([bytes.fromhex(p[0]) for p in pairs], [bytes.fromhex(p[1]) for p in pairs], key04)
        if key8 != None:
            key = bytes.fromhex(''.join(k0) + ''.join(k4) + key8)
            print('KEY: ', key.hex())
```

And we get our key: `KEY:  d9c9158616cc4e3feb0c413b`. Now, all that is left is to set `MASK` and `key` accordingly, and run our SMarT1 solution to decrypt the message!

`b'irisctf{if_you_didnt_use_a_smt_solver_thats_cool_too}\x00\x00\x00'`

Oh... the intended solution was with Z3 :) Check out [sera's writeup](https://github.com/Seraphin-/ctf/blob/master/irisctf2023/smart.md) for the Z3 based solution!

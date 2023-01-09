# AES-BAD-256
## Crypto (485 points): by sera

> I heard that some common block cipher modes have lots of footguns - using none (ECB) results in the legendary ECB Penguin, while others are vulnerable to bit flipping and padding attacks, so I made my own that would never fall to such a technique.

## Attachments:
- `nc aes.chal.irisc.tf 10100`
- [chal.py](assets/aesbad/chal.py)

## Solution

### The Algorithm

This challenge revolves around breaking a modified AES block cipher mode. If we take a look at the source code provided, we see that we are able to request the encryption of arbitrary plaintexts, and our goal is to submit ciphertext that will decrypt to some JSON with `"type": "flag"`.

To begin, let's take a look at how the encryption scheme works with blocks. After looking at the `encrypt` function, the following scheme is used:

- Right pad the message with null bytes to a length of 256.
- Split up the message into large blocks of size 256.
- For each of these blocks:
  - Loop through the `PERMUTATION` list. For each element `p` of `PERMUTATION`, take the 16 bytes `block[p::16]` (every 16th byte, starting at the `p`th).
  - Concatenate the previous set of 256 bytes back into a new block.
- Combine all of the new blocks, and encrypt using AES-ECB.

This seems pretty daunting, so let's take a look at an example. If `PERMUTATION` is `[1, 3, 5, 7, 9, 11, 13, 15, 0, 2, 4, 6, 8, 10, 12, 14]`, and we are given the message `abcdefghijklmnopqrstuvwxyz`, then we proceed as follows:

```
Original message:

index: 0123456789abcdef0123456789
msg:   abcdefghijklmnopqrstuvwxyz 
```

Iteration 0 (i = 0):  
`PERMUTATION[i] = 1`  
Inner list comprehension: `block[0 * 16 + 1], block[1 * 16 + 1], block[2 * 16 + 1], ...`  
This gives us `"br\x00\x00\x00\x00..."`, since `b` and `r` are the two bytes at the `1` indices of each group of 16 bytes.

Iteration 1 (i = 1):  
`PERMUTATION[i] = 3`  
Inner list comprehension: `block[0 * 16 + 3], block[1 * 16 + 3], block[2 * 16 + 3], ...` = `"dt\x00\x00\x00\x00..."`

We can continue through the remaining iterations to get the scrambled message:

`br\x00\x00\x00...\x00\x00\x00dt\x00\x00\x00...\x00\x00\x00fv\x00\x00\x00...\x00\x00\x00......`

From here, we encrypt this with AES-ECB.

### The Attack

In this challenge, the key realization is that since we are utilizing AES-ECB, we can "piece together" a message just by putting together the correct blocks. How do we find the correct blocks? First, let's think about a simpler case, where `PERMUTATION` is `[0, 1, 2, 3, 4, 5, ..., 15]`. Now, if we group up our message as blocks of 16, as shown below... (AA represents the bytes that encode the length of the message)

```
  0123456789abcdef
  ----------------
0|AA{"type": "echo
1|", "msg": "somem
2|essagehere"}
```

When we perform the permutation operation, we get the following data that will be encrypted with ECB:

```
  0123456
  -------
0|A"e
1|A,s
2|{ s
3|""a
4|tmg
5|yse
6|pgh
7|e"e
8|":r
9|: e
a| ""
b|"s}
c|eo
d|cm
e|he
f|om
```

If we take a look at the code that parses a command, we realize it utilizes `strict=False`. With a bit of experimentation, I found out this means that it will successfully parse the JSON `{"type: "echo", "type": "flag"}` as `{type: "flag"}`. Now, all we have to do is stitch together a message. We can create various echo commands that get us the correct parts of the message, shown below.

```
aim to send
0123456789abcdef|0123456789abcdef|0123456789abcdef|0
AA{"type": "echo|", "msg": "aaaa"|, "type": "flag" }

                      0123456789abcdef0123456789abcdef0123456789abcdef
                      v               v               v
0| A",  (A = 0x2c)  | AB{"type": "echo", "msg": "aaaaa,aaaaaaaaaaaa"}
                       v               v               v
1| B,"  (B = 0x00)  | AB{"type": "echo", "msg": "aaaaa,"}
                        v               v               v
2| { t              | AB{"type": "echo", "msg": "aaaaaaatype"}
                         v               v               v
3| ""y              | AB{"type": "echo", "msg": "aaaaaaatype"}
                          v               v               v
4| tmp              | AB{"type": "echo", "msg": "aaaaaaatype"}
                           v               v               v
5| yse              | AB{"type": "echo", "msg": "aaaaaaatype"}
                            v               v               v
6| pg"              | AB{"type": "echo", "msg": "aaaaaaatype"}
                             v               v               v
7| e":              | AB{"type": "echo", "msg": "aaaaaaaaaaaa:"}
                              v               v               v
8| ":"              | AB{"type": "echo", "msg": "aaaaaaaaaaaa:"}
                               v               v               v
9| : f              | AB{"type": "echo", "msg": "aaaaaaaaaaaaaaflag"}
                                v               v               v
a|  "l              | AB{"type": "echo", "msg": "aaaaaaaaaaaaaaflag"}
                                 v               v               v
b| "aa              | AB{"type": "echo", "msg": "aaaaaaaaaaaaaaflag"}
                                  v               v               v
c| eag              | AB{"type": "echo", "msg": "aaaaaaaaaaaaaaflag"}
                                   v               v               v
d| ca"              | AB{"type": "echo", "msg": "aaaaaaaaaaaaaaflag"}
                                    v               v               v
e| ha}              | AB{"type": "echo", "msg": "aaaaaaaaaaaaaaflag"}
                                     v               v               v
f| o"               | AB{"type": "echo", "msg": "aaaa"}
```

### The Permutation

The previous section assumed that we don't have to worry about the permutation. Now, we have to work out what the permutation is, so that we are able to extract the correct blocks!

Since we are able to make an infinite number of queries, we don't have to worry about running out of queries. Thus, we can start by sending 16 different messages.

```
   0123456789abcdef0123456789abcdef0123456789abcdef
   v               v               v               v
o| AB{"type": "echo", "msg": "AAAAAaaaaaaaaaaaaaaaa"}
   v               v               v               v
0| AB{"type": "echo", "msg": "AAAAAbaaaaaaaaaaaaaaa"}
    v               v               v               v
1| AB{"type": "echo", "msg": "AAAAAabaaaaaaaaaaaaaa"}
     v               v               v               v
2| AB{"type": "echo", "msg": "AAAAAaabaaaaaaaaaaaaa"}
      v               v               v               v
3| AB{"type": "echo", "msg": "AAAAAaaabaaaaaaaaaaaa"}
......
```

When we look at the transposed messages (with no transpose for now), we get the following:
```
  0123456
  -------
0|A"b
1|A,a
2|{ a
3|""a
4|tma
5|ysa
6|pga
7|e"a
8|":a
9|: a
a| "a
b|"Aa
c|eAa
d|cAa
e|hAa
f|oAa
```

If we have a transpose of, say, `[1, 3, 5, 7, ..., 0, 2, 4, 6, ...]` (as before), the transposed message becomes:

```
  0123456
  -------
0|A"a
1|A,b
2|{ a
3|""a
4|tma
5|ysa
6|pga
7|e"a
8|":a
9|: a
a| "a
b|"Aa
c|eAa
d|cAa
e|hAa
f|oAa
```

Since each row is encrypted with ECB, we can find the value of `TRANSPOSE` by sending 17 different messages, and determining which block changes. The code is shown below:

```python
def echo(msg):
  conn.recvuntil(b'> ')
  conn.sendline(b"1")
  conn.sendline(msg.encode())
  conn.recvuntil(b'> ')
  result = conn.recvline()
  return result

def findPerm():
  head = "a" * 5
  body = "a" * 16
  original = echo(head + body)
  blocks = [original[i:i+32] for i in range(0, len(original), 32)]

  perm = [0 for _ in range(16)]

  for i in range(16):
    newbody = "a" * 16
    newbody = newbody[0:i] + "b" + newbody[i+1:]
    modified = echo(head + newbody)
    newblocks = [modified[j:j+32] for j in range(0, len(original), 32)]
    diff = [j for j, (x, y) in enumerate(zip(blocks, newblocks)) if x != y][0]
    perm[diff] = i
  return perm
```

From here, now that we have `perm`, we can utilize our set of messages from before...

```python
messages = [
    "aaaaa,aaaaaaaaaaaa",
    "aaaaa,",
    "aaaaaaatype",
    "aaaaaaatype",
    "aaaaaaatype",
    "aaaaaaatype",
    "aaaaaaatype",
    "aaaaaaaaaaaa:",
    "aaaaaaaaaaaa:",
    "aaaaaaaaaaaaaaflag",
    "aaaaaaaaaaaaaaflag",
    "aaaaaaaaaaaaaaflag",
    "aaaaaaaaaaaaaaflag",
    "aaaaaaaaaaaaaaflag",
    "aaaaaaaaaaaaaaflag",
    "aaaa"
]

def loadMessages():
    newBlocks = ["" for _ in range(16)]
    for i in range(16):
        result = echo(messages[i])
        print(f'Round ${i+1}')
        print(result.decode())
        print()
        blocks = [result[j:j+32] for j in range(0, len(result), 32)]
        for j in range(16):
            if perm[j] == i:
                newBlocks[j] = blocks[j].decode()
                print(f'Selected block: {blocks[j].decode()}')
    return ''.join(newBlocks)
```

From here, all that is left is to combine everything together!

```python
from pwn import *

conn = remote('aes.chal.irisc.tf', 10100)

""" for PoW
print(conn.recv())
solution = input(">")
conn.sendline(solution)
"""

perm = findPerm()
result = loadMessages()
print()
print('Result:')
print(result)

conn.interactive()
```

Here's the output of a sample run (since the key changes each time):

```
(many more blocks before this but removing for clarity)
Selected block: 038f3e2e46871c31c114106f55c9ffe9
Round $16
0f140c15151455d061cb3da66055fb8de2fd789a5245b739432b8f0bfc796725d39bb95cce289cd89e8568fd61ff3801763f8ee8991f50715d0e497a9d1b8eeb78727ea97e2cb8b9081cc470e86133b3cc9e6cdd05a91644f506c4b8fd82d70745fffa7a6892468d130d345b617ac3293d326cabfac11cf68e3288cf16c99a48b49b327deb8b73eb0d065e993fb48815c3d6e052c21693de9d24d7589a64adbfa2ca6975e3c742ff738bc4bae9fefb2f100036986a0149535e9624e2224121daa3aad66acb7cdb8816759d741619e18cb2d44fdd0f3fc5c5ae28e50a60402b1ce4cc4c30044218072daf9ecce1f6a15ef95eaa34817cb43b17b18a5294a31e04


Selected block: e4cc4c30044218072daf9ecce1f6a15e

Result:
058632e128b207e5d2391dd19a1de24362d7387ba47286a3497533905d2e9dcc18fd9325152863655411a94781263f18038f3e2e46871c31c114106f55c9ffe9ec27cfeffb6bd6bde094ac3b4afb2d5fb4c899d79836ebae5fe27ff6843209a0fc4cc5156404796e7f778732fe270ba3632929f724e4f01bd996f6de4bcc140785a805eec32e48c69867075d69f462bf156f5526141d729d5bd745f66e7cd0dc457fe9705cbf09d4ed28e15610273db9d7a3a0df54f729c50fc0470e692442fd82fb4466df0569f4a79e82fdb33813a9cb2e51adb7eeee3ece5c991a4749259ce4cc4c30044218072daf9ecce1f6a15e8696cdef081d5b72b385ee3fd907e7e5
[*] Switching to interactive mode

1. Get an echo command
2. Run a command
3. Exit

> $ 2
Give me a command.

(hex) > $ 058632e128b207e5d2391dd19a1de24362d7387ba47286a3497533905d2e9dcc18fd9325152863655411a94781263f18038f3e2e46871c31c114106f55c9ffe9ec27cfeffb6bd6bde094ac3b4afb2d5fb4c899d79836ebae5fe27ff6843209a0fc4cc5156404796e7f778732fe270ba3632929f724e4f01bd996f6de4bcc140785a805eec32e48c69867075d69f462bf156f5526141d729d5bd745f66e7cd0dc457fe9705cbf09d4ed28e15610273db9d7a3a0df54f729c50fc0470e692442fd82fb4466df0569f4a79e82fdb33813a9cb2e51adb7eeee3ece5c991a4749259ce4cc4c30044218072daf9ecce1f6a15e8696cdef081d5b72b385ee3fd907e7e5
irisctf{bad_at_diffusion_mode}
```

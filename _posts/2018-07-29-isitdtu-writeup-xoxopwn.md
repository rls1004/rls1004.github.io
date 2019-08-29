---
layout: post
title: "[ISITDTU 2018] xoxopwn Write-up"
image: /img/isitdtu_2018.png
tags: [CTF, write-up, pwnable]
---

## Keyword
`pwnable` `python`

---
### Analysis

> nc 178.128.12.234 10002
or
nc 178.128.12.234 10003

```sh
This is function x()>>> x
<function x at 0x7f88b0d2e050>
```
when you access the xoxo service, you can see the above statement by typing 'x'.<br><br>

It looks like a python jail.<br>
so basically, you can try typing `dir()` or `vars()`.
```sh
This is function x()>>> dir()
['a', 'xxx']
```
```sh
This is function x()>>> vars()
{'a': 'vars()', 'xxx': 'finding secret in o()'}
```
You can see that there is a function named `o` through the string stored in the variable `xxx`.<br>

```sh
This is function x()>>> dir(o)
['__call__', '__class__', '__closure__', '__code__', '__defaults__', '__delattr__', '__dict__', '__doc__', '__format__', '__get__', '__getattribute__', '__globals__', '__hash__', '__init__', '__module__', '__name__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'func_closure', 'func_code', 'func_defaults', 'func_dict', 'func_doc', 'func_globals', 'func_name']
```
Some of the attributes that the `o` function has are interesting, such as `func_code`.<br><br>

look inside.
```sh
This is function x()>>> dir(o.func_code)
['__class__', '__cmp__', '__delattr__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__le__', '__lt__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', 'co_argcount', 'co_cellvars', 'co_code', 'co_consts', 'co_filename', 'co_firstlineno', 'co_flags', 'co_freevars', 'co_lnotab', 'co_name', 'co_names', 'co_nlocals', 'co_stacksize', 'co_varnames']
```
`co_code` and `co_consts` look nice :P <br>

First, it is the result of `co_code`.
```sh
This is function x()>>> o.func_code.co_code
d}|jd}d}d}xLtt|D]8}|tt||t||t|A7}q4W||krdGHndGHdS
```
This is python bytecode.<br>
and, you can decode it.
```python
from pwn import *
import dis

r = remote('178.128.12.234', 10002)
r.recvuntil('>>> ')
r.sendline('o.func_code.co_code')

data = r.recvall()
print dis.dis(data)
```
```
    0 LOAD_CONST          1 (1)
    3 STORE_FAST          1 (1)
    6 LOAD_FAST           1 (1)
    9 LOAD_ATTR           0 (0)
    12 LOAD_CONST          2 (2)
    15 CALL_FUNCTION       1
    18 STORE_FAST          1 (1)
    21 LOAD_CONST          3 (3)
    24 STORE_FAST          2 (2)
    27 LOAD_CONST          4 (4)
    30 STORE_FAST          3 (3)
    33 SETUP_LOOP         76 (to 112)
    36 LOAD_GLOBAL         1 (1)
    39 LOAD_GLOBAL         2 (2)
    42 LOAD_FAST           0 (0)
    45 CALL_FUNCTION       1
    48 CALL_FUNCTION       1
    51 GET_ITER       
>>  52 FOR_ITER           56 (to 111)
    55 STORE_FAST          4 (4)
    58 LOAD_FAST           3 (3)
    61 LOAD_GLOBAL         3 (3)
    64 LOAD_GLOBAL         4 (4)
    67 LOAD_FAST           0 (0)
    70 LOAD_FAST           4 (4)
    73 BINARY_SUBSCR  
    74 CALL_FUNCTION       1
    77 LOAD_GLOBAL         4 (4)
    80 LOAD_FAST           2 (2)
    83 LOAD_FAST           4 (4)
    86 LOAD_GLOBAL         2 (2)
    89 LOAD_FAST           0 (0)
    92 CALL_FUNCTION       1
    95 BINARY_MODULO  
    96 BINARY_SUBSCR  
    97 CALL_FUNCTION       1
    100 BINARY_XOR     
    101 CALL_FUNCTION       1
    104 INPLACE_ADD    
    105 STORE_FAST          3 (3)
    108 JUMP_ABSOLUTE      52
>>  111 POP_BLOCK      
>>  112 LOAD_FAST           3 (3)
    115 LOAD_FAST           1 (1)
    118 COMPARE_OP          2 (==)
    121 POP_JUMP_IF_FALSE   132
    124 LOAD_CONST          5 (5)
    127 PRINT_ITEM     
    128 PRINT_NEWLINE  
    129 JUMP_FORWARD        5 (to 137)
>>  132 LOAD_CONST          6 (6)
    135 PRINT_ITEM     
    136 PRINT_NEWLINE  
>>  137 LOAD_CONST          0 (0)
    140 RETURN_VALUE   
None
```
Hmm..<br><br>

Next, it is the result of `co_consts`.
```sh
This is function x()>>> o.func_code.co_consts
(None, '392a3d3c2b3a22125d58595733031c0c070a043a071a37081d300b1d1f0b09', 'hex', 'pythonwillhelpyouopenthedoor', '', 'Open the door', 'Close the door')
```
The `co_consts` returns a tuple containing the literals used by the bytecode.<br>
Among these, `392a3d3c2b3a22125d58595733031c0c070a043a071a37081d300b1d1f0b09` and `pythonwillhelpyouopenthedoor` seem to be suspicious.<br><br>

`(1)` and `(3)` in the bytecode mean `392a3d3c2b3a22125d58595733031c0c070a043a071a37081d300b1d1f0b09` and `pythonwillhelpyouopenthedoor`. (index 1, index 3)<br>
According to the bytecode, this value must be equal after the `XOR` operation.<br>

---
### Solve
Xor the two values.<br>
Rotate if the length is short.
```python
a = '392a3d3c2b3a22125d58595733031c0c070a043a071a37081d300b1d1f0b09'
b = 'pythonwillhelpyouopenthedoor'
b += b[:len(a)/2-len(b)]

a = int(a,16)
b = int(b.encode('hex'),16)

result = hex(a^b)[2:-1].decode('hex')

print result
```
```
ISITDTU{1412_secret_in_my_door}
```

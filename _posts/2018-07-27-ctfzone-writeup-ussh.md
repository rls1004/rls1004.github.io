---
layout: post
title: "[CTFZone 2018] USSH 3.0 Write-up"
image: /img/ctfzone_2018.png
tags: [CTF, write-up, crypto]
---

## 요약
`crypto` `session` `AES` `CBC`

---
### Vuln

> nc crypto-01.v7frkwrfyhsjtbpfcppnu.ctfz.one 1337

```sh
Login: asdfasdf
Group: regular
asdfasdf, welcome to crypto sh

asdfasdf@crypto: $ help
Avaliable commands: ls, cat, id, session, help, exit
```
서비스에 접속하면 id 를 입력할 수 있고, Group 은 자동적으로 regular 로 설정된다. 그 후 쉘이 뜨는데 사용 가능한 명령어가 몇 개 있다.
```sh
asdfasdf@crypto: $ ls
flag.txt  info.txt  backup.sh

asdfasdf@crypto: $ cat flag.txt
cat: flag.txt: Permission denied.
Expect group=root
```
우선 ls 명령어를 통해 flag.txt 파일이 있다는 것을 볼 수 있는데, cat 명령어로 읽으려고 하면 root 의 group 을 요구한다.
```sh
asdfasdf@crypto: $ id
uid=3(regular) gid=3(regular) groups=3(regular)

asdfasdf@crypto: $ session
Usage: session [OPTIONS]

  --get  Print session
  --set [SESSION]  Set session

asdfasdf@crypto: $ session --get
LRbbKN4KlLihxu2D3f12fg==:iK5+5EnwtDlMveNF1RIKWBxoR+mwLrAm+qesFm3gDtU=
```
id 명령로 현재의 권한을 확인해보면 regular 로 설정되어 있는 것을 확인할 수 있다. session 명령어로는 현재의 세션을 가져오거나 새로 설정할 수 있는데 \-\-get 옵션으로 세션을 가져와보면 base64 로 인코딩되어 있고 이를 복호화하면 알 수 없는 바이너리 값들이 나온다.<br>
특징을 살펴보면 `:` 을 기준으로 왼쪽은 항상 16 바이트 길이이고 오른쪽은 32 부터 시작해서 id 값이 길어질 때마다 16 의 배수 길이로 증가한다.
```sh
asdfasdf@crypto: $ session --set LRbbKN4KlLihxu2D3f12fg==:iK5+5EnwtDlMveNF1RIKWBxoR+mwLrAm+qesFm3gDta=
Error: PKCS7 padding is incorrect
```
\-\-set 옵션을 이용해서 원래 세션의 오른쪽 값 중 마지막 바이트를 임의로 바꿔줬더니 놀랍게도 `PKCS7 padding is incorrect` 에러가 발생한다.<br>
그 외에도 왼쪽의 16 바이트를 변경하며 시도해보면 `IV must be 16 bytes long`, 오른쪽의 32 바이트를 변경하며 시도해보면 `Input strings must be a multiple of 16 in length` 라는 흥미로운 에러들이 발생한다.<br><br>

PKCS7 에 대해서 검색해보니 **AES** 암호에서 사용하는 패딩 기법이었고, IV 가 등장하는 것으로 보아 **CBC** 모드를 사용했음을 알 수 있다.

<center><img src="/img/ctfzone_ussh_1.png" class="effect"></center>

<center><img src="/img/ctfzone_ussh_2.png" class="effect"></center>

CBC 모드는 블록 암호화 방식중 하나로, 각 블록은 암호화 되기 전에 이전 블록의 암호화 결과와 XOR 되며, 첫 블록의 경우에는 초기화 벡터(IV)가 사용된다.<br>
복호화 될 때는 복호화 된 후에 이전 블록의 암호문과 **XOR** 되어 최종적인 평문 블록을 만든다. 그렇기 때문에 암호 블록을 조작하여 대상 평문 블록을 조작할 수도 있다.
<br><br><br>

### Solve
일단 평문 데이터가 어떤 형식으로 되어있는지를 모르니 IV 의 값을 무작위로 바꿔보면서 평문이 바뀌는지 확인해본다.
```python
from pwn import *

r = remote('crypto-01.v7frkwrfyhsjtbpfcppnu.ctfz.one', 1337)

def login(id):
	r.recvuntil(':')
	r.sendline(id)

id = 'a'*10
login(id)

r.recvuntil('$')
r.sendline('session --get')
session = r.recvuntil('\n')
session = session.split(':')
iv = session[0].decode('base64')
ct = session[1].decode('base64')

r.recvuntil('$')
for i in range(len(iv)):
	print ""
	n_iv = iv[:i]+"A"+iv[i+1:]
	n_session = n_iv.encode('base64')[:-1]+":"+ct.encode('base64')[:-1]
	r.sendline('session --set '+n_session)
	print ("[%03d]"%i) + r.recvuntil('$')
	r.sendline('id')
	print ("[%03d]"%i) + r.recvuntil('$')

r.interactive()
```
```
[009] Ýaaaaaaaaa@crypto: $
[009] uid=3(regular) gid=3(regular) groups=3(regular)
Ýaaaaaaaaa@crypto: $

[010] a:aaaaaaaa@crypto: $
[010] uid=3(regular) gid=3(regular) groups=3(regular)
a:aaaaaaaa@crypto: $

[011] aa\x0caaaaaaa@crypto: $
[011] uid=3(regular) gid=3(regular) groups=3(regular)
aa\x0caaaaaaa@crypto: $

[012] aaa×aaaaaa@crypto: $
[012] uid=3(regular) gid=3(regular) groups=3(regular)
aaa×aaaaaa@crypto: $

[013] aaaa?aaaaa@crypto: $
[013] uid=3(regular) gid=3(regular) groups=3(regular)
aaaa?aaaaa@crypto: $

[014] aaaaa#aaaa@crypto: $
[014] uid=3(regular) gid=3(regular) groups=3(regular)
aaaaa#aaaa@crypto: $

[015] aaaaaa¿aaa@crypto: $
[015] uid=3(regular) gid=3(regular) groups=3(regular)
aaaaaa¿aaa@crypto: $
```
놀랍게도 IV의 마지막 7 바이트가 id 의 첫 7 바이트에 영향을 준다.<br>
id 와 group 값을 구분하는 문자를 알아내기 위해 다시 한 번 brute force 를 시도했다.
```python
for i in range(0x100):
	n_iv = iv[:12]+chr(i^ord('a')^ord(iv[12]))+iv[13:]
	n_session = n_iv.encode('base64')[:-1]+":"+ct.encode('base64')[:-1]
	r.sendline('session --set '+n_session)
	print "[%03d]"%i + r.recvuntil(': $')
```
```
[036] aaa$aaaaaa@crypto: $
[037] aaa%aaaaaa@crypto: $
[038] aaa@crypto: $
[039] aaa'aaaaaa@crypto: $
[040] aaa(aaaaaa@crypto: $
```
나머지는 다 id 의 중간 값이 바뀌지만 `&` 일 때는 뒤의 문자열이 무시된다. 흠 이거로군..
<br><br>
그렇다면 `?????aaaaaaaaaa&??????` 이런식으로 평문이 구성되어 있을텐데, `&`를 없애버리면 뒤쪽의 문자열이 id 값으로 출력될 수 있지 않을까?
```python
n_iv = iv[:13]+chr(ord('&')^ord('a')^ord(iv[13]))+iv[13+1:]
```
```sh
aaaa@crypto: $
session: Invalid session
```
안됨.. group 을 지정하는 부분이 없어지면 안되나 봄
<br><br><br><br>
그냥 무작정 brute force 하면서 regular 위치 알아내기
```python
for i in range(len(ct)):
        r.recvuntil(': $')
        n_ct = ct[:i]+"a"+ct[i+1:]
        n_session = iv.encode('base64')[:-1]+":"+n_ct.encode('base64')[:-1]
        r.sendline('session --set '+n_session)
        print r.recvuntil(': $')
        r.sendline('id')
        print "[%03d]"%i+r.recvuntil('\n')
```
```sh
session: Invalid session
aaaa@crypto: $
[014] uid=3(regular) gid=3(regular) groups=3(regular)

Error: PKCS7 padding is incorrect
aaaa@crypto: $
[015] uid=3(regular) gid=3(regular) groups=3(regular)
```
regular 가 바뀌게 되는 위치를 찾지 못한다. 전부 다 `Invalid session` 아니면 `PKCS7 padding is incorrect` 에러가 발생한다. 여기서 다시 생각을 좀 해야한다.

<center><img src="/img/ctfzone_ussh_33.png" class="effect"></center>

첫 번째 평문 블록이 가지고 있을 데이터를 생각해보면 위 그림처럼 표현할 수 있다.<br>
CBC 모드에 의하면 첫 번째 암호 블록을 복호화하고 IV 와 XOR 한 값이 첫 번째 평문 블록이 되는데, IV 의 열 번째 바이트부터 id 값에 영향을 줬으니 첫 번째 평문 블록의 열 번째 바이트부터 7 bytes 는 실제 id 값에 해당하는 데이터를 가지고 있다.<br>
하지만 id 값으로 4 bytes 길이인 'aaaa'를 입력했으므로 여기에는 id 값 이외에 group 에 대한 정보가 들어가게 될것이다. 또한 id와 group 정보를 구분하는 <span style="color:#cf3030">구분자(&)</span>도 들어가게 된다.<br><br>
첫 번째 암호 블록을 조작하면 두 번째 암호 블록이 복호화되었을 때 첫 번째 암호 블록과 XOR 되므로 평문을 조작할 수 있지만, 첫 번째 암호 블록의 복호화 결과는 알 수 없는 값으로 바뀌게 된다. 이 과정에서 id와 group 정보를 구분하는 <span style="color:#cf3030">구분자(&)</span> 또한 사라지게 되어 `Invalid session` 이라는 에러가 발생하는 것으로 보인다.<br><br>
두 번째 암호 블록에는 나머지 데이터들과 <span style="color:#cf3030">패딩 값</span>이 들어간다. 암호화할 데이터의 길이가 16의 배수에 맞지 않다면 빈 공간을 패딩으로 채우는데, 2 byte 가 모자라다면 <span style="color:#cf3030">02 02</span> 이라는 패딩을, 4 bytes 가 모자라다면 <span style="color:#cf3030">04 04 04 04</span> 라는 패딩을 채운다. 따라서 이 암호 블록이 조작될 경우, 패딩이 어긋나게 되어 `PKCS7 padding is incorrect` 라는 에러가 발생하게 된다.<br><br>
이러한 정보들을 조합했을 때, "구분자(&)와 패딩 값을 암호화 하고 있지 않은 블록이면서 다음 블록이 group 에 대한 정보를 가지고 있는 암호 블록" 을 조작해야 원하는 결과를 얻을 수 있다!

<center><img src="/img/ctfzone_ussh_4.png" class="effect"></center>

대략 이런 구조를 만들어서 두 번째 암호 블록을 조작해야 한다.<br>
위와 같은 구조를 만들기 위해서 id 값을 늘려줘야 하는데, 첫 번째 블록에 들어갈 7 bytes 와 두 번재 블록에 들어갈 16 bytes 를 더해서 최소 <span style="color:#cf3030">23 bytes</span> 길이의 id 값을 입력해야 한다 :P <br><br>

```python
id = 'a'*23
login(id)
...
for i in range(16,16+16): # second block range
        r.recvuntil(': $')
        n_ct = ct[:i]+"b"+ct[i+1:]
        n_session = iv.encode('base64')[:-1]+":"+n_ct.encode('base64')[:-1]
        r.sendline('session --set '+n_session)
        print r.recvuntil(': $')
        r.sendline('id')
        print "[%03d]"%i+r.recvuntil('\n')
```
```
aaaaaaa!|\x00Ñâh~\x14ÃøÂ\x18ÚO\x1b@crypto: $
[023] uid=3(¤egular) gid=3(¤egular) groups=3(¤egular)

aaaaaaa^×Ð¬Ø29\x04#hÓ8\x07Ò×@crypto: $
[024] uid=3(rêgular) gid=3(rêgular) groups=3(rêgular)

aaaaaaa07î\x15\x16tHk.þu»@crypto: $
[025] uid=3(reular) gid=3(reular) groups=3(reular)

aaaaaaaKrö\x1e'ÆÔÅ\x07înW¨k@crypto: $
[026] uid=3(regJlar) gid=3(regJlar) groups=3(regJlar)

aaaaaaay\x07àj\x16\x158NVån}@crypto: $
[027] uid=3(reguÉar) gid=3(reguÉar) groups=3(reguÉar)

aaaaaaa b<¨NÐ¶ßUÐÞø-@crypto: $
[028] uid=3(regul¯r) gid=3(regul¯r) groups=3(regul¯r)

aaaaaaa>r#åÁòN«ò|éÞ®Q@crypto: $
[029] uid=3(regulaÓ) gid=3(regulaÓ) groups=3(regulaÓ)
```
다시 결과를 확인해보면 23 번째 바이트부터 한 바이트씩 'regular'에 영향을 주고 있다.<br>
```python
solve = ''
solve += chr(ord(ct[23])^ord('r')^ord('r'))
solve += chr(ord(ct[24])^ord('e')^ord('o'))
solve += chr(ord(ct[25])^ord('g')^ord('o'))
solve += chr(ord(ct[26])^ord('u')^ord('t'))

n_ct = ct[:23]+solve+ct[23+len(solve):]
```
```
uid=3(rootlar) gid=3(rootlar) groups=3(rootlar)
```
regualr 를 root 로 바꾸는 방법은 간단하다. `XOR 연산의 원리`를 이용하면 되는데, 원래의 평문 값을 XOR 해주면 0이 되므로 여기에 원하는 값을 다시 XOR 해주면 된다.<br>
결과 값으로 (rootlar) 이 나왔다. 위에 있는 lar 은 필요 없는 값인데 공백으로 바꿔주면 (root&nbsp;&nbsp;&nbsp;) 가 된다. (root) 랑은 다른 값이다.<br><br>
여기에서는 사전에 미리 알아냈던 <span style="color:#cf3030">구분자(&)</span>를 이용하여 뒤쪽의 값을 무시할 수 있다.
```python
from pwn import *

r = remote('crypto-01.v7frkwrfyhsjtbpfcppnu.ctfz.one', 1337)

def login(id):
        r.recvuntil(':')
        r.sendline(id)

id = 'a'*23
login(id)

r.recvuntil('$')
r.sendline('session --get')
session = r.recvuntil('\n')
session = session.split(':')
iv = session[0].decode('base64')
ct = session[1].decode('base64')

r.recvuntil('$')

solve = ''
solve += chr(ord(ct[23])^ord('r')^ord('r'))
solve += chr(ord(ct[24])^ord('e')^ord('o'))
solve += chr(ord(ct[25])^ord('g')^ord('o'))
solve += chr(ord(ct[26])^ord('u')^ord('t'))
solve += chr(ord(ct[27])^ord('l')^ord('&'))

n_ct = ct[:23]+solve+ct[23+len(solve):]

n_session = iv.encode('base64')[:-1]+":"+n_ct.encode('base64')[:-1]
r.sendline('session --set '+n_session)

r.interactive()
```
```
aaaaaaaþÂÉâ°3IÙI(£Ó·3@crypto: $ $   id
uid=0(root) gid=0(root) groups=0(root)
aaaaaaaþÂÉâ°3IÙI(£Ó·3@crypto: $ $   ls
flag.txt  info.txt  backup.sh
aaaaaaaþÂÉâ°3IÙI(£Ó·3@crypto: $ $   cat flag.txt
ctfzone{2e71b73d355eac0ce5a90b53bf4c03b2}
```
ctfzone{2e71b73d355eac0ce5a90b53bf4c03b2}
<br>

### Ref
https://ko.wikipedia.org/wiki/블록_암호_운용_방식
https://en.wikipedia.org/wiki/Padding_(cryptography)

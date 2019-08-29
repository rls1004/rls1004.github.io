---
layout: post
title: "[ISITDTU 2018] aes_cnv Write-up"
image: /img/isitdtu_2018.png
tags: [CTF, write-up, crypto]
---

## Keyword
`crypto` `AES` `bit flipping`

---
### Analysis

> nc 35.185.111.53 13337

```sh
Give me content of your request you want to encrypt: d
Your encypted: P+WoLz2DQBAxj/N3s2xBhmz4wT3mnEiH9gjZ7tOjDHiHfFZismM8IknFW4IipKXO
Give me encrypted of your request: P+WoLz2DQBAxj/N3s2xBhmz4wT3mnEiH9gjZ7tOjDHiHfFZismM8IknFW4IipKXO
Your request: d
```
문자열을 입력하면 해당 문자열에 대한 암호문을 알려준다. request 로 암호문을 보내면 그 값을 복호화한다.<br>
<br>
```python
if req.startswith("Give me the flag"):
		self.request.sendall("Sorry! We don't support it!\n")
		exit()
self.request.sendall("Your encypted: ")
self.request.sendall(self.game.encode(req)+"\n")
self.request.sendall("Give me encrypted of your request: ")
enc = self.request.recv(BUFF_SIZE).strip()
try:
		dec = self.game.decode(enc)
except:
		self.request.sendall("Error!\n")
		exit()
self.request.sendall("Your request: %s\n" %dec)
if dec.startswith("Give me the flag"):
		self.request.sendall("This is flag: %s\n" %__FLAG__)
```
소스 코드 상에서 살펴보면 requset 를 decode 했을 때 `Give me the flag` 로 시작하는 문자열이라면 flag 를 출력해준다.<br>
"Give me the flag"의 encode 값을 알아내야 하는데 "Give me the flag"로 시작하는 문자열을 입력하면 encode 를 수행하지 않고 종료한다.<br>
<br>
```python
def encrypt(self, plain_text, iv):
		"""
		Encrypt plaint text
		"""
		assert len(iv) == 16
		plain_text = pad(plain_text)
		assert len(plain_text)%BLOCK_SIZE == 0
		cipher_text = ''
		aes = AES.new(self.key, AES.MODE_ECB)
		h = iv
		for i in range(len(plain_text)//BLOCK_SIZE):
				block = plain_text[i*16:i*16+16]
				block = xor(block, h)
				cipher_block = aes.encrypt(block)
				cipher_text += cipher_block
				h = md5(cipher_block).digest()
		return b64encode(iv+cipher_text)
```
encode 함수는 encrypt 함수를 호출한다. 여기서 AES ECB 모드를 사용하는데, 각각의 블록을 이전 블록의 MD5 해시(첫 번째 블록일 경우는 IV)와 XOR 하고 각 블록 마다 ECB 모드로 암호화한다.
```python
def decrypt(self, cipher_text):
		"""
		Decrypt cipher text
		"""
		cipher_text = b64decode(cipher_text)
		assert len(cipher_text)%BLOCK_SIZE == 0
		iv = cipher_text[:16]
		cipher_text = cipher_text[16:]
		aes = AES.new(self.key, AES.MODE_ECB)
		h = iv
		plain_text = ''
		for i in range(len(cipher_text)//BLOCK_SIZE):
				block = cipher_text[i*16:i*16+16]
				plain_block = aes.decrypt(block)
				plain_block = xor(plain_block, h)
				plain_text += plain_block
				h = md5(block).digest()
		return unpad(plain_text)
```
decrypt 할 때는 ECB 모드로 복호화한 값을 IV 혹은 이전 블록의 MD5 해시와 XOR 하여 plain_text 를 만든다.<br>
<br>
첫 번째 블록의 평문은 IV 와 XOR 된 값이기 때문에 IV 값을 조작하면 첫 번째 블록의 decrypt 결과를 원하는대로 바꿔줄 수 있다.<br>
여기서 한 블록의 크기는 16 으로 정의하고 있는데 마침 "<span style="color:#cf3030">G</span>ive me the flag"가 16 글자다.<br>
"<span style="color:#cf3030">g</span>ive me the flag" 등 의 입력을 보내서 encode 된 값을 가져온 다음 IV 값을 조작하여 plain_text 가 "Give me the flag"가 되도록 만들어주면 된다.

---
### Solve
<br>
```python
from pwn import *

r = remote('35.185.111.53', 13337)

r.recvuntil('encrypt: ')
r.sendline('give me the flag')
r.recvuntil('encypted: ')

dd = r.recvuntil('\n')
aa = dd.decode('base64')
aa = chr(ord(aa[0])^ord('g')^ord('G'))+aa[1:]

r.sendline(aa.encode('base64')[:-1])

print r.recvall()
```
**ISITDTU{chung96vn_i5_v3ry_h4nds0m3}**

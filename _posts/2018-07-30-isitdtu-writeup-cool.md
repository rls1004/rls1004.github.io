---
layout: post
title: "[ISITDTU 2018] cool Write-up"
image: /img/isitdtu_2018.png
tags: [CTF, write-up, reversing]
---

## Keyword
`reversing` `md5` `xor`

---
### Analysis

<br>

<center><img src="/img/isitdtu_cool_1.JPG" class="effect"></center>

The length of the string to be entered must be 28.
<br><br>

<center><img src="/img/isitdtu_cool_2.JPG" class="effect"></center>

and then, there is a function that compares the partial parts of the input value ...<br>
I named it `md5cmp_400CE5`<br><br>

<center><img src="/img/isitdtu_cool_3.JPG" class="effect"></center><br>

<center><img src="/img/isitdtu_cool_4.JPG" class="effect"></center>

This function does not directly compare the input value with a specific string. Generates an `md5 hash` for the input value and compares it to a specific string.<br><br>

To find the first 12 bytes, we need to decrypt the three MD5 hashes shown above.

<center><img src="/img/isitdtu_cool_5.JPG" class="effect"></center>

I used hashkiller.<br><br>

Compares 12 bytes to each hash value, and then compares 1 byte to 33. it is '!'.<br>
It is not finished yet.<br>
<br>

<center><img src="/img/isitdtu_cool_6.JPG" class="effect"></center>

Last 15 bytes.<br>
From the 14th character, the result of `xor` from the first character to the nth character must be the same as the specific value(byte_6020A8).<br>

<center><img src="/img/isitdtu_cool_7.JPG" class="effect"></center>


---
### Solve
<br>
```python
flag = "fl4g_i5_h3r3!"
data = "7D4D2344360276036F5B2F46761839".decode('hex')

m = 0

for c in flag:
	m = m ^ ord(c)

for c in data:
	flag += chr( m ^ ord(c) )
	m = m ^ ( m ^ ord(c) )

print flag
```
**ISITDTU{fl4g_i5_h3r3!C0ngr4tul4ti0n!}**

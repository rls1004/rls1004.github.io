---
layout: post
title: "[Defenit CTF] Write-up"
image: /img/202006/defenit.png
tags: [CTF, write-up, pwnable, web, reversing, misc]
---


지난 주말에 <font color='ORANGERED'>최동민 1인 팀(Dongmin Oneman Show)</font>으로 Defenit CTF에 참가했습니다.  
<center><img src="/img/202006/defenit_score.png" class="effect"></center>

Misc - QR Generator, Minesweeper  
Reversing - MixMix  
Web - Tar Analyzer, BabyJS  
Pwnable - BitBit  

에 대한 write-up 입니다.

---
# Misc

## QR Generator

<center><img src="/img/202006/defenit_qr_00.PNG" class="effect"></center><br>
<center><img src="/img/202006/defenit_qr_01.PNG" class="effect"></center>

서버에 접속하면 QR 코드가 0과 1로 표현되어 출력됩니다. QR 코드가 나타내는 값을 짧은 시간 안에 입력해야 하는 문제고 100개의 stage가 있습니다. 제한 시간이 짧으므로 손으로 풀 순 없고 코드를 짜야 합니다.  

우선 0과 1로 표현된 QR 코드를 이미지로 변환했습니다. 사실 이 부분이 귀찮아서 비트 배열 자체로 처리할 수 있는 방법을 찾고 있었는데 팀원분이 이미지 변환 코드를 올려주셨습니다.  
```python
from pwn import *
from qrcode.image.pil import PilImage
from pyzbar.pyzbar import decode
from PIL import Image

p = remote("qr-generator.ctf.defenit.kr", 9000)

print(p.recvuntil("What is your Hero's name? "))
p.sendline(" ")

for it in range(100):
  print it,
  p.recvuntil("< QR >\n")
  
  qrbit = []
  
  data = p.recvline()[:-2]
  data = data.decode('ascii').split(' ')
  qrbit.append(data)
  cnt = len(data)
  
  for i in range(cnt-1):
    data = p.recvline()[:-2]
    data = data.decode('ascii').split(' ')
    qrbit.append(data)

  image_factory = PilImage
  im = image_factory(2, cnt, 20)
  for r in range(cnt):
    for c in range(cnt):
      if int(qrbit[r][c]):
        im.drawrect(r, c)
  im.save('test.png')
```
QR 코드의 크기가 매번 달라지기 때문에 비트 배열의 1행을 읽어와서 비트의 개수를 알아내고 cnt에 저장해서 사용했습니다.  

<center><img src="/img/202006/defenit_qr_03.PNG" class="effect" width="45%"><img src="/img/202006/defenit_qr_04.PNG" class="effect" width="45%" style="margin-left:3%"></center>  
이미지가 올바르게 생성되는 것을 확인할 수 있습니다.  

이제 이미지의 QR 코드를 디코딩해야 합니다. QR 코드와 관련된 다양한 라이브러리가 존재하지만 `Zxing`을 제외하고는 무슨 이유인지 문제에서 생성한 QR 코드를 제대로 해석하지 못합니다. `Zxing`은 구글에서 제공하는 오픈소스로 Zebra Crossing의 약자입니다. QR코드 스캔 어플리케이션의 대다수가 이를 이용하고 있다고 합니다.  

전체 코드입니다. zxing은 python3 부터 지원되는데 pwntool은 아직 python3를 지원하지 않는 줄 알고 코드를 따로 짰습니다. ㅎㅁㅎ  
```python
# ex_qr.py
from pwn import *
from qrcode.image.pil import PilImage
from pyzbar.pyzbar import decode
from PIL import Image

p = remote("qr-generator.ctf.defenit.kr", 9000)

print(p.recvuntil("What is your Hero's name? "))
p.sendline(" ")

for it in range(100):
  #print it,
  p.recvuntil("< QR >\n")
  
  qrbit = []
  
  data = p.recvline()[:-2]
  print data
  data = data.decode('ascii').split(' ')
  qrbit.append(data)
  cnt = len(data)
  
  for i in range(cnt-1):
    data = p.recvline()[:-2]
    print data
    data = data.decode('ascii').split(' ')
    qrbit.append(data)

  image_factory = PilImage
  im = image_factory(2, cnt, 20)
  for r in range(cnt):
    for c in range(cnt):
      if int(qrbit[r][c]):
        im.drawrect(r, c)
  im.save('test.png')
  raw_input()
  
  cmd = ['python3', 'ex_qr2.py']
  fd_popen = subprocess.Popen(cmd, stdout=subprocess.PIPE).stdout
  data = fd_popen.read().strip()
  fd_popen.close()
  
  p.sendline(data)

p.interactive()
```
```python
# ex_qr2.py
import zxing

reader = zxing.BarCodeReader()
barcode = reader.decode('./test.png')
print(barcode.raw)
```

## Minesweeper
<center><img src="/img/202006/defenit_min_00.PNG" class="effect"></center><br>
<center><img src="/img/202006/defenit_min_01.PNG" class="effect"></center>

지뢰찾기 게임입니다. 맵은 항상 16x16 이고 1분 안에 게임을 클리어하면 flag를 주는 문제입니다.  

<center><img src="/img/202006/defenit_min_02.PNG" class="effect" width="45%"><img src="/img/202006/defenit_min_03.PNG" class="effect" width="45%" style="margin-left:3%"></center>

열에 해당하는 알파벳과 행에 해당하는 숫자 조합으로 입력을 하면 해당 위치를 클릭하게 됩니다. 뒤에 'f'를 붙이면 해당 위치에 flag를 표시할 수 있습니다. 룰은 일반 지뢰찾기 게임과 동일합니다. 각 셀에 있는 숫자는 인접한 8개의 셀에 존재하는 폭탄의 수를 나타냅니다.  

<center><img src="/img/202006/defenit_min_04.PNG" class="effect"></center>


저는 지뢰찾기 고수여서 33초만에 클리어하고 flag를 받았습니다.

# Reversing

## MixMix

<center><img src="/img/202006/defenit_mixmix_00.PNG" class="effect"></center>

문제 바이너리와 out.txt 파일이 주어집니다. out.txt에는 알 수 없는 문자들이 쓰여져 있는데, 문제의 바이너리에 어떤 입력을 주었을 때 out.txt 와 같은 결과를 만들어낼 수 있는지 분석하는 문제입니다.  

<center><img src="/img/202006/defenit_mixmix_06.PNG" class="effect"></center>

먼저 main 함수를 살펴봤습니다. 사용자에게 길이가 32 이하인 문자열을 입력 받고 `sub_1188()` 함수를 호출하여 format에 데이터를 만들어냅니다. 그리고 format을 out.txt 파일에 저장합니다.  

<center><img src="/img/202006/defenit_mixmix_07.PNG" class="effect"></center>

`sub_1188()` 함수에서는 여러 함수들을 호출하면서 몇몇 배열을 설정하고 사용자의 입력 값을 변화시킵니다. 먼저 `sub_DCF()`, `sub_9E0()` 에서는 rand 함수를 이용하여 난수 배열을 생성하고 이를 뒤섞는데, 하드 코딩 된 고정 된 값의 seed를 사용하기 때문에 섞인 배열은 사용자의 입력값과 관계없이 항상 같은 값이 같은 순서대로 저장됩니다. 저는 그 배열에 `rand_202040` 이라는 이름을 붙였습니다.  

<center><img src="/img/202006/defenit_mixmix_02.PNG" class="effect"></center>

`sub_E6C()` 함수에서는 사용자가 입력한 데이터를 한글자씩, 최하위 비트부터 차례로 1비트씩 가져와서 배열에 저장합니다. 배열의 원소 하나 당 한 개의 비트를 저장하게 됩니다.

<center><img src="/img/202006/defenit_mixmix_03.PNG" class="effect"></center>

`sub_FC5()` 함수에서는 위에서 만든 비트 배열을 정해진 순서로 섞습니다. 정해진 순서라고 표현한 것은 rand_202040 배열이 항상 같은 값을 같기 때문입니다.

<center><img src="/img/202006/defenit_mixmix_04.PNG" class="effect"></center>

`sub_108B()` 함수에서는 그걸 한 번 더 섞습니다.. 여기엔 202C40과 202C44 배열도 사용되는데, 둘 다 rand_202040을 이용하여 만들어진 배열이기 때문에 두 배열 역시 항상 같은 값을 같은 순서로 저장하고 있습니다.

<center><img src="/img/202006/defenit_mixmix_05.PNG" class="effect"></center>

`sub_F01()` 함수에서는 최종적으로 섞여진 비트 배열을 8개씩 끊어서, 0번째 비트가 최상위 비트가 되고 7번째 비트가 최하위 비트가 되는 순서로 8비트를 1바이트로 합칩니다.

<center><img src="/img/202006/defenit_mixmix_01.PNG" class="effect"></center>

`sub_B2A()` 함수에서 format 데이터의 최종적인 모습이 만들어집니다. 위의 과정에서 최종적으로 만들어진 배열이 a1이고 format이 저장될 곳이 a2 입니다. v16_255 배열은 항상 같은 값을 같은 순서로 저장하고 있는 배열입니다.  

v16_255 배열에서 가져온 값에 어떤 연산을 수행하고 그 값을 a1에 있는 값과 xor 연산을 수행하여 a2 배열에 저장합니다. a2의 값은 out.txt에 주어져있고 v16_255의 값도 항상 일정하여 알아낼 수 있는 값이므로 a1 배열의 값도 알아낼 수 있습니다. 여기서부터 흐름을 거꾸로 따라가면서 입력 값을 알아낼 수 있습니다.  
```python
xor_arr = [0xf7, 0xa2, 0x82, 0x49, 0x8b, 0xfc, 0xea, 0x28, 0x8e, 0x92, 0x4b, 0x86, 0x51, 0x73, 0xd7, 0xa9, 0xa5, 0xa9, 0x38, 0xea, 0x69, 0xa3, 0x8b, 0xe7, 0xac, 0x17, 0x09, 0x6a, 0x8b, 0xbf, 0xfd, 0x15]

shuffle = [0x48, 0x54, 0x04, 0x9c, 0x88, 0x38, 0x94, 0x2c, 0xa8, 0xb4, 0x18, 0xdc, 0xc8, 0x98, 0xd4, 0x8c, 0x14, 0x28, 0x44, 0x08, 0x4c, 0x34, 0x40, 0x3c, 0xf0, 0xf8, 0x64, 0xfc, 0xe8, 0x00, 0xf4, 0x20, 0x78, 0x68, 0x58, 0x70, 0x5c, 0x74, 0x50, 0x7c, 0xc4, 0xd0, 0xa4, 0xc0, 0xac, 0xd8, 0xa0, 0xcc, 0x90, 0x1c, 0xb8, 0x84, 0xbc, 0x10, 0xb0, 0x80, 0x0c, 0x6c, 0xe4, 0x30, 0xec, 0x60, 0xe0, 0x24, 0x148, 0x154, 0x104, 0x19c, 0x188, 0x138, 0x194, 0x12c, 0x1a8, 0x1b4, 0x118, 0x1dc, 0x1c8, 0x198, 0x1d4, 0x18c, 0x114, 0x128, 0x144, 0x108, 0x14c, 0x134, 0x140, 0x13c, 0x1f0, 0x1f8, 0x164, 0x1fc, 0x1e8, 0x100, 0x1f4, 0x120, 0x178, 0x168, 0x158, 0x170, 0x15c, 0x174, 0x150, 0x17c, 0x1c4, 0x1d0, 0x1a4, 0x1c0, 0x1ac, 0x1d8, 0x1a0, 0x1cc, 0x190, 0x11c, 0x1b8, 0x184, 0x1bc, 0x110, 0x1b0, 0x180, 0x10c, 0x16c, 0x1e4, 0x130, 0x1ec, 0x160, 0x1e0, 0x124, 0x248, 0x254, 0x204, 0x29c, 0x288, 0x238, 0x294, 0x22c, 0x2a8, 0x2b4, 0x218, 0x2dc, 0x2c8, 0x298, 0x2d4, 0x28c, 0x214, 0x228, 0x244, 0x208, 0x24c, 0x234, 0x240, 0x23c, 0x2f0, 0x2f8, 0x264, 0x2fc, 0x2e8, 0x200, 0x2f4, 0x220, 0x278, 0x268, 0x258, 0x270, 0x25c, 0x274, 0x250, 0x27c, 0x2c4, 0x2d0, 0x2a4, 0x2c0, 0x2ac, 0x2d8, 0x2a0, 0x2cc, 0x290, 0x21c, 0x2b8, 0x284, 0x2bc, 0x210, 0x2b0, 0x280, 0x20c, 0x26c, 0x2e4, 0x230, 0x2ec, 0x260, 0x2e0, 0x224, 0x348, 0x354, 0x304, 0x39c, 0x388, 0x338, 0x394, 0x32c, 0x3a8, 0x3b4, 0x318, 0x3dc, 0x3c8, 0x398, 0x3d4, 0x38c, 0x314, 0x328, 0x344, 0x308, 0x34c, 0x334, 0x340, 0x33c, 0x3f0, 0x3f8, 0x364, 0x3fc, 0x3e8, 0x300, 0x3f4, 0x320, 0x378, 0x368, 0x358, 0x370, 0x35c, 0x374, 0x350, 0x37c, 0x3c4, 0x3d0, 0x3a4, 0x3c0, 0x3ac, 0x3d8, 0x3a0, 0x3cc, 0x390, 0x31c, 0x3b8, 0x384, 0x3bc, 0x310, 0x3b0, 0x380, 0x30c, 0x36c, 0x3e4, 0x330, 0x3ec, 0x360, 0x3e0, 0x324]

input_relate_shuffle = [41, 36, 116, 232, 24, 214, 145, 67, 139, 45, 61, 98, 117, 50, 136, 234, 194, 79, 131, 233, 103, 43, 172, 169, 111, 143, 199, 19, 163, 173, 95, 102, 229, 89, 21, 90, 47, 17, 78, 97, 85, 22, 204, 11, 128, 66, 5, 46, 13, 0, 93, 130, 42, 185, 59, 142, 63, 65, 161, 138, 213, 137, 73, 105, 18, 251, 221, 34, 192, 62, 60, 76, 86, 68, 198, 141, 64, 170, 177, 20, 155, 190, 244, 186, 120, 1, 216, 148, 236, 80, 238, 237, 174, 31, 113, 118, 107, 71, 188, 208, 51, 16, 180, 218, 87, 110, 147, 7, 140, 55, 108, 152, 14, 191, 44, 196, 37, 243, 124, 23, 126, 220, 122, 215, 109, 193, 171, 12, 2, 119, 211, 104, 92, 240, 230, 121, 217, 70, 88, 9, 30, 206, 6, 53, 94, 207, 133, 178, 202, 249, 195, 112, 69, 252, 15, 38, 175, 25, 127, 77, 189, 91, 162, 82, 29, 153, 187, 54, 132, 114, 239, 176, 56, 165, 179, 184, 159, 254, 33, 151, 32, 197, 144, 168, 49, 226, 167, 212, 210, 222, 181, 224, 75, 146, 135, 228, 245, 219, 106, 101, 156, 52, 149, 248, 209, 160, 58, 40, 125, 129, 99, 83, 84, 3, 200, 164, 35, 28, 74, 39, 57, 203, 242, 8, 72, 227, 134, 183, 115, 253, 182, 4, 223, 205, 26, 123, 48, 150, 100, 235, 246, 10, 27, 250, 247, 158, 231, 81, 241, 225, 255, 157, 154, 201, 96, 166]

output = open('out.txt.0').read()
output = list(output)

a1 = [0 for _ in range(32)]

for i in range(32):
  a1[i] = xor_arr[i] ^ ord(output[i])

shuffle_2 = [0 for _ in range(32*8)]

for i in range(32):
  tmp = a1[i]
  for k in range(8):
    shuffle_2[i*8+7-k] = tmp & 1
    tmp = tmp >> 1

shuffle_ir = [0 for _ in range(32*8)]
for i in range(len(shuffle)):
  shuffle_ir[shuffle[i]/4] = shuffle_2[i]

input_relate = [0 for _ in range(32*8)]
for i in range(len(shuffle)):
  input_relate[input_relate_shuffle[i]] = shuffle_ir[i]

dd = []

for i in range(32):
  tmp = 0
  for k in range(8):
    tmp += input_relate[i*8+k] << k
  dd.append(tmp)

print ''.join(map(chr,dd))
```

```sh
$ python ex_mixmix.py 
Defenit{m1x_r4nd_c0lumn_r0w_rc4}
```


# Web

## Tar Analyzer

<center><img src="/img/202006/defenit_tar_00.PNG" class="effect"></center>

소스 코드와 서버가 주어집니다.

<center><img src="/img/202006/defenit_tar_01.PNG" class="effect"></center>

*.tar 파일을 제출하면 압축을 해제하여 보여주는 웹 서비스 입니다. 팀원분이 취약점을 찾으셨습니다.  

<center><img src="/img/202006/defenit_tar_02.PNG" class="effect"></center>

심볼릭 링크를 tar로 압축하면 다른 경로에 대한 접근이 가능했습니다. 루트 디렉토리(/)에 대한 심볼릭 링크 파일인 root를 생성하고 ex_tar.tar로 압축하여 업로드 했습니다.

<center><img src="/img/202006/defenit_tar_03.PNG" class="effect"></center>

'제출' 버튼을 클릭하면 file을 보여주는 화면이 나타납니다.  

<center><img src="/img/202006/defenit_tar_04.PNG" class="effect"></center>

여기서 root를 클릭하면 Not Found 라는 페이지가 뜹니다.  

<center><img src="/img/202006/defenit_tar_05.PNG" class="effect"></center>

루트 디렉토리에 접근했는지 확인하기 위해 /root/etc/passwd 경로에 접근해봤습니다. root는 심볼릭 링크이기 때문에 실제로는 /etc/passwd에 대한 접근입니다. 접근 결과, passwd 파일이 내려받아 졌습니다.  

<center><img src="/img/202006/defenit_tar_06.PNG" class="effect"></center>

그 다음에 race condition을 해야할지 고민했는데 flag는 flag.txt에 있지 않을까 싶어서 /flag.txt에 접근했더니 flag를 읽을 수 있었습니다. flag 내용을 보니 race condition이 intended solution인 것 같습니다.  

## BabyJS

<center><img src="/img/202006/defenit_babyjs_00.PNG" class="effect"></center>

소스 코드와 서버가 주어집니다.  

<center><img src="/img/202006/defenit_babyjs_01.PNG" class="effect"></center>

write라는 메뉴만 있는 심플한 구조입니다.  

<center><img src="/img/202006/defenit_babyjs_12.PNG" class="effect"></center>

코드를 살펴보면, write 메뉴를 통해서 "FLAG" 문자열이 포함되지 않으면서 200자 이하의 글을 쓸 수 있고 이게 html 파일로 만들어집니다. 그리고 페이지가 렌더링 될 때 `{FLAG, 'apple': 'mint'}`라는 데이터도 넘어가는데, write 한 html 파일에서 이 내용을 읽을 수 있으면 됩니다.

<center><img src="/img/202006/defenit_babyjs_04.PNG" class="effect"></center>  

이 웹 서비스는 Handlebars 라는 템플릿 엔진을 사용하고 있습니다. 템플릿 인젝션을 테스트하기 위해 `apple`의 값을 요청했습니다.  

<center><img src="/img/202006/defenit_babyjs_05.PNG" class="effect"></center>

템플릿 인젝션을 통해 `apple`에 저장된 `mint`를 가져왔습니다.  

<center><img src="/img/202006/defenit_babyjs_06.PNG" class="effect"></center>

"FLAG" 문자열에 대한 필터링이 있으니 일단 `#each`를 이용해서 모든 key 값을 가져왔습니다.  

<center><img src="/img/202006/defenit_babyjs_07.PNG" class="effect"></center>

`FLAG`와 `apple` 외에도 다른 key들이 존재합니다.  

<center><img src="/img/202006/defenit_babyjs_08.PNG" class="effect"></center>

{% raw %}
`{{.}}`을 이용해서 각 key가 가진 value를 요청했습니다.  
{% endraw %}

<center><img src="/img/202006/defenit_babyjs_09.PNG" class="effect"></center>

하지만 결과는 Cannot convert object to primitive value 라는 에러를 내뿜습니다.  

<center><img src="/img/202006/defenit_babyjs_10.PNG" class="effect"></center>

알고 보니 value가 없는 경우가 있어서 에러가 발생한 것이었습니다. `#if`를 이용해서 `this.length`에 대한 조건을 추가했습니다.  

<center><img src="/img/202006/defenit_babyjs_11.PNG" class="effect"></center>

가져왔습니다.

# Pwnable

## BitBit

<center><img src="/img/202006/defenit_bitbit_00.PNG" class="effect"></center>

대회 끝나기 1시간 전에 붙잡았는데 Description을 읽지 않고 풀어서 너무 아쉬웠던 문제였습니다. pwnable이어서 당연히 쉘 따는 문제로 생각해서 heap overflow 취약점을 찾았고 fread의 eof를 이용하여 free를 트리거 시킬 수 있는 것을 알아내고 뭐 그런 분석을 하고 있다가 대회가 종료됐습니다.  

오늘 문득 Description을 보니 쉘 따는 문제가 아니라 프로그램이 정상 동작 하도록 바이너리를 패치하는 문제였습니다 ㅠㅠ..!!  

<center><img src="/img/202006/defenit_bitbit_02.PNG" class="effect"></center>

주어진 바이너리는 bmp 파일의 경로를 인자로 주면 해당 파일을 읽어서 비어있는 네모와 채워진 네모로 표현해줍니다.  

<center><img src="/img/202006/defenit_bitbit_01.PNG" class="effect"></center>

바이너리의 내용 중 문제가 되는 부분인데요, bmp 파일의 내용을 파싱하여 색상 테이블 개수를 사이즈로 하여 메모리를 할당합니다. 그 후에 각 테이블에 다시 메모리를 할당하여 RGB 데이터를 읽어옵니다.  

여기서 이상한 점을 눈치챌 수 있는데요, 두 번째 for문에서 색상 테이블 개수만큼 메모리가 할당되기 때문에 처음에 할당해야하는 메모리 크기는 `[색상 테이블 개수]`가 아니라 `[색상 테이블 개수 * 8]`입니다.  

<center><img src="/img/202006/defenit_bitbit_03.PNG" class="effect"></center>

size 값을 적당히 패치해야 하는데 저는 패치할 코드로 memory_allocation_401260() 함수를 선택했습니다. 모든 메모리 할당에 대해서 size가 8배로 늘어나겠지만 문제가 되진 않을테니까요.  

<center><img src="/img/202006/defenit_bitbit_04.PNG" class="effect"></center>
<center><img src="/img/202006/defenit_bitbit_06.PNG" class="effect" style="margin-top:3%"></center>

패치 전 입니다.

<center><img src="/img/202006/defenit_bitbit_05.PNG" class="effect"></center>
<center><img src="/img/202006/defenit_bitbit_07.PNG" class="effect" style="margin-top:3%"></center>

패치 후 입니다.  

<center><img src="/img/202006/defenit_bitbit_08.PNG" class="effect"></center>

서버와 연결할 때는 패치한 바이너리를 어딘가에 올려놓고 http 링크를 입력해야 합니다. 저는 github에 임시로 올려놨습니다.  

바이너리 패치 문제인 것을 깨닫고 패치해서 flag 받기까지 3분 걸렸습니다. 다음부터는 Description 좀 잘 읽어야겠습니다 ㅠㅠ  


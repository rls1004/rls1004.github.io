---
layout: post
title: "[POX 2018 Final] amugeona Write-up"
image: /img/pox_2018.PNG
tags: [CTF, write-up, pwn, pwnable, windows]
---

## Keyword
`pwnable`

---
### Description

<br>

윈도우 포너블 문제.<br>
amugeona.exe 파일이 주어졌다.<br>
amugeona.exe는 파일 정보를 파싱해서 보여주는 간단한 프로그램이다.<br>
`*.rls` 확장자에 대해서는 특별하게 파싱한다.<br>
<br>
문제 서버에는 메일 서비스가 동작하고 있고 공격 대상에게 피싱 메일을 보내는 컨셉이다.<br>
메일에는 사용자가 입력한 파일 바이너리가 `[랜덤해시].rls`로 첨부된다.<br>

<br><br><br>

### Vulnerability


`*.rls` 파일의 구조는 다음과 같이 간단하다.


* <b>전체적인 구조</b>

| offset | size | desc |
| :------ | :--- |:--- |
| 0 | 4 | signature ('1004') |
| 4 | 4 | 파일 총 크기<br>실제 크기와 다르면 파싱 종료 |
| 8 | 4 | unused |
| 12 | 4 | 파일 총 개수 |
| 16 | - | 파일 정보<br>파일 총 개수만큼 반복 |

* <b>파일 정보에 대한 구조</b>

| offset | size | desc |
| :------ | :--- |:--- |
| 0 | 4 | 파일명 길이(n) |
| 4 | n | 파일명 |
| n+4 | 4 | 파일 크기(m) |
| n+8 | m | 파일 데이터 |

<br>
<br>
<br>

<center><img src="/img/pox_2018_final_amugeona.png" class="effect"></center>

일반 파일을 열면 파일 이름, 파일 크기, 파일 이름에 대한 3개의 해시 값을 보여준다.<br>
`*.rls` 파일은 여러 파일의 정보를 담고 있는 파일이며 amugeona.exe로 해당 파일을 open할 경우, 내부에 저장된 각 파일의 정보를 보여준다.<br>
<br>

<center><img src="/img/pox_2018_final_amugeona_1.png" class="effect"></center>

3개의 파일 정보를 담고 있는 `*.rls`파일을 열면 위처럼 세 개의 파일 정보가 나온다.<br>
<br>
`+` 버튼을 사용하여 다른 파일을 추가로 open 할 수 있지만 최대 17개까지만 open이 가능하다.<br>
하지만 `*.rls` 파일에 담긴 파일 개수에 대해서는 제한하고 있지 않기 때문에 17개 이상의 파일 open이 가능해진다.<br>
<br>
파일의 데이터(제목, 크기, 해시)는 구조체로 저장되고, 하나의 구조체는 0x280의 크기를 갖는다.<br>
제목, 크기, 해시1, 해시2, 해시3은 각각 0x80만큼의 크기를 차지한다.<br>
파일 개수를 0x666667로 조작하면 0x280 * 0x666667 = 0x1 00000180 = 0x180 으로 <span style="color:#cf3030">integer overflow</span>가 발생한다.<br>
첫 번째 파일의 데이터를 저장하려고 하면 0x180 크기의 힙 영역에 0x280 만큼의 데이터를 정하려 하기 때문에 <span style="color:#cf3030">heap overflow</span>가 발생한다.<br>
이를 이용하여 다른 객체를 덮어쓸 수 있고, 함수 포인터 또한 덮어쓸 수 있다.<br>
<br>
나머지는 stack pivoting 후 ROP를 사용하여 리버스 쉘 획득 후 flag를 읽으면 된다.<br>
두 번째 파일의 제목 데이터가 객체의 함수 포인터를 덮어쓰게 되는데, <br>제목의 중간 값부터 객체를 덮기 때문에 ROP에 사용할 수 있는 총 바이트 수는 63 바이트 정도로 제한이 있다.<br>

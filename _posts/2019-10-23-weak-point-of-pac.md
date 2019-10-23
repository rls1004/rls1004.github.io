---
layout: post
title: "Weak Poin of PAC"
image: /img/pac_2_17.png
tags: [iOS, mitigation, PAC, PAuth, weak_point]
---

# PAuth Protection
공격자는 대상 프로세스에서 올바르게 Sign 된 포인터를 조작할 수 없습니다. Sign 된 포인터를 만들기 위해서는 Signing Key 와 Context 가 필요한데 Signing key 는 모든 프로세스마다 유니크합니다. Context 또한 모든 프로세스에게 private 한 값입니다. **Signing Key** 와 **Context** 를 알 수 없는 공격자는 대상 프로세스에서 올바르게 인식 될 Sign 된 포인터를 만들어내지 못하는 것입니다.  
  
하지만 만약 어떤 두 프로세스에서 동일한 Signing Key 와 Context 가 사용된다면 어떨까요?  
해당 프로세스간 포인터 대체 공격(Pointer Substitution Attack)이 가능하고 PAuth-protection 은 실패하게 됩니다.  
  
그렇다면 iOS 에서는 Signing Key 와 Context 를 올바르게 보호할까요?  

# Experiments
실험을 통해 확인해 보도록 하겠습니다.
서로 다른 개발자 계정으로 개발된 프로그램 A 와 B가 있습니다.
A 에서 Sign 된 포인터를 만들고 B 에서 그것을 검증해보도록 하겠습니다.
( 모든 Key 와 2개의 Context 를 이용했습니다)  

<center><img src="/img/pac_2_17.png" class="effect"></center>  

**A 에서 Sign 된 포인터가 B 에서 올바르게 검증될 수 있을까요?**

## (1-1) A-iKey & Zero Context
먼저, 인스트럭션 포인터를 검증하는 A-Key 에 대한 실험입니다. 실험에 사용된 A 프로세스의 코드는 아래와 같습니다.
```c
asm (
	"LDR X0, =0x12345678\n\t"
	"PACIZA X0\n\t"
	"AUTIZA X0"
);
```
- **PACIZA X0**  
APIAKey 와 Zero Context 를 사용하여 X0 에 대한 PAC 계산 후 삽입
- **AUTIZA X0**  
APIAKey 와 Zero Context 를 사용하여 X0 에 대한 PAC 검증 후 PAC 제거

X0 에 0x12345678 이라는 값을 넣고, 이 값에 대해 Sign 하여 PAC 값을 삽입하고 다시 Auth 하여 PAC 값을 제거할 것입니다.

<center><img src="/img/pac_2_00.png" class="effect" width="400px"></center>  

A 프로세스에서는 잘 동작합니다.

B 프로세스에서는 A 프로세스에서 Sign 한 값이 올바르게 Auth 될 수 있는지 실험해보겠습니다.
B 프로세스의 코드는 아래와 같습니다.  
```c
asm (
	"AUTIZA X0"
);
```
디버거를 이용해서 X0 의 값을 A 에서 Sign 한 값으로 바꿔줄 것입니다.  

<center><img src="/img/pac_2_01.png" class="effect" width="400px"></center>  

X0 의 값을 A 프로세스가 Sign 한 값인 0x00440f8012345678 로 바꿔줬더니 Auth 결과가 올바르게 나왔습니다.

## (1-2) A-iKey & SP Context
이번에 Zero Context 대신 SP Context 를 사용했습니다. 실험에 사용된 A 프로세스의 코드는 아래와 같습니다.
```c
asm (
	"LDR X0, =0x12345678\n\t"
	"PACIA X0, SP\n\t"
	"AUTIA X0, SP"
);
```
- **PACIA X0, SP**  
APIAKey 와 SP Context 를 사용하여 X0 에 대한 PAC 계산 후 삽입
- **AUTIA X0, SP**  
APIAKey 와 SP Context 를 사용하여 X0 에 대한 PAC 검증 후 PAC 제거

비슷한 코드이지만 이번엔 Context 로 SP 레지스터의 값을 사용했습니다.  

<center><img src="/img/pac_2_02.png" class="effect" width="400px"></center>  

A 프로세스에서는 Sign 결과 0x001a348012345678 이라는 값이 나왔습니다. B 프로세스에서 이 값이 올바르게 Auth 될 수 있는지 확인해보겠습니다. 아래는 B 프로세스의 코드입니다.  
```c
asm (
	"AUTIA X0, SP"
);
```
이번에도 디버거를 이용해서 X0 와 SP 의 값은 A 프로세스에서 Sign 된 값과 Sign 에 사용된 SP 의 값으로 바꿔줄 것입니다.  

<center><img src="/img/pac_2_03.png" class="effect" width="400px"></center> 

이번에도 역시 Auth 결과가 올바르게 나왔습니다.

## (2-1) A-dKey & Zero Context
이번엔 데이터 포인터를 검증하는 A-Key 에 대한 실헙니다. 실험에 사용된 A 프로세스의 코드는 아래와 같습니다.
```c
asm (
	"LDR X0, =0x12345678\n\t"
	"PACDZA X0\n\t"
	"AUTDZA X0"
);
```
- **PACDZA X0**  
APDAKey 와 Zero Context 를 사용하여 X0 에 대한 PAC 계산 후 삽입
- **AUTDZA X0**  
APDAKey 와 Zero Context 를 사용하여 X0 에 대한 PAC 검증 후 PAC 제거  
<br>

<center><img src="/img/pac_2_04.png" class="effect" width="400px"></center>

아래는 검증에 사용된 B 프로세스의 코드입니다.
```c
asm (
	"AUTDZA X0"
);
```

X0 의 값은 A 프로세스에서 Sign 된 값으로 바꿔줍니다.  

<center><img src="/img/pac_2_05.png" class="effect" width="400px"></center>

이번에도 올바른 값으로 검증에 성공했습니다.  


## (2-2) A-dKey & SP Context
이번에 Zero Context 대신 SP Context 를 사용했습니다. 실험에 사용된 A 프로세스의 코드는 아래와 같습니다.
```c
asm (
	"LDR X0, =0x12345678\n\t"
	"PACDA X0, SP\n\t"
	"AUTDA X0, SP"
);
```
- **PACDA X0, SP**  
APDAKey 와 SP Context 를 사용하여 X0 에 대한 PAC 계산 후 삽입
- **AUTDA X0, SP**  
APDAKey 와 SP Context 를 사용하여 X0에 대한 PAC 검증 후 PAC 제거  
<br>

<center><img src="/img/pac_2_06.png" class="effect" width="400px"></center>

아래는 검증에 사용된 B 프로세스의 코드입니다.
```c
asm (
	"AUTDA X0, SP"
);
```

X0 의 값은 A 프로세스에서 Sign 된 값으로 바꾸고 SP 의 값은 그때 사용된 A 프로세스의 SP 값으로 바꿔줍니다. 

<center><img src="/img/pac_2_07.png" class="effect" width="400px"></center>

또 성공했습니다.  

## (3-1) B-iKey & Zero Context
이번에는 인스트럭션 포인터를 검증하는 B-Key 에 대한 실험입니다. 실험에 사용된 A 프로세스의 코드는 아래와 같습니다.
```c
asm (
	"LDR X0, =0x12345678\n\t"
	"PACIZB X0\n\t"
	"AUTIZB X0"
);
```
- **PACIZB X0**  
APIBKey 와 Zero Context 를 사용하여 X0 에 대한 PAC 계산 후 삽입
- **AUTIZB X0**  
APIBKey 와 Zero Context 를 사용하여 X0 에 대한 PAC 검증 후 PAC 제거  
<br>

<center><img src="/img/pac_2_08.png" class="effect" width="400px"></center>

검증에 사용된 B 프로세스의 코드입니다.
```c
asm (
	"AUTIZB X0"
);
```

X0 의 값은 A 프로세스에서 Sign 된 값으로 바꿔줍니다.  

<center><img src="/img/pac_2_09.png" class="effect" width="400px"></center>

이번엔 검증에 실패하여 PAC 값이 제거되지 못하고 error 값으로 대체되었습니다.

## (3-2) B-iKey & SP Context
이번에 Zero Context 대신 SP Context 를 사용했습니다. 실험에 사용된 A 프로세스의 코드는 아래와 같습니다.
```c
asm (
	"LDR X0, =0x12345678\n\t"
	"PACIB X0, SP\n\t"
	"AUTIB X0, SP"
);
```
- **PACIB X0, SP**  
APIBKey 와 SP Context 를 사용하여 X0 에 대한 PAC 계산 후 삽입
- **AUTIB X0, SP**  
APIBKey 와 SP Context 를 사용하여 X0 에 대한 PAC 검증 후 PAC 제거  
<br>

<center><img src="/img/pac_2_10.png" class="effect" width="400px"></center>

검증에 사용된 B 프로세스의 코드입니다.
```c
asm (
	"AUTIB X0, SP"
);
```

X0 의 값은 A 프로세스에서 Sign 된 값으로 바꾸고 SP 의 값은 그때 사용된 A 프로세스의 SP 값으로 바꿔줍니다. 

<center><img src="/img/pac_2_11.png" class="effect" width="400px"></center>

이번에도 검증에 실패했습니다.

## (4-1) B-dKey & Zero Context
이번에는 데이터 포인터를 검증하는 B-Key 에 대한 실험입니다. 실험에 사용된 A 프로세스의 코드는 아래와 같습니다.
```c
asm (
	"LDR X0, =0x12345678\n\t"
	"PACDZB X0\n\t"
	"AUTDZB X0"
);
```
- **PACDZB X0**  
APDBKey 와 Zero Context 를 사용하여 X0 에 대한 PAC 계산 후 삽입
- **AUTDZB X0**  
APDBKey 와 Zero Context 를 사용하여 X0 에 대한 PAC 검증 후 PAC 제거  
<br>

<center><img src="/img/pac_2_12.png" class="effect" width="400px"></center>

검증에 사용된 B 프로세스의 코드입니다.
```c
asm (
	"AUTDZB X0"
);
```

X0 의 값은 A 프로세스에서 Sign 된 값으로 바꿔줍니다.  

<center><img src="/img/pac_2_13.png" class="effect" width="400px"></center>

검증에 실패하여 PAC 값이 제거되지 못하고 error 값으로 대체되었습니다.

## (4-2) B-dKey & SP Context
이번에 Zero Context 대신 SP Context 를 사용했습니다. 실험에 사용된 A 프로세스의 코드는 아래와 같습니다.
```c
asm (
	"LDR X0, =0x12345678\n\t"
	"PACDB X0, SP\n\t"
	"AUTDB X0, SP"
);
```
- **PACDB X0, SP**  
APDBKey 와 SP Context 를 사용하여 X0 에 대한 PAC 계산 후 삽입
- **AUTDB X0, SP**  
APDBKey 와 SP Context 를 사용하여 X0 에 대한 PAC 검증 후 PAC 제거  
<br>

<center><img src="/img/pac_2_14.png" class="effect" width="400px"></center>

검증에 사용된 B 프로세스의 코드입니다.
```c
asm (
	"AUTDB X0, SP"
);
```

X0 의 값은 A 프로세스에서 Sign 된 값으로 바꾸고 SP 의 값은 그때 사용된 A 프로세스의 SP 값으로 바꿔줍니다. 

<center><img src="/img/pac_2_15.png" class="effect" width="400px"></center>

검증에 실패했습니다.


# Result
A 프로세스에서 각각의 Key 와 Context 를 이용해서 PAC 를 생성했습니다. 이를 B 프로세스에서 검증해보니 아래와 같은 결과가 나왔습니다.

||A-iKey|A-dKey|B-iKey|B-dKey|
|-|-|-|
|Zero Context|<font color='MEDIUMSEAGREEN'>Valid</font>|<font color='MEDIUMSEAGREEN'>Valid</font>|<font color='SALMON'>Invalid</font>|<font color='SALMON'>Invalid</font>|
|Concrete Context|<font color='MEDIUMSEAGREEN'>Valid</font>|<font color='MEDIUMSEAGREEN'>Valid</font>|<font color='SALMON'>Invalid</font>|<font color='SALMON'>Invalid</font>|

놀라운 결과입니다.  
A 프로세스에서 A-Key 를 이용해 Sign 된 포인터는 B 프로세스에서도 올바르게 검증됩니다. iOS 는 서로 다른 user-space 프로세스에서 동일한 키를 사용하는걸까요?  

**user-space 시스템 서비스**에서도 적용 가능할까요?  
이를 알아보기 위해서는 서비스에서 Sign 된 포인터가 사용자 프로세스에서 올바르게 검증되는지 확인해야 합니다. 실험을 위해서는 먼저 user-space 시스템 서비스에서 Sign 된 포인터 값을 가져와야 하는데요, 취약점을 이용해서 가져올 수 있습니다. 서비스 프로세스에서 Crash 가 발생하면, **Crash Log** 에 레지스터 값이 나타나게 되는데 그 중 Sign 된 포인터 값이 포함되어 있을 것입니다. (이 부분에 대해서는 직접 실험해보지 못했는데 해보게 되면 추후에 내용을 추가하겠습니다)  

DEFCON 의 발표에서는 logd 서비스에서 발생하는 **null-pointer dereference** 취약점을 이용했습니다. logd 의 `com.apple.logd` 서비스를 공격하여 Crash Log 를 획득한 뒤 X16 레지스터에 남아있는 Sign 된 포인터 값을 찾았습니다. 이 값은 BRAA X16, X17 을 통해 Sign 된 값이므로 Context 인 X17 레지스터의 값도 확인해야 합니다. 이렇게 찾은 값들을 사용자 프로세스에서 AUTIA 인스트럭션을 사용하여 검증하도록 실험하면 됩니다.  

실험 결과는 **검증 성공**이었습니다.  

즉, 사용자 프로세스와 시스템 서비스가 **동일한 A-Key 를 사용**한다는 것을 알 수 있었습니다. 하지만 Context 값이 private 하기 때문에 포인터 대체 공격은 불가능합니다. Key 관리에 문제가 있는 것은 맞지만 private 한 Context 가 프로세스간 포인터 대체 공격을 방지하고 있습니다. Context 값을 알지 않고는 시스템 서비스에서 올바르게 검증되는 Sign 된 포인터 값을 조작할 수 없습니다.  

..하지만 <font color='ORANGERED'>Zero Context</font> 라는 것이 존재합니다! BLRAA<font color='ORANGERED'>Z</font> / BRAA<font color='ORANGERED'>Z</font> / AUTI<font color='ORANGERED'>Z</font>A / AUTI<font color='ORANGERED'>Z</font>B / AUTD<font color='ORANGERED'>Z</font>A / AUTD<font color='ORANGERED'>Z</font>B 등 Zero Context 를 사용하여 포인터를 검증하는 명령어들이 있습니다.  


정리 해 봅시다!  
로컬의 공격자 (같은 디바이스 내 다른 프로세스) 는 공격자의 프로세스에서 Sign 된 포인터를 생성할 수 있습니다. 시스템 프로세스에서 그 포인터들이 사용될 때, Zero Context 를 사용한다면 검증을 통과할 수 있게 됩니다.  

[이전의 post](https://rls1004.github.io/2019-08-29-pac/) 에서 PAC 의 도입으로 인해 ROP/JOP 등의 Code Reuse Attack 이 엄청나게 어려워졌다고 말씀드렸었는데 이로 인해 JOP 가 다시 가능해집니다! A-iKey 와 Zero Context 를 사용하는 모든 branch 인스트럭션은 JOP 가젯으로써 활용될 수 있습니다. 또한 일부 가젯은 Context 레지스터를 제어하는데도 사용할 수 있을 것입니다.  

JOP 는 살아났는데 ROP 는 어떨까요? Return Address 는 함수 프롤로그에서 PACISP 에 의해 Sign 되고 에필로그에서 RETAB 에 의해 검증됩니다. B-iKey 를 사용하는 것인데 B-Key 는 프로세스마다 유니크합니다. 또한 Context 로는 SP 를 사용하기 때문에 private 합니다. 결론적으로 ROP 는 어렵습니다 :(  

# iOS user space 에서 PAuth 의 키 관리 문제

**A-Key** 는 디바이스마다 다릅니다. 각 디바이스가 부팅될 때마다 새로 바뀝니다. 같은 디바이스 내의 모든 프로세스는 동일한 A-Key 를 사용합니다.  
**B-Key** 는 프로세스마다 다릅니다. 프로세스가 재시작 될 때마다 새로 바뀝니다. 서로 다른 프로세스는 서로 다른 B-Key 를 사용합니다.  

## Performance 와 Security 사이의 Trade-off
공유 라이브러이에는 A-Key 를 이용해 Sign 되는 수 많은 함수 포인터가 있습니다. 공유 라이브러리의 메모리 영역은 새로운 프로세스를 생성할 때 **COW(Copy-on-Write)**[^cra] 방식을 사용합니다. 때문에 서로 다른 프로세스에서 서로 다른 A-Key를 이용해 포인터에 Sign 한다면 새로운 프로세스를 생성할 때 공유 라이브러리들을 복사하기 위해 더 많은 시간과 메모리 공간이 필요하게 됩니다.  

<center><img src="/img/pac_2_16.png" class="effect"></center>  

이러한 이유로 서로 다른 프로세스가 같은 A-Key 를 사용하고 있습니다.  


# Reference
- DEFCON 27, “HackPac: Hacking Pointer Authentication in iOS User Space”, Xiaolong Bai, Min (Spark) Zheng

[^cra]: https://drawings.jvns.ca/copyonwrite/ 
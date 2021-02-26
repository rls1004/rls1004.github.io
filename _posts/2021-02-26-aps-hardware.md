---
layout: post
title: "[APS] Hardware security and biometrics"
image: /img/apple-security.png
tags: [iOS, macOS, Apple]
---

Apple Platform Security 2021 FEB 버전의 보고서 중 **<span style="color:#3090C7">Hardware security and biometrics</span>** 장에서 Apple SoC security를 전반적으로 설명하는 내용이 새로 추가되었고 Secure Enclave의 내용이 크게 증가했으며 Hardware microphone disconnect에 다이어그램과 함께 약간의 설명이 추가되었습니다.  

이 포스팅에서는 **Secure Enclave**의 내용을 중점으로 번역 하고 마지막 부분에서 내용을 정리합니다.  

# Apple SoC security
Apple이 설계한 실리콘은 모든 Apple 제품에서 공통 아키텍처를 형성하며 Mac은 물론 iPhone, iPad, Apple TV, Apple watch를 지원한다. Apple의 세계적 수준의 실리콘 설계 팀은 10년 넘게 Apple Soc(System On Chip)을 구축하고 개선해 왔다. 그 결과, 보안 기능에서 업계를 선도하는 모든 디바이스에 대해 확장 가능한 아키텍처가 탄생했다. 보안 기능을 위한 이러한 공통 기반은 소프트웨어와 함께 작동하도록 자체 실리콘을 설계한 회사에서만 사용 가능하다.  

Apple 실리콘은 특별히 아래에 설명 된 시스템 보안 기능을 활성화하도록 설계 및 제작 되었다.  

|Feature|A10|A11, S3|A12, S4|A13, S5|A14, S6|M1|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|Kernel Integrity Protection|💚|💚|💚|💚|💚|💚|
|Fast Permission Restrictions| |💚|💚|💚|💚|💚|
|System Coprocessor<br>Integrity Protection| | |💚|💚|💚|💚|
|Pointer Authentication Codes| | |💚|💚|💚|💚|
|Page Protection Layer| |💚|💚|💚|💚|🙅‍♀️|

<span style="color:#3090C7">Page Protection Layer(PPL)</span>는 플랫폼이 서명되고 신뢰할 수 있는 코드만 실행 하도록 한다. 이것은 macOS에 적용할 수 없는 보안 모델이다.  

Apple이 설계한 실리콘은 특히 아래에 자세히 설명 된 데이터 보호 기능을 활성화 한다.  

|Feature|A10|A11, S3|A12, S4|A13, S5|A14, M1, S6|
|:-:|:-:|:-:|:-:|:-:|:-:|
|Sealed Key Protection (SKP)|💚|💚|💚|💚|💚|
|recoveryOS - All Data Protection<br>Classes protected|💚|💚|💚|💚|💚|
|Alternate boots of DFU, Diagnostics,<br>and Update - Class A, B,<br>and C data protected| | |💚|💚|💚|

# Secure Enclave

## 개요

**Secure Enclave**는 Apple Soc(System On Chip)에 통합된 보안 전용 서브 시스템이다. Secure Enclave는 추가 보안 계층을 제공하기 위해 메인 프로세서와 격리되며, 애플리케이션 프로세서 커널이 손상되는 경우에도 민감한 사용자 데이터를 안전하게 유지하도록 설계되었다. 이는 SoC와 동일한 설계 원칙을 따른다. (하드웨어 신뢰 루트를 설정하는 boot ROM, 효율적이고 안전한 암호화 작업을 위한 AES 엔진 및 메모리 보호). Secure Enclave에는 스토리지가 포함되어 있지 않지만 애플리케이션 프로세서 및 운영체제에서 사용하는 NAND 플래시 스토리지와는 별도로 연결된 스토리지에 정보를 안전하게 저장하는 메커니즘이 있다.  

![secure enclave components](/img/202102/secure-enclave-components.png)

Secure Enclave는 다음과 같은 버전의 iPhone, iPad, Mac, Apple TV, Apple Watch, HomePod 하드웨어에 적용된다.
- iPhone 5s or later
- iPad Air or later
- MacBook Pro computers with Touch Bar (2016 and 2017) that contain the Apple T1 Chip
- Intel-based Mac computers that contain the Apple T2 Security Chip
- Mac computers with Apple silicon
- Apple TV HD or later
- Apple Watch Series 1 or later
- HomePod and HomePod mini

## Secure Enclave Processor

**Secure Enclave Processor**는 Secure Enclave를 위한 주요 컴퓨팅 성능을 제공한다. 확실한 격리를 위해 Secure Enclave Processor는 <span style="color:#3090C7">Secure Enclave 전용</span>으로만 사용된다. 이를 통해 공격 대상 소프트웨어와 동일한 실행 코어를 공유하는 악성 소프트웨어에 의존하는 부 채널 공격을 방지 할 수 있다.  

Secure Enclave Processor는 Apple이 커스텀한 버전의 L4 마이크로 커널을 실행한다. 클럭 및 전력 공격으로부터 보호하는데 도움이 되도록 낮은 클럭 속도에서 효율적으로 작동하도록 설계되었다. A11 및 S4에서 시작된 Secure Enclave Processor에는 <span style="color:#3090C7">Memory Protection Engine</span>과 anti-replay 기능이 있는 <span style="color:#3090C7">암호화 된 메모리</span>, <span style="color:#3090C7">secure boot</span>, <span style="color:#3090C7">전용 난수 생성기</span>, 자체 <span style="color:#3090C7">AES 엔진</span>이 포함 된다.

## Memory Protection Engine
Secure Enclave는 디바이스의 DRAM 메모리의 전용 영역에서 작동한다. 다중 보호 계층은 애플리케이션 프로세서에서 Secure Enclave 보호 메모리를 분리한다.  

디바이스가 시작되면, Secure Enclave Boot ROM은 메모리 보호 엔진에 대한 임의의 임시 메모리 보호 키를 생성한다. Secure Enclave가 전용 메모리 영역에 write 할 때마다 메모리 보호 엔진은 Mac XEX(xor-encrypt-xor) 모드에서 AES를 사용하여 메모리 블록을 암호화하고 메모리에 대한 CMAC(Cipherbased Message Authentication Code) 인증 태그를 계산한다. 메모리 보호 엔진은 <span style="color:#3090C7">암호화 된 메모리</span>와 함께 <span style="color:#3090C7">인증 태그</span>를 저장한다. Secure Enclave가 메모리를 read 할 때 메모리 보호 엔진은 인증 태그를 확인한다. 인증 태그가 일치하면 메모리 보호 엔진이 메모리 블록을 해독한다. 태그가 일치하지 않으면 메모리 보호 엔진이 Secure Enclave에 error 신호를 보낸다. 메모리 인증 오류 후 Secure Enclave는 시스템이 재부팅 될 때까지 <span style="color:#3090C7">요청 수락을 중지</span>한다.  

**Apple A11 및 S4 SoC 부터** 메모리 보호 엔진은 Secure Enclave 메모리에 대한 replay 보호 기능을 추가한다. 보안적으로 중요한 데이터의 replay를 방지하기 위해 메모리 보호 엔진은 인증 태그와 함께 메모리 블록에 대한 <span style="color:#3090C7">nonce</span> 값을 저장한다.  nonce는 CMAC 인증 태그에 대한 추가 tweak으로 사용된다. 모든 메모리 블록에 대한 nonce는 Secure Enclave 내의 전용 SRAM에 있는 무결성 트리를 사용하여 보호 된다. write의 경우, 메모리 보호 엔진은 nonce와 무결성 트리의 각 level을 SRAM까지 업데이트 한다. read의 경우, nonce와 무결성 트리의 각 level을 확인한다. <span style="color:#3090C7">nonce 불일치</span>는 인증 태그 불일치와 유사하게 처리된다.  

**Apple A14, M1 및 이후 SoC에서** 메모리 보호 엔진은 두 개의 임시 메모리 보호 키를 지원한다. 첫 번째는 <span style="color:#3090C7">Secure Enclave의 private 데이터</span>에 사용되며 두 번째는 <span style="color:#3090C7">Secure Neural Engine과 공유되는 데이터</span>에 사용된다.  

메모리 보호 엔진은 Secure Enclave에 대해 인라인으로 투명하게 작동한다. Secure Enclave는 암호화 되지 않은 일반 DRAM처럼 메모리를 읽고 쓰는 반면, Secure Enclave 외부의 관찰자는 암호화되고 인증 된 메모리 버전만 볼 수 있다. 그 결과 성능이나 소프트웨어 복잡성의 절충없이 강력한 메모리 보호가 가능하다.  

## Secure Enclave Boot ROM

Secure Enclave에는 전용 **Secure Enclave Boot ROM**이 포함되어 있다. 애플리케이션 프로세서 Boot ROM과 마찬가지로 Secure Enclave Boot ROM은 Secure Enclave에 대한 <span style="color:#3090C7">하드웨어 신뢰 루트를 설정하는 변경 불가능한 코드</span>이다.  

시스템이 시작되면, iBoot는 전용 메모리 영역을 Secure Enclave에 할당한다. 메모리를 사용하기 전에 Secure Enclave Boot ROM은 메모리 보호 엔진을 초기화 하여 Secure Enclave 보호 메모리의 암호화 보호기법을 제공한다.  

그런 다음 애플리케이션 프로세서는 **sepOS** 이미지를 Secure Enclave Boot ROM으로 보낸다. sepOS 이미지를 Secure Enclave 보호 메모리에 복사한 후 Secure Enclave Boot ROM은 이미지의 암호화 해시 및 서명을 확인하여 sepOS가 디바이스에서 실행되도록 승인되었는지 확인한다. sepOS 이미지가 디바이스에서 실행 되도록 올바르게 서명 된 경우, Secure Enclave Boot ROM은 sepOS에게 제어를 전송한다. 서명이 유효하지 않은 경우, Secure Enclave Boot ROM은 다음 칩 재설정까지 <span style="color:#3090C7">Secure Enclave를 더이상 사용하지 못하게 한다</span>.  

**Apple A10 이상 SoC에서** Secure Enclave Boot ROM은 sepOS의 해시를 전용 레지스터에 저장하여 잠근다(lock). Public Key Accelerator는 운영체제 바인딩(OS-bound) 키에 이 해시를 사용한다.  

## Secure Enclave Boot Monitor

**Apple A13 이상 SoC에서** Secure Enclave에는 부팅 된 sepOS의 해시의 보다 강력한 무결성을 보장하는 **Boot Monitor**가 포함되어 있다.  

시스템 시작시, Secure Enclave Processor의 SCIP(System Coprocessor Integrity Protection) 설정은 <span style="color:#3090C7">Secure Enclave Processor가 Secure Enclave Boot ROM 이외의 코드를 실행하지 못하도록</span>한다. Boot Monitor는 Secure Enclave가 SCIP 설정을 직접 수정하는 것을 방지한다. 로드 된 sepOS를 실행 가능하게 만들기 위해 Secure Enclave Boot ROM은 로드 된 sepOS의 주소와 사이즈를 Boot Monitor에게 request로 보낸다. request를 받으면, Boot Monitor는 Secure Enclave Processor를 재설정하고, 로드 된 sepOS를 해시하고, 로드 된 sepOS의 실행을 허용하도록 SCIP 설정을 업데이트하고, 새로 로드 된 코드 내에서 실행을 시작한다. 시스템이 계속 부팅 됨에 따라 새 코드를 실행할 때 마다 동일한 프로세스가 사용된다. Boot Monitor는 부팅 프로세스의 실행중인 해시를 매번 업데이트 한다. Boot Monitor는 실행중인 해시에 중요한 보안 매개 변수도 포함한다.  

부팅이 완료되면 부팅 모니터가 실행중인 해시를 마무리하고 이를 Public Key Accelerator로 전송하여 운영체제 바인딩(OS-bound) 키에 사용한다. 이 프로세스는 <span style="color:#3090C7">Secure Enclave Boot ROM의 취약점이 있더라도 운영체제 키 바인딩을 우회할 수 없도록 설계</span>되었다.  

## True Random Number Generator

**TRNG(True Random Number Generator)**는 <span style="color:#3090C7">안전한 랜덤 데이터</span>를 생성하는데 사용된다. Secure Enclave는 임의의 암호화 키, 임의의 키 시드, 또는 기타 엔트로피를 생성할 때마다 TRNG를 사용한다. TRNG는 CTR_DRBG(카운터 모드의 블록 암호 기반 알고리즘)로 처리 된 multiple [링 오실레이터](https://en.wikipedia.org/wiki/Ring_oscillator){:target="_blank"}를 기반으로 한다.  

## Root Cryptographic Keys

Secure Enclave에는 고유의 ID(UID) Root Cryptographic Key가 포함되어 있다. UID는 디바이스마다 고유하며 디바이스의 다른 식별자와 관련이 없다.  

무작위로 생성된 UID는 제조시 SoC에 통합된다. **A9 SoC부터**, UID는 제조 과정에서 Secure Enclave TRNG에 의해 생성되고 오로지 Secure Enclave에서 실행되는 스프트웨어 프로세스를 사용하여 퓨즈에 기록 된다. 이 프로세스는 제조 중에 UID가 디바이스 외부에서 보이지 않도록 보호하므로 Apple 또는 공급 업체가 접근하거나 저장할 수 없다.  

sepOS는 UID를 사용하여 디바이스별 secret을 보호한다. UID는 데이터를 특정 디바이스에 암호화 방식으로 연결할 수 있게 해준다. 예를 들어, 파일 시스템을 보호하는 키 계층에는 UID가 포함되어 있으므로 내부 SSD 스토리지가 <span style="color:#3090C7">물리적으로 한 디바이스에서 다른 디바이스로 이동하면 파일에 접근할 수 없다</span>. 기타 보호되는 디바이스 별 secret에는 Touch ID 또는 Face ID 데이터가 포함된다. Mac에서는 AES 엔진에 연결된 완전한 <span style="color:#3090C7">내부 저장소만</span>이 이러한 수준의 암호화를 받는다. 예를 들어, USB를 통해 연결된 외부 저장 장치나 2019 Mac Pro에 추가된 PCIe 기반 저장 장치는 이러한 방식으로 암호화되지 않는다.  

Secure Enclave에는 지정된 SoC를 사용하는 모든 디바이스에 공통적인 디바이스 GID(그룹 ID)도 있다. (예: Apple A14 SoC를 사용하는 모든 디바이스가 동일한 GID를 공유함)  

UID 및 GID는 JTAG(Joint Test Action Group) 또는 기타 디버깅 인터페이스를 통해 사용할 수 없다.  

## Secure Enclave AES Engine

**Secure Enclave AES 엔진**은 AES 암호를 기반으로 대칭 암호화를 수행하는데 사용되는 하드웨어 블록이다. AES 엔진은 타이밍 및 정적 전력 분석(SPA, Satic Power Analysis)을 사용한 정보 유출을 방지하도록 설계 되었다. **A9 SoC부터** AES 엔진에는 동적 전력 분석(DPA, Dynamic Power Analysis) 방지도 포함된다.  

AES 엔진은 하드웨어 및 소프트웨어 키를 지원한다. 하드웨어 키는 Secure Enclave UID 또는 GID에서 파생된다. 이러한 키는 AES 엔진 내에 있으며 sepOS 소프트웨어에서도 볼 수 없다. 소프트웨어는 하드웨어 키를 사용하여 암호화 및 복호화 작업을 요청할 수 있지만 <span style="color:#3090C7">키를 추출 할 수는 없다</span>.  

**Apple A10 및 최신 SoC**의 AES 엔진에는 UID 또는 GID에서 파생된 키를 여러가지로 변화 시키는 lockable 시드 비트가 포함되어 있다. 이를 통해 디바이스의 작업 모드에 따라 데이터 접근을 조정할 수 있다. 예를 들어, lockable 시드 비트는 DFU(Device Firmware Update) 모드에서 부팅 할 때 암호로 보호 된 데이터에 대한 접근을 거부하는데 사용된다. 자세한 내용은 Passcodes and passwords에서 참고할 수 있다.  

## AES Engine

Secure Enclave가 있는 모든 Apple 디바이스에는 NAND(비휘발성) 플래시 스토리지와 메인 시스템 메모리 간의 직접 메모리 접근(DMA, direct memory access)에 내장된 전용 AES256 암호화 엔진(AES Engine)이 있어 파일 암호화를 매우 효율적으로 만든다. **A9 이상의 A 시리즈 프로세서에서** 플래시 스토리지 하위 시스템은 DMA 암호화 엔진을 거친 사용자 데이터를 포함한 메모리에만 접근 권한이 부여된 격리된 bus에 있다. 

부팅시 sepOS는 TRNG를 사용하여 임시 래핑 키를 생성한다. Secure Enclave는 전용 와이어를 사용하여 키를 AES 엔진에 전송하고 Secure Enclave 외부의 소프트웨어에서 접근하지 못하도록한다. 그런 다음 sepOS는 임시 래핑 키를 사용하여 애플리케이션 프로세서 파일 시스템 드라이버에서 사용할 파일 키를 래핑할 수 있다. 파일 시스템 드라이버가 파일을 읽거나 쓸 때 래핑 된 키를 AES 엔진으로 보내어 키의 래핑을 해제한다. AES 엔진은 <span style="color:#3090C7">래핑되지 않은 키를 소프트웨어에 노출하지 않는다</span>.

*Note* : AES 엔진은 Secure Enclave 및 Secure Enclave AES 엔진과는 별개의 구성 요소이지만 작동은 아래와 같이 Secure Enclave와 밀접하게 연결되어 있다.

![aes engine](/img/202102/aes-engine.png)

AES 엔진은 DMA 경로에서 line-speed 암호화를 지원하여 데이터를 스토리지에 쓰고 읽을 때 효율적인 암호화 및 암호 해독을 수행한다.  

## Public Key Accelerator

**PKA(Public Key Accelerator)**는 비대칭 암호화 작업을 수행하는데 사용되는 하드웨어 블록이다. PKA는 RSA, ECC(Elliptic Curve Cryptography) 서명, 암호화 알고리즘을 지원한다. PKA는 SPA 및 DPA와 같은 타이밍 및 부채널 공격을 사용한 정보 유출을 방지하도록 설계되었다.  

PKA는 소프트웨어 및 하드웨어 키를 지원한다. 하드웨어 키는 Secure Enclave UID 또는 GID에서 파생된다. 이러한 키는 PKA 내에 있으며 sepOS 소프트웨어에서도 볼 수 없다.  

**A13 SoC** 부터 PKA의 암호화 구현은 공식 검증 기술을 사용하여 수학적으로 올바른 것으로 입증 되었다.  

**Apple A10 및 이후의 SoC**에서 PKA는 <span style="color:#3090C7">SCP(Sealed Key Protection)</span>라고도 하는 OS 바인딩 키를 지원한다. 이러한 키는 디바이스의 UID와 디바이스에서 실행되는 sepOS의 해시의 조합을 사용하여 생성 된다. 해시는 Secure Enclave Boot ROM 또는 **Apple A13 이상 SoC**의 Secure Enclave Monitor에서 제공된다. 이러한 키는 특정 Apple 서비스에 대한 요청을 생성 할 때 <span style="color:#3090C7">sepOS 버전 확인</span>에 사용되며 사용자 승인 없이 시스템에 중요한 변경이 있을 경우 자료에 대한 접근을 방지하여 <span style="color:#3090C7">패스코드로 보호된 데이터의 보안을 향상</span>시키는데도 사용된다.  

## Secure nonvolatile storage

Secure Enclave에는 전용 **보안 비휘발성 스토리지**가 장착되어 있다. 보안 비휘발성 스토리지는 전용 I2C bus를 사용하여 Secure Enclave에 연결되므로 <span style="color:#3090C7">Secure Enclave에서만 접근</span>할 수 있다. 모든 사용자 데이터 암호화 키는 Secure Enclave 비휘발성 저장소에 저장된 엔트로피에 뿌리를 두고 있다.  

**A12, S4 및 이후 SoC**가 있는 디바이스에서 Secure Enclave는 엔트로피 저장을 위해 **Secure Storage Component**와 쌍을 이룬다. Secure Storage Component는 변경 불가능한 ROM 코드, 하드웨어 난수 생성기, 디바이스 별 고유 암호화 키, 암호화 엔진 및 물리적 변조 감지와 함께 자체적으로 설계되었다. Secure Enclave 및 Secure Storage Component는 <span style="color:#3090C7">엔트로피에 대한 독점 접근</span>을 제공하는 암호화되고 인증 된 프로토콜을 사용하여 통신한다.  

**2020년 가을 이후에 출시 된 디바이스**에는 2세대 Secure Storage Component가 장착되어 있다. 2세대 보안 스토리지 구성 요소에는 **counter lockbox**가 추가되었다. 각 counter lockbox는 128비트의 <span style="color:#3090C7">솔트</span>, 128비트의 <span style="color:#3090C7">패스코드 검증기</span>, 8비트의 <span style="color:#3090C7">counter</span>, 8비트의 <span style="color:#3090C7">최대 시도 횟수</span>를 저장한다. counter lockbox에 대한 접근은 암호화되고 인증 된 프로토콜을 통해 이루어진다.  

counter lockbox는 패스코드로 보호 된 사용자 데이터를 잠금 해제 하는데 필요한 <span style="color:#3090C7">엔트로피</span>를 보유한다. 사용자 데이터에 접근 하려면 쌍을 이루는 Secure Enclave는 사용자의 패스코드와 Secure Enclave의 UID로부터 올바른 패스코드 엔트로피 값을 가져와야 한다. 페어링 된 Secure Enclave가 아닌 소스에서 전송 된 잠금 해제 시도로는 사용자의 패스코드를 알 수 없다. 패스코드 시도 제한을 초과하면 (예: iPhone에서 10회 시도) 패스코드로 보호 된 데이터가 Secure Storage Component에 의해 <span style="color:#3090C7">완전히 지워진다</span>.  

counter lockbox를 생성하기 위해 Secure Enclave는 Secure Storage Component에 패스코드 엔트로피 값과 최대 시도 횟수 값을 보낸다. Secure Storage Component는 난수 생성기를 사용하여 솔트 값을 생성한다. 그런 다음 제공된 패스코드 엔트로피, secure storage component의 고유 암호화 키, 솔트 값으로부터 패스코드 검증기 값과 lockbox 엔트로피 값을 파생한다. Secure Storage Component는 count는 0으로, 제공된 최대 시도 횟수 값, 파생 된 패스코드 검증기 값, 솔트 값으로 counter lockbox를 초기화 한다. 그런 다음 Secure Storage Component는 생성 된 lockbox 엔트로피 값을 Secure Enclave에 반환한다.

이후에 counter lockbox에서 lockbox 엔트로피 값을 찾기 위해 Secure Enclave는 Secure Storage Component에 <span style="color:#3090C7">패스코드 엔트로피</span>를 보낸다. Secure Storage Component는 먼저 lockbox의 카운터를 증가시킨다. 증가 된 카운터가 최대 시도 값을 초과하는 경우 Secure Storage Component는 counter lockbox를 완전히 지운다. 최대 시도 횟수에 도달하지 않은 경우 Secure Storage Component는 counter lockbox를 생성하는데 사용된 것과 동일한 알고리즘을 사용하여 <span style="color:#3090C7">패스코드 검증기 값</span>과 <span style="color:#3090C7">lockbox 엔트로피 값</span>을 파생하려고 시도한다. 파생 된 패스코드 검증기 값이 저장된 패스코드 검증기 값과 일치하면 Secure Storage Component는 lockbox 엔트로피 값을 Secure Enclave로 반환하고 카운터를 0으로 재설정한다.  

패스코드로 보호 된 데이터에 접근하는데 사용되는 키는 counter lockbox에 저장된 엔트로피에 뿌리를 두고 있다. 자세한 정보는 Data Protection overview를 참고.  

보안 비휘발성 스토리지는 Secure Enclave의 모든 <span style="color:#3090C7">anti-replay 서비스</span>에 사용된다. Secure Enclave의 anti-replay 서비스는 다음과 같은 anti-replay 바운더리를 표시하는 이벤트에 대한 데이터를 취소하는데 사용된다. (이에 국한된 것은 아님)
- 패스코드 변경
- Touch ID 또는 Face ID 활성화/비활성화
- Touch ID 지문 또는 Face ID 얼굴 추가/제거
- Touch ID 도는 Face ID 재설정
- Apple Pay 카드 추가/제거
- 모든 컨텐츠 및 설정 지우기

**Secure Storage Component가 없는 아키텍처**에서는 EEPROM(electrically erasable programmable read-only memory)을 사용하여 Secure Enclave를 위한 보안 스토리지 서비스를 제공한다. Secure Storage Component와 마찬가지로 EEPROM은 Secure Enclave에서만 연결 및 접근 할 수 있지만 전용 하드웨어 보안 기능을 포함하지 않으며 엔트로피(물리적 연결 특성 제외) 또는 counter lockbox 기능에 대한 독점 접근을 보장하지 않는다.  

## Secure Neural Engine

**Face ID가 있는 디바이스**에서 **Secure Neural Engine**은 2D 이미지와 depth 맵을 사용자 얼굴의 수학적 표현으로 변환한다.  

**A11~A13 SoC**에서 Secure Neural Engine은 Secure Enclave에 통합된다. Secure Neural Engine은 고성능을 위해 직접 메모리 접근(DMA)을 사용한다. sepOS 커널의 제어 하에 있는 입출력 메모리 관리 장치(IOMMU, input-output memory management unit)는 인증 된 메모리 영역에 대한 이러한 직접 접근을 제한한다.  

**A14 및 M1** 부터 Secure Neural Engine은 애플리케이션 프로세서의 Neural Engine에서 보안 모드로 구현된다. 전용 하드웨어 보안 컨트롤러는 애플리케이션 프로세서와 Secure Enclave의 task를 전환하며, 각 전환에서 Neural Engine 상태를 재설정해 Face ID 데이터를 안전하게 유지한다. 전용 엔진은 메모리 암호화, 인증, 접근 제어를 적용한다. 동시에, 별도의 암호화 키와 메모리 범위를 사용하여 Secure Neural Engine을 승인 된 메모리 영역으로 제한한다.  

## Power and clock monitors

모든 전자 장치는 제한된 전압 및 주파수 범위 내에서 작동하도록 설계되었다. 이 범위 밖에서 작동하면 전자 장치가 오작동 할 수 있으며 보안 제어를 우회 할 수 있다. 전압 및 주파수가 안전한 범위를 유지하도록 하기 위해 Secure Enclave는 **모니터링 회로**와 함께 설계되었다. 이러한 모니터링 회로는 Secure Enclave 보다 훨씬 더 큰 작동 범위를 갖도록 설계되었다. 모니터가 잘못된 작동 지점을 감지하면 Secure Enclave의 clock이 자동으로 중지되고 다음 SoC 재설정까지 다시 시작되지 않는다.  

## Secure Enclave feature summary

*Note* : 2020년 가을에 처음 출시 된 A12, A13, S4, S5 제품에는 2세대 Secure Storage Component가 있다. 이러한 SoC를 기반으로 하는 이전 제품에는 1 세대 Secure Storage Component가 있다.  

|SoC|Memory Protection<br>Engine|Secure Storage|AES Engine|PKA|
|:-:|:-:|:-:|:-:|:-:|
|A8|암호화, 인증|EEPROM|⭕|❌|
|A9|암호화, 인증|EEPROM|DPA 방지|⭕|
|A10|암호화, 인증,<br>replay 방지|EEPROM|DPA 방지,<br>lockable 시드 비트|OS-bound 키|
|A11|암호화, 인증,<br>replay 방지|EEPROM|DPA 방지,<br>lockable 시드 비트|OS-bound 키|
|A12<br>(2020 가을 이전)|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>1 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키|
|A12<br>(2020 가을 이후)|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>2 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키|
|A13<br>(2020 가을 이전)|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>1 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키,<br>Boot Monitor|
|A13<br>(2020 가을 이후)|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>2 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키,<br>Boot Monitor|
|A14|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>2 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키,<br>Boot Monitor|
|S3|암호화, 인증|EEPROM|DPA 방지,<br>lockable 시드 비트|⭕|
|S4|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>1 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키|
|S5<br>(2020 가을 이전)|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>1 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키|
|S5<br>(2020 가을 이후)|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>2 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키|
|S6|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>2 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키|
|T2|암호화, 인증|EEPROM|DPA 방지,<br>lockable 시드 비트|OS-bound 키|
|M1|암호화, 인증,<br>replay 방지|Secure Storage<br>Component<br>2 세대|DPA 방지,<br>lockable 시드 비트|OS-bound 키,<br>Boot Monitor|  

# 요약

기회가 된다면 나중에 이쁘게 정리 할 것입니다..

![hardware security and biometrics revision summary](/img/202102/part-1-summary.png)

**애플리케이션 프로세서**에서 sepOS 이미지를 **Secure Enclave Boot ROM**에 전달하면 Secure Enclave Boot ROM은 이미지의 해시와 서명을 확인하여 승인된 것인지 검증합니다. 검증 되었다면 로드 된 sepOS의 주소와 사이즈를 **Secure Enclave Boot Monitor**에게 전송하고, Secure Enclave Boot Monitor는 Secure Enclave 프로세서를 재설정하고 **sepOS**의 실행을 허용합니다.  

**Secure Enclave AES Engine**에는 UID 또는 GID에서 파생 된 하드웨어 키를 지원하며, sepOS에서는 이 키를 이용한 암호화 및 복호화를 요청 할 수 있지만 키를 추출 할 수는 없습니다.  

부팅시 sepOS는 **TRNG**를 사용하여 임시 래핑 키를 생성하고, 이를 **AES 엔진**에 전송합니다. sepOS는 임시 래핑 키를 사용하여 파일 시스템 드라이버에서 사용할 파일 키를 래핑 할 수 있고, 파일 시스템 드라이버는 파일을 읽거나 쓸 때 래핑 된 키를 AES 엔진으로 보내어 키의 래핑을 해제합니다.  

**PKA**는 비대칭 암호를 수행하고 SCP라고 하는 OS 바인딩 키를 지원하며, 이 키는 UID와 sepOS 해시를 이용하여 생성 됩니다. 이 키는 특정 서비스 요청 시 sepOS 버전을 확인하는데 사용되며 사용자 승인 없이 중요한 변경 사항이 발생했을 경우 자료에 대한 접급을 방지합니다.  

Secure Enclave 전용 **비휘발성 스토리지**는 Secure Enclave에서만 접근 할 수 있으며, 모든 사용자 데이터 암호화 키는 여기에 저장된 엔트로피에 뿌리를 두고 있습니다. 2세대 스토리지 부터 **counter lockbox**가 추가 되었습니다. Secure Enclave는 사용자의 패스코드와 UID로부터 패스코드 엔트로피 값을 계산하여 전송하고, **Secure Storage Components**는 패스코드 엔트로피 값 등을 이용하여 lockbox를 생성합니다. lockbox에는 lockbox 엔트로피 값과 패스코드 검증기 값이 생성 됩니다. 생성된 lockbox의 패스코드 검증기 값이 미리 저장된 올바른 값과 일치하는지 확인하여 패스코드를 검증하게 됩니다. 일치한다면 lockbox 엔트로피 값을 반환합니다.  

Hardware microphone disconnect에 대한 내용도 다이어그램과 함께 약간의 내용이 추가 되었지만 읽지 않았습니다.  

# Related Post
- [Apple Platform Security Revision](/2021-02-26-apple-platform-security-revision/){:target="_blank"}
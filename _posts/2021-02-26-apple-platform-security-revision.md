---
layout: post
title: "Apple Platform Security Revision"
image: /img/apple-security.png
tags: [iOS, macOS, Apple]
---

# Apple Platform Security 2021 FEB ver.
애플은 매년 자사의 보안 기능과 기술을 설명하는 **Apple Platform Security** 보고서를 공개하고 있습니다. 2021년 2월 최신판이 드디어 공개 됐는데요, 폐쇄적이었던 애플이 이번 보고서에서는 핵심 보안 기술들에 대한 상세한 설명과 그림을 추가 했습니다. 공개 된 버전에는 최신 운영체제 버전과 M1 칩셋에 대한 설명들이 추가되었고, 향후 macOS 버전에서 커널 익스텐션인 kexts에 대한 지원을 중단할 것이라는 것도 알 수 있습니다.
>  ... This
is why developers are being strongly encouraged to adopt system extensions **before kext
support is removed from macOS for future Mac computers with Apple silicon.** (Reduced Security policy 내용 중 일부)

**Document revision history**에 의하면 February 2021 버전의 변경 사항은 다음과 같습니다.  
- iOS 14.3 / iPadOS 14.3 / macOS 11.1 / tvOS 14.3 / watchOS 7.2 에 대한 업데이트
- 추가된 Topic :
  - Memory safe iBoot implementation
  - Boot process for a Mac with Apple silicon
  - Boot modes for a Mac with Apple silicon
  - Startup Disk security policy control for a Mac with Apple silicon
  - LocalPolicy signing-ky creaation and management
  - Contents of a LocalPolicy file for a Mac with Apple
silicon
  - Signed system volume security in macOS
  - Apple Security Research Device
  - Password Monitoring
  - IPv6 security
  - Car keys security in iOS
- 업데이트 된 Topic :
  - Secure Enclave
  - Hardware microphone disconnect
  - recoveryOS and diagnostics environments for an
Intel-based Mac
  - Direct memory access protections for Mac
computers
  - Kernel extensions in macOS
  - System Integrity Protection
  - System security for watchOS
  - Managing FileVault in macOS
  - App access to saved passwords
  - Password security recommendations
  - Apple Cash security in iOS, iPadOS, and watchOS
  - Secure Business Chat using the Messages app
  - Wi-Fi privacy
  - Activation Lock security
  - Apple Configurator 2 security

Revision에 표시 된 내용은 목차명이 바뀌고 내용 변화가 별로 없는 것들도 포함 되어 있어서 이런 것들은 제외하고 따로 비교해서 정리해본 내용은 아래와 같습니다.

![Apple Platform Security revision](/img/202102/Apple Platform Security revision v2.png)

페이지 수만 비교해도 157 페이지에서 196 페이지로 늘어났습니다. <font color="#3090C7">Apple SoC security</font>에 대한 내용과 <font color="#3090C7">Apple Security Research Device</font> / <font color="#3090C7">Car key security in iOS</font> / <font color="#3090C7">IPv6 security</font>는 이전에 없던 topic들 입니다. 새로 추가 된 내용도 많은데 특히 <font color="#3090C7">Secure Enclave</font>와 <font color="#3090C7">Secure boot</font>의 내용이 눈에 띄게 상세해졌고, <font color="#3090C7">Hardware microphone diconnect</font>에서는 다이어그램이 추가되었습니다. 아직 문서를 다 읽어보진 않았지만 <font color="#3090C7">Encryption in macOS</font> 목차가 <font color="#3090C7">FileVault</font>로 바뀌면서 몇 가지 삭제 된 내용들이 있었습니다. 또, <font color="#3090C7">Apple security and privacy certifications</font>에 대한 내용이 문서 마지막에 포함되었었는데 이번부터 **Security Certifications and Compliance Center**라는 50페이지에 달하는 별도의 문서로 새로 작성되었습니다.  

앞으로 심심할 때마다 읽어보면서 주요 내용을 정리해볼 계획입니다.

# REF.
- [Apple Platform Security](https://manuals.info.apple.com/MANUALS/1000/MA1902/ko_KR/apple-platform-security-guide-kh.pdf){:target="_blank"} - 이 링크는 새로운 보고서가 나오면 최신 보고서로 변경 됩니다.

# Related Post
- [[APS] Hardware security and biometrics](/2021-02-26-aps-hardware/){:target="_blank"}
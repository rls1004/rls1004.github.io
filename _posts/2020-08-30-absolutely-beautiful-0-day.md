---
layout: post
title: "Absolutely beautiful 0-day"
image: /img/apple.PNG
tags: [iOS, 0-day, sandbox, XML]
---

Siguza가 올린 이 트윗을 보고 여러가지로 놀랐습니다. iOS 연구를 하면서 Siguza를 굉장히 많이 들어봤는데 그런 Siguza의 첫 번째 0-day라는 점에서 흥미로웠고, "absolute best"라고 표현한 것도 관심이 갈 수밖에 없었습니다.  

Siguza의 블로그에도 [해당 취약점에 대한 글](https://siguza.github.io/psychicpaper/
)이 올라왔는데 친절한 설명이 되어 있습니다. 이 포스팅은 이 글을 보고 정리한 내용입니다.  

<center>
<img src="/img/202008/psychic_paper_01.png" width='80%'/>
</center>

Siguza는 해당 취약점에 "Psychic Paper"라는 이름을 붙였습니다. 닥터 후가 맨날 들고 다니는 종이라는데 닥터 후를 안봐서 잘 모르겠네요.  

## Overview

이 익스플로잇을 Sandbox Escape를 위한 가장 우아한 익스플로잇이라고 표현하고 있습니다.  

<center>
<img src="/img/202008/psychic_paper_00.png" width='60%'/>
</center>

공격자의 프로세스가 Security check를 모두 통과할 수 있고, 또한 다른 프로세스들이 바라보기에는 광범위한 자격증명을 가졌다고 인식하게 만들 수 있습니다.  

## Background

이 취약점은 어려운 배경 지식 없이도 이해하기 쉽습니다.  
XML Parser에서 발생하기 취약점이기 때문에 XML에 대해서 알아야 합니다.  
  
**+ XML 문법?**  
- `<tag></tag>` 또는 `<tag/>`
- tag 안에 `a="b"` 와 같이 속성을 가질 수 있음
- `<?...?>` : Processing instructions
- `<!DOCTYPE …>` : Document type declarations
- `<!-- -->` : 주석
- Full XML SPEC : [https://www.w3.org/TR/xml/](https://www.w3.org/TR/xml/)  
  
**+ XML은 파싱하기 어렵다!**  

<a href="https://xkcd.com/1144/">
<img src="https://imgs.xkcd.com/comics/tags.png"/>
</a>

- `<mis>matched</tags>`
- `<attributes that=“are never closed>`
- `<tags that are never closed`
- `<!>`
  
**+ Property list, Plist?**  
Property list(Plist)는 serialized data를 저장하기 위한 범용 포맷입니다.  
Plist는 다양한 형태로 작성되지만 Apple에서는 bplist 바이너리 형식과 XML 기반 형식을 사용합니다.  
<br>
<center>
<img src="/img/202008/psychic_paper_02.png" width='80%'/>
</center>

Plist 파일은 iOS와 macOS에서 `configuration files`, `package properties`, `코드 서명`에 사용 됩니다.  

<br>
<center>
<img src="/img/202008/psychic_paper_03.png" width='80%'/>
</center>
  
여기서 코드 서명에 주목합시다. iOS에서 바이너리를 실행하려면 `AppleMobileFileIntegrity(AMFI)`라는 커널 extension에 유효한 코드 서명이 있어야 합니다. 그렇지 않은 바이너리는 즉시 종료됩니다.  

<br>
<center>
<img src="/img/202008/psychic_paper_04.png" width='80%'/>
</center>

코드 서명은 `hashsum`으로 식별됩니다. hashsum은 다음 두 가지 방법 중 하나로 유효성 검사를 할 수 있습니다.  
1. hash는 커널에 미리 알려질 수 있다 (`ad-hoc`). iOS 시스템 앱 및 데몬에 사용되며 커널에서 직접 `알려진 hash 모음`과 비교하여 간단히 확인한다.
2. `유효한 코드 서명 인증서`로 서명해야 한다. 이는 모든 third party 앱에 사용되며, 이 시나리오에서는 AMFI가 userland 데몬 `amfid`를 호출하여 필요한 모든 검사를 실행한다.  

이 중 후자에 사용되는 코드 서명 인증서는 두 가지 형태로 제공 됩니다.
1. `App Store 인증서`. 이것은 Apple 자체에 의해서만 작성되며 이러한 방식으로 서명하려면 앱이 App Store 검사를 통과해야 한다.
2. `개발자 인증서`. 무료 "7일" 인증서, "일반" 개발자 인증서, 엔터프라이즈 배포 인증서 일 수 있다.

<br>
<center>
<img src="/img/202008/psychic_paper_05.png" width='80%'/>
</center>

개발자 인증서를 사용할 경우, 해당 앱에는 Xcode가 가져올 수 있는 파일인 `provisioning profile`이 필요합니다. (Payload/Your.app/embedded.mobileprovision)  
이 파일은 Apple이 자체 서명한 것으로, 유효 기간, 디바이스 목록, 유효한 개잘자 계정, 앱에 적용해야하는 모든 제한 사항을 지정하고 있습니다.  

**+ Example for valid XML plist and provisioning profiles!**  
<center>
<div style="width:48%; display: inline-block;vertical-align: middle;">
<img src="/img/202008/psychic_paper_06.png"/>
<figcaption>XML plist</figcaption></div>
<div style="width:48%; display: inline-block;vertical-align: middle;">
<img src="/img/202008/psychic_paper_07.png"/>
<figcaption>provisioning profiles</figcaption></div>
</center>
<br>

**+ App Sandboxing?**  
이제 앱 샌드박싱에 대해서 간단히 살펴봅시다.  

<center>
<img src="/img/202008/psychic_paper_08.png" width='80%'/>
</center>

표준 UNIX 환경에서는 UID를 통해 보안 경계를 검사합니다. 한 UID의 프로세스는 다른 UID의 리소스에 접근할 수 없으며 `privileged`로 간주되는 모든 리소스에 접근하기 위해서는 `UID 0(root)`가 필요합니다.  
iOS와 macOS는 UID도 사용하지만 `entitlements`라는 개념도 도입했습니다.
entitlements는 바이너리에 적용할 properties와 privileges의 목록입니다.
entitlements가 존재하는 경우, XML plist 파일 형식으로 바이너리의 코드 서명에 포함되며 다음과 같은 모습입니다.
```xml
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>task_for_pid-allow</key>
    <true/>
</dict>
</plist>
```
이는 바이너리가 `task_for_pid-allow` entitlements를 보유함을 의미합니다. iOS에서는 일반적으로 mach trap인 task_for_pid()를 허용하지 않습니다. 하지만 이 entitlements를 보유한 경우에는 이를 호출할 수 있습니다.  
이러한 entitlements는 iOS와 macOS 전체에서 검사되며 수천가지가 넘는 것들이 있습니다. (조나단 레빈이 이를 검색할 수 있도록 모든 항목의 [카탈로그](http://newosxbook.com/ent.jl)를 작성 함)  

<center>
<div style="width:40%; display: inline-block;vertical-align: top;margin-right:100px">
<img src="/img/202008/psychic_paper_09.png"/>
</div>
<div style="width:40%; display: inline-block;vertical-align: top;">
<img src="/img/202008/psychic_paper_10.png"/>
</div>
</center>

중요한 것은 iOS의 모든 third party 앱이 가능한한 적은 수의 파일, 서비스 및 커널 API에 접근할 수 있는 컨테이너 형 환경에 배치되는데 entitlements를 사용하여 해당 컨테이너에 구멍을 뚫거나 컨테이너를 제거할 수 있게 됩니다.  
<br>

## The Bug

**IOTaskHasEntitlement(task_t task, const char * entitlement)**
<center>
<img src="/img/202008/psychic_paper_11.png" width='70%'/>
<figcaption>iokit/bsddev/IOKitBSDInit.cpp
</figcaption>
</center>

특정 entitlements를 가지고 있는지 확인하는 함수입니다.  
`copyClientEntitlements(..)` 호출을 통해 어떤 객체를 가져올 수 있는지 확인하고 true 혹은 false를 리턴하게 됩니다.  

**IOUserClient::copyClientEntitlements(task_t task, const char * entitlement)**
<center>
<img src="/img/202008/psychic_paper_12.png" width='70%'/>
<figcaption>iokit/Kernel/IOUserClient.cpp
</figcaption>
</center>

여기서는 같은 이름의 `copyClientEntitlements(..)`를 호출하여 task가 가지고 있는 모든 entitlements 객체를 가져오고, 그 중에서 확인하려는 entitlement에 대한 객체가 있는지 확인합니다.  

**IOUserClient::copyClientEntitlements(task_t task)**  
<center>
<img src="/img/202008/psychic_paper_13.png" width='70%'/>
<figcaption>iokit/Kernel/IOUserClient.cpp
</figcaption>
</center>

XML로 작성 된 plist를 파싱하고 unserialize 하여 entitlements 객체를 생성합니다.  

**parser difference**  
이 버그의 핵심은 parser difference 입니다.  

<center>
<img src="/img/202008/psychic_paper_14.png" width='70%'/>
</center>

iOS에는 특이하게도 최소 4개의 plist parser가 존재합니다. 하지만 이 4개의 parser가 동일한 plist에 대해서 항상 동일한 결과를 반환하지 못 합니다. XML을 올바르게 파싱하기가 매우 어렵기 때문에 유효한 XML에 대해서는 모든 parser가 올바른 결과를 반환하지만, 잘못된 XML 형식에 대해서는 동일하지 않은 데이터를 반환할 수 있습니다.  

이것이 <font color='SALMON'>parser difference</font> 입니다.  
parser difference에 의해 parser 마다 서로 다른 결과를 반환할 수 있고, 이것이 시스템에 걸친 설계 결함이 됩니다.  

<center>
<img src="/img/202008/psychic_paper_15.png" width='80%'/>
</center>

중요한 내용을 정리해봅시다.  
entitlements를 분석하는데는 4개의 parser가 **모두** 사용됩니다.  
그 중 amfid가 사용하는 parser는 CoreFoundation의 **CFPropertyListCreateWithData** 입니다.  
**parser difference**로 인해 4개의 parser가 동일한 파일에 대해 항상 동일한 데이터를 반환하지 않습니다.  

※ OSUnserializeXML과 IOCFUnserialize가 항상 동일한 데이터를 반환했으므로 이 게시물의 나머지 부분에서는 이를 동등한 것으로 간주함  
※ 간결함을 위해 OSUnserializeXML/IOCFUnserialize는 "IOKit", CFPropertyListCreateWithData는 "CF", xpc_create_from_plist는 "XPC"라고 함

## The exploit

이 버그를 익스플로잇 하는 가장 우아한 방법이 [트위터](https://twitter.com/s1guza/status/1255641164885131268)에 올라왔습니다.  
<center>
<img src="/img/202008/psychic_paper_16.png" width='80%'/>
</center>

여기서 흥미로운 토큰은 `<!--->`과 `<!-->`이며, XML 스펙에 따르면 유효한 XML 토큰이 아닙니다.  
그럼에도 불구하고 IOKit, CF, XPC는 모두 위의 XML/plist를 수용하는데 정확히 같은 방식으로 파싱 하는 것이 아닙니다.  

[plparser](https://github.com/Siguza/psychicpaper)를 사용하면 다음과 같은 결과를 확인할 수 있습니다.  
<center>
<img src="/img/202008/psychic_paper_17.png" width='80%'/>
</center>

amfid가 CF를 사용하여 provisioning profiles에서 허용하지 않는 권한이 포함되어 있는지 여부를 확인하면 아무런 entitlements를 가지고 있지 않은 것으로 확인됩니다.  
하지만 커널이나 데몬에서 특정 작업에 대한 권한을 확인하려고 할 때는 모든 권한을 가지고 있다고 확인하게 됩니다.  

**Code: IOKit parser**  
<center>
<img src="/img/202008/psychic_paper_18.png" width='70%'/>
</center>

`<!--->` 토큰을 분석할 때, `nextChar()`를 통해 문자를 하나씩 가져오며 `<!--` 까지 가져왔을 때 주석의 시작으로 인식합니다.  

**Code: CF parser**
<center>
<img src="/img/202008/psychic_paper_19.png" width='70%'/>
</center>

CF에서는 `!`를 만났을 때, 그 뒤에 `--`가 따라온다면 포인터를 2만큼 증가시킵니다. 이렇게 되면 `--` 중 두 번째 문자를 다시 가리키게 되어 중복이 발생합니다. `<!--->`에서 `-` 하나가 더 인식되어서 `<!--`과 `-->`이 되기 때문에 주석의 시작과 끝을 모두 인식하게 됩니다.  

**PoC: IOKit parser**  
<center>
<img src="/img/202008/psychic_paper_20.png" width='70%'/>
</center>

IOKit에 의하면 초록색 네모로 묶여진 `-><1`라는 문자가 주석으로 인식됩니다.  

**PoC: CF parser**
<center>
<img src="/img/202008/psychic_paper_21.png" width='70%'/>
</center>

CF는 `<!--->`에서 주석의 시작과 끝이 모두 포함된 것으로 인식했기 때문에 그 다음에 나타나는 `<!--`를 주석의 시작으로, 마지막에 나타나는 `-->`를 주석의 끝으로 인식합니다.  
그 사이에 들어간 모든 문자들은 주석이 됩니다.  

정리 해보면 CF를 사용하는 amfid에게는 드러나지 않고 IOKit과 XPC에서는 파싱 되는 entitlements를 작성할 수 있습니다. 이를 통해 provisioning profiles에서 허용하지 않는 작업을 몰래 숨겨놓을 수 있습니다.  

## Conclusion

sandbox escape를 위해 필요한 entitlements 입니다.
- com.apple.private.security.no-container
  - sandbox가 프로세스에 어떤 profile도 적용하지 못함
  - mobile이 액세스 할 수 있는 모든 위치에서 읽고 쓸 수 있음
  - 수 많은 시스템 콜을 실행하고, 이전에는 허용되지 않았던 수 백 개의 드라이버 및 사용자 영역 서비스와 통신 가능
  - 사용자 데이터에 관한 보안은 더 이상 존재하지 않음
- task_for_pid-allow
  - mobile로 실행되는 모든 프로세스의 task port 조회 가능
  - 이를 사용하여 프로세스 메모리를 읽고 쓰거나 쓰레드 레지스터 상태를 직접 가져오거나 설정 가능
- platform-application
  - Apple 바이너리의 task port에 대해서는 위의 작업을 수행할 수 없지만 바이너리를 Apple 바이너리로 인식하게 하여 우회 가능
  - (UPDATE 10. May 2020 : iOS 11에서 패치 되어 복잡해짐)

**Patch**  
iOS 13.4의 [패치](https://support.apple.com/ko-kr/HT211102) 내용입니다.  
<center>
<img src="/img/202008/psychic_paper_22.png" width='70%'/>
</center>

Linux Henze의 버그 리포트로 인해 entitlements 검사가 다소 강화 되었습니다. 버그의 자세한 내용을 알 순 없지만 parser dirrerence를 악용하여 amfid를 우회하는 bplist과 관련 있다고 추측 되며, psychic paper는 13.4 패치에서는 살아남았지만 13.5 beta 3에서 패치 되었습니다.  

iOS 13.5 beta 3에서는 AMFI.kext와 amfid에 `AMFIUnserializeXML`이라는 새로운 함수가 도입 되었습니다. 이 함수는 `OSUnserializeXML`과 `CFPropertyListCreateWithData`의 결과를 비교하여 동일한지 확인하는 역할을 합니다. 잘못된 토큰을 이용해 어떤 것이든 몰래 집어넣으려고 하면 AMFI에서 프로세스를 중단시키고 syslog에 보고합니다.  

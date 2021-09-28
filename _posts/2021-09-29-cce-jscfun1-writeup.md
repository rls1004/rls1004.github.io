---
layout: post
title: "[CCE 2021] jscfun1 Write-up"
image: /img/cce_2021.png
tags: [CTF, write-up, misc, javascript]
---

# jscfun1

jsc 바이너리, 도커 파일, 관련 라이브러리, 스크립트 파일 등이 주어집니다.  
jsfun1.py 는 서버에서 동작하는 스크립트 입니다.  

## 문제 : jsfun1.py

```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-

import os, sys
import random, tempfile

jsCodeTemplate = """
const getDescriptors = Object.getOwnPropertyDescriptors;

{userCode}

;

;(()=> {\{
    return function(obj) {\{
    let flag = "{flag}";
    if (typeof obj != 'object')
        return false;
    
    for (key in getDescriptors(obj))
        return false;

    for (key in obj)
        return false;

    if (obj.get != 'function() {\{ return a; }\}')
        return false;

    if (obj.set != 'function(val) {\{ a = 17171717; }\}')
        return false;
    
    val = {randNumber};
    obj.set(val);

    if (obj.get() != val)
        return false;

    if ((!('flag' in obj)) && obj.flag == 27272727)
        print(flag);
    
    return false;
    }\}
}\})()(myObj);
"""

if __name__ == "__main__":
    with open("/flag", "r") as f:
        flag = f.read().strip()

    print("input the code (code has to end with <EOF>)")

    userCode = ""
    while "<EOF>" not in userCode:
        userCode += sys.stdin.read(1)

    userCode = userCode.split("<EOF>")[0]
    if len(userCode) > 280:
        exit(0)

    userJsCode = jsCodeTemplate.format(userCode=userCode, randNumber=random.getrandbits(64), flag=flag)
    with tempfile.NamedTemporaryFile(delete=False) as fp:
        fp.write(bytes(userJsCode, 'utf8'))
        fp.flush()
        tempName = fp.name

    fd = os.popen("/jsc " + tempName)
    print(fd.read())
    os.remove(tempName)
```

위 파이썬 코드에서 요구하는 몇 가지 조건을 만족하는 myObj 객체를 Javascript 코드로 작성하는 문제입니다. 조건은 다음과 같습니다.  

1. 객체의 타입은 `object` 이어야 함
2. `Object.getOwnPropertyDescriptors`는 객체의 모든 프로퍼티에 대한 디스크립터를 반환하는데 이 값이 없어야함
3. 열거 가능한 프로퍼티를 갖지 않아야 함 (`key in obj`)
4. obj.get과 obj.set이 각각 문자열 `function() { return a; }`, `function(val) { a = 17171717; }`와 일치해야 함
5. obj.set(val)을 통해 설정한 값을 obj.get()을 통해 가져올 수 있어야 함
6. `'flag'`라는 열거 가능한 속성을 포함하지 않으면서, obj.flag의 값이 `27272727` 이어야 함

## 풀이

Proxy 를 이용해서 myObj 객체를 생성함으로써 문제를 해결할 수 있습니다.  

Object의 프로퍼티 정보에 접근 할 때는 내부적으로 `ownKeys`가 호출됩니다.  
이를 이용해서 Proxy 객체의 핸들러에 `ownKeys` 트랩을 정의하여 빈 배열을 반환하도록 작성합니다.  

비슷한 원리로 obj.set, obj.get 프로퍼티에 접근 할 때는 `get` 트랩을 이용해서 조건에 맞는 문자열을 각각 반환하게 합니다. (get, set 메서드의 toString을 정의 해도 됨)  

obj.set(val)에서는 변수에 값을 설정하고 obj.get()에서는 변수의 값을 반환 하도록 함수를 각각 작성 합니다.  

`ownKeys` 트랩을 이용했기 때문에 obj 객체는 열거 가능한 'flag' 프로퍼티를 갖지 않습니다. 하지만 flag 프로퍼티에 접근 할 때 `get` 트랩이 호출되어 원하는 값을 반환하게 작성할 수 있습니다.  

```javascript
a=1;
c=1;
obj={};
s=(v)=>{a=v};
g=()=>a;
myObj=new Proxy(obj, {
    ownKeys: function(target) {
        return [];
        },
    get: function(t, p) {
        if (p=='flag')
            return 27272727;
        if (c==2) {
            c+=1;
            return s;
        }
        if (c==3)
            return g;
        if (p=='get')
            return 'function() { return a; }';
        c+=1;
        return 'function(val) { a = 17171717; }'
    }
})
```

cce2021{48f4f6690015c3990b78237e0491f334a3c58784224c3c9dbf8aa8f9b1e1a4fa2c99a4557ed9719e1975b2371e3422943fec5adc72456e3878f9e9}
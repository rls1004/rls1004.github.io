---
layout: post
title: "checkCapsLock() analysis"
image: /img/202002/caps.PNG
tags: [capslock, programming, javascript]
---

학교 포털 사이트에 로그인하려고 하면 패스워드 입력 중 항상 거슬리던 것이 있습니다.  
패스워드에 대문자를 입력하게 되면 "Caps Lock이 켜져있습니다. 확인하세요!"라는 메시지가 뜨는 것입니다.  
하지만 저는 Caps Lock을 키지 않았고 대문자를 입력했을 뿐입니다.  
어떻게 된 것인지 분석하다가 신기한 것을 발견해서 글을 쓰게 됐습니다.  

# 문제 상황

![caps alert](/img/202002/caps_00.png)

패스워드 입력 시 Caps Lock을 검사하는 것은 로그인 실수를 방지하기 위한 방법 중 하나입니다.  
하지만 제 경우엔 패스워드에 대문자가 들어갔을 뿐인데 Caps Lock이 켜져 있다고 인식됐습니다.  
로그인 할 때마다 입력이 끊기게 되어 매우 번거롭습니다.  

# 분석 1

코드 중에 잘못된 부분이 있을 것으로 생각됩니다.  
코드를 살펴봤습니다.  

```html
<label for="userpw">패스워드</label><input type="password" name="password" id="userpw" value="" class="input-1"  style="width:75px;"  tabindex="2" onKeyPress="return checkCapsLock(event)" /> 
```
```javascript
var capsLockCheck = true;

function checkCapsLock( e ) {
	 var myKeyCode=0;
	 var myShiftKey=false;
	 var myMsg='Caps Lock이 켜져있습니다. 확인하세요!';

	 // Internet Explorer 4+
	 if ( document.all ) {
	  myKeyCode=e.keyCode;
	  myShiftKey=e.shiftKey;

	 // Netscape 4
	 } else if ( document.layers ) {
	  myKeyCode=e.which;
	  myShiftKey=( myKeyCode == 16 ) ? true : false;

	 // Netscape 6
	 } else if ( document.getElementById ) {
	  myKeyCode=e.which;
	  myShiftKey=( myKeyCode == 16 ) ? true : false;

	 }

	 // Upper case letters are seen without depressing the Shift key, therefore Caps Lock is on
	 if ( ( myKeyCode >= 65 && myKeyCode <= 90 ) && !myShiftKey && capsLockCheck) {
	  alert( myMsg );
	  capsLockCheck = false;
	 // Lower case letters are seen while depressing the Shift key, therefore Caps Lock is on
	 } else if ( ( myKeyCode >= 97 && myKeyCode <= 122 ) && myShiftKey  && capsLockCheck) {
	  alert( myMsg );
	  capsLockCheck = false;

	 }
	}
```
브라우저에 따라 event.keyCode 또는 event.which를 사용해야 하기 때문에 if문을 나눠놓은 것 같습니다. 이건 키 코드의 종류를 가져옵니다.  
문제는 shift 키의 정보를 다룰 때 발생합니다. Internet Explorer 4+ 의 경우에는 event에서 shiftKey의 정보를 가져오지만 나머지 브라우저의 경우에는 which로 가져온 키 코드의 정보를 16(shift에 해당하는 코드)과 비교합니다.  
하지만 `onKeyPress` 이벤트의 경우에는 shift 키를 처리하지 않고 **결과 문자**만 다루기 때문에 shift에 해당하는 키 코드는 전달되지 않습니다.  

이로 인해 myShiftKey는 항상 false가 됩니다.  
shift를 검사하고 나면, shift를 누르지 않은 상태로 대문자가 쓰여진 경우와 shift를 누른 상태에서 소문자가 쓰여진 경우에 대해 Caps Lock이 걸렸다고 alert를 띄웁니다.  
shift는 항상 눌리지 않은 것으로 인식되어 대문자가 입력되면 항상 alert가 뜨게 됩니다.  


# 분석 2

그렇다면 Internet Explorer에서는 검사가 잘 되어야 하는거 아닌가요?  

![caps alert ie](/img/202002/caps_01.png)  

그렇지 않았습니다.  

```html
<label for="userpw">패스워드</label><input type="password" name="password" id="userpw" value="" class="input-1"  style="width:75px;"  tabindex="2" onKeyPress="console.log(event.keyCode, event.shiftKey);" /> 
```
![event key](/img/202002/caps_02.png)  

IE에서 html 코드를 수정하고 keyCode와 shiftKey를 확인한 모습입니다.  
a와 A를 입력한 것인데 javascript 코드가 이 값 중 A(keyCode: 65, shiftKye: true)에 대해 처리한다면 myKeyCode가 65 이상, 90 이하이고 myShiftKey가 true 이기 때문에 if문의 조건이 false가 되어 alert가 뜨지 않아야 합니다.  

여기서 `if (document.all)`을 의심해볼 수 있습니다.  
document.all에 undefined나 null이 들어갔다면 다른 브라우저에서 처럼 잘못 된 코드로 처리하게 됩니다.  

![document.all](/img/202002/caps_03.png)  

하지만 document.all에는 값이 있었고, 값이 있더라도 false를 반환합니다. &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<strong><font color='SALMON'><---- 신기한 부분</font></strong>  

ECMAScript에서는 참/거짓을 다음과 같은 기준으로 정의하고 있습니다.  

|Argument type|result|
|--|--|
|undefined|false|
|null|false|
|boolean|same as input|
|number|+0, -0, NaN -> false; otherwise -> true|
|string|aempty string -> false; otherwise -> true|
|object|true|

하지만 DOM 객체 중 이 기준에 대한 한 가지 예외가 있습니다.  
바로 document.all 입니다.  

document.all은 DOM의 요소에 접근하는 비표준 방법이며 더이상 사용되지 않는 방식이고 현재는 document.getElementById를 사용합니다.  
하지만 호환성 문제로 대부분의 브라우저는 document.all을 계속 지원하고 있습니다.  

```javascript
if (document.all) {
  // code that uses `document.all`, for ancient browsers
} else if (document.getElementById) {
  // code that uses `document.getElementById`, for “modern” browsers
}
```
이때, 위와 같은 코드로 이전 브라우저와 모던 브라우저를 구분하는데  
모던 브라우저에서도 document.all을 지원하기 때문에 조건이 항상 true가 되는 문제가 발생했습니다.  

```javascript
if (document.getElementById) {
  // code that uses `document.getElementById`, for “modern” browsers
} else if (document.all) {
  // code that uses `document.all`, for ancient browsers
}
```
코드를 이렇게 작성한다면 문제가 없지만, 기존의 많은 코드가 첫 번째 코드를 사용해서 처리하고 있습니다.  
이 문제를 해결하는 가장 간단한 방법이 document.all을 false로 만드는 것 입니다.  

종합해서 보면 Internet Explorer 4 이상의 낡은 브라우저에서만 Caps Lock 검사가 제대로 이루어질 것으로 보입니다. 어쨌든 궁금증은 해결됐습니다.

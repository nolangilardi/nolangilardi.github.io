---
title: "Decompiling WASM"
description: "A little note to have easier time decompiling WASM before directly reading WASM very simple instruction sets."
pubDate: "2024-08-19"
tags: ["wasm", "reverse-engineering"]
---

## The Problem

Have you ever looked at WASM decompiled code using `wasm2c` from wabt toolkit and thought, "there has to be a better way"?
While WASM's instruction set keeps things simple (except for those vectors and floating-point numbers!),
the stack machine design can sometimes lead to less-than-optimal code after decompilation.

Take a look at this code snippet: (taken from revsite2 challenge LITCTF 2024)

```c
static void w2c_visit_ad(Z_revsite2_instance_t* instance) {
  // [SNIP]
  FUNC_PROLOGUE;
  /* push rbp */
  /* mov rbp, rsp */
  u32 w2c_i0, w2c_i1, w2c_i2;
  u64 w2c_j0, w2c_j1;
  w2c_i0 = instance->w2c_g0;
  w2c_l0 = w2c_i0;
  /* sub rsp, N */
  w2c_i0 = 2048u;
  w2c_l1 = w2c_i0;
  w2c_i0 = w2c_l0;
  w2c_i1 = w2c_l1;
  w2c_i0 -= w2c_i1;
  w2c_l2 = w2c_i0;
  w2c_i0 = w2c_l2;
  instance->w2c_g0 = w2c_i0;
  // [SNIP]
```

This might look familiar if you've dabbled in C/C++ before. It's just a standard function prologue, but thanks to WASM's design,
it ends up a bit bloated. What could potentially be a single instruction gets broken down into multiple steps.
It's readable, sure, but not exactly optimized.

## The Compiler's Magic Touch

This is where the beauty of compilers comes in. Just like with any other code,
we can use tools like `gcc` or `clang` with their handy `-O` flag to optimize our C code from WASM decompiler.

```sh
pacman -S wabt
# convert wasm to c with wabt toolkit wasm2c
wasm2c file.wasm -o file.c
# wasm-c runtime api
git clone https://github.com/WebAssembly/wasm-c-api
# -O3 max optimization, -g include wasm2c structs, wasm-rt-impl for default wasm runtime implementations
gcc -g \
    -O3 \
    -I ./wasm-c-api/include \
    -I . \
    -I /usr/share/wabt/wasm2c \
    /usr/share/wabt/wasm2c/wasm-rt-impl.c \
    file.c
```

and after loading the compiled binary to IDA, we get a very clean and optimized output.

```c
void __fastcall Z_revsite2Z_visit_ad(Z_revsite2_instance_t *instance)
{
  // [SNIP]
  rbp = instance->w2c_g0;
  rsp = rbp - 2048;
  instance->w2c_g0 = rsp;
  *&instance->w2c_memory.data[rsp + 2040] = *(&score + instance->w2c_memory.data) + 123456789;
  if ( *(&chksum + instance->w2c_memory.data) == *&instance->w2c_memory.data[rsp + 2040] )
  {
    ++*(&chksum + instance->w2c_memory.data);
    *(&chksum2 + instance->w2c_memory.data) += 3
                                             * *(&score + instance->w2c_memory.data)
                                             * *(&score + instance->w2c_memory.data)
                                             + 5 * *(&score + instance->w2c_memory.data)
                                             + 3;
    *(&key + instance->w2c_memory.data) += 3
  // [SNIP]
```

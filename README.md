# c2goasm: C to Go Assembly 

## Introduction

This is a tool to convert assembly as generated by a C/C++ compiler into Golang assembly. It is meant to be used in combination with [asm2plan9s](https://github.com/minio/asm2plan9s) in order to automatically generate pure Go wrappers for C/C++ code (that may for instance take advantage of compiler intrinsics and/or templates).

Mode of operation:
```
$ c2goasm -a /path/to/c-project/build/SimdAvx2Detection.cpp.s SimdAvx2Detection_amd64.s
```

You can optionally nicely format the code using [asmfmt](https://github.com/klauspost/asmfmt) by passing in an `-f` flag. 

This project has been developed as part of developing a Go wrapper around [Simd](https://github.com/ermig1979/Simd). However it should also work with other projects and libraries.

## Command line options
```
$ c2goasm --help
Usage of c2goasm:
  -a	Immediately invoke asm2plan9s
  -c	Compact byte codes
  -f	Format using asmfmt
  -s	Strip comments
```

## A simple example

Here is a simple C function doing an AVX2 intrinsics computation:
```
void MultiplyAndAdd(float* arg1, float* arg2, float* arg3, float* result) {
    __m256 vec1 = _mm256_load_ps(arg1);
    __m256 vec2 = _mm256_load_ps(arg2);
    __m256 vec3 = _mm256_load_ps(arg3);
    __m256 res  = _mm256_fmadd_ps(vec1, vec2, vec3);
    _mm256_storeu_ps(result, res);
}
```

Compiling into assembly gives the following
```
__ZN14MultiplyAndAddEPfS1_S1_S1_: ## @_ZN14MultiplyAndAddEPfS1_S1_S1_
## BB#0:
        push          rbp
        mov           rbp, rsp
        vmovups       ymm0, ymmword ptr [rdi]
        vmovups       ymm1, ymmword ptr [rsi]
        vfmadd213ps   ymm1, ymm0, ymmword ptr [rdx]
        vmovups       ymmword ptr [rcx], ymm1
        pop           rbp
        vzeroupper
        ret
```

Running `c2goasm` will generate the following Go assembly (eg. saved in `MultiplyAndAdd_amd64.s`)
```
//+build !noasm !appengine
// AUTO-GENERATED BY C2GOASM -- DO NOT EDIT

TEXT ·_MultiplyAndAdd(SB), $0-32

	MOVQ vec1+0(FP), DI
	MOVQ vec2+8(FP), SI
	MOVQ vec3+16(FP), DX
	MOVQ result+24(FP), CX

	LONG $0x0710fcc5             // vmovups    ymm0, yword [rdi]
	LONG $0x0e10fcc5             // vmovups    ymm1, yword [rsi]
	LONG $0xa87de2c4; BYTE $0x0a // vfmadd213ps    ymm1, ymm0, yword [rdx]
	LONG $0x0911fcc5             // vmovups    yword [rcx], ymm1

	VZEROUPPER
	RET
```

This needs to be accompanied by the following Go code (in `MultiplyAndAdd_amd64.go`)
```
//go:noescape
func _MultiplyAndAdd(vec1, vec2, vec3, result unsafe.Pointer)

func MultiplyAndAdd(someObj Object) {

	_MultiplyAndAdd(someObj.GetVec1(), someObj.GetVec2(), someObj.GetVec3(), someObj.GetResult()))
}
```

And as you may have gathered the amd64.go file needs to be in place in order for the arguments names to be derived (and allow `go vet` to succeed).

## Benchmark against cgo

Running `benchmark_test.go` in resp. `test/` and `cgocmp/` gives the following result 

```
$ benchcmp ../cgocmp/cgo.out c2goasm.out 
benchmark                      old ns/op     new ns/op     delta
BenchmarkMultiplyAndAdd-12     383           9.48          -97.52%
```

## Internals

The basic process is to (in the prologue) setup the stack and registers as how the C code expects this to be the case, and upon exiting the subroutine (in the epilogue) to revert back to the golang world and pass a return value back if required. In more details:
- Define assembly subroutine with proper golang decoration in terms of needed stack space and overall size of arguments plus return value. 
- Function arguments are loaded from the golang stack into registers and prior to starting the C code any arguments beyond 6 are stored in C stack space.
- Stack space is reserved and setup for the C code. Depending on the C code, the stack pointer maybe aligned on a certain boundary (especially needed for code that takes advantages of SIMD instructions such as AVX etc.).
- A constants table is generated (if needed) and any `rip`-based references are replaced with proper offsets to where Go will put the table. 

## Limitations

- Arguments need (for now) to be 64-bit size, meaning either a value or a pointer (this requirement will be lifted)
- Generarally no `call` statements (thus inline your C code) with the exception of a few functions such as `memset` and `memcpy` (see `clib_amd64.s`)

## Generate assembly from C/C++

For eg. projects using cmake, here is how to see a list of assembly targets
```
$ make help | grep "\.s"
```

To see the actual command to generate the assembly
```
$ make -n SimdAvx2BgraToGray.s
```

## Compatible compilers

The following compilers have been tested:
- `clang` (Apple LLVM version) on OSX/darwin
- `clang` on linux

Compiler flags:
```
-masm=intel -mno-red-zone -mstackrealign -mllvm -inline-threshold=1000 -fno-asynchronous-unwind-tables -fno-exceptions -fno-rtti
```

| Flag                              | Explanation                                        |
|:----------------------------------| :--------------------------------------------------|
| `-masm=intel`                     | Output Intel syntax for assembly                   |
| `-mno-red-zone`                   | Do not write below stack pointer (avoid [red zone](https://en.wikipedia.org/wiki/Red_zone_(computing)))  |
| `-mstackrealign`                  | Use explicit stack initialization                  |
| `-mllvm -inline-threshold=1000`   | Higher limit for inlining heuristic (default=255)  |
| `-fno-asynchronous-unwind-tables` | Do not generate unwind tables (for debug purposes) |
| `-fno-exceptions`                 | Disable exception handling                         |
| `-fno-rtti`                       | Disable run-time type information                  |

The following flags are only available in `clang -cc1` frontend mode (see [below]()):

| Flag                              | Explanation                                                        |
|:----------------------------------| :------------------------------------------------------------------|
| `-fno-jump-tables`                | Do not use jump tables as may be generated for `select` statements |

#### `clang` vs `clang -cc1` 

As per the clang [FAQ](https://clang.llvm.org/docs/FAQ.html#driver), `clang -cc1` is the frontend, and `clang` is a (mostly GCC compatible) driver for the frontend. To see all options that the driver passes on to the frontend, use `-###` like this:

```
$ clang -### -c hello.c
"/usr/lib/llvm/bin/clang" "-cc1" "-triple" "x86_64-pc-linux-gnu" etc. etc. etc.
```

#### Command line flags for clang

To see all command line flags use either `clang --help` or `clang --help-hidden` for the clang driver or `clang -cc1 -help` for the frontend.

#### Further optimization and fine tuning

Using the LLVM optimizer ([opt](http://llvm.org/docs/CommandGuide/opt.html)) you can further optimize the code generation. Use `opt -help` or `opt -help-hidden` for all available options.

An option can be passed in via `clang` using the `-mllvm <value>` option, such as `-mllvm -inline-threshold=1000` as discussed above.

Also LLVM allows you to tune specific functions via [function attributes](http://llvm.org/docs/LangRef.html#function-attributes) like `define void @f() alwaysinline norecurse { ... }`.

#### What about GCC support?

For now GCC code will not work out of the box. However there is no reason why GCC should not work fundamentally (PRs are welcome).

## Resources

- https://github.com/golang/go/files/447163/GoFunctionsInAssembly.pdf
- https://godbolt.org/

# shape-basics

Minimal generic functions used to read the shape body, the per-instantiation wrapper, and the dictionary out of the `go build -gcflags=-S` output.

## Files

- `main.go` defines `First[T]`, `StoreFirst[T]`, `Greet[T Stringer]` (a local `Stringer` interface) and pointer receivers `UserA`, `UserB`.
- `multi.go` defines `DoRW[T RW]` (a multi-method interface constraint), `MakeAndFirst[T any]` (uses `make`, so the dictionary supplies runtime type info), and `Outer[T fmt.Stringer]` calling `Inner[T fmt.Stringer]` (a sub-dictionary case).

## What to look for in the assembly

```sh
go build -gcflags="-S -l" -o /dev/null . 2> asm.txt
```

A curated excerpt of the actual output (the `First` / `StoreFirst` / `Greet` symbols) is committed as [`asm-excerpt.txt`](asm-excerpt.txt) for reference.

`main.First[go.shape.*uint8]` is the single shape body shared by every pointer instantiation. `main.First[go.shape.int]` is a separate shape body for `int`. Symbols such as `main.First[*int]` or `main.First[*string]` are per-instantiation wrappers. They load the static dictionary address and tail-call the shape body. `main..dict.Greet[*main.UserA]` is the dictionary symbol itself, referenced through an `R_ADDRARM64` relocation.

`StoreFirst[T]` is the easiest place to see the shape boundary. The pointer-shape body emits `runtime.writeBarrier` and `runtime.gcWriteBarrier2` before the store. The int-shape body does not. That difference in emitted instructions is the machine-level reason `int` and `*int` cannot share a shape.

For `DoRW[T RW]`, the dictionary holds two function pointers (one for `Read`, one for `Write`), referenced from the body at offsets 0 and 8.

For `Outer` calling `Inner`, the first entry of the outer dictionary is a pointer to the inner dictionary, which the outer body loads and passes through as the inner call's `.dict` argument.

## Tested on

Go 1.26.x, darwin/arm64.

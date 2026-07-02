# shape-effectiveness

Measures how much shape sharing actually reduces compiled code. The generic function has a non-trivial body, and it is instantiated with three groups of concrete types.

## The function

```go
func Process[T fmt.Stringer](items []T) string {
    s := "items:\n"
    for i, item := range items {
        s += fmt.Sprintf("  [%d] %s\n", i, item.String())
    }
    return s
}
```

## The three groups

Ten pointer types `*P1` through `*P10`, each a distinct named struct.

Ten value types `V1` through `V10` with a common underlying type, each defined as `struct{ v int }`. The type names differ but the underlying type is identical.

Five value types `D1` through `D5` with distinct underlying types: `struct{a int}`, `struct{b int}`, `struct{x,y int}`, `struct{s string}`, and `struct{x int; y string}`.

## How to inspect

```sh
go build -gcflags="-S -l" -o /dev/null . 2> asm.txt
grep -E '^main\.Process\[go\.shape' asm.txt
```

Across the 25 concrete instantiations there are only 7 shape bodies. One body covers the pointer group (`go.shape.*uint8`). One body covers the V group (`go.shape.struct { main.v int }`). The D group splits into 5 separate bodies because the underlying types differ.

## What it shows

The shape grouping rule in Go 1.18+ is "same underlying type, or both pointer types". The ten pointer types collapse to a single body. The ten named structs with an identical underlying type also collapse to a single body. The five structs with different field layouts each get their own body.

The number of shape bodies (7 out of 25 instantiations) is platform independent, because grouping is decided by the compiler before code generation. Byte sizes depend on the architecture: on Go 1.26.1 the bodies are byte-identical between darwin/arm64 and linux/arm64, and around 15% larger on linux/amd64 due to the different instruction encoding. Measured against a hypothetical fully monomorphised build of the same code, the total `Process` body size comes out around 71% smaller on arm64 and around 72% smaller on amd64. The exact ratio depends on the function body.

This is a deliberately favourable case for shape sharing. The OOPSLA 2022 paper [Generic Go to Go](https://songlh.github.io/paper/generic.pdf) observes in §6.4 that in their realistic benchmarks they "do not observe the reuse of method implementations" because the Go 1.18 grouping rule is conservative.

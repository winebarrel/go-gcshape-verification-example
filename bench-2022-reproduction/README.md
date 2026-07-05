# bench-2022-reproduction

Reproduces two well-known 2022 generics benchmarks on Go 1.26.x and compares the numbers with the originals.

The two articles are:

- PlanetScale, Vicent Marti, [Generics can make your Go code slower](https://planetscale.com/blog/generics-can-make-your-go-code-slower) ([Wayback Machine snapshot](https://web.archive.org/web/20220522015937/https://planetscale.com/blog/generics-can-make-your-go-code-slower), 2022-03-30)
- DoltHub, Andy Arthur, [Generics and Value Types in Golang](https://www.dolthub.com/blog/2022-04-01-fast-generics/) (2022-04-01)

## Running

```sh
go test -bench=. -benchmem -benchtime=2s -count=3
```

The repository also has a manual [Benchmark workflow](https://github.com/winebarrel/go-gcshape-verification-example/actions/workflows/bench.yml). Running it executes the same command on `ubuntu-latest` (x86) and `ubuntu-24.04-arm` and prints a combined comparison table in the job summary.

## Files

`dolthub.go` does binary search over `ArrayVal` (a value receiver) and `ArrayRef` (a pointer receiver), through an `Array` interface and through an `Array`-constrained generic function. The code is taken directly from the article.

`planetscale.go` defines `EscapeMonomorphized`, `EscapeIface`, `EscapeGeneric[W io.ByteWriter]`, and `EscapeGenericSuper[W IBuffer]`. The original article is a single-page application and the source could not be extracted directly, so this file is reconstructed from the prose. The structure matches what the article describes, but small differences in the function body are possible.

`bench_test.go` drives the five PlanetScale variants and the four DoltHub variants.

The five PlanetScale variants exercise the following:

| Name | What is measured |
|---|---|
| `Monomorphized` | `func Escape(*strings.Builder, []byte)`. Concrete type, no generics, no interface. |
| `Iface` | `func Escape(io.ByteWriter, []byte)`. Interface, no generics. |
| `GenericWithPointer` | `EscapeGeneric[W io.ByteWriter](w W, ...)` called with `*strings.Builder`. W is inferred as the concrete pointer type. |
| `GenericWithExactIface` | The same function, called with the value already typed as `io.ByteWriter`. W is inferred as `io.ByteWriter` itself. |
| `GenericWithSuperIface` | A second generic function constrained on the wider interface `IBuffer`, called with the value typed as `IBuffer`. This triggers the `runtime.assertI2I` path that the original article identified as the slowest case. |

## Headline result

The cost ratios versus `Monomorphized` were measured on Go 1.26.x in three environments: the two GitHub-hosted Linux runners (via the Benchmark workflow, [run](https://github.com/winebarrel/go-gcshape-verification-example/actions/runs/27466670328)) and a local Apple M4 Pro.

| Variant | 2022 (PlanetScale) | linux/amd64 | linux/arm64 | darwin/arm64 (M4 Pro) |
|---|---|---|---|---|
| Monomorphized | 1.00x | 1.00x | 1.00x | 1.00x |
| Iface | 1.35x | 1.70x | 1.46x | 2.05x |
| GenericWithPointer | 1.42x | 1.72x | 1.53x | 2.07x |
| GenericWithExactIface | 1.91x | 2.55x | 1.89x | 1.41x |
| GenericWithSuperIface | 3.48x | 2.56x | 1.90x | 1.41x |

Two things hold on every platform tested. Every indirect variant is still slower than the monomorphized function, and `GenericWithSuperIface` now costs the same as `GenericWithExactIface`. In 2022 the SuperIface case paid a large extra penalty for the `runtime.assertI2I` path (3.48x against 1.91x). That gap is gone.

Everything else is platform dependent. On the Linux runners the ordering matches 2022, with the interface-typed generic variants slowest at about 1.9x on arm64 and about 2.6x on amd64. On the M4 the ordering flips, and the interface-typed variants (about 1.41x) come out faster than `Iface` and `GenericWithPointer` (about 2.05x). No single machine's ratios, including these, should be quoted as the cost of Go generics in general.

The DoltHub observation reproduces on all platforms. Generic search over a value type runs at 25 to 37 percent less time than interface-based search over the same value type. The allocation column makes the mechanism explicit: the generic value-type case is `0 B/op`, the others are `24 B/op`.

## Notes

The PlanetScale reconstruction is structural, not byte-for-byte. Treat it as the same setup, not an exact replay of the original code.

Relative ratios vary widely with the CPU, as the table above shows, and even the ordering of the variants can flip between platforms. Measure on the platforms you care about before drawing conclusions. Shared CI runners can also land on different hardware generations between runs, so quote results together with the run link and the reported CPU.

The `strings.Builder` inside `EscapeMonomorphized` triggers internal resizing, so the allocation column here does not match the original article's allocation column. The ordering is still informative, but treat the absolute alloc numbers with caution.

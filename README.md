# go-gcshape-verification-example

Verification code for the Go 1.18+ generics implementation (GC Shape Stenciling with dictionary passing). The code is set up so that you can read the compiled assembly and confirm what the design documents describe. It also measures how much shape sharing reduces binary size, and reproduces two well-known 2022 benchmarks on Go 1.26.

## Layout

The repository is a multi-module workspace. Each subdirectory is an independent Go module.

- `shape-basics/` contains minimal generic functions (`First`, `StoreFirst`, `Greet` and a few multi-method / make / nested-generic variants). You inspect the `go build -gcflags=-S` output to see the shape body, the per-instantiation wrapper, and the dictionary.
- `shape-effectiveness/` measures how much shape sharing reduces compiled code. It instantiates the same generic function with three groups of concrete types and counts the resulting shape bodies.
- `bench-2022-reproduction/` reproduces benchmarks from PlanetScale ([Generics can make your Go code slower](https://planetscale.com/blog/generics-can-make-your-go-code-slower) / [Wayback](https://web.archive.org/web/20220522015937/https://planetscale.com/blog/generics-can-make-your-go-code-slower), 2022-03-30) and DoltHub ([Generics and Value Types in Golang](https://www.dolthub.com/blog/2022-04-01-fast-generics/), 2022-04-01) on Go 1.26.x and compares the numbers with the original reports.

## Usage

Each module is self-contained. From inside a module directory:

```sh
# Get the assembly for the shape body, the wrapper, and the dictionary
go build -gcflags="-S -l" -o /dev/null . 2> asm.txt

# Run the benchmarks (bench-2022-reproduction only)
go test -bench=. -benchmem -benchtime=2s -count=3
```

Tested on Go 1.26.x. Benchmark results, including the ratios between variants, differ noticeably between platforms; see `bench-2022-reproduction/README.md` for numbers from linux/amd64, linux/arm64, and darwin/arm64.

The benchmarks can also be run on GitHub-hosted runners via the [Benchmark workflow](https://github.com/winebarrel/go-gcshape-verification-example/actions/workflows/bench.yml), which runs them on both an x86 and an arm64 Linux runner and prints a combined comparison table in the run summary.

## Notes

This is study material rather than a library. The Go 1.18+ shape grouping rule is "same underlying type, or both pointer types", and each benchmark is set up so that the consequences of that rule are visible at the assembly level.

The PlanetScale benchmark was reconstructed from the article's prose because the original is a single-page application and the code could not be extracted directly. The DoltHub reproduction uses the source code as published in the article.

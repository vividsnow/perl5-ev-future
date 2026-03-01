# EV::Future

Minimalist and high-performance async control flow for Perl's `EV` event loop, implemented in XS for maximum speed and minimal overhead.

## Project Overview

`EV::Future` provides three control flow primitives:
- `parallel(\@tasks, \&final_cb [, $unsafe])`: Executes all tasks concurrently.
- `parallel_limit(\@tasks, $limit, \&final_cb [, $unsafe])`: Executes tasks concurrently with at most `$limit` in-flight.
- `series(\@tasks, \&final_cb [, $unsafe])`: Executes tasks sequentially, one at a time.

All three are implemented in C (XS) to offload task management and callback handling, ensuring low latency and high throughput.

### Technologies
- **Perl**: Main interface and module logic.
- **XS (C)**: Core implementation of control flow logic.
- **EV**: The event loop used for asynchronous operations.
- **ExtUtils::MakeMaker**: Build system.

## Building and Running

### Prerequisites
- Perl 5.10.0 or higher.
- `EV` module (v4.37+).
- A C compiler and `make`.

### Build Commands
```bash
perl Makefile.PL
make
```

### Running Tests
```bash
make test
# OR
prove -b t/
```

### Cleaning Up
```bash
make clean
```

## Benchmarks

1000 synchronous tasks x 5000 iterations (`bench/comparison.pl`):

### Parallel (iterations/sec)

| Module | Rate |
|--------|-----:|
| EV::Future (unsafe) | 4,386 |
| EV::Future (safe) | 2,262 |
| AnyEvent cv (begin/end) | 1,027 |
| Future::XS (wait_all) | 982 |
| Promise::XS (all) | 32 |

### Parallel limit=10 (iterations/sec)

| Module | Rate |
|--------|-----:|
| EV::Future (unsafe) | 4,673 |
| EV::Future (safe) | 2,688 |
| Future::Utils fmap_void | 431 |

### Series (iterations/sec)

| Module | Rate |
|--------|-----:|
| EV::Future (unsafe) | 5,000 |
| AnyEvent cv (stack-safe) | 3,185 |
| EV::Future (safe) | 2,591 |
| Future::XS (chain) | 893 |
| Promise::XS (chain) | 809 |

Safe mode allocates a per-task CV for double-call protection and wraps each dispatch in `G_EVAL`. Unsafe mode reuses a single shared CV and skips `G_EVAL`, roughly doubling throughput.

To run benchmarks:
```bash
perl -Mblib bench/benchmark.pl
perl -Mblib bench/comparison.pl
```

## Examples

Practical examples are provided in the `eg/` directory:
- `eg/curl_parallel.pl`: Fetches multiple URLs concurrently (with limit) using `AnyEvent::YACurl`.
- `eg/redis_parallel.pl`: Demonstrates concurrent Redis operations using `EV::Hiredis`.
- `eg/etcd_series.pl`: Demonstrates sequential etcd operations using `EV::Etcd`.

To run examples:
```bash
perl -Ilib eg/curl_parallel.pl                    # requires AnyEvent::YACurl
perl -Ilib eg/redis_parallel.pl                   # requires EV::Hiredis + Redis
perl -Ilib eg/etcd_series.pl                      # requires EV::Etcd + etcd
```

## Development Conventions

### XS Implementation
- The core logic resides in `Future.xs`.
- Context management is handled via `parallel_ctx`, `plimit_ctx` and `series_ctx` structs.
- Memory management is strictly handled using Perl's `Newx` and `Safefree` with proper reference counting (`SvREFCNT_inc`/`dec`).
- It uses the `EVAPI` to interface with the `EV` loop.

### Testing
- Tests are located in the `t/` directory.
- `t/01-basic.t` covers parallel, parallel_limit, series, stress tests, edge cases, synchronous completion, exceptions, and deep recursion.
- Use `Test::More` for all test scripts.

### Coding Style
- Follow standard Perl XS conventions.
- Ensure all XS functions are properly documented in the `.pm` file using POD.
- Minimize overhead in the XS layer to maintain high performance.

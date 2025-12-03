---
theme: /home/gaurav/workspace/others/gdg-berlin/gopher.json
author: ""
paging: "@GDG Golang, Berlin"
date: Dec 3, 2025
---

# TinyGo — WebAssembly beyond the Web

```sh

```

---

# TinyGo — WebAssembly beyond the Web

```sh
❯ tree agenda
.
├── WebAssembly
│   ├── Overview
│   └── Wasm Module
├── Standard Go + Wasm
├── TinyGo
│   ├── Overview
│   └── Beyond the Web
└── Demo
```

---

# TinyGo — WebAssembly beyond the Web

❯ $whoami

```yaml
Name:   Gaurav Gahlot
Role:   Staff Software Engineer @IONOS Cloud
X:      @_gauravgahlot
GitHub: @gauravgahlot
Web:    https://gauravgahlot.in/
OSS:
  - Akri (maintainer, CNCF Sandbox)
  - Tinkerbell (ex-maintainer, CNCF Sandbox)
  - falcosidekick
  - fission
  ...
```

---

## WebAssembly - Overview

📌 WebAssembly is neither

```sh
❌Web
❌Assembly
```

---

## WebAssembly - Overview

📌Just another _byte code_ format.

```

```

| Property        | Description                           |
| --------------- | ------------------------------------- |
| **Security**    | Sandboxed execution environment       |
| --------------- | ------------------------------------- |
| **Performance** | Near native execution speed           |
| --------------- | ------------------------------------- |
| **Polyglot**    | Supports a wide set of language       |
| --------------- | ------------------------------------- |
| **Portability** | Cross-platform and cross-architecture |
| --------------- | ------------------------------------- |

---

## WebAssembly - Overview

📌Just another _byte code_ format.

```

```

```
~~~graph-easy
[Java Program] - compilation -> [Java Byte Code]
[Java Byte Code] - execution -> [JVM]
[JVM] -> [ARM]
[JVM] -> [x86]
~~~
```

---

## WebAssembly - Overview

📌Just another _byte code_ format.

```

```

```
~~~graph-easy
[Any Program] - compilation -> [Wasm Byte Code]
[Wasm Byte Code] - execution -> [Wasm Runtime]
[Wasm Runtime] -> [ARM]
[Wasm Runtime] -> [x86]
~~~
```

---

## WebAssembly - Wasm Module

📌Two representations

1. Text format (`foo.wat`)

```rust
(module
  (func (export "run")
        (result i32)
    i32.const 2
    return))
```

```
~~~graph-easy
[foo.wat] - wat2wasm -> [foo.wasm]
~~~
```

---

## WebAssembly - Wasm Module

📌Two representations

1. Text format (`foo.wat`)

```rust
(module
  (func (export "run")
        (result i32)
    i32.const 2
    return))
```

2. Binary format (`foo.wasm`)

```rust
00000000: 286d 6f64 756c 650a 2020 2866 756e 6320  (module.  (func
00000010: 2865 7870 6f72 7420 2272 756e 2229 0a20  (export "run").
00000020: 2020 2020 2020 2028 7265 7375 6c74 2069         (result i
00000030: 3332 290a 2020 2020 6933 322e 636f 6e73  32).    i32.cons
00000040: 7420 320a 2020 2020 7265 7475 726e 2929  t 2.    return))
00000050: 0a0a
```

---

## WebAssembly - Wasm Module

```

```

```
+---------------------- Runtime (Host) ---------------------------+
|                                                                 |
|                                  +--- Wasm Module (Guest)---+   |
|                                  |                          |   |
|      [Export: run] <-- exports --|------  [Function]        |   |
|             ^                    |                          |   |
|             |                    +--------------------------+   |
|          invokes                                                |
|         function                                                |
|             |                                                   |
+-----------------------------------------------------------------+
              |
           ┌──────┐
           │ User │
           └──────┘
```

---

## Standard Go + Wasm

📌In the browser

```yaml
- Supported: "Yes"
- Uses: 'syscall/js' package
- Build: GOOS=js GOARCH=wasm go build -o main.wasm
```

📌Outside the browser

```yaml
- Supported: "Yes"
- Uses: WebAssembly System Interface (WASI)
- Build: GOOS=wasip1 GOARCH=wasm go build -o main.wasm
```

---

## Standard Go + Wasm

📌In the browser

```yaml
- Supported: "Yes"
- Uses: 'syscall/js' package
- Build: GOOS=js GOARCH=wasm go build -o main.wasm
```

📌Outside the browser

```yaml
- Supported: "Yes"
- Uses: WebAssembly System Interface (WASI)
- Build: GOOS=wasip1 GOARCH=wasm go build -o main.wasm
```

📌Requirements

```sh
❌not building for the web
❌don't need WASI support
```

---

## TinyGo

- An LLVM based compiler for Go
- Supports both - JS (browser), and WASI (server, edge)

- Memory manager `-gc`:
  - `none`: no memory management
  - `leaking`: allcate; never free
  - `conservative`: mark/sweep GC

- Scheduling (concurrency) `-scheduler`:
  - depends on platform
  - `none`: no goroutines/channels
  - `tasks`: similar to Go scheduler
  - `asyncify`: used for WebAssembly

---

## Beyond the Web

Custom build configuration (`wasm-unknown.json`) for TinyGo:

```json
{
  "inherits": ["wasm"],
  "llvm-target": "wasm32-unknown-unknown",
  "gc": "leaking",
  "scheduler": "none",
  "libc": "none",
  "default-extldflags": ""
}
```

```sh
$ tinygo build -o compute.wasm -target=wasm-unknown -no-debug -opt=z code/compute.go
```

---

| Property                              | Description                                                                               |
| ------------------------------------- | ----------------------------------------------------------------------------------------- |
| inherits: [wasm]                      | Start with default wasm configuration                                                     |
| ------------------------------------- | -------------------------------------                                                     |
| llvm-target: wasm32-unknown-unknown   | Identifier for a 32-bit WASM target with an "unknown" vendor and "unknown" OS/environment |
| ------------------------------------- | -------------------------------------                                                     |
| gc: leaking                           | Disables the garbage collector and all associated runtime code                            |
| ------------------------------------- | -------------------------------------                                                     |
| scheduler: none                       | Disables the scheduler                                                                    |
| ------------------------------------- | -------------------------------------                                                     |
| libc: none                            | The compiled code to handle all low-level calls directly or rely on host for imports      |
| ------------------------------------- | -------------------------------------                                                     |
| default-extldflags: ""                | Clear any default linker flags                                                            |
| ------------------------------------- | -------------------------------------                                                     |

---

## References

- [WebAssembly](https://webassembly.org/)
- [WebAssembly by Example](https://wasmbyexample.dev/home.en-us.html)
- [TinyGo - WebAssembly](https://tinygo.org/docs/guides/webassembly/)
- [TinyGo - Important Build Options](https://tinygo.org/docs/reference/usage/important-options/)
- [TinyGo - Playground](https://play.tinygo.org/)

---

## Thank you!!

📌Slides

- [Web](https://gauravgahlot.in/talks)
- [GitHub](https://github.com/gauravgahlot/talks)

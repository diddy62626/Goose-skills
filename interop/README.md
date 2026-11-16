# interop

**Working with data** — The C boundary in both directions: extern fn, extern "lib", export fn, sidecars and CMake.

4 skills. Load any of them as `interop/<skill>/skill.md`.

- [`goose-export-to-c`](interop/export-to-c/skill.md) — export fn — Goose as a library called from C: > export fn makes a Goose function callable from C; the compiler emits a C wrapper. Build with --standalone --header hello.h -o hello.c (or --shared for a .so/.dll); the host calls goose_init once (which runs Goose main), then the exports; goose_exit releases and allows reload. Slice results come back as pointer + length out-parameter.
- [`goose-extern-fn`](interop/extern-fn/skill.md) — extern fn — calling C from Goose: > extern fn binds a Goose signature to a C symbol with no body; the call compiles to a direct C call. What may cross: scalars, flat fixed-size values, T& pointers, T[:] as {data,len}, and u8[>..]& builders. What is rejected at the declaration. Where symbols resolve (JIT: math.h only; builds: your C and libraries).
- [`goose-extern-lib-and-ffi`](interop/extern-lib-and-ffi/skill.md) — extern "lib" blocks and the ffi module — binding shared libraries by name: > extern "SDL2" { fn SDL_Init(flags: u32) -> i32; } opens a shared library at program start and binds the members called — no C to compile. Handles are u64, pointers to Goose-owned data are T&, structs cross packed, and ffi::cstr/bytes copy library-owned memory into Goose storage. Sidecar dylibs maps platform filenames.
- [`goose-sidecar-own-c`](interop/sidecar-own-c/skill.md) — Sidecar foo.build — a program or package with C of its own: > C that belongs to the program goes in a sidecar foo.build beside foo.goose: c_sources/c_include/c_defines/c_flags/c_libs (each with platform suffixes), c_headers adapters using the generated types, dylibs mappings, and output naming. Packages use the same sidecar with cmake_target for CMake projects.

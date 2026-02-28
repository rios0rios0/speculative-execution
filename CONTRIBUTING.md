# Contributing

Contributions are welcome. By participating, you agree to maintain a respectful and constructive environment.

For coding standards, testing patterns, architecture guidelines, commit conventions, and all
development practices, refer to the **[Development Guide](https://github.com/rios0rios0/guide/wiki)**.

## Prerequisites

- [GCC](https://gcc.gnu.org/) 7+ (with x86 intrinsics support for `_mm_clflush`, `__rdtscp`)
- [CMake](https://cmake.org/) 3.13+
- [Make](https://www.gnu.org/software/make/) or [Ninja](https://ninja-build.org/)
- A Linux x86-64 system (required for `/proc/kallsyms`, `/proc/pagemap`, and inline AT&T assembly)

## Development Workflow

1. Fork and clone the repository
2. Create a branch: `git checkout -b feat/my-change`
3. Create a build directory and generate the build files:
   ```bash
   mkdir build && cd build
   cmake ..
   ```
4. Build all targets (Spectre, Meltdown, and Tests):
   ```bash
   make
   ```
5. Run a specific exploit or test:
   ```bash
   ./build/jan_horn        # Spectre PoC (Jann Horn)
   ./build/cache_time      # Cache timing test
   ./build/paboldin        # Meltdown PoC
   ```
6. Build the standalone Meltdown exploit (alternative):
   ```bash
   cd Meltdown/paboldin
   make -f Makefile.txt
   ./run.sh
   ```
7. Commit following the [commit conventions](https://github.com/rios0rios0/guide/wiki/Life-Cycle/Git-Flow)
8. Open a pull request against `main`

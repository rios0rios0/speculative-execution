# Speculative Execution Repository

**ALWAYS FOLLOW THESE INSTRUCTIONS FIRST.** Only fallback to additional search and context gathering if the information in these instructions is incomplete or found to be in error.

This repository contains educational proof-of-concept (PoC) implementations of the Meltdown (CVE-2017-5754) and Spectre (CVE-2017-5753) CPU vulnerabilities. It includes reference exploits from published research (Jann Horn et al., paboldin, rootkea) alongside original cross-process Spectre variants and supporting test utilities, all written in C99 for x86/x86-64 Linux.

## Quick Reference

**Essential Commands:**
- `mkdir build && cd build && cmake .. && make` - Build all targets (~10-30s depending on CPU)
- `./build/jan_horn` - Run Spectre v1 PoC (Jann Horn)
- `./build/cache_time` - Run cache timing test
- `cd Meltdown/paboldin && make -f Makefile.txt && ./run.sh` - Build and run Meltdown standalone
- `gcc --version && cmake --version` - Check prerequisites

**CMake Build Targets:**

| Target | Source | Description |
|--------|--------|-------------|
| `jan_horn` | `Spectre/jann_horn_et_al/script.c` | Canonical Spectre v1 PoC |
| `rootkea` | `Spectre/rootkea/script.c` | Spectre v1 with custom secret |
| `paboldin` | `Meltdown/paboldin/script.c` | Meltdown kernel memory reader |
| `victim` | `Spectre/rios0rios0/victim.c` | Cross-process victim (single-shot) |
| `victim2` | `Spectre/rios0rios0/victim2.c` | Cross-process victim (looping) |
| `attacker` | `Spectre/rios0rios0/attacker.c` | Cross-process Spectre attacker |
| `cache_time` | `Tests/CacheTime.c` | Cache hit/miss timing |
| `kernel_table` | `Tests/KernelTable.c` | Fork/virtual memory test |
| `virt_address` | `Tests/VirtAddress.c` | Virtual-to-physical translation |
| `size_of` | `Tests/SizeOf.c` | sizeof vs strlen demo |

**Technology Stack:** C99 (`_GNU_SOURCE`), inline x86/x86-64 AT&T assembly, `rdtscp`/`rdtsc`, `clflush` (`_mm_clflush`), `sigaction`, `sched_setaffinity`, CMake 3.13+

**No CI/CD pipelines and no automated tests** -- this is a research/educational project.

## Working Effectively

### Prerequisites

- **GCC 7+** with x86 intrinsics (`_mm_clflush`, `__rdtscp`, `__builtin_ia32_*`)
- **CMake 3.13+**
- **GNU Make** or **Ninja**
- **Linux x86-64** (required for `/proc/kallsyms`, `/proc/pagemap`, inline AT&T assembly)

### Bootstrap and Setup

```bash
git clone https://github.com/rios0rios0/speculative-execution.git
cd speculative-execution
mkdir build && cd build
cmake ..
make
```

### Core Commands That Work

```bash
# Verify prerequisites
gcc --version                    # GCC 7+
cmake --version                  # CMake 3.13+

# Build all targets
mkdir build && cd build
cmake ..                         # Configure (~1-2s)
make                             # Build all (~10-30s)

# Run Spectre PoC (Jann Horn)
./build/jan_horn                 # Reads "The Magic Words are Squeamish Ossifrage."

# Run cache timing test
./build/cache_time               # Shows cached (~5-30 cycles) vs uncached (~100-300 cycles)

# Run Meltdown standalone build
cd Meltdown/paboldin
make -f Makefile.txt             # Standalone build with -O2 -msse2
./run.sh                         # Reports VULNERABLE or NOT VULNERABLE

# Run cross-process Spectre (two terminals)
./build/victim2                  # Terminal 1: prints PID and virtual address
./build/attacker <pid> <vaddr>   # Terminal 2: run attacker with victim's output
```

### Standalone Meltdown Build (Alternative)

```bash
cd Meltdown/paboldin
# Optional: auto-detect rdtscp support
bash detect_rdtscp.sh            # Generates rdtscp.h
make -f Makefile.txt             # Produces ./script binary
./run.sh                         # Full vulnerability test
```

## Repository Structure

```
speculative-execution/
├── CMakeLists.txt                          # Build configuration (10 targets, C99)
├── README.md                               # Project documentation
├── CONTRIBUTING.md                         # Development guidelines
├── LICENSE                                 # MIT License
│
├── Meltdown/
│   └── paboldin/
│       ├── script.c                        # Meltdown PoC: reads kernel memory via speculative execution
│       ├── Makefile.txt                    # Standalone Makefile with -O2 -msse2 flags
│       ├── run.sh                          # Locates linux_proc_banner and runs the exploit
│       └── detect_rdtscp.sh                # Generates rdtscp.h with rdtscp or rdtsc+mfence
│
├── Spectre/
│   ├── jann_horn_et_al/
│   │   └── script.c                        # Canonical Spectre v1 PoC (bounds check bypass)
│   ├── rootkea/
│   │   └── script.c                        # Spectre v1 variant with custom secret string
│   └── rios0rios0/
│       ├── attacker.c                      # Cross-process Spectre attacker via /proc/pagemap
│       ├── victim.c                        # Victim process (single-shot, exits after one call)
│       └── victim2.c                       # Victim process (loops, prints PID and vaddr)
│
└── Tests/
    ├── CacheTime.c                         # Cache hit/miss timing across 10 cache lines
    ├── CacheTime2.c                        # Extended timing with L1/L2/L3 size constants
    ├── VirtAddress.c                       # Virtual-to-physical address translation via pagemap
    ├── SizeOf.c                            # sizeof vs strlen demonstration
    └── KernelTable.c                       # Fork/virtual memory inheritance test
```

## Architecture and Design Patterns

### Flush+Reload Side Channel (all exploits)

All exploits in this repository use the same fundamental technique:

1. **Flush** -- evict all 256 entries of the probe array from the CPU cache using `clflush` / `_mm_clflush`
2. **Speculate** -- trigger speculative execution that indexes into the probe array using the secret byte as the index (multiplied by 512 or 4096 to span separate cache lines)
3. **Reload** -- time access to each of the 256 probe array entries using `rdtscp`; the entry that loads significantly faster than the threshold reveals the secret byte value

### Branch Predictor Mistraining (Spectre v1)

- Runs 30 iterations; 5 in-bounds accesses per 1 out-of-bounds attack (`j % 6 == 0` selects the attack)
- Bit-twiddling avoids conditional jumps that would retrain the predictor:
  ```c
  x = ((j % 6) - 1) & ~0xFFFF;
  x = x | (x >> 16);  // 0 for attack iteration, else valid index
  ```
- Probe array striding: `array2[array1[x] * 512]` or `array2[array1[x] * 4096]`

### Meltdown Signal Recovery

- Exploits speculative execution past a faulting kernel memory load
- Installs a `sigaction` handler that modifies `RIP`/`EIP` in the signal context to jump past the speculative gadget
- Adaptive threshold: calibrates the cache hit/miss boundary by sampling 1,000,000 cached/uncached accesses and computing the geometric mean

### Cross-Process Spectre (rios0rios0)

- Parses `/proc/<pid>/pagemap` to translate virtual addresses to physical page frame numbers (PFNs)
- Uses the `PagemapEntry` struct to extract the 54-bit PFN
- Victim processes print their own PID and virtual address for the attacker to consume

### Cache Timing Constants (CacheTime2.c)

```c
#define CACHE_L1_ALL_SIZE (32)    // 32 KB (Yarom2015MappingTI)
#define CACHE_L1_LINE_SIZE (64)   // 64 B cache line
#define CACHE_L2_ALL_SIZE (256)   // 256 KB
#define CACHE_L3_ALL_SIZE (4096)  // 4 MB
#define CACHE_HIT_THRESHOLD (80)  // assume cache hit if time <= threshold
```

## Technology Stack

| Component | Technology |
|-----------|------------|
| Language | C99 with GNU extensions (`_GNU_SOURCE`, `#define _GNU_SOURCE`) |
| Assembly | Inline x86/x86-64 AT&T syntax for speculative load gadgets |
| Timing | `rdtscp` / `rdtsc` hardware timestamp counters |
| Cache manipulation | `clflush` via `_mm_clflush` intrinsic (`<immintrin.h>`) |
| Signal handling | `sigaction` with `SA_SIGINFO` for SIGSEGV recovery |
| CPU pinning | `sched_setaffinity` to pin processes to CPU 0 |
| Memory translation | `/proc/<pid>/pagemap` with `PagemapEntry` struct (bit 63 = present, bits 0-54 = PFN) |
| Build system | CMake 3.13+ (`cmake_minimum_required(VERSION 3.13)`) |
| C standard | `set(CMAKE_C_STANDARD 99)` |
| Target architectures | x86-64 primary, i386 supported |
| Target OS | Linux only |

## CI/CD Pipeline

There are **no CI/CD pipelines** in this repository. It is a standalone research project with no automated build, test, or deployment workflows.

## Development Workflow

1. Fork and clone the repository
2. Create a feature branch: `git checkout -b feat/my-change`
3. Create the build directory and configure:
   ```bash
   mkdir build && cd build
   cmake ..
   ```
4. Build all targets:
   ```bash
   make
   ```
5. Test your changes manually by running the relevant binary
6. Commit following the [commit conventions](https://github.com/rios0rios0/guide/wiki/Life-Cycle/Git-Flow)
7. Open a pull request against `main`

For full coding standards, testing patterns, architecture guidelines, and commit conventions, refer to the **[Development Guide](https://github.com/rios0rios0/guide/wiki)**.

## Coding Conventions

- **Language standard:** C99 (`set(CMAKE_C_STANDARD 99)`). Only the Meltdown source (`Meltdown/paboldin/script.c`) defines `_GNU_SOURCE` (for `ucontext`/`REG_RIP`); the Spectre and test sources do not
- **Assembly syntax:** AT&T syntax for inline `asm` blocks (GCC default)
- **Probe array sizing:** 256 entries × 512 or 4096 bytes per entry (to span cache lines)
- **Cache line stride:** Minimum 512 bytes between probe array entries (typically 4096 for safety)
- **Timing reads:** Always use `rdtscp` (serializing) over `rdtsc` where the hardware supports it; use `detect_rdtscp.sh` to generate `rdtscp.h` for the Meltdown standalone build
- **No dynamic memory allocation** in hot speculative paths
- **Signal handlers** must be async-signal-safe; only modify `uc_mcontext` registers, do not call libc functions

## Common Tasks and Troubleshooting

### Build Errors

```bash
# Missing cmake
sudo apt-get install cmake

# Missing GCC with intrinsics
sudo apt-get install gcc

# cmake version too old (need 3.13+)
cmake --version
# If too old, download from https://cmake.org/download/
```

### Exploit Not Working / NOT VULNERABLE

Modern systems have CPU and kernel mitigations that prevent these exploits from succeeding:
- **Meltdown:** Kernel Page Table Isolation (KPTI / KAISER) prevents userspace from accessing kernel mappings
- **Spectre v1:** Compiler-inserted `lfence` barriers and array index masking disrupt the timing window
- **Cross-process Spectre:** Process isolation and `/proc/pagemap` restrictions (require `CAP_SYS_ADMIN` or reduced permissions on newer kernels) limit the attack surface

The exploits may still demonstrate the side-channel technique even on patched systems but will not successfully leak secrets.

### Adjusting Cache Timing Thresholds

If `cache_time` shows no clear separation between cached and uncached accesses, adjust the threshold in the relevant source:

```c
// In Spectre sources
#define CACHE_HIT_THRESHOLD 80  // Reduce if cached accesses are slow on your CPU

// In Meltdown (paboldin)
// The threshold is calibrated dynamically via 1M sample geometric mean
```

### Cross-Process Spectre Setup

```bash
# Terminal 1: start the looping victim
./build/victim2
# Output: PID=12345  array1=0x7fff...  secret=0x7fff...

# Terminal 2: run the attacker with victim's addresses
./build/attacker 12345 0x7fff...
```

### Reading `/proc/pagemap` (VirtAddress / attacker)

On Linux kernels ≥ 4.0, reading `/proc/<pid>/pagemap` for other processes requires `CAP_SYS_PTRACE`. Run as root or adjust `/proc/sys/kernel/perf_event_paranoid` for testing:

```bash
sudo ./build/attacker <pid> <vaddr>
# or
sudo ./build/virt_address
```

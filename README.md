<h1 align="center">Speculative Execution</h1>
<p align="center">
    <a href="https://github.com/rios0rios0/speculative-execution/releases/latest">
        <img src="https://img.shields.io/github/release/rios0rios0/speculative-execution.svg?style=for-the-badge&logo=github" alt="Latest Release"/></a>
</p>

A collection of proof-of-concept implementations and supporting test utilities for the Meltdown (CVE-2017-5754) and Spectre (CVE-2017-5753) CPU vulnerabilities. Includes reference exploits from published research (Jann Horn et al., paboldin, rootkea) alongside original cross-process Spectre variants, all built for educational study of speculative execution side-channel attacks on x86 processors.

**Language:** C (C99) with inline x86/x86-64 assembly | **Build:** CMake 3.13+ | **Status:** Educational/Research (2019)

## Features

### Meltdown (CVE-2017-5754)

- **Kernel memory reader (paboldin)** -- reads arbitrary kernel memory from userspace by exploiting speculative execution past a faulting load. Uses a SIGSEGV handler to redirect execution after the faulting instruction, combined with a Flush+Reload cache side-channel to extract speculatively accessed bytes one at a time
- **Automatic vulnerability detection** -- the `run.sh` script locates the `linux_proc_banner` kernel symbol via `/proc/kallsyms` or `System.map`, reads 10 bytes from that address, and compares against the expected `/proc/version` format string to determine if the system is vulnerable
- **Adaptive cache threshold** -- dynamically calibrates the cache hit/miss timing threshold by measuring 1,000,000 cached and uncached memory accesses and computing the geometric mean

### Spectre (CVE-2017-5753 -- Variant 1: Bounds Check Bypass)

- **Jann Horn et al. reference** -- the canonical Spectre v1 proof-of-concept that mistrains the branch predictor with in-bounds accesses, then speculatively reads out-of-bounds secret data through a cache side-channel using `array2[array1[x] * 512]`
- **Rootkea variant** -- functionally similar to the Jann Horn PoC but with a different secret string (`"The password is rootkea"`) and a secret length of 23 bytes
- **Cross-process attacker/victim (rios0rios0)** -- an original two-process Spectre variant:
  - `victim.c` / `victim2.c` -- victim processes that expose their PID and virtual addresses of `array1` and `secret`, then either exit or loop indefinitely
  - `attacker.c` -- reads `/proc/<pid>/pagemap` to translate virtual addresses to physical addresses, then attempts a cross-process Spectre attack using `clflush` cache eviction and `rdtscp` timing

### Test Utilities

- **CacheTime** -- measures memory access times across 10 cache lines (each 4096 bytes apart), demonstrating the timing difference between cached and uncached accesses using `_mm_clflush` and `__rdtscp`
- **CacheTime2** -- extended cache timing test with L1/L2/L3 cache size constants and `__clear_cache` intrinsic for bulk cache flushing
- **VirtAddress** -- resolves virtual addresses to physical addresses by parsing `/proc/<pid>/pagemap`, using the `PagemapEntry` struct to extract the 54-bit page frame number (PFN)
- **SizeOf** -- demonstrates the difference between `sizeof` and `strlen` for character arrays vs. string literals
- **KernelTable** -- explores process forking and virtual memory inheritance, showing how parent and child processes share the same virtual address space layout but have independent memory contents

## Technologies

| Component | Technology |
|-----------|------------|
| Language | C99 with GNU extensions (`_GNU_SOURCE`) |
| Assembly | Inline x86/x86-64 AT&T syntax (speculative load gadgets) |
| Timing | `rdtscp` / `rdtsc` hardware timestamp counters |
| Cache manipulation | `clflush` (via `_mm_clflush` intrinsic) |
| Signal handling | `sigaction` with `SA_SIGINFO` for SIGSEGV recovery |
| CPU pinning | `sched_setaffinity` to pin to CPU 0 |
| Build system | CMake 3.13+ |
| Target architectures | x86-64, i386 |

## Project Structure

```
CMakeLists.txt                          # Build configuration (13 targets)
Meltdown/
  paboldin/
    script.c                            # Meltdown PoC: reads kernel memory via speculative execution
    Makefile.txt                        # Standalone Makefile with -O2 -msse2 flags
    run.sh                              # Locates linux_proc_banner and runs the exploit
    detect_rdtscp.sh                    # Generates rdtscp.h with the appropriate timing function
                                        #   (rdtscp if supported, otherwise rdtsc + mfence)
Spectre/
  jann_horn_et_al/
    script.c                            # Canonical Spectre v1 PoC (bounds check bypass)
  rootkea/
    script.c                            # Spectre v1 variant with custom secret string
  rios0rios0/
    attacker.c                          # Cross-process Spectre attacker using /proc/pagemap
    victim.c                            # Victim process (single-shot, exits after one call)
    victim2.c                           # Victim process (loops, prints PID and vaddr)
Tests/
  CacheTime.c                           # Cache hit/miss timing measurement
  CacheTime2.c                          # Extended cache timing with L1/L2/L3 size constants
  VirtAddress.c                         # Virtual-to-physical address translation via pagemap
  SizeOf.c                              # sizeof vs strlen demonstration
  KernelTable.c                         # Fork-based virtual memory inheritance test
```

## Technical Details

### Flush+Reload Side Channel

All exploits use the Flush+Reload technique:
1. **Flush** -- evict all 256 entries of a probe array from the cache using `clflush`
2. **Speculate** -- trigger speculative execution that indexes into the probe array using the secret byte as the index (multiplied by 512 or 4096 to span cache lines)
3. **Reload** -- time access to each of the 256 probe array entries using `rdtscp`; the entry that loads significantly faster reveals the secret byte value

### Branch Predictor Mistraining (Spectre)

The Spectre attacks mistrain the branch predictor with 5 in-bounds calls per 1 out-of-bounds call (30 iterations, `j%6==0` selects the attack). Bit-twiddling (`x = ((j%6)-1) & ~0xFFFF; x = x | (x >> 16)`) avoids conditional jumps that would retrain the predictor.

### Meltdown Signal Recovery

The Meltdown exploit handles the inevitable SIGSEGV from the faulting kernel memory access by installing a `sigaction` handler that modifies the instruction pointer (`RIP`/`EIP`) in the signal context to jump past the speculative gadget, allowing execution to continue and the cache side-channel to be measured.

## Installation

```bash
git clone https://github.com/rios0rios0/speculative-execution.git
cd speculative-execution
mkdir build && cd build
cmake ..
make
```

### Meltdown (standalone build)

```bash
cd Meltdown/paboldin
make -f Makefile.txt
./run.sh
```

## Usage

### Spectre (Jann Horn)

```bash
./build/jan_horn
# Reads the secret string "The Magic Words are Squeamish Ossifrage."
# via speculative execution, printing each byte with confidence scores
```

### Meltdown

```bash
cd Meltdown/paboldin
./run.sh
# Locates linux_proc_banner in kernel memory, attempts to read 10 bytes
# Reports VULNERABLE or NOT VULNERABLE
```

### Cache Timing Test

```bash
./build/cache_time
# Prints CPU cycle counts for accessing 10 array elements,
# showing the difference between cached (~5-30 cycles) and uncached (~100-300 cycles) accesses
```

**Warning:** These exploits require specific unpatched CPU/kernel configurations to succeed. Modern systems with KPTI (Kernel Page Table Isolation) and Spectre mitigations will report NOT VULNERABLE.

## Contributing

Contributions are welcome. Please open an issue or submit a pull request.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build

```bash
mkdir build && cd build && cmake .. && make
```

Requires GCC 7+, CMake 3.13+, Linux x86-64. Produces 10 binaries in `build/`. No CI/CD. No automated tests -- all verification is manual.

Meltdown has a standalone build: `cd Meltdown/paboldin && make -f Makefile.txt && ./run.sh`

## Architecture

All exploits use the Flush+Reload side channel: flush a 256-entry probe array from cache, trigger speculative execution that indexes it by the secret byte (x512 or x4096 stride), then time reloads with `rdtscp` to find the cached entry.

- **Spectre v1** mistrains the branch predictor with 5 in-bounds accesses per 1 out-of-bounds attack. Bit-twiddling avoids conditional jumps that would retrain the predictor.
- **Meltdown** uses `sigaction` to recover from SIGSEGV after reading kernel memory; modifies `RIP`/`EIP` in the signal context to continue past the faulting instruction.
- **Cross-process Spectre** translates virtual addresses via `/proc/<pid>/pagemap` (`CAP_SYS_PTRACE` required on kernels >= 4.0).

## Conventions

- C99 (`set(CMAKE_C_STANDARD 99)`). Only the Meltdown source (`Meltdown/paboldin/script.c`) defines `_GNU_SOURCE`, for `ucontext`/`REG_RIP`; the Spectre and test sources do not.
- Inline assembly uses AT&T syntax.
- Probe arrays: 256 entries x 512 or 4096 bytes to span separate cache lines.
- Timing: prefer `rdtscp` (serializing) over `rdtsc`.
- No dynamic allocation in speculative paths.
- Signal handlers must be async-signal-safe -- only modify `uc_mcontext` registers.

## Caveats

Modern kernels with KPTI and Spectre mitigations prevent successful exploitation. The code demonstrates the technique on unpatched systems.

<!-- chlog:start -->
## Changelog (chlog) — MANDATORY

If the repository you are working in uses chlog (a `.chlog.yaml` or `.chlog.yml`
config file, or a `.changes/` directory, exists at the project root), the
following is binding and ALWAYS applies: whenever you make ANY change, you MUST
create a changelog fragment as part of the same change — automatically, without
being asked, before committing.

- Do NOT edit CHANGELOG.md directly; it is generated from fragments.
- Create the fragment with:
  `chlog new --kind <Kind> --body "<imperative description>"`
- Valid kinds: Added, Changed, Deprecated, Removed, Fixed, Security
- Choose the kind that best matches the change (e.g., new feature → Added,
  bug fix → Fixed, behavior change → Changed, removal → Removed, security fix → Security).
- If the change is backward-INCOMPATIBLE with the public API (a breaking
  change), you MUST add the `--breaking` flag:
  `chlog new --kind <Kind> --breaking --body "<description>"`.
  This is the ONLY thing that triggers a major version bump — the kind alone
  never does (per SemVer, major = incompatible change). When unsure whether a
  change breaks compatibility, ask the user instead of guessing.
- Fragments are YAML files in `.changes/unreleased/`; stage them with your commit.
- `chlog check` fails the build when a fragment is missing — never skip it.
<!-- chlog:end -->

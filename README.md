# Fix Erlang/OTP AddressSanitizer Memory Leaks

## Problem Statement

When building Erlang/OTP 26.2.5.12 with AddressSanitizer (ASAN) enabled, the build process fails due to memory leaks detected in build-time utilities. These leaks cause the Erlang compiler (erlc) to crash during compilation of the `megaco` library.

## Background

We discovered this while building an ASAN-instrumented Erlang VM for fuzzing EMQX (an Erlang MQTT broker). ASAN is needed to detect memory errors in EMQX's C NIFs during fuzzing.

## Specific Leaks Detected

### Leak 1: dyn_erl.c:209 in find_prog()
```
Direct leak of 57 byte(s) in 1 object(s) allocated from:
    #0 0x7f939ea3a77b in __interceptor_strdup ../../../../src/libsanitizer/asan/asan_interceptors.cpp:439
    #1 0x55d72815fc6a in find_prog ../unix/dyn_erl.c:209
    #2 0x55d72816015d in main ../unix/dyn_erl.c:366
    #3 0x7f939e80f249  (/lib/x86_64-linux-gnu/libc.so.6+0x27249)

SUMMARY: AddressSanitizer: 57 byte(s) leaked in 1 allocation(s).
```

**File**: `erts/etc/unix/dyn_erl.c`
**Function**: `find_prog()` at line 209
**Issue**: Result of `strdup()` is never freed

### Build Failure Location
The build fails when compiling `lib/megaco/src/flex`:
```
make[5]: *** [/build/otp_src_26.2.5.12/make/x86_64-pc-linux-gnu/otp.mk:136: ../../ebin/megaco_flex_scanner.beam] Error 1
```

The erlc compiler terminates with error when ASAN leak detection is enabled.

## Current Workaround

Set `ASAN_OPTIONS="detect_leaks=0"` during build to disable leak detection:

```dockerfile
RUN export ASAN_OPTIONS="detect_leaks=0" && \
    make -j$(nproc) && \
    make install
```

This allows the build to complete but doesn't fix the underlying issue.

## Goal

1. Fix the memory leak(s) in Erlang/OTP build utilities
2. Submit a pull request to erlang/otp
3. Enable ASAN leak detection during build (currently disabled)

## The Fix

The memory leak occurs in `erts/etc/unix/dyn_erl.c` in the `main()` function's `--realpath` code path (lines 365-373).

**Root Cause:** The `find_prog()` function returns a `strdup()`'d string that must be freed by the caller. In the normal execution path, this memory is correctly freed at line 403 with `efree(abspath)`. However, in the `--realpath` path, the function returns early without freeing the allocated memory.

**Fix:** Add `efree(abspath);` before the `return 0;` statement in the `--realpath` block.

```c
// Before (leaks memory):
printf("%s", abspath);
return 0;

// After (fixed):
printf("%s", abspath);
efree(abspath);
return 0;
```

The fix is available in:
- `fix/dyn_erl.c` - The patched source file
- `fix/0001-Fix-memory-leak-in-dyn_erl.c-find_prog.patch` - Patch for upstream submission

## Implementation Plan

### Phase 1: Reproduce and Analyze
- [x] Clone erlang/otp repository (use OTP-26.2.5.12 tag)
- [ ] Build with ASAN to reproduce the leak
- [x] Examine `erts/etc/unix/dyn_erl.c:209` (find_prog function)
- [x] Identify all strdup() calls that aren't freed
- [x] Check if there are other similar leaks in the codebase

### Phase 2: Fix the Leak
- [x] Add appropriate free() call(s) in dyn_erl.c
- [x] Ensure the fix doesn't break any error paths
- [ ] Test that the fixed code still works correctly
- [ ] Verify the leak is gone with ASAN

### Phase 3: Test Thoroughly
- [ ] Build Erlang with the fix and ASAN leak detection enabled
- [ ] Run Erlang's test suite: `make tests`
- [ ] Verify no regressions
- [ ] Check for any other ASAN warnings during full build

### Phase 4: Submit Upstream
- [ ] Create a clean commit with proper message
- [ ] Fork erlang/otp on GitHub
- [ ] Create a branch for the fix
- [ ] Submit pull request
- [ ] Respond to review feedback

## Technical Details

### Build Command That Triggers Leak
```bash
cd otp_src_26.2.5.12
export CFLAGS="-fsanitize=address -fno-omit-frame-pointer -g -O1"
export LDFLAGS="-fsanitize=address"
export ASAN_OPTIONS="detect_leaks=1"  # This will cause failure
./configure --prefix=/usr/local --enable-jit --without-javac --without-odbc
make -j$(nproc)
```

### Files to Investigate
- `erts/etc/unix/dyn_erl.c` - Primary leak location
- Look for similar patterns in:
  - `erts/etc/unix/cerl.c`
  - Other utilities in `erts/etc/`

### Resources
- Erlang/OTP repository: https://github.com/erlang/otp
- Contribution guidelines: https://github.com/erlang/otp/blob/master/CONTRIBUTING.md
- ASAN documentation: https://github.com/google/sanitizers/wiki/AddressSanitizer

## Expected Impact

**Severity**: Low (build-time only, not runtime)
**Scope**: Affects anyone building Erlang with ASAN
**Fix complexity**: Trivial (add free() call)
**Upstream interest**: Moderate (improves ASAN compatibility)

## Notes

- This is a build-time leak in a short-lived utility, not a runtime leak in the Erlang VM
- The leak doesn't affect production Erlang systems
- However, fixing it enables proper ASAN builds of Erlang, which is valuable for security research and fuzzing
- The fix is likely a one-liner adding a free() call

## Related Context

This issue was discovered while working on an MQTT fuzzing project.

Full ASAN build log available in this repository at `asan-build-failure.log`.

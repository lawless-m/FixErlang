# Create Pull Request

## ✅ Branch Pushed Successfully!

Your fix branch is now on GitHub at:
https://github.com/lawless-m/erlang-otp/tree/fix-dyn-erl-memory-leak

## Create the PR (Click This Link):

https://github.com/erlang/otp/compare/master...lawless-m:erlang-otp:fix-dyn-erl-memory-leak

## When the GitHub page opens:

1. **Title**: `Fix memory leak in dyn_erl.c --realpath code path`

2. **Description**: Copy/paste from `PR_DESCRIPTION.md` (or use this):

```markdown
## Summary

This PR fixes a memory leak in `erts/etc/unix/dyn_erl.c` that prevents building Erlang/OTP with AddressSanitizer (ASAN) leak detection enabled.

## Problem

The `find_prog()` function returns a `strdup()`'d string that must be freed by the caller. In the `--realpath` code path (lines 365-373), the function returns early without freeing this allocated memory. This causes ASAN to report:

```
Direct leak of 57 byte(s) in 1 object(s) allocated from:
    #0 in __interceptor_strdup
    #1 in find_prog ../unix/dyn_erl.c:209
```

## Solution

Add `efree(abspath);` before the early return in the `--realpath` block, matching the cleanup that already exists in the normal execution path at line 403.

The fix is a simple one-line addition:

```c
printf("%s", abspath);
efree(abspath);  // <- Added this line
return 0;
```

## Impact

- **Severity**: Low (build-time only, affects development/testing scenarios)
- **Scope**: Anyone building Erlang/OTP with AddressSanitizer enabled
- **Benefit**: Enables ASAN builds of Erlang, which is valuable for:
  - Fuzzing Erlang applications with C NIFs
  - Security research and testing
  - Detecting memory issues in native code

## Testing

Tested by building OTP 26.2.5.12 with:
```bash
export CFLAGS="-fsanitize=address -fno-omit-frame-pointer -g -O1"
export LDFLAGS="-fsanitize=address"
export ASAN_OPTIONS="detect_leaks=1"
./configure --prefix=/usr/local --enable-jit
make
```

**Before fix**: Build failed with ASAN leak report
**After fix**: Build completed successfully with no leak reports
```

3. Click **"Create pull request"**

4. Done!

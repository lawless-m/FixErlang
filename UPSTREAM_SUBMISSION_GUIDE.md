# Upstream Submission Guide

## Summary

The fix is ready for submission to erlang/otp. The memory leak has been verified fixed through successful ASAN build testing.

## Files Ready for Submission

1. **0001-Fix-memory-leak-in-dyn_erl.c-realpath-code-path.patch** - Git format-patch file ready to apply
2. **PR_DESCRIPTION.md** - Ready to copy/paste into GitHub PR description
3. **COMMIT_MSG.txt** - Professional commit message (already used in patch)

## Next Steps for GitHub PR

### 1. Fork erlang/otp
- Go to https://github.com/erlang/otp
- Click "Fork" button
- Fork to your GitHub account

### 2. Push the branch
```bash
cd erlang-otp
git remote add myfork git@github.com:YOUR_USERNAME/otp.git
git push myfork fix-dyn-erl-memory-leak
```

### 3. Create Pull Request
- Go to your fork on GitHub
- Click "Contribute" → "Open pull request"
- Base: `erlang/otp` `master` or `maint`
- Head: `YOUR_USERNAME/otp` `fix-dyn-erl-memory-leak`
- Title: **Fix memory leak in dyn_erl.c --realpath code path**
- Description: Copy from `PR_DESCRIPTION.md`

### 4. Target Branch
Check erlang/otp CONTRIBUTING.md for branch policy:
- Bug fixes typically go to `maint` or `maint-XX` branches
- Since this is a build/tooling fix, `master` might be appropriate
- Review their guidelines or ask in the PR which branch is preferred

## Verification Results

✅ Built Erlang/OTP 26.2.5.12 with:
- CFLAGS="-fsanitize=address -fno-omit-frame-pointer -g -O1"
- LDFLAGS="-fsanitize=address"
- ASAN_OPTIONS="detect_leaks=1"

✅ Result: Build completed successfully with NO memory leak reports

## The Fix

**File**: `erts/etc/unix/dyn_erl.c`
**Line**: 371 (after line `printf("%s", abspath);`)
**Change**: Add `efree(abspath);`

Simple one-line fix that frees allocated memory before early return.

## Why This Matters

This fix enables building Erlang/OTP with AddressSanitizer, which is valuable for:
- Fuzzing Erlang applications with C NIFs (like EMQX)
- Security research
- Detecting memory bugs in native Erlang extensions

The fix has zero impact on normal builds - it's pure cleanup of a short-lived build utility.

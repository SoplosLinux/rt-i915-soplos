# Changelog — rt-i915-soplos

## 1.0.0 — 2026-09-20

Initial release.

### Added

- `0001-rt-i915-7.2.patch` — lets the i915 driver build and run on
  PREEMPT_RT for Linux 7.2. Removes `depends on !PREEMPT_RT` from `DRM_I915`
  and stops i915 taking `spinlock_t` locks with interrupts disabled in the
  vblank evasion path, the display tracepoints and `execlists_dequeue_irq()`.

### Notes

- Applies clean to the 7 touched files of Linux 7.2.6, and compiles without
  errors or warnings with PREEMPT_RT and without it, for both i915 and xe.
- Not boot-tested yet.
- Soplos RT kernels 7.0.3 through 7.2.6 shipped without `CONFIG_DRM_I915`
  because of the upstream dependency; this patch is the fix for that.
- Written by Soplos, not a rebase. Not submitted upstream.

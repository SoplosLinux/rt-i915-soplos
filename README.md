# rt-i915-soplos — i915 on PREEMPT_RT patch for Soplos Linux

Makes the Intel i915 graphics driver usable on PREEMPT_RT kernels. Written
and maintained by Soplos, it is not a rebase of another project's patch.

> Not yet wired into **soplos-kernel-installer** — `core/downloader.py` still
> only downloads a third-party `rt-i915` patch when the machine doing the
> build has an Intel GPU, which never happens on the build VM. See "Status"
> below.

---

## Why this patch exists

Mainline declares `DRM_I915` as `depends on !PREEMPT_RT`. Every PREEMPT_RT
kernel therefore drops the driver silently at `make olddefconfig`, with no
error. Soplos RT kernels 7.0.3 through 7.2.6 all shipped without
`CONFIG_DRM_I915`.

A machine with an Intel iGPU then has no KMS driver: it boots past plymouth
to a black screen with a blinking cursor. Observed on a Haswell machine
(`8086:0416`) running `7.2.5-soplos-rt-v3`.

The reason for the dependency is that i915 takes `spinlock_t` locks with hard
interrupts disabled. That is fine on a regular kernel, and a
sleeping-in-atomic bug on PREEMPT_RT, where `spinlock_t` is a sleeping lock.

---

## What the patch does

- Removes `depends on !PREEMPT_RT` from `DRM_I915`.
- **Vblank evasion** (`intel_pipe_update_start/end`, the legacy cursor update,
  the scanout position query and the evade loop) runs with interrupts
  disabled and calls `intel_vblank_section_enter()`, which takes
  `uncore->lock`. On RT the section now keeps interrupts enabled. Evasion
  becomes best effort: an atomic update preempted past its vblank shows up as
  an "Atomic update failure" for that one frame, instead of sleeping in atomic
  context. Xe shares these files and is left unchanged.
- **Display tracepoints** sample the scanline in `TP_fast_assign`, which takes
  `uncore->lock`. Tracepoint probes run with preemption disabled, so on RT the
  scanline is reported as 0.
- **`execlists_dequeue_irq()`** disabled interrupts around a section that takes
  `sched_engine->lock`. On RT the lock serialises the submission and the
  tasklet is never reentered, so the interrupt disabling is skipped.

Everything else in i915 that disables interrupts only touches seqcounts or
lockdep state and needs no change. A non-RT kernel builds to the same code as
before.

---

## Patch files

| File | Kernel versions | Base |
|------|-----------------|------|
| `patches/0001-rt-i915-7.2.patch` | Linux 7.2.x | written against 7.2.6 |

One file per `major.minor` line, as with the other Soplos patch repos.

---

## Status

- **7.2.6:** the patch applies clean to the 7 touched files, and compiles
  without errors or warnings with PREEMPT_RT and without it, for both i915
  and xe. With PREEMPT_RT, `DRM_I915=m` survives `olddefconfig`. Without RT
  it expands to the same code as the original.
- **Not boot-tested.** Do not publish an RT kernel with this patch until it
  has been booted on real hardware with `DEBUG_ATOMIC_SLEEP` and
  `PROVE_LOCKING` enabled and `dmesg` checked. `soplos-kernel-installer`
  forces `DEBUG_KERNEL=n`, so a test kernel like that has to be built by hand.

### Not covered yet

- `GEM_BUG_ON(!irqs_disabled())` in `i915_request.c`, and the
  `in_atomic()` / `irqs_disabled()` checks in `intel_engine_cs.c` and
  `intel_guc.h`. The asserts are only built with `DRM_I915_DEBUG_GEM`. The
  other two are worth revisiting for GuC platforms.
- Not audited: `preempt_disable()` and `local_bh_disable()` sites in
  `intel_engine_cs.c`, `intel_reset.c`, `intel_engine_heartbeat.c`,
  `i915_gem_wait.c` and `vlv_iosf_sb.c`, and the GuC submission code.
- GVT was not compiled: the config used for the compile check did not
  enable `DRM_I915_GVT`.
- The execlists change cannot be exercised on Haswell, which is the only
  hardware available for testing so far.

---

## Applying the patch manually

```bash
cd /path/to/linux-7.2.6
patch -p1 < /path/to/patches/0001-rt-i915-7.2.patch
```

---

## Modified files

| File | Change |
|------|--------|
| `drivers/gpu/drm/i915/Kconfig` | Drops `depends on !PREEMPT_RT` |
| `drivers/gpu/drm/i915/display/intel_vblank.h` | RT-aware interrupt helpers and `intel_trace_scanline()` |
| `drivers/gpu/drm/i915/display/intel_vblank.c` | Scanout position, scanline query and evade loop use the helpers |
| `drivers/gpu/drm/i915/display/intel_crtc.c` | `intel_pipe_update_start/end` use the helpers |
| `drivers/gpu/drm/i915/display/intel_cursor.c` | Legacy cursor update uses the helpers |
| `drivers/gpu/drm/i915/display/intel_display_trace.h` | Tracepoints use `intel_trace_scanline()` |
| `drivers/gpu/drm/i915/gt/intel_execlists_submission.c` | `execlists_dequeue_irq()` skips interrupt disabling on RT |

---

## Kernel variants using this patch

Intended for every Soplos `rt` variant. It only changes behaviour when
`CONFIG_PREEMPT_RT` is enabled.

---

## License

GPL-2.0 (inherited from the Linux kernel).

# Firefox hardware video decode via AMD iGPU (REVERTED)

**Status: tried 2026-08-01, did not work, reverted same day.** Left below
for reference in case this is revisited. Current permanent state is
software decode only (`media.hardware-video-decoding.enabled = false`),
no wrapper/launcher override in place.

## Context

`media.hardware-video-decoding.enabled = false` was set 2026-08-01 to stop
Firefox RDD-process SIGSYS crashes (nvidia VDPAU driver issuing ioctls the
RDD seccomp sandbox doesn't allowlist — see memory `project_firefox_crash`).
That fixed the crashes but pushed video decode to the CPU, causing YouTube
stutter.

This box has two GPUs:
- `01:00.0` RTX 4090 (nvidia proprietary driver) — drives the actual
  display, is the crash source for hw decode.
- `12:00.0` AMD Raphael iGPU (`amdgpu` driver, kernel driver already
  loaded) — alive but idle, not driving any display. Confirmed via
  `lspci -k -s 12:00.0` and `/sys/class/drm/renderD128/device/vendor` =
  `0x1002` (AMD). Its render node is `/dev/dri/renderD128`; nvidia's is
  `/dev/dri/renderD129`.

Plan: point Firefox's hardware decode at the AMD iGPU's VA-API path
instead of nvidia's VDPAU path. Decode-only workload, doesn't touch
display output, so no conflict expected with the 4090 doing normal
rendering/compositing.

## Steps taken

1. `sudo dnf install mesa-va-drivers-freeworld libva-utils` — Fedora's own
   repo omits AMD's VA-API codec bits (patent-encumbered), RPM Fusion
   nonfree has them. Does not touch any nvidia package.
2. Verify: `vainfo --display drm --device /dev/dri/renderD128` should list
   H.264/VP9/AV1 decode profiles for the `radeonsi` driver.
3. Point Firefox at that node specifically — it does not auto-prefer an
   idle iGPU over the active discrete GPU. Set env vars for the Firefox
   launch (desktop file or shell wrapper):
   - `LIBVA_DRIVER_NAME=radeonsi`
   - `MOZ_DISABLE_RDD_SANDBOX` is NOT used — sandbox stays on, we're
     changing which GPU is decoding, not disabling the sandbox that
     caused the original crash.
   - Device selection pinned so libva opens `renderD128` and not `129`.
4. Re-enable in `about:config` (profile `c4079wj3.adminbill`):
   `media.hardware-video-decoding.enabled = true`,
   `media.rdd-ffmpeg.enabled = true` (or equivalent ffmpeg/vaapi pref).
5. Test YouTube playback, confirm via `about:support` → Media section
   that the decoder backend shows `radeonsi`/AMD, not nvidia, and watch
   `journalctl` for any RDD SIGSYS recurrence.

## Rollback

If this causes crashes, graphical glitches, or anything else weird:

1. Revert the about:config pref:
   `media.hardware-video-decoding.enabled = false` in
   `~/.mozilla/firefox/c4079wj3.adminbill/prefs.js` — this is the
   known-good, confirmed-stable (if stuttery) state from earlier
   2026-08-01.
2. Remove/unset the `LIBVA_DRIVER_NAME` and render-node env var overrides
   from the Firefox launch wrapper/desktop file.
3. No need to uninstall `mesa-va-drivers-freeworld` / `libva-utils` —
   inert if Firefox isn't configured to use them, safe to leave.
4. Restart Firefox.

## Outcome (2026-08-01)

Did not work. After completing all the steps above and testing YouTube
playback:

- `nvidia-smi dmon -s u` showed the nvidia **decoder engine flat at 0%**
  during playback. The GPU activity visible in monitoring tools (`sm`/
  `mem` at 17-42%) was normal display-compositing load, not decode — the
  4090 drives the monitor regardless of which chip decodes, so it always
  shows some activity during video playback.
- The RDD (decode) process had **zero GPU device file descriptors open**
  — not the AMD node, not nvidia, nothing. It was running pure software
  decode the whole time (~18.5% CPU on that one process confirms it).
- Conclusion: Mozilla's own gfx feature blocklist appears to suppress
  hardware video decode outright on this system, most likely keyed on
  nvidia being the primary/display adapter, independent of the
  `LIBVA_DRIVER_NAME`/`LIBVA_DRM_DEVICE` env vars used here — those only
  influence libva's own auto-device-pick, and decode never got far enough
  to reach that logic.
- Bypassing the blocklist (`media.hardware-video-decoding.force-enabled`)
  was considered but **not attempted** — real risk that Firefox's GPU
  selection opens a specific render node itself (ignoring the env
  override) and picks nvidia again, reintroducing the original SIGSYS
  crash. Decided not worth the risk; reverted instead.

## Revert steps taken

1. Closed Firefox gracefully (`SIGTERM`, confirmed exit).
2. Set `media.hardware-video-decoding.enabled = false` in
   `~/.mozilla/firefox/c4079wj3.adminbill/prefs.js`.
3. Deleted `~/.local/share/applications/org.mozilla.firefox.desktop`
   (launcher override) and `~/.local/bin/firefox-amd-decode` (wrapper
   script) — no trace of either left.
4. Relaunched via plain `/usr/bin/firefox`, confirmed no `LIBVA_*` env
   vars present on the running process.
5. Left `mesa-va-drivers-freeworld` / `libva-utils` packages installed —
   inert, harmless, no need to remove.

Current permanent state: software decode only, accepting the CPU-decode
stutter as the tradeoff for RDD-crash stability. Do not consider the
original crash fix "resolved" without 2-3 weeks of confirmed crash-free
video playback, per the standing rule in `project_firefox_crash` memory
(an earlier "resolved" note on that crash was wrong once already).

## If revisiting this later

The actual open questions are:
- Can the gfx hw-decode blocklist be identified/inspected (e.g. via
  `about:support` → Graphics, or `MOZ_LOG=PlatformDecoderModule:5`) to
  confirm it's really nvidia-primary-GPU-keyed and not something else?
- Does Firefox's RDD process pick a render node itself (bypassing
  `LIBVA_DRM_DEVICE`), and if so, is there a Firefox-level pref (not a
  libva env var) to pin the decode GPU explicitly?
Both need answering before trying `force-enabled` again — otherwise it's
a coin flip on reintroducing the original crash.

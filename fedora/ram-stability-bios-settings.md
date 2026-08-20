# Memory + CPU tuning — 4 mismatched DIMMs (6 ranks) on B650

**Resolved 2026-08-18.** Settled on **96 GB @ 3600, all 32 threads, verified.**
Earlier drafts of this file carried two conclusions that turned out wrong ("a bad DIMM", then
"4400 is the answer") — both are corrected below rather than deleted.

Hardware: ASRock B650 PG Lightning (BIOS 4.10) · Ryzen 9 7950X (16C/32T, 2 CCDs) · 96 GB non-ECC =
2×32 GB `CMK64GX5M2B5600C40` (**dual-rank**, CL40) + 2×16 GB `CMK32GX5M2B5600C36` (**single-rank**,
CL36) = **4 DIMMs, 6 ranks on 2 channels**. The kits differ in rank structure, CAS latency *and*
density, so each channel carries one dual-rank and one single-rank module — asymmetric loading
against a single common termination profile.

## Current setting

**96 GB @ 3600**, DRAM Frequency manual, EXPO disabled, everything else Auto.
Verified 10 min at full 32 threads: **37,314 MB/s, 0 errors**.
**The multi-hour soak is still outstanding.**

## The finding

**Four populated slots (6 ranks) cannot hold 4800 MT/s on this board. Neither kit is faulty — do not RMA.**

At 4800 each bit gets ~208 ps; at 3600, ~278 ps. Capacitive load plus stub reflections from a
daisy-chain board with **all four slots filled** consume more of that budget than remains — the
populated-slot count matters more than the rank count here. The data line with the least slack — bit 17 here — goes negative first and fails
repeatably, which imitates a dead chip exactly.

## Verified configurations — note the thread count column

| configuration | speed | threads | result | bandwidth |
|---|---|---|---|---|
| all 4 DIMMs | 4800 | **32** | **FAIL** 49 s, then 100 s | — |
| **all 4 DIMMs** | **3600** | **32** | **PASS** ← current | **37,314 MB/s** |
| 2×16 GB alone | 4800 | 32 | PASS | — |
| 2×32 GB alone | 4800 | 32 | PASS | 27,555 MB/s |
| 2×32 GB alone | 3600 | 32 | PASS | 20,564 MB/s |
| 2×32 GB alone | 5600 EXPO | — | **no POST** | — |
| all 4 DIMMs | 4400 | 8 | pass — **INVALID** | 45,506 |
| all 4 DIMMs | 3600 | 8 | pass — **INVALID** | 31,044 |

The two 8-thread rows were measured while a BIOS failsafe had silently crippled the CPU (below).
They are kept only as a warning; they prove nothing.

**Measured scaling** (2 DIMMs, 32t): +33% clock → +34% bandwidth, near linear. So 3600→4400 on
four DIMMs ≈ +22%. Also note **4 DIMMs @3600 (37.3 GB/s) beats 2 DIMMs @4800 (27.6 GB/s)** — more
ranks means more bank parallelism, recovering most of what the lower clock costs.

## What 3600 costs in practice

Near zero here. LLM weights live in the 4090's VRAM; ComfyUI is GPU-bound; desktop and browser
work is imperceptible. Gaming at 2560×1440 with a 4090 is GPU-bound (1–5%). Compilation 2–5%.
Memory bandwidth would only bind on **CPU** inference, which this box doesn't do.

## The CPU failsafe incident — read this before diagnosing anything

**A failed POST silently disabled one CCD and SMT. 32 threads became 8, with no warning, no
message, and no log entry.** It survived multiple reboots and BIOS edits unnoticed for ~2.5 hours,
and invalidated every test run during that window.

Triggered by the 2×32 GB EXPO-5600 no-POST attempt → ASRock failsafe recovery.

Symptoms:
```
nproc                 8          (should be 32)
Core(s) per socket    8          Thread(s) per core: 1
L3 cache              32 MiB (1 instance)      7950X has 64 MiB across 2 CCDs
sensors               no Tccd2                 second chiplet powered off
smt control           notsupported             firmware not exposing SMT
/proc/cmdline         clean                    not a maxcpus/nosmt issue
```

**Not hardware damage:** zero MCEs, surviving cores boosting 4.9 GHz, temps cool. Bad memory
cannot damage a CPU — signal-integrity failures are *timing* faults with no overvoltage, and VSOC
was never raised. Silicon damage is messy and throws MCEs; this was a clean chiplet-granularity
disable, exactly the granularity a BIOS option operates at.

**Fix: Load Optimized Defaults** (F9, or Exit → Load UEFI Defaults), then re-set DRAM frequency.
Restored 32 threads, 64 MiB L3 (2 instances), Tccd2 reporting. Hunting SMT/Core Control in AMD CBS
was unnecessary.

## Verification standard

```
sudo stressapptest -M 65536 -s 600 -W 2>&1 | tee ~/sat-$(date +%m%d-%H%M).log
```

**Before trusting any run, confirm BOTH dimensions:**
```
nproc                                             # expect 32
sudo dmidecode -t memory | grep 'Configured Memory Speed'
```

Ten minutes catches gross failure. **It is not a verdict** — this hardware passed months of
ordinary use at 4800 while silently corrupting data. Trusting a setting needs **4+ hours, ideally
an overnight `-s 28800` run on an idle machine**. Any `Hardware Error` or `miscompare` kills that
config. Do not use the machine during a run: a test holding 64 of 93 GiB crashes healthy
applications and reads as a hardware symptom.

## Filesystem: clean — the scrub errors were false positives

The first scrub (run while memory was failing) reported 21 checksum errors including one
"uncorrectable" against `~/backup/projects-20260331.tar.gz`. A **read-only** scrub on a stable
config later found **zero errors across 1.47 TiB**, and that tarball reads fully from disk with
caches dropped and passes `gzip -t`. Data profile is `single`, so btrfs could never have repaired
it — passing now proves it was never corrupt. The scrub was checksumming **in faulty RAM**.

**Never trust a scrub run on suspect memory, in either direction.** Use
`sudo btrfs scrub start -r /` (read-only) when memory is in doubt — it detects without writing,
so a bit-flip during "repair" can't overwrite a good block.

## Diagnostic gotchas learned the hard way

- A **single consistent failing bit does not mean a bad chip** — a marginal bus fails one lane
  first too. Only swapping modules distinguishes them.
- A **no-POST does not mean a dead stick.** Hand-tuned settings built for one memory config,
  applied to another, will refuse to boot. Set the memory section to **Auto** before concluding.
- **Auto picks conservative speeds silently.** Always confirm the trained speed before comparing runs.
- On **non-ECC** memory, "no MCE/EDAC errors" is not evidence of health.
- **Verify thread count and memory speed together** — one silent BIOS change invalidated hours.

## Outstanding

- Overnight soak at 3600; one miscompare → drop lower
- No backup or git-mirror pushes to server1 until that soak passes
- Optional: retest 4400 at 32 threads if the ~22% is ever wanted (low value for this workload)
- Original BIOS tweak values were never photographed and are gone (BIOS was reset)

Phone card: https://claude.ai/code/artifact/77e5dd18-48a6-4ce0-8265-f0cd1cec7a37
Memory: `project_ram_mixed_kits`

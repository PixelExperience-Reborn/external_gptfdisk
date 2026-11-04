# P2 — Android17 GPT fdisk: recovery interface and source owner

**Date:** 2026-11-04. **Status:** reviewed source/API boundary only; Android17 recovery is not compiled or tested.

## Provenance

| Project | Revision | Evidence |
| --- | --- | --- |
| Historical PE `external_gptfdisk`, branch `seventeen` | `b09f809a6688356fe79c07219a279efee5645d7f`, 2024 | PE/Android14 inherited `sgdisk` and `libgptf`, added a `libsgdisk` shared library that renames `main` to `sgdisk_main`, then made it visible to `//bootable/recovery/volume_manager`. |
| AOSP17 `platform/external/gptfdisk`, tag `android-17.0.0_r1` | `e6867acc464878771f1cc139e14e058238ace238`, tree `1a8cb9d96252b68e3e6686e543f79c6b628bed40` | Official `Android.bp` exports an `sgdisk` binary, a `libgptf` static library, and `libgptf_fuzzer_lib`. It does **not** define the historical `libsgdisk` entry point. |
| AOSP17 effective checkout intent | `external/gptfdisk` = `platform/external/gptfdisk` | Staged PE reference manifest selects official AOSP17 owner. |

Official: https://android.googlesource.com/platform/external/gptfdisk/+/refs/tags/android-17.0.0_r1/Android.bp

## Historical delta is **not redundant for every recovery implementation**

PE commit [`b09f809`](https://github.com/PixelExperience-Reborn/external_gptfdisk/commit/b09f809a6688356fe79c07219a279efee5645d7f) specifically exposed `libsgdisk` to the former `bootable/recovery/volume_manager`. Historical `Android.bp` also set `recovery_available: true` for `sgdisk` and `libgptf` and built `libsgdisk` by compiling `sgdisk.cc` with `-Dmain=sgdisk_main`.

That surface is not provided by the official Android17 `Android.bp` inspected in this audit. **Do not silently promise compatibility** with a PE/third-party recovery consumer. Conversely, do not import the 2024 fork into the base platform just to expose a deprecated interface.

## Architecture decision

1. **Default platform owner stays AOSP17**; do not activate historical `external_gptfdisk` in any staged base manifest.
2. At the recovery component stage, inspect its exact `Android.bp` and C++ calls for `libsgdisk` / `sgdisk_main` / `libgptf`. If absent, the old shim is unneeded.
3. If a legitimate recovery consumer exists, evaluate migration to current supported `sgdisk` invocation or create a **small, isolated compatibility module** against AOSP17 `libgptf` with symbol visibility, partition-safety and error-path tests. No `main`-renaming hack without build validation.
4. Device testing must cover GPT read/backup/restore and negative cases on disposable images; do **not** write to actual user partition tables or unlock critical partitions for this audit.

**Verified:** historical `Android.bp` and commit diff, official AOSP17 target definitions, manifest source owner. **Unverified:** effective Soong recovery consumers, compilation, partition image round-trip, recovery boot on hardware.

This is a dependency flag for later recovery work, **not a reason to enable older code unconditionally**.

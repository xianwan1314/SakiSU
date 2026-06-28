# DEVLOG: vivo / iQOO compatibility

> This file records the vivo/iQOO behavior that is implemented in code. It is
> not a design wish list and should be kept in sync with `boot_patch.rs`,
> `KsuCli.kt`, and `Install.kt`.

## Scope

The manager-side switch is intentionally simple: **"去除vr或适配vivo特性"**.
When enabled, SakiSU uses one backend command, `boot-patch-vivo`, and lets
`ksud` decide what the selected image actually needs.

vivo/iQOO GKI devices commonly need two independent operations:

| Image | Action |
|---|---|
| `init_boot.img` / compatible boot ramdisk | Inject `kernelsu.ko` and `ksuinit`; use the `_vivo` KMI/LKM variant. |
| `vendor_boot.img` | Remove `vr.ko` and its `modules.*` references only; do not inject KernelSU files. |

Turning the vivo switch off restores the normal SakiSU flow.

## Backend

`userspace/ksud/src/boot_patch.rs` keeps vivo handling partition-agnostic:

1. `patch_vivo()` adds `vr.ko` to `remove_module` if it is not already present.
2. `patch_vivo()` then calls the normal `patch()` path.
3. `patch()` loads the ramdisk cpio before loading embedded LKM resources.
4. If the cpio contains `lib/modules/*.ko`, the image is treated as
   `vendor_boot` and `no_install` is enabled automatically.
5. `remove_vendor_modules()` discovers every `lib/modules/<version>-gki/` root
   dynamically and cleans `modules.load`, `modules.dep`, `modules.softdep`, and
   `modules.load.recovery`.

The backend also exposes:

```text
ksud boot-info classify-image <image>
```

It prints one of:

- `vendor_boot`
- `init_boot`
- `unknown`

The classification uses the same boot-image and cpio parsing strategy as the
patch path, so the manager can make UI decisions without duplicating parser
logic.

## Manager Flow

`manager/app/.../ui/util/KsuCli.kt` copies a selected image into cache and calls
`boot-info classify-image` when classification is needed.

`manager/app/.../ui/screen/Install.kt` uses the result like this:

- SelectFile + vivo ON + classified `vendor_boot`: skip the KMI dialog and run
  rmvr through `boot-patch-vivo`.
- SelectFile + vivo ON + classified `init_boot` or `unknown`: keep the KMI
  dialog. This preserves the safe path for LKM injection.
- DirectInstall + selected partition `vendor_boot`: skip the KMI dialog.
- Other GKI install paths with no custom `.ko`: keep the KMI dialog.

When vivo mode is enabled and a KMI string is selected, `installBoot()` appends
`_vivo` unless the selected KMI already has that suffix. This is harmless for
vendor_boot rmvr because the backend skips LKM resource loading on that path.

## Expected User Flow

```text
init_boot.img:
  Install -> SelectFile -> choose init_boot.img
  -> choose androidXX-Y.Z_vivo KMI
  -> boot-patch-vivo injects KernelSU LKM

vendor_boot.img:
  Install -> SelectFile -> choose vendor_boot.img
  -> no KMI dialog
  -> boot-patch-vivo removes vr.ko and modules.* references only
```

## CI Signing State

The current signing path is intentionally left as-is:

- `build-manager.yml` generates or materializes one keystore for the batch.
- The certificate size/hash is passed to LKM workflows as
  `KSU_EXPECTED_PR_BUILD_SIZE/HASH`.
- `kernel/manager/apk_sign.c` requires the v2 certificate to match a trusted
  key; if v3 or v3.1 blocks are present, their certificates must match too.
- `userspace/ksud/src/apk_sign.rs` mirrors this relaxed policy for
  manager-side signature reporting.

Do not reintroduce the old "force v2-only signing" assumption unless the kernel
verifier policy changes again.

## Lessons Kept

- Do not gate rmvr on `partition == "vendor_boot"`; SelectFile does not expose a
  partition dropdown.
- Do not make the vivo switch mean "rmvr only"; init_boot still needs LKM
  injection.
- Do not hard-code `lib/modules/6.1-gki`; vivo devices also ship 5.10, 5.15,
  6.6, and other layouts.
- Keep the exact `base.apk` match in `kernel/manager/throne_tracker.c`; prefix
  matching accepts `base.apk.prof` and breaks manager detection.

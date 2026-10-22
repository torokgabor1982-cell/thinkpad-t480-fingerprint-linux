# Build and reproduction notes

This is a developer-oriented prototype, not an installer. Keep an alternative password login available. The public patch deliberately excludes the original machine-specific importer, device-record numbers, local data search paths and private installation scripts.

## Exact tested baseline

- libfprint MR !626: https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/626
- Base commit: `1cf22b89c335b1b63efcccb1113234be9ba2e9d0`
- Patch: [legacy-validity-identities.patch](patches/legacy-validity-identities.patch)

The patch modifies only `validity_verify.c` and `tests/test-validity.c`. It contains reviewable experimental representation choices, not a promise of acceptance upstream.

## Inspect and build

Fetch the MR from the official libfprint repository. For example, in an isolated checkout:

```bash
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git libfprint-validity
cd libfprint-validity
git fetch origin refs/merge-requests/626/head
git checkout --detach 1cf22b89c335b1b63efcccb1113234be9ba2e9d0
git apply --check /path/to/legacy-validity-identities.patch
git apply /path/to/legacy-validity-identities.patch
meson setup build \
  -Ddrivers=validity \
  -Ddoc=false \
  -Dintrospection=false \
  -Dinstalled-tests=false \
  -Dudev_rules=disabled \
  -Dudev_hwdb=disabled
meson compile -C build
./build/tests/test-validity
```

Fedora build dependencies used locally included meson, ninja-build, gcc-c++, pkgconf-pkg-config, glib2-devel, libgusb-devel and openssl-devel. Exact package names differ elsewhere. The test program should report 202 passing tests at this baseline.

## Before any hardware run

The development driver also requires external device data files, and may require a firmware file. These are not supplied here. Do not install Ubuntu binary library packages on Fedora simply to obtain those files.

Review the development driver's pairing behavior first. The local hardware trial set `pair_state.check_only = TRUE` in `open_pair()` so it would refuse new pairing instead of initializing/erasing the sensor. This is a separate local safeguard and NOT part of the identity patch. Do not infer that arbitrary driver probes are non-destructive.

Stop other reader daemons before an isolated hardware trial. Do not use factory-reset, clear-storage or enrollment commands as a diagnostic shortcut on shared Windows/Linux storage.

## What to look for

Run the isolated identify example with relevant logging only after reviewing the preceding requirements. The confirmed diagnostic symptom was a chip-reported match followed by rejection during gallery correlation. Preserve the distinction between:
- a real scan/matcher rejection;
- a matched record with an incompatible binary identity;
- a matched legacy subtype missing from the gallery mapping;
- successful device identification that still has no authorized Linux user binding.

A successful example is not equivalent to a working PAM integration.

## Linux user binding and deployment

A device may contain other people's prints. A local import must include only records whose ownership has been established and explicitly authorized. Do not copy the original machine's database record IDs. Do not accept all device matches as the current user.

For this machine, a private library directory and a fprintd-only systemd library-path override avoided replacing the packaged library. Per-user serialized references were imported separately. Synthetic UI slots were distinct from the raw device subtype carried in the patched metadata.

This repository does not provide a universal migration/import utility. Future fprintd/libfprint updates can require rebuilding or migration; no long-term update-compatibility claim is made.

## Do not post raw logs

Some debug modes dump private key material, SID data, identifiers or biometric information. Report sanitized decision messages, software versions and synthetic fixtures instead.

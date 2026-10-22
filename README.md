# ThinkPad T480 / Synaptics 06cb:009a: legacy fingerprint identity correlation fix

Tested on Fedora 44 KDE, Plasma 6.7.5, fprintd 1.94.5 and kernel 7.2.11-200.fc44.x86_64, on 2026-10-22.

## Scope
This is a single-machine case study and a prototype patch against the development Validity driver in [libfprint MR !626](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/626), commit 1cf22b89c335b1b63efcccb1113234be9ba2e9d0. It is not a claim that released upstream libfprint supports this device, a general installer, or a security audit.

## Confirmed failure
The reader held fingerprints created by Windows and python-validity. The native development driver opened the reader, and diagnostics showed:
```
Identify: chip matched user_dbid=<record> subtype=245,
which is no print in the gallery -- no match
NOT IDENTIFIED!
```
A chip-level match was being rejected during software correlation.

### Binary identities
list_get_user_recv converted legacy identity bytes to a GVariant string using g_strndup. Binary Windows SID identities can contain embedded NUL bytes. This does not preserve the complete identity bytes and length expected by print_owns_record.

### Legacy finger subtypes
The finger mapping only covered subtypes 1–10. This reader also held subtypes 245–248 (0xf5–0xf8). Reducing these to an unknown anatomical finger loses the original subtype.

## Prototype correction
The patch preserves identity bytes and original subtype in a (qay) GVariant for legacy records, while retaining textual identity compatibility. Authentication still requires the returned user record ID, exact identity bytes and length, exact subtype, and presence of that subtype in the returned record. An arbitrary chip match or first gallery entry must not be accepted as the requested user's fingerprint.

The patch includes an intermediate ay compatibility branch and diagnostic record descriptions from the local experiment. Maintainers may prefer a cleaner versioned print representation and further migration tests before merging.

## Validation
- 202 Validity tests passed: the existing 201 plus one new regression test.
- The new test covers a correct binary identity/subtype-245 match, wrong subtype, wrong returned user record ID, and changed identity bytes.
- Six consecutive same-finger identify trials succeeded after the fixes, with matches alternating between two legacy records.
- Left index, right index and right thumb worked in the example program.
- After importing local references for the user's known records, official Fedora fprintd with the private patched library worked for sudo and KDE screen unlock, including after reboot (user-reported).
- One system-service trial had 3 successes in 4 attempts. This is not a recognition-rate benchmark.
- Final Windows operation was not re-tested after the Linux changes.
- Compatibility with future distribution/library updates remains untested.

## Troubleshooting path
python-validity/open-fprintd could enroll, but verification was unreliable. USB disconnections were also observed. Re-enrollment, recalibration and disabling autosuspend did not reliably resolve the problem. Not every earlier symptom was proven to share the native driver's correlation bug.
A separate experimental Rust implementation established TLS successfully but did not produce reliable identification in these trials.

## Integration lessons
Use fprintd-verify -f any when testing every imported reference. Without -f any, the official CLI selected the first stored finger, making correct matches to other records appear to fail.
Legacy reference slots may have anatomical labels that do not reflect their original Windows fingers.
Machine-specific record IDs must not be copied to another reader. Assigning shared device records to a Linux account requires verifying their ownership.
The final setup did not require factory-resetting or erasing the existing sensor records.
The local hardware trial additionally forced pairing into check-only mode. That safety change is NOT included in the identity-only patch; review automatic pairing before using the development driver on valuable existing device data.

## Login and KWallet
The final working configuration retained password-based disk unlock and initial graphical login, with fingerprints for sudo and screen unlock. An experimental initial-login PAM override was reverted: it triggered the fingerprint flow after submitting the password field and left KWallet locked.
Do not treat this report as a recommendation to copy PAM configuration from another distribution.

## Privacy and reproducibility
No biometric images/templates, private keys, real SIDs, serial numbers, personal paths or raw debug dumps are included. The synthetic SID bytes in the regression test are test fixtures, not a real user's identity.
Do not publish the original conversation, full debug logs or the machine-specific installation/backup directory.
The patch needs maintainer review before being treated as a supported fix.

## Related projects
- [libfprint MR !626](https://gitlab.freedesktop.org/libfprint/libfprint/-/merge_requests/626)
- [python-validity](https://github.com/uunicorn/python-validity)
- [Experimental Rust driver](https://github.com/wdes/Synaptics-Metallica-MIS-rs)


## Files in this repository
- [Prototype patch and regression test](patches/legacy-validity-identities.patch)
- [Build and reproduction notes](REPRODUCING.md)
- [Magyar esetleírás](README.hu.md)

This investigation and prototype were developed with AI assistance and validated by the laptop owner through hardware trials. This repository is not affiliated with Lenovo, Fedora or the libfprint project. It does not redistribute the external device data package or sensor firmware.

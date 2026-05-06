# copyfail-rs v0.1.1

Bugfix release. Fixes a false-negative on RHEL-family kernels (RHEL 9, Rocky 9, AlmaLinux 9, CentOS Stream 9, Oracle Linux 9, Amazon Linux 2023).

## Issue fixed

[#1](https://github.com/diemoeve/copyfail-rs/issues/1): false negative from RHEL 9 (reported by @badfiles).

On kernels built with `CONFIG_CRYPTO_USER_API_AEAD=y` (built-in instead of module), the host-vulnerability gate aborted with exit 3 ("host kernel does not appear vulnerable") even though `--mode detect --check` correctly reported `VERDICT: VULNERABLE`. The two modes disagreed on the same kernel.

### Root cause

`host_kernel_appears_vulnerable()` checked only `algif_aead` in `/proc/modules` OR the `authencesn` template in `/proc/crypto`. Neither signal fires on RHEL-family kernels:

- `algif_aead` is compiled into the kernel (`=y`), so it never appears in `/proc/modules` (modules listing only enumerates loaded `.ko` files).
- `authencesn(hmac(sha256),cbc(aes))` is a lazy-instantiated template, registered on first userspace AF_ALG bind, absent until then.

Both signals false, gate trips, exit 3.

The same broken signal pair was also used in `SuVector::applicable()` and `PasswdVector::applicable()`, so both vectors reported NOT APPLICABLE on the same hosts.

### Fix

The pre-exploit gate (and the `su` / `passwd` vector applicability checks) now consult the detect-mode verdict, which reads `/boot/config-$(uname -r)`. `CONFIG_CRYPTO_USER_API_AEAD=y` returns `Verdict::Vulnerable`, gate returns true. Falls back to the legacy OR-of-signals when the verdict is `Unknown` (containers without `/boot`, locked-down kernels).

### Verification

- 7 new unit tests in `tests/detect_check_test.rs` covering the gate behaviour:
  - `gate_true_on_rhel9_builtin_with_lazy_template`: exact issue #1 fingerprint
  - `gate_true_on_module_loaded`, `gate_false_when_mitigated`, `gate_false_when_config_n`
  - `gate_legacy_fallback_when_unknown_with_module_loaded`, `gate_legacy_fallback_when_unknown_with_template_present`, `gate_false_when_unknown_and_no_signals`
- End-to-end smoke test on Rocky 9.7 (kernel `5.14.0-611.5.1.el9_7`):
  - v0.1.0: `--mode exploit` returns exit 3 (reproduces #1)
  - v0.1.1: `--vector list` reports "Kernel vulnerable: yes", `su` + `passwd` APPLICABLE; `--vector auto --dry-run` selects "would execute: su"

## Known limitation surfaced during testing

On Rocky 9 with `authselect`-managed minimal `system-auth` (no `default=bad` line, no `faillock authfail` line), the `pam` vector reports NOT APPLICABLE. The current Fedora killshot heuristic targets the older PAM-with-faillock stack. Tracked as a follow-up. `su` and `passwd` vectors remain available on these hosts.

## Pre-built binaries

| Asset | Target | Size |
|-------|--------|------|
| `copyfail-x86_64-musl` | x86_64-unknown-linux-musl | 108 KB |
| `copyfail-aarch64-musl` | aarch64-unknown-linux-musl | 97 KB |
| `copyfail-armv7-musleabihf` | armv7-unknown-linux-musleabihf | 87 KB |
| `checksums.txt` | sha256 sums | (267 B) |

```
$ sha256sum -c checksums.txt
copyfail-x86_64-musl: OK
copyfail-aarch64-musl: OK
copyfail-armv7-musleabihf: OK
```

## Upgrade

```
$ sha256sum copyfail-x86_64-musl
e241d17b588dfd939011f72efadadeb66b5c3a08ef8692cebf441969b73568a5  copyfail-x86_64-musl
```

Drop-in replacement. No CLI surface changes, no breaking changes. Anyone affected by issue #1 should upgrade.

## Credits

- Bug report and reproduction: @badfiles (GitHub issue #1)

## License

MIT.

# Security Policy

## Supported versions

Security fixes are provided for the latest released version of PathForge (Base)
and PathForge Pro on the Unity Asset Store.

| Version | Supported |
|---|---|
| Latest | Yes |
| All older releases | No |

## Reporting a vulnerability

**Do not open a public issue for security problems.**

1. Use GitHub's private vulnerability reporting:
   **Report a vulnerability** (Issues tab → *Security* → *Report a vulnerability*).
   This sends the report to the maintainers only.
2. Fallback: [info@birdblu.com](mailto:info@birdblu.com) with **Security** in
   the subject line.

PathForge ships a native plugin (Rust core behind a C ABI). Reports on the
native layer — loader, FFI boundary, or any memory-safety issue — are handled
the same way.

## What to include

- Asset and version, Unity version, platform, scripting backend (Mono/IL2CPP).
- Minimal repro steps and the observed effect.
- Your preferred contact for the response (GitHub handle or E-Mail).

## Response

Reports are acknowledged individually. Fixes ship with the next release; the
release notes summarize the change without disclosing details before the fix
is widely available. If you found a vulnerability, no public credit is made
unless you ask for it.

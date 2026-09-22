---
layout: default
is_doc: true
title: Third Party Notices
permalink: /Third Party Notices.html
---

# Third Party Notices — PathForge

**None.** The PathForge native kernel and the C# binding have **zero third-party
runtime dependencies**. The Rust core crate has no external `crates.io`
dependencies; the FFI crate depends only on the core crate (both first-party).

The only Unity dependencies are Unity's own built-in modules, which are part of
the engine, not third-party packages:

- `com.unity.modules.tilemap`
- `com.unity.modules.ai`

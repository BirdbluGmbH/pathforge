---
layout: default
is_doc: true
title: FAQ
permalink: /docs/FAQ.html
base: ../
---

# PathForge — FAQ

## General

**Q: What game genres is PathForge designed for?**
A: Tower defense, RTS, colony sims, mobile strategy, and any game where multiple agents share destinations.

**Q: Is PathForge a replacement for Unity NavMesh?**
A: No. PathForge is a route computation engine. It works with your own graph — tilemap, waypoint, mesh, or custom. We provide adapters for common Unity patterns but do not include NavMesh generation.

**Q: Does PathForge include local avoidance?**
A: No. PathForge produces routes; you handle collision avoidance separately.

**Q: Can I use PathForge in a lockstep multiplayer game?**
A: Yes. PathForge is fully deterministic: same graph and same query always produce the same result.

## Technical

**Q: How large a graph can PathForge handle?**
A: PathForge uses 32-bit vertex IDs and weights, supporting up to ~4 billion vertices. Practical limits depend on available memory.

**Q: How do I handle graph changes at runtime?**
A: Dispose the old solver, create a new one, and rebuild from the updated `IGraphSource`. See the "Dynamic Obstacles" section in [Getting Started](GettingStarted.html). The `GraphVersion` property helps track changes for cache invalidation.

**Q: Which platforms are supported?**
A: See the Installation section in the [API Reference](API-Reference.html).

**Q: Does it work with IL2CPP?**
A: Yes. All FFI types are blittable (primitive structs, sequential layout). No `Reflection.Emit` or dynamic proxies are used.

**Q: What C# version does PathForge require?**
A: C# 9.0 / .NET Standard 2.1 (Unity 6 baseline).

## Performance

**Q: When should I use fields instead of individual path queries?**
A: When 5+ agents share the same target, a single field computation is faster than individual queries. See the "From One Agent to Many" section in [Getting Started](GettingStarted.html) for the decision tree.

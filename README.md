# PathForge — native, deterministic pathfinding for Unity

PathForge is a **native Rust pathfinding engine for Unity** (UPM package). It
computes routes over the graph of your world — a single fast, exact path, or one
shared distance field that many agents read from. It is a **route engine, not a
movement controller**: it tells your units which way to go; you still animate,
steer, and resolve collisions. It is the GPS, not the car.

## Quick start
```csharp
using UnityEngine;
using UnityEngine.Tilemaps;

var builder = new GridGraphBuilder(grid, tilemap, 20u, 15u);   // your Tilemap
var solver  = new DnskSolver(builder.NumVertices);
solver.BuildFrom(builder, HeuristicMode.Landmarks);

uint[] buffer = new uint[512];                 // declare once, reuse — zero GC per query
var path = solver.QueryPath(startVertex, targetVertex, buffer);
if (path.Reachable)
{
    // buffer[0 .. path.PathLength-1] = the route (vertex IDs); path.Distance = total cost
}
```
Every PathForge type is in the **global namespace** — there is no `using` for a
PathForge namespace. The buffer is yours to supply and reuse, so the kernel
allocates nothing per query.

## What it does
- **Single path** — `DnskSolver.QueryPath(source, target, buffer)`: one fast, exact route.
- **Shared field for crowds** — `DnskField.ComputeField` / `ComputeWeightedField`, then `NextStep` / `BatchNextStep`: one computation serves every agent, in one native call per frame.
- **Graph adapters** — Grid, Tilemap (per-tile terrain weights), Hex, Waypoint, triangle Mesh (terrain), and Unity's baked NavMesh.
- **Bring your own graph** — implement the single `IGraphSource` interface for any custom graph.
- **Deterministic** — integer math, fixed tie-breaks: same input, same route, on every platform, every run. Safe for replays and multiplayer lockstep.
- **Zero allocation per query** — you supply a `uint[]` buffer; the kernel writes the route into it.
- **Self-diagnose** — `DnskNative.Available` / `DnskNative.Diagnose()` answer the common "plugin won't load" question.

## Which tier
The rule is about **targets, not headcount**: agents heading to *independent*
destinations (one path each) → **Base**, any number of them. A crowd sharing one
— or a few — destinations (one field, many readers) → **Pro.** Pro builds on and
depends on Base.

| | **PathForge (Base)** | **PathForge Pro** |
|---|---|---|
| Routes | single-agent `QueryPath` | single path **plus** shared fields |
| Many agents | one search per agent (independent targets) | one field, `BatchNextStep` for the whole group |
| Targets | one | one, or a weighted set (nearest-of-N) |

## Platforms
Requires **Unity 6 (6000.0 or later)**; **Mono** and **IL2CPP** both supported.

| Platform | Architecture | |
|---|---|---|
| Windows | x86_64 | included |
| macOS | x86_64 + Apple Silicon (arm64) | included |
| Linux | x86_64 + ARM64 | included |
| Android | ARM64 | included |

**Not included:** Windows 32-bit, Windows ARM64, iOS, WebGL.

## Numbers
Measured against **our internal reference baseline** (a plain A* used in our own
benchmarks) on identical maps — not against any commercial product:
- 0 KB allocated per query vs ~205 KB in that baseline.
- ~1.9× faster route time than that baseline (40 agents / 100 routes).

## Integration
A drop-in route engine for the navigation you already have — it accepts your
Grid, Tilemap, Hex layout, Waypoint set, triangle Mesh, or Unity's baked NavMesh
and routes directly over them. For triangle meshes there is no grid and no extra
bake step. When the world changes at runtime there is no incremental update:
rebuild the solver from your updated graph (or swap a pre-built field) and every
agent on it reroutes. Nine sample scenes ship in the package.

---

## Documentation
Full docs: [birdblugmbh.github.io/pathforge](https://birdblugmbh.github.io/pathforge/)
(rendered in this repo under `docs/`, and shipped in-package under
`Documentation~/`).
[Getting Started](docs/GettingStarted.html) · [API Reference](docs/API-Reference.html) ·
[Bring Your Graph](docs/BringYourGraph.html) · [Troubleshooting](docs/Troubleshooting.html) ·
[FAQ](docs/FAQ.html) · [License](LICENSE.html)

For AI assistants: a compact, LLM-oriented summary is in
[llms.txt](llms.txt).

## Getting help — this is the issue tracker
This repository is the support channel for both assets. Support runs through the
[issue tracker](https://github.com/BirdbluGmbH/pathforge/issues); please file bugs,
feature requests, and questions there. Three templates are available:

- [Bug report](https://github.com/BirdbluGmbH/pathforge/issues/new?template=bug_report.yml) — something in PathForge (Base or Pro) is not working as expected.
- [Feature request](https://github.com/BirdbluGmbH/pathforge/issues/new?template=feature_request.yml) — a suggestion for a new feature or behavior.
- [Question](https://github.com/BirdbluGmbH/pathforge/issues/new?template=question.yml) — API usage, setup, or a "does PathForge do X?" question.

For "how do I" questions, check the [troubleshooting guide](docs/Troubleshooting.html)
and the rest of the docs first — most setup questions are answered there. Security
reports go privately to [info@birdblu.com](mailto:info@birdblu.com) (see
[SECURITY.md](SECURITY.md)).

**Scope.** In scope: plugin loading and ABI issues, supported-platform builds, API
usage, samples, deterministic behavior. Out of scope: Unity engine bugs, conflicts
with other third-party assets, custom game logic.

Purchased on the Unity Asset Store — this site/repo is an information resource,
not a checkout.

[Impressum](https://birdblugmbh.github.io/pathforge/#impressum)

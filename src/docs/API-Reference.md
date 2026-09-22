---
layout: default
is_doc: true
title: API Reference
permalink: /docs/API-Reference.html
base: ../
---

# PathForge — API Reference

> Complete method and type reference. For tutorials, patterns, and game-specific examples, see [Getting Started](GettingStarted.html).

---

## Installation

### Prerequisites

- Unity 6 (6000.0 or later)
- C# 9.0 / .NET Standard 2.1 (Unity 6 baseline)
- Native plugin included for your platform

### Supported Platforms

| Platform | Plugin | Status |
|---|---|---|
| Windows x86_64 | `dnsk_ffi.dll` | Included |
| Linux x86_64 | `libdnsk_ffi.so` | Included |
| Linux ARM64 | `libdnsk_ffi.aarch64.so` | Included |
| macOS x86_64 | `libdnsk_ffi.x86_64.dylib` | Included |
| macOS ARM64 (Apple Silicon) | `libdnsk_ffi.arm64.dylib` | Included |
| Android ARM64 | `libdnsk_ffi.so` | Included |
| iOS | — | Not supported |

Not included: Windows 32-bit, Windows ARM64, iOS, WebGL. See
[Troubleshooting](Troubleshooting.html) for plugin-loading issues on each platform.

**Mono and IL2CPP:** compatible with both script backends — all FFI types are
blittable, no `Reflection.Emit`.

### Package Import

Import via Package Manager or by placing the package folder in `Packages/`. The native
plugin loads automatically — no setup required.

---

## DnskSolver

Main entry point. Owns one graph in native memory. Implements `IDisposable`.

### Constructors

| Signature | Description |
|---|---|
| `DnskSolver(uint numVertices)` | Create solver for N vertices. Checks the native plugin's version on construction — throws `InvalidOperationException` on mismatch. |

### Methods

| Method | Description |
|---|---|
| `AddEdges(uint[] sources, uint[] targets, uint[] weights)` | Add directed edges. All three arrays must be same length. Must be called before `Build()`. |
| `Build(HeuristicMode mode)` | Finalize graph and build internal structures. Can only be called once. Default: `HeuristicMode.Landmarks`. |
| `BuildFrom(IGraphSource source, HeuristicMode mode)` | Convenience: calls `AddEdges` + `Build` using the provided `IGraphSource`. |
| `QueryPath(uint source, uint target, uint[] pathBuffer)` | Single-agent path query. Writes path vertices into `pathBuffer`. Returns `PathResult`. Zero managed allocations. |
| `CreateField()` | Create a `DnskField` handle for field operations. Requires `Build()` to have been called. |
| `Dispose()` | Release native memory. Call in `OnDestroy()`. After disposal, all operations throw `ObjectDisposedException`. |

### Properties

| Property | Type | Description |
|---|---|---|
| `GraphVersion` | `uint` | Monotonic version number. Use for cache invalidation. |

---

## DnskField  *(Pro)*

Field operations on a built solver. Created via `solver.CreateField()`. Does not need separate disposal — shares the solver's native memory.

> **Pro only.** `DnskField` and every method below bind to the `dnsk_ffi_pro` native
> module, which only the **PathForge Pro** package installs. On a Base-only install,
> calling any of these throws `DnskProRequiredException` (a clear "install Pro" message,
> not a cryptic P/Invoke error). Single-query routing (`DnskSolver.QueryPath`) works on
> Base.

### Methods

| Method | Description |
|---|---|
| `ComputeField(uint target, uint[] distancesOut, bool[]? exploredOut = null)` | Single-target reverse field. `distancesOut[v]` = distance from `v` to target. `uint.MaxValue` = unreachable. Optional `exploredOut` bitmap marks reachable cells. |
| `ComputeWeightedField(uint[] targets, uint[]? costs, uint[] distancesOut, uint[] rootsOut)` | Multi-target Voronoi field. `rootsOut[v]` = vertex ID of nearest target. Optional `costs` array (one per target) weights targets differently. Pass `null` for equal weights. |
| `ComputeFieldWatched(uint target, byte[] watchedBitmap, uint watchedLeft, uint[] distancesOut)` | Partial field with early termination. `watchedBitmap` is indexed by vertex ID (set `1` for watched vertices). `watchedLeft` is the count of watched vertices still unreached. Flood stops when `watchedLeft` reaches 0. Pass `null` bitmap with `watchedLeft = 0` for a full flood. |
| `NextStep(uint vertex)` | Return next vertex toward target from computed field. Returns `FieldStep`. Throws `InvalidOperationException` on blocked/unreachable vertices. |
| `BatchNextStep(uint[] vertices, uint[] nextOut, bool[] terminalOut)` | Batch next-step for multiple vertices. All three arrays must have same length. Returns count of valid results. Unreachable/blocked vertices are silently skipped. Single FFI call. |
| `ExtractPath(uint start, uint[] pathBuffer, uint maxSteps)` | Extract full path from `start` vertex to target via field. Returns actual number of vertices written into `pathBuffer`. `maxSteps` caps the path length. |
| `Root(uint vertex)` | Which target vertex this cell belongs to. Only valid after `ComputeWeightedField()`. |
| `BatchQuery(uint[] sources, uint[] targets, uint[] distancesOut, uint[]? rootsOut)` | Batch distance queries for source-target pairs. All arrays must have same length. Optional `rootsOut` returns nearest target per pair. |

---

## HeuristicMode

| Value | Description |
|---|---|
| `Dijkstra` | Plain Dijkstra. Use for mesh/NavMesh graphs or small grids. |
| `Landmarks` | Landmark-based heuristic. Use for grid/hex graphs — typically the fastest option on regular layouts. |

---

## PathResult

Returned by `DnskSolver.QueryPath()`.

| Field | Type | Description |
|---|---|---|
| `Distance` | `uint` | Total path cost (sum of edge weights). `uint.MaxValue` if target is unreachable. |
| `PathLength` | `int` | Number of vertices written into the path buffer (including source and target). |
| `Reachable` | `bool` | `true` if a path exists from source to target. |

---

## FieldStep

Returned by `DnskField.NextStep()`.

| Field | Type | Description |
|---|---|---|
| `NextVertex` | `uint` | The next vertex on the path toward the target. |
| `IsTerminal` | `bool` | `true` if the current vertex IS the target (no further steps needed). |

---

## IGraphSource

Interface for providing graph data to the solver. Engine-free (no `UnityEngine` dependencies).

| Member | Type | Description |
|---|---|---|
| `NumVertices` | `uint` | Total vertex count. Vertex IDs are 0 to `NumVertices - 1`. |
| `NumEdges` | `uint` | Total directed edge count. |
| `FillEdges(uint[] sources, uint[] targets, uint[] weights)` | `void` | Write edge data into pre-allocated arrays. Each undirected connection should be emitted as two directed edges (forward + reverse). All three arrays have length `NumEdges`. |
| `IsBlocked(uint vertex)` | `bool` | Whether a vertex is impassable. Used by utility classes, not the solver core. |

---

## Graph Builder Adapters

All implement `IGraphSource`.

### GridGraphBuilder

Tilemap adapter. **Tile present = blocked.**

| Constructor | Description |
|---|---|
| `GridGraphBuilder(Grid grid, Tilemap tilemap)` | Auto-size from `tilemap.cellBounds`. 4-connectivity. |
| `GridGraphBuilder(Grid grid, Tilemap tilemap, uint cols, uint rows)` | Explicit size. 4-connectivity. |
| `GridGraphBuilder(Grid grid, Tilemap tilemap, uint cols, uint rows, bool diagonal)` | Explicit size. `diagonal: true` = 8-connectivity (cardinals weight 10, diagonals 14, corner-cutting guarded). |

| Method | Description |
|---|---|
| `VertexToWorld(uint vertex)` | Convert vertex ID to world-space cell center. |

### TilemapGraphBuilder

Tilemap adapter. **Tile present = walkable.** Supports per-tile weights.

| Constructor | Description |
|---|---|
| `TilemapGraphBuilder(Tilemap tilemap)` | Auto-bounds. 4-connectivity. |
| `TilemapGraphBuilder(Tilemap tilemap, BoundsInt bounds, bool diagonal)` | Explicit bounds. Optional 8-connectivity. |

| Method | Description |
|---|---|
| `SetTileWeight(TileBase tile, uint weight)` | Set movement cost multiplier for a tile type. |

### HexGraphBuilder

Pointy-top hex grid (odd-row-offset).

| Constructor | Description |
|---|---|
| `HexGraphBuilder(uint cols, uint rows, Func<uint,uint,bool> blocked, float cellWidth, float rowHeight)` | Predicate-based blocked check. Default cell dimensions: `1.0` width, `0.866` row height (regular unit hexes). |

| Method | Description |
|---|---|
| `HexToWorld(uint vertex)` | Convert vertex ID to world-space hex center. |

### WaypointGraphBuilder

Fully connected graph from Transform array.

| Constructor | Description |
|---|---|
| `WaypointGraphBuilder(Transform[] waypoints)` | Default connection distance: 100 units. |
| `WaypointGraphBuilder(Transform[] waypoints, float connectionDistance)` | Waypoints within `connectionDistance` are connected. Edge weight = Euclidean distance. |

| Method | Description |
|---|---|
| `VertexToWorld(uint vertex)` | Return world position of waypoint at given index. |

### MeshGraphBuilder

Triangle-adjacency graph from a Unity `Mesh`. Each triangle is a vertex.

| Constructor | Description |
|---|---|
| `MeshGraphBuilder(Mesh mesh)` | All triangles walkable. |
| `MeshGraphBuilder(Mesh mesh, Func<int,bool>? triangleBlocked)` | Predicate: triangle index → blocked. |

| Method | Description |
|---|---|
| `TriangleCentroid(uint vertex)` | Return mesh-local centroid of triangle. Transform by the mesh object's transform for world space. |

### NavMeshGraphBuilder

Triangle-adjacency graph from Unity's baked NavMesh.

| Constructor | Description |
|---|---|
| `NavMeshGraphBuilder()` | All NavMesh areas walkable. |
| `NavMeshGraphBuilder(Func<int,bool>? areaBlocked)` | Predicate: NavMesh area index → blocked. |

| Method | Description |
|---|---|
| `TriangleCentroid(uint vertex)` | Return world-space centroid of NavMesh triangle. |

---

## Unity Helpers

### DnskPathFollower

`MonoBehaviour` for frame-by-frame path following via `QueryPath`.

| Property | Type | Description |
|---|---|---|
| `Solver` | `DnskSolver` | Reference to the solver. |
| `StartVertex` | `uint` | Starting vertex ID. |
| `TargetVertex` | `uint` | Destination vertex ID. |
| `MoveSpeed` | `float` | World units per second. Default: 5. |
| `IsFollowing` | `bool` | `true` while agent is moving along a path. |
| `PathRemaining` | `int` | Steps left in current path. |

| Method | Description |
|---|---|
| `StartPath()` | Compute path and begin following. |
| `StartPath(uint startVertex)` | Compute path from a specific vertex. |
| `SetVertexToWorld(Func<uint,Vector3> converter)` | Set custom vertex-to-world converter (required for grid builders). |

### DnskFieldCache  *(Pro)*

Caches computed fields keyed by `(target, graphVersion, mode)`. **Pro only** — it wraps
`DnskField`, so it requires the PathForge Pro package.

| Method | Description |
|---|---|
| `GetOrCompute(FieldSignature sig, Func<FieldSignature,FieldData> compute)` | Return cached field or compute and cache. |
| `Invalidate()` | Clear all cached fields. |
| `IncrementGraphVersion()` | Increment version and clear cache. |

| Property | Type | Description |
|---|---|---|
| `HitCount` | `int` | Cache hit count. |
| `MissCount` | `int` | Cache miss count. |
| `GraphVersion` | `uint` | Current cached graph version. |

### FieldSignature

Cache key struct.

| Field | Type | Description |
|---|---|---|
| `Target` | `uint` | Target vertex ID. |
| `GraphVersion` | `uint` | Solver graph version. |
| `Mode` | `HeuristicMode` | Heuristic mode used. |

### FieldData

Cached field result struct.

| Field | Type | Description |
|---|---|---|
| `Distances` | `uint[]` | Distance array. |
| `Roots` | `uint[]` | Root array (weighted field). |

### DnskNative

Static diagnostics for the native plugin layer.

| Member | Type | Description |
|---|---|---|
| `Available` | `bool` | `true` if the native plugin loaded and ABI version matches. |
| `LoadError` | `string?` | Human-readable load failure reason. `null` if loaded. |
| `OK` | `int` | FFI status code: success (0). |
| `InvalidInput` | `int` | FFI status code: invalid input (1). |
| `Panic` | `int` | FFI status code: Rust panic (2). |

| Method | Description |
|---|---|
| `Diagnose()` | Return detailed diagnostic string (probed paths, load status, Win32 error codes). |

---

## Lifecycle & Error Handling

### Solver Lifecycle

```
new DnskSolver(N) → AddEdges() → Build() → [QueryPath() | CreateField()] → Dispose()
```

- The solver **must** be disposed. Call `Dispose()` in `OnDestroy()`.
- `Build()` can only be called once. Create a new solver for graph changes.
- `CreateField()` requires `Build()` to have been called.
- Field handles share the solver's native memory — they do NOT need separate disposal.

### Native Plugin Loading

The native plugin loads automatically before the first scene load — no setup
required. If it does not load, see [Troubleshooting](Troubleshooting.html).

### Error Table

| Situation | Behavior |
|---|---|
| No path exists | `QueryPath` returns `PathResult.Reachable = false`, `Distance = uint.MaxValue` |
| Native call fails | `InvalidOperationException` with FFI status code |
| `NextStep` on blocked vertex | `InvalidOperationException` — wrap in try/catch in per-frame loops |
| Graph not built yet | `InvalidOperationException` on any query or field operation |
| Solver already disposed | `ObjectDisposedException` |
| ABI version mismatch | `InvalidOperationException` at `DnskSolver` construction |
| `pathBuffer` too small | Path is silently truncated to buffer capacity |

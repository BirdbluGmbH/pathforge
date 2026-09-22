---
layout: default
is_doc: true
title: Getting Started
permalink: /docs/GettingStarted.html
base: ../
---

# PathForge — Getting Started

> Fast paths for one. Shared fields for many.

PathForge is a native pathfinding engine for Unity. You give it a map. It tells your
units how to move. The difference: PathForge runs in compiled Rust code, allocates
zero managed memory per query, and lets you route hundreds of agents with a single
computation.

---

## Installation

PathForge is a Unity UPM package. Import it via the Package Manager or by placing
the package folder in your project's `Packages/` directory.

### What You Need

- Unity 6 (6000.0 or later)
- C# scripting basics (you've written a `MonoBehaviour` before)
- A scene with something to navigate on (a Tilemap, a Mesh, or a set of waypoints)

### What PathForge Is and Isn't

| Is | Isn't |
|---|---|
| A route computation engine | A movement controller (no animation, no steering) |
| Works with Tilemap, Mesh, NavMesh, Waypoints, Hex | A replacement for Unity NavMesh baking |
| Zero GC allocations per query | A local avoidance system (agents still collide — you handle that) |
| Deterministic (same input = same output) | A behavior tree or decision-making system |

**Think of it this way:** PathForge is the GPS. You still drive the car.

---

## Your First Path

Let's get one enemy walking from point A to point B on a grid.

### Step 1: Set Up Your Grid

Create a Grid and Tilemap in your scene (GameObject → 3D Object → Grid). Place some
obstacle tiles where you want walls.

### Step 2: Write the Script

Attach this script to any GameObject in your scene and assign the references in the
Inspector:

```csharp
using UnityEngine;
using UnityEngine.Tilemaps;

public class MyPathfinding : MonoBehaviour
{
    [SerializeField] private Grid grid;
    [SerializeField] private Tilemap tilemap;
    [SerializeField] private Transform enemy;
    [SerializeField] private Transform target;

    private DnskSolver solver;
    private GridGraphBuilder builder;

    // Reused across queries — never allocate inside Update()
    private uint[] pathBuffer = new uint[512];

    void Start()
    {
        // 20 columns, 15 rows — must match your Tilemap size
        // GridGraphBuilder convention: tile present = blocked, empty = walkable
        builder = new GridGraphBuilder(grid, tilemap, 20u, 15u);

        solver = new DnskSolver(builder.NumVertices);
        solver.BuildFrom(builder, HeuristicMode.Landmarks);
    }

    void Update()
    {
        if (Input.GetMouseButtonDown(0))
        {
            uint startVertex = WorldToVertex(enemy.position);
            uint targetVertex = WorldToVertex(target.position);

            var result = solver.QueryPath(startVertex, targetVertex, pathBuffer);

            if (result.Reachable)
            {
                Debug.Log($"Path found! {result.PathLength} steps, cost {result.Distance}");
                // pathBuffer[0..PathLength-1] holds the vertex IDs of the route
            }
            else
            {
                Debug.Log("No path — target is unreachable");
            }
        }
    }

    void OnDestroy()
    {
        solver?.Dispose();
    }

    // vertex = x + y * columns (like reading a chess board left-to-right, top-to-bottom)
    uint WorldToVertex(Vector3 position)
    {
        int x = Mathf.FloorToInt(position.x);
        int y = Mathf.FloorToInt(position.y);
        return (uint)(x + y * 20);
    }

    Vector3 VertexToWorld(uint vertex)
    {
        uint x = vertex % 20;
        uint y = vertex / 20;
        return grid.GetCellCenterWorld(new Vector3Int((int)x, (int)y, 0));
    }
}
```

**Why these values?**

| Value | Meaning |
|---|---|
| `20u, 15u` | Grid dimensions — columns and rows. The `u` means `uint` (unsigned integer). Vertex IDs are always positive whole numbers, like cell numbers on a chess board. |
| `512` (buffer size) | Maximum path length in cells. For a 20×15 grid (300 cells), 512 is more than enough. A safe upper bound is `cols × rows`. |
| `HeuristicMode.Landmarks` | Smart shortcuts for grid-based maps. Typically 2-5× faster than plain Dijkstra. Use `HeuristicMode.Dijkstra` only for triangle meshes or very small grids. |

**Why a buffer?**

PathForge doesn't allocate memory for each query. You provide a `uint[]` array and
the kernel writes the path directly into it. This means zero garbage collection
pressure — declare `pathBuffer` as a field, never inside `Update()`.

### How Vertex IDs Work

The grid is a 2D surface flattened into a single list of cells:

```
  x→
  0  1  2  3  4  ...  19
0 ┌──┬──┬──┬──┬──┐
  │0 │1 │2 │3 │4 │   vertex = x + y * 20
1 ├──┼──┼──┼──┼──┤   (20 = number of columns)
  │20│21│22│23│24│
  └──┴──┴──┴──┴──┘
  ↑
  y
```

The `GridGraphBuilder` provides a `VertexToWorld()` helper (vertex → world
position) so you don't have to do the "vertex to world" math yourself. The
reverse direction — "world to vertex" — is your game's job, because it depends on
how *you* place agents on the grid (snapping to cell centers, clamping to bounds,
rounding). The quick-start example above shows the standard one-line version
(`x + y * columns`). If you use a different grid convention, that's the one place
you adapt it.

---

## From One Agent to Many

### Decision tree: query or field?

```
How many agents need a route, and where do they go?

├── 1 agent → 1 target .................... QueryPath()
│        (one search, one path)
│
├── A few agents → DIFFERENT targets ...... QueryPath() per agent
│        (or BatchQuery() to group same-target pairs)
│
└── 5+ agents → SAME (or a few) targets .... ComputeField() /
         ComputeWeightedField()  +  BatchNextStep()
         (one computation serves every agent, every frame)
```

Rule of thumb: **the more agents share a destination, the more the field wins.**
One agent → query. A crowd → field.

The single-query pattern works fine for a few units. But what happens when you have
50 enemies all heading toward the same base?

**The naive approach:** 50 separate `QueryPath()` calls. Each one runs a full search.
On a large map, that's 50 searches competing for CPU time — visible frame stutters
at wave start.

**The PathForge approach:** Compute a **field** once. Every agent reads its next step
from that field. One computation. Zero per-frame search cost.

### What Is a Field?

Imagine standing on any cell of your map and looking toward the base. A field tells
you "which neighbor is closer to the base?" — for every cell, simultaneously.

```
Before field:                    After ComputeField(base):
┌──┬──┬──┐                       ┌──┬──┬──┐
│  │  │  │                       │← │← │↓ │  Each cell knows which
├──┼──┼──┤                       ├──┼──┼──┤  neighbor leads toward the
│  │  │  │  →  one computation   │← │· │← │  base. "·" is the base.
├──┼──┼──┤                       ├──┼──┼──┤
│  │  │· │                       │↓ │↑ │↑ │
└──┴──┴──┘                       └──┴──┴──┘
```

Once computed, asking "which way should I go from here?" is just looking up the
direction stored in that cell — no search, no calculation.

### Complete Example: Many Agents, One Field

```csharp
using UnityEngine;
using UnityEngine.Tilemaps;

public class ManyAgents : MonoBehaviour
{
    [SerializeField] private Grid grid;
    [SerializeField] private Tilemap tilemap;
    [SerializeField] private Transform[] enemies;
    [SerializeField] private Transform basePosition;

    private DnskSolver solver;
    private GridGraphBuilder builder;
    private DnskField field;

    // Field output — distance from each cell to the base
    private uint[] distances;

    // Batch arrays — filled each frame, one native call for all agents
    private uint[] batchVerts;
    private uint[] batchNext;
    private bool[] batchTerminal;

    void Start()
    {
        builder = new GridGraphBuilder(grid, tilemap, 20u, 15u);
        solver = new DnskSolver(builder.NumVertices);
        solver.BuildFrom(builder, HeuristicMode.Landmarks);

        // Create field and compute ONCE
        field = solver.CreateField();
        distances = new uint[builder.NumVertices];
        field.ComputeField(WorldToVertex(basePosition.position), distances);

        // Allocate batch arrays once
        int count = enemies.Length;
        batchVerts = new uint[count];
        batchNext = new uint[count];
        batchTerminal = new bool[count];
    }

    void Update()
    {
        int count = enemies.Length;

        // Fill current positions
        for (int i = 0; i < count; i++)
            batchVerts[i] = WorldToVertex(enemies[i].position);

        // ONE native call resolves all agents
        int valid = field.BatchNextStep(batchVerts, batchNext, batchTerminal);

        // Apply results
        for (int i = 0; i < valid; i++)
        {
            if (batchTerminal[i])
            {
                // Agent reached the base — handle arrival
                Debug.Log($"Enemy {i} reached the base!");
            }
            else
            {
                Vector3 targetPos = builder.VertexToWorld(batchNext[i]);
                enemies[i].position = Vector3.MoveTowards(
                    enemies[i].position, targetPos, 3f * Time.deltaTime);
            }
        }
    }

    void OnDestroy()
    {
        solver?.Dispose();
    }

    uint WorldToVertex(Vector3 position)
    {
        int x = Mathf.FloorToInt(position.x);
        int y = Mathf.FloorToInt(position.y);
        return (uint)(x + y * 20);
    }
}
```

**Why `uint[] distances`?**

The field computation fills this array with the distance from each cell to the target.
`distances[v] = uint.MaxValue` means that cell cannot reach the target (blocked off).
You don't have to use this array — it's optional output that lets you visualize
reachability or build heatmaps.

**The numbers:** On a 40-agent fleet, `BatchNextStep()` is typically 10-30× faster
than 40 separate `NextStep()` calls because the cross-language call overhead is paid
once, not 40 times.

---

## Genre Patterns

### Tower Defense

**Scenario:** Enemies spawn at the edge and walk toward your base. Waves of 50-200
units. The map has walls and corridors that shape enemy movement.

```csharp
// Build once at scene start
var builder = new GridGraphBuilder(grid, tilemap, cols, rows);
solver = new DnskSolver(builder.NumVertices);
solver.BuildFrom(builder, HeuristicMode.Landmarks);

// Compute field to the base — ONE computation serves every enemy, every wave
var field = solver.CreateField();
uint[] distances = new uint[builder.NumVertices];
field.ComputeField(baseVertex, distances);

// Each frame: batch all enemies into one call
int valid = field.BatchNextStep(enemyVerts, nextVerts, terminals);
for (int i = 0; i < valid; i++)
{
    if (terminals[i])
        HandleEnemyArrival(i);
    else
        MoveEnemyTo(i, nextVerts[i]);
}
```

**Why this works:** The field is computed once. Adding more enemies costs nothing.
Adding more waves costs nothing. The only cost is the initial flood, which runs in
compiled code and typically takes microseconds.

### Colony Sim / RTS

**Scenario:** Workers need to find the nearest resource, storage, or exit. There are
multiple destinations, and each worker should go to the closest one.

```csharp
// Multiple targets: resource nodes, storage buildings, exits
uint[] targets = new uint[] { resourceA, resourceB, storage, exit1, exit2 };
uint[] distances = new uint[builder.NumVertices];
uint[] roots = new uint[builder.NumVertices];

var field = solver.CreateField();
// null = equal weight for all targets (no preference)
field.ComputeWeightedField(targets, null, distances, roots);

// Each worker: find nearest target, then follow the field
foreach (var worker in workers)
{
    uint myTarget = field.Root(worker.currentVertex); // nearest target
    var step = field.NextStep(worker.currentVertex);
    worker.MoveTo(step.NextVertex);
}
```

**What `roots` tells you:** After `ComputeWeightedField`, the `roots` array divides
the map into zones. Every cell knows which target it "belongs to." Like drawing a map
where each region is colored by the nearest city.

**The `costs` parameter:** Pass `null` for equal weight. Pass a `uint[]` (one value
per target) to weight targets differently — for example, a storage building that's
half full gets a lower cost, attracting more workers.

### Dynamic Obstacles

**Scenario:** A mudslide blocks a road. Enemies need to reroute instantly.

```csharp
// When the world changes, rebuild the solver
solver.Dispose(); // free old memory
solver = new DnskSolver(builder.NumVertices);
solver.BuildFrom(builder, HeuristicMode.Landmarks);

// Recompute the field — enemies reroute automatically
field = solver.CreateField();
field.ComputeField(baseVertex, distances);
```

**Advanced:** The `SupplyRun` sample shows dual-solver swapping — pre-build both the
"calm" and "flooded" graphs, then swap the active field in one line. Zero rebuild
cost at runtime.

---

## Graph Adapters

PathForge ships with six adapters. Choose the one that matches your map.

### GridGraphBuilder — Tilemap with Obstacles

The most common starting point. Your Tilemap has obstacle tiles (walls, water,
cliffs). Empty cells are walkable.

```csharp
// 4-directional movement (up, down, left, right)
var builder = new GridGraphBuilder(grid, tilemap, 20u, 15u);

// 8-directional movement (adds diagonals)
var builder = new GridGraphBuilder(grid, tilemap, 20u, 15u, diagonal: true);
```

**Convention:** Tile present = blocked. Empty cell = walkable.

**Diagonal mode:** When enabled, diagonal moves cost 14 vs 10 for cardinal moves
(an integer approximation of √2). Diagonals cannot cut through corners — both
adjacent cardinal cells must be walkable.

### TilemapGraphBuilder — Tilemap with Terrain Types

Same as GridGraphBuilder but inverted convention: tile present = walkable. Supports
per-tile weights (different movement costs for different terrain).

```csharp
var builder = new TilemapGraphBuilder(tilemap, tilemap.cellBounds, diagonal: false);
builder.SetTileWeight(sandTile, 3u);  // sand is 3× slower
builder.SetTileWeight(roadTile, 1u);  // roads are fast
```

### HexGraphBuilder — Hexagonal Grids

For pointy-top hex grids (odd-row-offset layout, which Unity's hex tilemaps use).

```csharp
// Blocked cells defined by a predicate
Func<uint, uint, bool> isBlocked = (x, y) => obstacleSet.Contains((x, y));
var builder = new HexGraphBuilder(20u, 15u, isBlocked);
```

Each hex connects to up to 6 neighbors. All edges have equal weight by default.

### WaypointGraphBuilder — Sparse Navigation Points

For games where movement happens along specific routes (highways, train lines,
corridor systems).

```csharp
Transform[] waypoints = FindObjectsOfType<WaypointMarker>()
    .Select(w => w.transform).ToArray();
var builder = new WaypointGraphBuilder(waypoints, connectionDistance: 100f);
```

Waypoints within `connectionDistance` of each other are connected. Edge weight =
Euclidean distance between waypoints.

### MeshGraphBuilder — Triangle Mesh Navigation

For terrain meshes, custom surfaces, or any 3D geometry where agents walk on
triangles. Each triangle is a navigation node; adjacent triangles (sharing an edge)
are connected.

```csharp
Mesh terrainMesh = terrainMeshFilter.mesh;
var builder = new MeshGraphBuilder(terrainMesh);
var solver = new DnskSolver(builder.NumVertices);
solver.BuildFrom(builder, HeuristicMode.Dijkstra); // Dijkstra for meshes
```

**Why Dijkstra?** Landmark shortcuts need a grid-like structure. Triangle meshes
don't have that regularity, so landmarks provide no benefit.

### NavMeshGraphBuilder — Unity's Baked NavMesh

Same triangle-adjacency core as MeshGraphBuilder, but reads from Unity's baked
NavMesh. Use this when you already have a NavMesh and want PathForge's performance.

```csharp
// All areas walkable
var builder = new NavMeshGraphBuilder();

// Exclude specific areas (e.g., block "Not Walkable" area)
var builder = new NavMeshGraphBuilder(areaIndex => areaIndex == NavMeshArea.NotWalkable);
```

---

## The MonoBehaviour Shortcut

For simple single-agent pathfinding, PathForge includes `DnskPathFollower` — a
`MonoBehaviour` that handles path following automatically:

```csharp
var follower = enemy.AddComponent<DnskPathFollower>();
follower.Solver = solver;
follower.StartVertex = startVertex;
follower.TargetVertex = targetVertex;
follower.MoveSpeed = 5f;            // World units per second
follower.SetVertexToWorld(builder.VertexToWorld);
follower.StartPath();
```

The follower computes the path and moves the agent frame-by-frame. Check
`follower.IsFollowing` to see if it's still moving, and `follower.PathRemaining`
for steps left.

---

## Common Mistakes

### "My path is empty"

Check `result.Reachable` first — if false, the target is blocked off. If true but
`PathLength` is 0, the start and target are the same vertex. If `PathLength` is
smaller than expected, your buffer might be too small (increase it to `cols × rows`).

### "NextStep throws an exception every frame"

`NextStep()` throws on blocked or unreachable vertices. In per-frame loops, check
reachability before calling:

```csharp
uint vertex = GetAgentVertex(i);
if (distances[vertex] != uint.MaxValue)
{
    var step = field.NextStep(vertex);
    MoveAgent(step.NextVertex);
}
// else: agent is on a blocked cell — teleport or skip
```

This avoids exceptions entirely. If you can't check distances, wrap in try/catch,
but prefer the distance check — exceptions in hot loops are expensive.

### "Agents don't move after I change the map"

The field is stale. When you add or remove obstacles, you must rebuild the solver
and recompute the field. There is no incremental update.

### "I'm getting GC spikes"

You're probably allocating `pathBuffer` inside a loop. Declare it as a field and
reuse it:

```csharp
// BAD — allocates every frame
void Update() { solver.QueryPath(a, b, new uint[512]); }

// GOOD — one allocation, reused forever
uint[] buffer = new uint[512];
void Update() { solver.QueryPath(a, b, buffer); }
```

### "The plugin won't load"

Check `DnskNative.Available`. If false, call `DnskNative.Diagnose()` for details.
Common causes: wrong platform binary, missing VC++ runtime (Windows), or the
native library isn't in the expected folder.

---

## Memory and Cleanup

Always dispose your solver when the scene unloads:

```csharp
void OnDestroy()
{
    solver?.Dispose();
}
```

Field handles (`DnskField`) share the solver's memory — they don't need separate
disposal. When the solver is disposed, all its fields are freed too.

---

## What's Next?

- **[API Reference](API-Reference.html)** — Complete method documentation, parameter
  details, return types, error table
- **[Bring Your Graph](BringYourGraph.html)** — How to implement `IGraphSource` for
  custom graph structures (adjacency list, CSR, etc.)
- **[FAQ](FAQ.html)** — Platform support, IL2CPP, mobile tips, deterministic
  multiplayer
- **Samples** — The `Samples~/` folder contains nine demos (BaseDemo, ColonySim,
  TowerDefense, etc.) that show each pattern in action with visuals

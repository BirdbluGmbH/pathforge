---
layout: default
is_doc: true
title: Bring Your Graph
permalink: /docs/BringYourGraph.html
base: ../
---

# PathForge — Bring Your Graph

PathForge accepts any graph through the `IGraphSource` interface. This allows you to integrate with existing navigation systems or build custom graph structures.

## IGraphSource Interface

```csharp
public interface IGraphSource
{
    uint NumVertices { get; }
    uint NumEdges { get; }
    void FillEdges(uint[] sources, uint[] targets, uint[] weights);
    bool IsBlocked(uint vertex);
}
```

### Contract

- `NumVertices` must return the total number of vertices (0-indexed IDs)
- `NumEdges` must return the exact number of directed edges
- `FillEdges` writes edge data into the provided arrays. All three arrays have the same length equal to `NumEdges`.
- `IsBlocked` returns `true` for cells/vertices that should not be traversed

## Example: Adjacency List

```csharp
public class AdjacencyListGraph : IGraphSource
{
    private readonly Dictionary<uint, List<(uint target, uint weight)>> _adj;
    private readonly HashSet<uint> _blocked;
    private readonly uint _vertexCount;

    public uint NumVertices => _vertexCount;

    public uint NumEdges
    {
        get
        {
            uint count = 0;
            foreach (var kvp in _adj) count += (uint)kvp.Value.Count;
            return count;
        }
    }

    public void FillEdges(uint[] sources, uint[] targets, uint[] weights)
    {
        int i = 0;
        foreach (var kvp in _adj)
        {
            foreach (var (t, w) in kvp.Value)
            {
                sources[i] = kvp.Key;
                targets[i] = t;
                weights[i] = w;
                i++;
            }
        }
    }

    public bool IsBlocked(uint vertex) => _blocked.Contains(vertex);
}
```

## Example: Pre-computed CSR

```csharp
public class CsrGraph : IGraphSource
{
    private readonly uint[] _offsets;
    private readonly uint[] _targets;
    private readonly uint[] _weights;
    private readonly uint _vertexCount;

    public uint NumVertices => _vertexCount;
    public uint NumEdges => (uint)_targets.Length;

    public void FillEdges(uint[] sources, uint[] targets, uint[] weights)
    {
        int i = 0;
        for (uint v = 0; v < _vertexCount; v++)
        {
            for (int j = _offsets[v]; j < _offsets[v + 1]; j++)
            {
                sources[i] = v;
                targets[i] = _targets[j];
                weights[i] = _weights[j];
                i++;
            }
        }
    }

    public bool IsBlocked(uint vertex) => false;
}
```

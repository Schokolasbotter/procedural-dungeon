# Procedural Dungeon

> A dungeon generator built on branching random walkers and bitmask autotiling — carve the
> space, resolve the walls, then populate it with a player, treasure, traps, and enemy camps.

**Engine:** Unity · **Language:** C# · **Role:** Solo · **Built:** 2024

---

## The Pipeline

Generation runs as an ordered sequence, each stage consuming the last stage's output:

| Stage | What it does |
|---|---|
| `PopulateGrid()` | Random walkers carve open space into a `bool[,]` grid |
| `CreateFloorTiles()` | Instantiates floor geometry for open cells, with a chance of scatter props |
| `PlaceWalls()` | Resolves each solid cell to a wall shape via neighbour bitmask |
| `FillOutsideWithDirt()` | Surrounds the dungeon with fill and boundary colliders |
| `SpawnPlayer()` · `SpawnCoinTrap()` · `SpawnEnemyCamp()` · `SpawnEnemy()` | Populates content into valid cells |

Keeping generation and decoration in separate passes means the layout algorithm can be swapped
without touching anything downstream — every later stage reads only the boolean grid.

---

## Branching Random Walkers

The layout is a **drunkard's walk**, with one addition that matters.

Five walkers are seeded deterministically — one at each edge midpoint and one at the centre —
so the carved region always spans the map rather than clustering wherever the RNG happened to
start:

```csharp
new RandomWalker(1, gridSize / 2, gridSize, grid),
new RandomWalker(gridSize / 2, 1, gridSize, grid),
new RandomWalker(gridSize - 2, gridSize / 2, gridSize, grid),
new RandomWalker(gridSize / 2, gridSize - 2, gridSize, grid),
new RandomWalker(gridSize / 2, gridSize / 2, gridSize, grid)
```

Each walker takes 100 steps, flipping its current cell to open. At **step 51 a walker spawns a
new walker at its own position**, capped at `gridSize` total:

```csharp
if (randomWalkers[i].stepCount == 51 && randomWalkers.Count < gridSize)
    randomWalkers.Add(new RandomWalker((int)walker.position.x, (int)walker.position.y, gridSize, grid));
```

That mid-life split is what makes the result a *branching cave network* rather than five
independent corridors. Because children spawn on top of their parent's path, every branch is
connected to the region that produced it by construction — the generator never has to run a
connectivity pass or discard unreachable rooms.

Movement uses C# relational patterns over a single random draw, with bounds checks that clamp
rather than wrap, so walkers cannot escape the grid:

```csharp
switch (randomValue)
{
    case < 0.25f: if (position.x - 1 <= 0) { break; } position.x--; break;
    case < 0.5f:  if (position.x + 1 >= gridSize-1) { break; } position.x++; break;
    ...
}
```

---

## Bitmask Autotiling

Once the grid is carved, every solid cell needs the wall mesh that matches its exposed faces.
Rather than branching over cases, each cell samples its four orthogonal neighbours and
accumulates a bitfield:

```csharp
if (grid[i, j + 1]) { WallIndex += 1; }   // north
if (grid[i + 1, j]) { WallIndex += 2; }   // east
if (grid[i, j - 1]) { WallIndex += 4; }   // south
if (grid[i - 1, j]) { WallIndex += 8; }   // west

GameObject Wall = Instantiate(Walls[WallIndex], position, Walls[WallIndex].transform.rotation, WallsContainer.transform);
```

Four bits give 16 possible neighbour configurations, indexing directly into a 16-entry prefab
list. Adding a new wall style means authoring 16 prefabs — no code change, and no conditional
logic to extend.

> **A correction to my own source comment:** the code calls this "Marching Cube Algorithm." It
> isn't. Marching cubes is 3D isosurface extraction across 256 cases; this is **marching
> squares / bitmask autotiling** — the 2D, 16-case relative. Same family, different algorithm,
> and worth naming correctly.

---

## What I'd Do Differently

- **The termination condition is fragile.** The carve loop ends on
  `finishedSteps &= randomWalkers[randomWalkers.Count - 1].stepCount < 100` — it only ever
  inspects the *last* walker in the list. It works because children are appended and finish
  last, but it's an accident of list ordering rather than a stated invariant. Checking that
  *all* walkers are done would say what's actually meant.
- **Mesh bounds are queried inside the double loop.**
  `floorTiles[0].GetComponent<MeshFilter>().sharedMesh.bounds` is recomputed for every cell —
  `gridSize²` component lookups for a value that never changes. It belongs above the loop.
- **No pooling.** At `gridSize = 100` this instantiates up to 10,000 GameObjects in `Start()`,
  which is a visible hitch. Pooling, or merging static tiles into a combined mesh, would fix
  both the spike and the draw-call count.
- **Dead code.** A commented-out grid-update block still sits inside `PopulateGrid()`.
- **Walkers share a mutable `bool[,]` reference** with no coordination. It happens to be safe
  because carving is idempotent — writing `true` twice is harmless — but that's load-bearing
  luck, not design.

---

## Related

Companion project: [terrainGeneration](https://github.com/Schokolasbotter/terrainGeneration) —
procedural terrain via frequency-domain spectral synthesis and Diamond-Square.

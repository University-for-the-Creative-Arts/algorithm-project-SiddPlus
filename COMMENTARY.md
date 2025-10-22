This system implements a procedural generation module for Unreal Engine, primarily focused on creating a randomized, heightmap-based terrain mesh and populating it with static meshes and actors.

## System Overview and Implementation

The core of the system is the `AProceduralGeneration` Actor, which uses Unreal's `UProceduralMeshComponent` to construct the terrain geometry at runtime.

1.  Multiplayer Synchronization: The system is designed for deterministic multiplayer environments. The map generation is driven by a single integer `Seed`, which is a replicated property (`UPROPERTY(ReplicatedUsing = OnRep_Seed)`).
    * The server (`HasAuthority()`) generates a seed based on the current time in `BeginPlay()` and sets the replicated variable.
    * Setting the seed triggers the `OnRep_Seed()` function on all clients (and is manually called on the server).
    * `OnRep_Seed()` calls `GenerateMap()`, ensuring all instances of the game use the same seed, thus generating an identical, synchronized world.

2.  Generation Pipeline: The `GenerateMap()` function orchestrates the process:
    * It initializes the deterministic random stream: `FMath::RandInit(Seed)`.
    * It calculates a persistent `NoiseOffset` using the seeded random stream.
    * It calls `CreateVertices()` and `CreateTriangles()` to build the mesh.
    * It calculates mesh normals and tangents using `UKismetProceduralMeshLibrary::CalculateTangentsForMesh`.
    * It creates the final mesh section: `ProceduralMesh->CreateMeshSection(...)`.
    * It calls `SetupHISMComponents()` and `PopulateObjects()`.

## Procedural and Generative Aspects

The approach uses several procedural techniques to create a varied, repeatable world.

* Terrain Generation: Vertices are created in a grid, and their Z-coordinate is determined by Perlin Noise via `FMath::PerlinNoise2D`. This creates smooth, natural-looking elevation from a 2D input space. The use of a consistent `Seed` and a randomly generated, persistent `NoiseOffset` ensures variation between sessions but identical terrain for all players within a session.
    $$Z = \text{PerlinNoise2D}((i \cdot \text{NoiseScale} + \text{NoiseOffset}.X), (j \cdot \text{NoiseScale} + \text{NoiseOffset}.Y)) \cdot \text{ZMultiplier}$$
* Object Placement (Algorithms): Objects are populated using two primary methods, both relying on the seeded random stream:
    1.  Limited-Count Spawning: Items with a `MaxSpawnCount > 0` are placed randomly until the count is met or max attempts are reached. The function `GetRandomValidSpawnLocation` repeatedly picks a random vertex, checks its slope angle against `MaxSlopeAngle`, and verifies collision-free placement using a spatial grid (see below) before returning a valid spawn point.
    2.  Density Spawning: Items with `MaxSpawnCount == 0` are checked at *every terrain vertex* within a safe inner border. They are spawned if a random value (`FMath::FRand()`) is less than their defined `Density` value, allowing for uniform ground cover (like grass or rocks).
* Border Spawning: Actors and meshes intended for the map's edges are spawned in a specific loop that selects a random number of borders and then attempts to place objects at predetermined grid-fraction locations (Left, Middle, Right) along those borders, adding a random planar offset for variation.

---

## Tools and Data Structures

* Unreal Engine APIs:
    * `UProceduralMeshComponent`: Used to generate the primary terrain mesh.
    * `UHierarchicalInstancedStaticMeshComponent (HISM)`: Used for efficient rendering of large numbers of repeated meshes (trees, rocks) by grouping them into single draw calls, improving performance.
    * `FMath::PerlinNoise2D`: The core algorithm for terrain height generation.
    * `FMath::RandInit / FMath::RandRange`: Used for deterministic randomness in noise offset, object placement, rotation, and scale.
* Spatial Grid (`SpawnedObjectGrid`): To efficiently check for object overlap, a `TMap<FIntPoint, TArray<FSpawnedObjectInfo>>` is used. This structure maps 2D grid coordinates to a list of objects within that cell. `IsLocationClear` checks the object's grid cell and its 8 neighbors, minimizing the number of collision checks.
* Raycasting (`GetTerrainPointAtWorldLocationXY`): After an initial random position is found on the vertex grid, a raycast (`LineTraceComponent`) is performed from above the terrain down to the generated procedural mesh. This provides a more accurate ground collision point and the correct surface normal for objects to be rotated and aligned to the terrain's slope.
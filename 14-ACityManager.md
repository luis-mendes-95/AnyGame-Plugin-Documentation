# ACityManager

> **Module:** AnyGame World | **Header:** `World/City/Manager/CityManager.h` | **Parent:** `AActor`

## Overview

`ACityManager` is the **procedural city generator** — it reads a spline, detects directional segments, spawns road modules (straight, curve, cross, start/end), places spawn zones, and optimizes everything with HISM. Supports editor real-time preview with instance limits.

## Design Principles

**Spline-Driven:** City layout follows a `USplineComponent`. Direction changes become curves, overlapping roads become crossings.

**HISM Optimization:** All static meshes are batched into `UHierarchicalInstancedStaticMeshComponent` for draw call reduction.

**Data Asset Driven:** Road types and spawn zones are defined in `UCityBlockAsset` data assets, not hardcoded.

**Editor Preview Limit:** Prevents editor freeze by capping instances during real-time construction script.

**World Reacts:** CityManager is a World Actor — it doesn't contain gameplay logic, only spawns environment.

## Architecture Overview

```
ACityManager
     │
     ├── CitySpline (USplineComponent)
     │
     ├── CityBlocksAssets[] (UCityBlockAsset references)
     │
     └── GenerateCity()
              │
              ├── DetectSegments() → FSegmentData[]
              │
              ├── SpawnRoadSegments()
              │   ├── Start module
              │   ├── Curve modules (direction change)
              │   ├── Straight modules
              │   └── End module
              │
              ├── RemoveOverlappingRoadInstances()
              │   └── Replace overlaps with Cross modules
              │
              └── SpawnZones (per module)
                  ├── SpawnZoneMeshes() → HISM
                  ├── SpawnZoneActors() → AActor
                  └── SpawnReferenceActorMeshes() → Extract from BP
```

## Road Module Types

| Type | Asset Keyword | Purpose |
|------|---------------|---------|
| **Straight** | `Road` (excluding `Curve`) | Main road segments |
| **Curve** | `RoadCurve` | 90° direction changes |
| **Cross** | `RoadCross` | Intersection (auto-generated at overlaps) |
| **StartEnd** | `RoadStartEnd` | Spline endpoints |

Modules are found by `FindRoadType(Keyword, Exclude)` scanning `CityBlocksAssets`.

## Properties

### Settings

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `LoadRadius` | `float` | 3000 | HISM visibility culling radius |

### Components

| Property | Type | Purpose |
|----------|------|---------|
| `CitySpline` | `USplineComponent*` | Main spline defining city layout |

### Data Assets

| Property | Type | Purpose |
|----------|------|---------|
| `CityBlocksAssets` | `TArray<TSoftObjectPtr<UCityBlockAsset>>` | All city block definitions |
| `RoadVariations` | `TArray<TSoftObjectPtr<UCityBlockAsset>>` | Straight road variations (cyclic) |
| `RoadCurveVariations` | `TArray<TSoftObjectPtr<UCityBlockAsset>>` | Curve variations (cyclic) |
| `RoadCrossVariations` | `TArray<TSoftObjectPtr<UCityBlockAsset>>` | Cross variations |
| `RoadStartEndVariations` | `TArray<TSoftObjectPtr<UCityBlockAsset>>` | Start/End variations |

### Runtime

| Property | Type | Purpose |
|----------|------|---------|
| `SpawnedActors` | `TArray<AActor*>` | All spawned actors (for cleanup) |
| `SpawnedMeshes` | `TArray<UHISMC*>` | All HISM components |
| `RuntimeDecals` | `TArray<UDecalComponent*>` | Spawned decals |
| `RuntimeParticles` | `TArray<UParticleSystemComponent*>` | Spawned particles |
| `RoadModuleToSpawnedMeshes` | `TMap<int32, TArray<FHISMInstanceRef>>` | Module → instances mapping |

### Editor Preview

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `bAutoGenerate` | `bool` | true | Real-time generation in editor |
| `bEnableEditorPreviewLimit` | `bool` | true | Cap instances in editor |
| `MaxInstancesPreview` | `int32` | 2000 | Max instances in editor preview |
| `EditorPreviewSpawnCount` | `int32` | 0 | Current preview instance count |

## Key Structures

### FSegmentData

```cpp
struct FSegmentData
{
    int32 StartIndex;      // Spline point start
    int32 EndIndex;        // Spline point end
    FVector SnapDir;       // Snapped direction (±X or ±Y)
    float SegmentLength;   // Total length in units
};
```

### FHISMInstanceRef

```cpp
struct FHISMInstanceRef
{
    UHierarchicalInstancedStaticMeshComponent* HISM;
    int32 InstanceIndex;
};
```

## Generation Pipeline

### 1. DetectSegments

Analyzes spline and groups points by cardinal direction:

```cpp
TArray<FSegmentData> DetectSegments(int32 NumPoints)
{
    // For each spline point:
    // 1. Calculate direction to next point
    // 2. Snap to nearest cardinal (±X or ±Y)
    // 3. If direction changes → new segment
    // 4. Accumulate segment length
    
    // Returns array of segments with:
    // - Start/End indices
    // - Snapped direction
    // - Total length
}
```

### 2. SpawnRoadSegments

Places road modules along detected segments:

```cpp
void SpawnRoadSegments(...)
{
    // 1. Spawn START module (recessed by 1 module length)
    
    // 2. For each segment:
    //    a. If direction changed from previous → spawn CURVE
    //    b. Calculate NumModules = SegmentLength / RoadLength
    //    c. Spawn STRAIGHT modules
    //    d. For each module → process SpawnZones
    
    // 3. Spawn END module (rotated 180°)
}
```

### 3. RemoveOverlappingRoadInstances

Detects overlaps and replaces with crossings:

```cpp
void RemoveOverlappingRoadInstances(...)
{
    while (bHasOverlap)
    {
        // 1. Get all road instance positions
        // 2. Find pairs closer than RoadLength * 0.5
        // 3. Mark for removal (+ adjacent instances)
        // 4. Remove from HISM (highest index first)
        // 5. Spawn CROSS at overlap center
        // 6. Process cross SpawnZones
    }
}
```

## Spawn Zone System

Each road module has associated `FSpawnZone` definitions:

```cpp
struct FSpawnZone
{
    TArray<UStaticMesh*> PossibleMeshes;
    TArray<TSubclassOf<AActor>> PossibleActors;
    FBox Bounds;
    bool bRandomSpawn;
    bool bAttachSequentially;
    bool bAlignAlongY;
    bool bRandomRotation;
    FRotator SpawnRotation;
    float Intensity;
    float AttachedSpacing;
};
```

### SpawnZoneMeshes

Three spawn modes:

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Random** | `bRandomSpawn = true` | Random positions within bounds, intensity-scaled attempts |
| **Sequential** | `bAttachSequentially = true` | Line along X or Y axis with spacing |
| **Grid** | Default | Grid pattern, spacing based on intensity |

### SpawnZoneActors

Same three modes, but spawns full actors instead of HISM instances.

### SpawnReferenceActorMeshes

Extracts components from a Blueprint "reference actor":

```cpp
void SpawnReferenceActorMeshes(AActor* ReferenceActor, ...)
{
    // 1. Spawn temp invisible actor
    // 2. Recursively gather components (including ChildActors)
    // 3. Extract and add to HISM:
    //    - UStaticMeshComponent
    //    - UInstancedStaticMeshComponent
    //    - UHierarchicalInstancedStaticMeshComponent
    // 4. Duplicate light components:
    //    - URectLightComponent
    //    - UPointLightComponent
    // 5. Spawn interactable actors:
    //    - AChair
    //    - ASoundBox
    // 6. Copy spline components
    // 7. Spawn decals and particles
    // 8. Destroy temp actor
}
```

## HISM Management

### CreateHISM

```cpp
UHierarchicalInstancedStaticMeshComponent* CreateHISM(UStaticMesh* Mesh)
{
    UHISMC* NewHISM = NewObject<UHISMC>(this);
    NewHISM->SetStaticMesh(Mesh);
    NewHISM->SetupAttachment(RootComponent);
    NewHISM->RegisterComponent();
    return NewHISM;
}
```

### GetOrCreateHISM

```cpp
UHISMC* GetOrCreateHISM(UStaticMesh* Mesh, TMap<UStaticMesh*, UHISMC*>& Cache)
{
    if (UHISMC** Found = Cache.Find(Mesh))
        return *Found;
    
    UHISMC* NewHISM = CreateHISM(Mesh);
    Cache.Add(Mesh, NewHISM);
    return NewHISM;
}
```

### UpdateHISMVisibility (Runtime Culling)

```cpp
void UpdateHISMVisibility()
{
    FVector PlayerLocation = GetPlayerPawn()->GetActorLocation();
    
    for (UHISMC* HISM : SpawnedMeshes)
    {
        float Distance = FVector::Dist(PlayerLocation, HISM->Bounds.Origin);
        bool bShouldBeVisible = Distance < LoadRadius;
        
        HISM->SetVisibility(bShouldBeVisible, true);
        HISM->SetComponentTickEnabled(bShouldBeVisible);
        HISM->SetCollisionEnabled(bShouldBeVisible ? QueryOnly : NoCollision);
    }
}
```

## Editor Preview Limit

Prevents editor freeze during real-time generation:

```cpp
bool IsEditorPreviewWorld() const
{
    return World->WorldType == Editor || World->WorldType == EditorPreview;
}

bool HasEditorPreviewLimit() const
{
    return bEnableEditorPreviewLimit && IsEditorPreviewWorld() && MaxInstancesPreview > 0;
}

bool ConsumeEditorPreviewBudget(int32 Count)
{
    if (!HasEditorPreviewLimit()) return true;
    if (!CanSpawnMoreInEditorPreview(Count)) return false;
    
    EditorPreviewSpawnCount += Count;
    return true;
}
```

Every spawn call checks the budget:

```cpp
if (HasEditorPreviewLimit() && !ConsumeEditorPreviewBudget(1))
    return;  // Stop spawning
```

## Lifecycle

### BeginPlay (Runtime)

```cpp
void BeginPlay()
{
    ClearCity();
    LoadCityBlocksFromDataTable();
    GenerateCity();
}
```

### OnConstruction (Editor Real-time)

```cpp
void OnConstruction(const FTransform& Transform)
{
    if (!bAutoGenerate) return;
    if (!IsEditorPreviewWorld()) return;
    
    ResetEditorPreviewCounter();
    PreloadCityBlockAssets();
    GenerateCity();
}
```

### Tick

```cpp
void Tick(float DeltaTime)
{
    UpdateHISMVisibility();  // Runtime culling
    
    if (bDebugGeneral)
        DebugGeneral();
}
```

## Editor Buttons (Details Panel)

| Button | Function | Purpose |
|--------|----------|---------|
| `Editor_GenerateCity` | Full regeneration | Reset + preload + generate |
| `Editor_UpdateCity` | Same as generate | Alias for clarity |
| `Editor_ClearCity` | Remove all spawned | Clean slate |
| `Editor_ResetPreviewCounter` | Reset budget | Allow more spawns |
| `Editor_ValidateDebugSpline` | Draw debug | Visualize spline points |

## Quick Start

### 1. Create CityManager in level:

```
World Outliner → Add Actor → ACityManager
```

### 2. Edit spline in viewport:

```
Select CityManager → CitySpline → Add/Move points
- Straight segments = road modules
- Direction changes = curves
- Overlapping paths = crossings
```

### 3. Assign city block assets:

```
CityManager → Details:
├── CityBlocksAssets:
│   ├── DA_Road
│   ├── DA_RoadCurve
│   ├── DA_RoadCross
│   └── DA_RoadStartEnd
└── RoadVariations: (optional cyclic variations)
```

### 4. Configure editor preview:

```
CityManager → Details → AnyGame|City|Editor:
├── bAutoGenerate = true (real-time in editor)
├── bEnableEditorPreviewLimit = true
└── MaxInstancesPreview = 2000
```

### 5. Test generation:

```
Details Panel → AnyGame|City|Editor:
├── [Editor_GenerateCity] → Full generation
├── [Editor_ClearCity] → Clean up
└── [Editor_ValidateDebugSpline] → Debug visualization
```

### 6. Create UCityBlockAsset:

```cpp
// In UCityBlockAsset
USTRUCT()
struct FCityBlockAssetData
{
    FName BlockName;              // "Road", "RoadCurve", etc.
    bool bEnabled;
    FBlockMeshData BaseMeshData;  // Main mesh
    TArray<FBlockMeshData> OptionalMeshesData;
    TArray<FSpawnZone> SpawnZones;
    TArray<FSpawnZone> BuildingZones;
    TArray<TSubclassOf<AActor>> MeshReferenceActors;
};
```

## Performance Notes

**Editor:** Use `MaxInstancesPreview` to prevent freezing. Default 2000 is safe for most hardware.

**Runtime:** `UpdateHISMVisibility()` culls HISM by distance every tick. Adjust `LoadRadius` based on your view distance.

**HISM Batching:** All identical meshes share one HISM component. 1000 lamp posts = 1 draw call.

**Instance Removal:** When removing overlapping roads, instances are removed highest-index-first to avoid index invalidation.

## Architecture Compliance

✅ **World Reacts:** CityManager only spawns environment. No gameplay logic.

✅ **Anti-Lock-In:** Uses data assets. Remove the manager → remove the procedural city, static content remains.

✅ **Core Stability:** CityManager is in World module (extension), not Core.

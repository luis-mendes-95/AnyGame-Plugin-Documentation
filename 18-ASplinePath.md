# ASplinePath

> **Module:** AnyGame World | **Header:** `World/SplinePath/SplinePath.h` | **Parent:** `AActor`

## Overview

`ASplinePath` is an **editor-friendly spline actor** — it wraps `USplineComponent` with billboard icons at start/end points for easy visualization in the editor. Used for AI patrol routes, camera paths, or any gameplay that follows a path.

## Design Principles

**Editor Visualization:** Billboard icons mark start (green) and end (red) points. Icons auto-update when spline is edited.

**World Reacts:** SplinePath is a World Actor. It defines a path but doesn't contain gameplay logic.

**No Tick:** `PrimaryActorTick.bCanEverTick = false`. Pure data container.

**Runtime Minimal:** All billboard icons are `bHiddenInGame = true`. Zero visual overhead at runtime.

## Components

| Component | Type | Purpose |
|-----------|------|---------|
| `EditorIcon` | `UBillboardComponent` | Actor icon in editor (Root) |
| `Spline` | `USplineComponent` | The actual path |
| `StartIcon` | `UBillboardComponent` | Marks spline start point |
| `EndIcon` | `UBillboardComponent` | Marks spline end point |

## Properties

| Property | Type | Purpose |
|----------|------|---------|
| `Spline` | `USplineComponent*` | Public access to spline data |
| `EditorIcon` | `UBillboardComponent*` | Root billboard |
| `StartIcon` | `UBillboardComponent*` | Start point marker |
| `EndIcon` | `UBillboardComponent*` | End point marker |

## Editor Features

### Icon Auto-Update

Icons automatically reposition when spline points are moved:

```cpp
void OnConstruction(const FTransform& Transform)
{
    Super::OnConstruction(Transform);
    
#if WITH_EDITOR
    UpdateSplineIcons();
#endif
}

#if WITH_EDITOR
void UpdateSplineIcons()
{
    if (!Spline || Spline->GetNumberOfSplinePoints() < 2) return;
    
    // Get world positions of first and last points
    const FVector StartLocation = Spline->GetLocationAtSplinePoint(0, ESplineCoordinateSpace::World);
    const FVector EndLocation = Spline->GetLocationAtSplinePoint(
        Spline->GetNumberOfSplinePoints() - 1, 
        ESplineCoordinateSpace::World
    );
    
    // Move icons to match
    StartIcon->SetWorldLocation(StartLocation);
    EndIcon->SetWorldLocation(EndLocation);
}
#endif
```

### Custom Icons (Optional)

Icons can be loaded from plugin assets:

```cpp
#if WITH_EDITOR
void SetupEditorIcons()
{
    static ConstructorHelpers::FObjectFinder<UTexture2D> ActorIconObj(
        TEXT("/Plugin/AnyGame/Narrative/SplinePath/Icons/T_SplinePath_Actor")
    );
    static ConstructorHelpers::FObjectFinder<UTexture2D> StartIconObj(
        TEXT("/Plugin/AnyGame/Narrative/SplinePath/Icons/T_SplinePath_Start")
    );
    static ConstructorHelpers::FObjectFinder<UTexture2D> EndIconObj(
        TEXT("/Plugin/AnyGame/Narrative/SplinePath/Icons/T_SplinePath_End")
    );
    
    if (ActorIconObj.Succeeded()) EditorIcon->Sprite = ActorIconObj.Object;
    if (StartIconObj.Succeeded()) StartIcon->Sprite = StartIconObj.Object;
    if (EndIconObj.Succeeded()) EndIcon->Sprite = EndIconObj.Object;
}
#endif
```

## Lifecycle

```cpp
// Constructor: Create all components, no tick
ASplinePath()
{
    PrimaryActorTick.bCanEverTick = false;
    
    // Root billboard
    EditorIcon = CreateDefaultSubobject<UBillboardComponent>(TEXT("EditorIcon"));
    RootComponent = EditorIcon;
    EditorIcon->bHiddenInGame = true;
    
    // Spline component
    Spline = CreateDefaultSubobject<USplineComponent>(TEXT("Spline"));
    Spline->SetupAttachment(RootComponent);
    Spline->SetClosedLoop(false);
    Spline->bDrawDebug = true;
    
    // Start/End icons
    StartIcon = CreateDefaultSubobject<UBillboardComponent>(TEXT("StartIcon"));
    StartIcon->SetupAttachment(RootComponent);
    StartIcon->bHiddenInGame = true;
    
    EndIcon = CreateDefaultSubobject<UBillboardComponent>(TEXT("EndIcon"));
    EndIcon->SetupAttachment(RootComponent);
    EndIcon->bHiddenInGame = true;
    
#if WITH_EDITOR
    SetupEditorIcons();
#endif
}
```

## Quick Start

### 1. Place in level:

```
World Outliner → Add Actor → ASplinePath
```

### 2. Edit spline points:

```
Select SplinePath → Select Spline component → 
Add/Move/Delete points in viewport
```

### 3. Use with AI patrol:

```cpp
// In UCharAIComponent or Behavior Tree
ASplinePath* PatrolPath = ...;

if (PatrolPath && PatrolPath->Spline)
{
    USplineComponent* Spline = PatrolPath->Spline;
    
    // Get point count
    int32 NumPoints = Spline->GetNumberOfSplinePoints();
    
    // Get location at point
    FVector Location = Spline->GetLocationAtSplinePoint(PointIndex, ESplineCoordinateSpace::World);
    
    // Get location along spline (0-1)
    FVector LocationAlongSpline = Spline->GetLocationAtDistanceAlongSpline(Distance, ESplineCoordinateSpace::World);
    
    // Get direction at point
    FVector Direction = Spline->GetDirectionAtSplinePoint(PointIndex, ESplineCoordinateSpace::World);
}
```

### 4. Use with camera path:

```cpp
// In cinematic or camera system
void UpdateCameraAlongPath(float Progress)
{
    if (!CameraPath || !CameraPath->Spline) return;
    
    float SplineLength = CameraPath->Spline->GetSplineLength();
    float Distance = Progress * SplineLength;
    
    FVector Location = CameraPath->Spline->GetLocationAtDistanceAlongSpline(
        Distance, ESplineCoordinateSpace::World
    );
    
    FRotator Rotation = CameraPath->Spline->GetRotationAtDistanceAlongSpline(
        Distance, ESplineCoordinateSpace::World
    );
    
    CameraActor->SetActorLocationAndRotation(Location, Rotation);
}
```

### 5. Reference from other actors:

```cpp
// In any actor that needs a path
UPROPERTY(EditAnywhere, Category = "Path")
ASplinePath* FollowPath;

void BeginPlay()
{
    if (FollowPath && FollowPath->Spline)
    {
        // Start following the path
        StartFollowingSpline(FollowPath->Spline);
    }
}
```

## Common Spline Operations

### Get Total Length

```cpp
float Length = Spline->GetSplineLength();
```

### Get Point Count

```cpp
int32 NumPoints = Spline->GetNumberOfSplinePoints();
```

### Get Location at Point

```cpp
FVector Location = Spline->GetLocationAtSplinePoint(Index, ESplineCoordinateSpace::World);
```

### Get Location at Distance

```cpp
FVector Location = Spline->GetLocationAtDistanceAlongSpline(Distance, ESplineCoordinateSpace::World);
```

### Get Location at Time (0-1)

```cpp
FVector Location = Spline->GetLocationAtTime(Time, ESplineCoordinateSpace::World, true);
```

### Get Closest Point on Spline

```cpp
FVector ClosestPoint = Spline->FindLocationClosestToWorldLocation(WorldLocation, ESplineCoordinateSpace::World);
float InputKey = Spline->FindInputKeyClosestToWorldLocation(WorldLocation);
```

### Check if Closed Loop

```cpp
bool bClosed = Spline->IsClosedLoop();
```

## Integration with UCharAIComponent

SplinePath is used by the AI system for patrol and follow behaviors:

```cpp
// In UCharAIComponent (conceptual)
UPROPERTY(EditAnywhere, Category = "AI|Patrol")
ASplinePath* PatrolPath;

void StartPatrol()
{
    if (!PatrolPath || !PatrolPath->Spline) return;
    
    CurrentPatrolPointIndex = 0;
    MoveToPatrolPoint(CurrentPatrolPointIndex);
}

void OnReachedPatrolPoint()
{
    CurrentPatrolPointIndex++;
    
    if (CurrentPatrolPointIndex >= PatrolPath->Spline->GetNumberOfSplinePoints())
    {
        if (PatrolPath->Spline->IsClosedLoop())
            CurrentPatrolPointIndex = 0;
        else
            CurrentPatrolPointIndex = PatrolPath->Spline->GetNumberOfSplinePoints() - 1;
    }
    
    MoveToPatrolPoint(CurrentPatrolPointIndex);
}
```

## Architecture Compliance

✅ **World Reacts:** SplinePath defines a path. Other systems read it.

✅ **No Tick:** Pure data container with zero runtime overhead.

✅ **Editor-Friendly:** Visual icons for easy level design.

✅ **Anti-Lock-In:** Standard `USplineComponent`. Any system can read it directly.

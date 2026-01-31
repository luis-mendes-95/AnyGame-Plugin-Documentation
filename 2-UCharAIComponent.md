# UCharAIComponent

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/AI/CharAIComponent.h` | **Parent:** `UActorComponent`

## Overview

`UCharAIComponent` is a **dual-role behavior controller**: on the real Player, it records the movement trail; on NPCs, it executes AI logic (follow, combat, goto). Tick-based, no Behavior Trees.

## Design Principles

**Dual Role:** The same component exists on all characters. `IsRealPlayer()` decides whether to record trail or execute AI — never both.

**Trail > NavMesh:** NPCs follow the Player's exact path via spline stream. Guarantees identical traversal (stairs, jumps, doors) without pathfinding.

**Tag Matrix:** Factions are Actor Tags (`Player`, `Friend`, `Enemy`). "Who attacks whom" logic is resolved at compile-time, not in data assets.

**Input Bridge:** AI doesn't inject input directly. It signals *intent* (`WantsToStrafe`, `WantsToOrbit`) via interface — the user's template decides how to map it.

## States

```cpp
enum class EAIState : uint8
{
    Default,  // Idle / Wander
    Combat,   // Engaged with enemy
    Goto      // Following spline path
};
```

| State | Trigger | Behavior |
|-------|---------|----------|
| `Default` | No objective | Idle or random wander |
| `Combat` | `Enemies.Num() > 0` | Approach → Attack → Orbit loop |
| `Goto` | `GoToSpline()` called | Follows spline point-by-point |

**Follow** is not a state — it's an **override** that runs before the switch when `bIsFollowingPlayer && ActorHasTag("Friend")`.

## Systems

### 1. Follow System

Two mutually exclusive modes:

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Legacy** | `bFollowPlayerSpline = false` or has LoS | Goes directly to Player with separation |
| **Trail Spline** | `bFollowPlayerSpline = true` + LoS lost | Follows recorded trail points |

**Auto-Switch (LoS):**
```cpp
// Loses LoS for SwitchToTrailDelay → activates Trail
// Regains LoS for SwitchToDirectDelay → returns to Legacy
```

| Property | Default | Purpose |
|----------|---------|---------|
| `MinFollowDistance` | 200 | Minimum distance to Player |
| `MaxFollowDistance` | 350 | Starts following above this |
| `CatchUpDistance` | 750 | Skips points if too far |
| `bAutoTrailWhenNoLOS` | true | Enables auto-switch |

### 2. Trail Recorder (Player Only)

Records Player position at intervals, creates spline stream for NPCs to consume.

| Property | Default | Purpose |
|----------|---------|---------|
| `TrailSampleDistance` | 90 | Minimum distance between points |
| `TrailSampleInterval` | 0.10s | Sampling timer |
| `MaxTrailPoints` | 128 | Maximum points (circular queue) |

```cpp
// Only runs on the real Player
if (IsRealPlayer())
{
    EnsurePlayerTrailSpline();
    StartTrailRecorder();
}
```

### 3. Combat System

Internal state machine with per-frame decisions:

```cpp
enum class EAICombatDecision : uint8
{
    None,
    Approach,      // Slot available, out of range → walk to target
    AttackNear,    // Within NearAttackThreshold → melee attack
    AttackDistant, // Within DistantAttackThreshold → ranged attack
    Orbit          // Slot occupied or cooldown → circle the target
};
```

| Property | Default | Purpose |
|----------|---------|---------|
| `OrbitDistance` | 300 | Circle radius when orbiting |
| `OrbitTolerance` | 0 | Slack before radius correction |
| `RotationSpeed` | 5.0 | Rotation speed (interp) |

### 4. Avoidance System

Two layers: **separation** (other pawns) + **obstacle** (walls).

| Property | Default | Purpose |
|----------|---------|---------|
| `AvoidanceRadius` | 160 | Detection radius for other NPCs |
| `AvoidanceStrength` | 0.75 | Separation push force |
| `ObstacleTraceDistance` | 140 | Wall sweep distance |
| `ObstacleTraceRadius` | 28 | Sweep radius |
| `ObstacleMaxAttempts` | 7 | Alternate direction attempts |
| `ObstacleAngleStep` | 25° | Angular increment per attempt |

### 5. Auto-Interact (Traversal)

During Follow, detects obstacles and fires `Interact()` automatically (doors, ladders, etc).

| Property | Default | Purpose |
|----------|---------|---------|
| `bAutoInteractDuringFollow` | true | Enables the system |
| `FollowInteractCheckInterval` | 0.12s | Check frequency |
| `FollowInteractCooldown` | 0.80s | Cooldown between interactions |
| `FollowInteractPauseDuration` | 0.25s | Pauses movement after interact |

```cpp
// Simulates input "spam" like a player would
const int32 BurstCount = 4;
const float BurstInterval = 0.05f;
InputComp->Interact(); // + 3 scheduled via timer
```

## Input Bridge

Interface to decouple AI from the template's input system:

```cpp
UINTERFACE(BlueprintType)
class UAnyGameInputBridge : public UInterface { ... };

class IAnyGameInputBridge
{
    UFUNCTION(BlueprintNativeEvent)
    void AnyGame_SetWantsToStrafe(bool bEnable);

    UFUNCTION(BlueprintNativeEvent)
    void AnyGame_SetWantsToOrbit(bool bEnable);
};
```

**Usage:** Character Blueprint implements the interface and maps to its system (Lyra, ALS, custom).

```cpp
// AI signals intent (edge-triggered, no spam)
if (bNewWantsStrafe != bCachedWantsStrafe)
{
    IAnyGameInputBridge::Execute_AnyGame_SetWantsToStrafe(Owner, bNewWantsStrafe);
}
```

## Tag Matrix (Factions)

```cpp
// Who can attack whom
Player → Enemy ✓
Friend → Enemy ✓
Enemy  → Player ✓
Enemy  → Friend ✓
Enemy  → Enemy ✗
Friend → Player ✗
```

Enforced in `CanAttackByTagMatrix_AI()` — local helper, not exposed.

## Key Functions

| Function | Category | Purpose |
|----------|----------|---------|
| `FollowPlayer(APlayerCharacter*)` | Follow | Starts follow (requires `Friend` tag) |
| `StopFollowPlayer()` | Follow | Stops follow |
| `SetUseTrailFollow(bool)` | Follow | Manual Trail/Legacy toggle |
| `GoToSpline(AActor*, FName)` | Goto | Starts spline navigation |
| `PauseMovement(float)` | Movement | Pauses AI for duration |
| `DisengageCombat()` | Combat | Clears combat state |

## Lifecycle

```cpp
// BeginPlay: Configures role based on IsRealPlayer()
void BeginPlay()
{
    Character = Cast<APlayerCharacter>(GetOwner());
    
    if (IsRealPlayer())
    {
        EnsurePlayerTrailSpline();
        StartTrailRecorder();  // Player records
    }
    else
    {
        StopTrailRecorder();   // NPC doesn't record
    }
}

// TickComponent: Main state machine
void TickComponent(float DeltaTime, ...)
{
    if (!CanTick()) return;  // IsRealPlayer() returns false here
    
    // Friend Follow has priority (but Combat overrides)
    if (bIsFollowingPlayer && ActorHasTag("Friend"))
    {
        if (Enemies.Num() > 0) { UpdateCombatBehavior(); return; }
        UpdateFollowBehavior();
        return;
    }
    
    switch (CurrentState)
    {
        case EAIState::Default: HandleDefaultState(); break;
        case EAIState::Combat:  UpdateCombatBehavior(); break;
        case EAIState::Goto:    UpdateGotoBehavior(); break;
    }
}
```

## Quick Start

1. **Player setup:** Nothing. Trail recorder auto-enables if `IsRealPlayer()`.

2. **NPC Follower:**
```cpp
// In your NPC Blueprint or C++
NPC->Tags.Add("Friend");
NPC->GetAIComponent()->FollowPlayer(PlayerCharacter);
```

3. **NPC Enemy:**
```cpp
NPC->Tags.Add("Enemy");
// Combat starts automatically when Player enters CombatComponent range
```

4. **Spline Patrol:**
```cpp
NPC->GetAIComponent()->GoToSpline(MySplinePathActor, NAME_None);
```

5. **Debug:** `bDebugAIComponent` on `CharDebugComponent` + `bDebugFollowSpline` to visualize trail.

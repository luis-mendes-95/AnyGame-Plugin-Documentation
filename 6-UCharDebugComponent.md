# UCharDebugComponent

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Debug/CharDebugComponent.h` | **Parent:** `UActorComponent`

## Overview

`UCharDebugComponent` is a **runtime inspection tool** — it renders on-screen debug info for any component via toggle flags. No editor widgets, no ImGui — just `GEngine->AddOnScreenDebugMessage`.

## Design Principles

**Flag-Per-Component:** Each component has its own `bDebug[Name]` bool. Enable in Details panel or at runtime.

**Zero Cost When Off:** Debug functions only run when their flag is true. No overhead in shipping builds (use `#if WITH_EDITOR` if needed).

**Centralized:** All character debugging lives here. Components don't debug themselves — they expose state, this component reads it.

**Log Category Integration:** Some components (AI, Combat) use conditional log macros that check this component's flags.

## Debug Flags

| Flag | Target | Info Displayed |
|------|--------|----------------|
| `bDebugAIComponent` | `UCharAIComponent` | State, movement, combat, orbit, avoidance, spline |
| `bDebugAnimInstance` | `UCharAnimInstance` | Character ref, attachment state, hands occupied |
| `bDebugAnimComponent` | `UCharAnimComponent` | IK transforms |
| `bDebugAttachmentsComponent` | `UCharAttachmentsComponent` | State, slots, attached actors |
| `bDebugCombatComponent` | `UCharCombatComponent` | All flags, stats, targets, attacker slot, direction |
| `bDebugInputComponent` | `UCharInputComponent` | Reserved |
| `bDebugCharacterMovementComponent` | `UCharacterMovementComponent` | Mode, velocity, root motion, avoidance, floor |
| `bDebugInteractionComponent` | `UCharInteractionComponent` | Focus target, overlapping items, vehicle |
| `bDebugCareer` | `UCharCareerComponent` | Reserved |
| `bDebugSoundComponent` | `UCharSoundComponent` | Reserved |
| `bDebugWidgetComponent` | `UCharWidgetComponent` | Reserved |

## Debug Output Sections

### AI Component (`bDebugAIComponent`)

```
==== AI COMPONENT ====
Character: BP_Enemy_01
bIsAIActive: TRUE
CurrentState: EAIState::Combat

==== MOVEMENT ====
CurrentMoveDirection: (0.5, 0.8, 0.0)
TimeSinceDirectionChange: 1.23
ChangeDirectionInterval: 3.50

==== MOVEMENT PAUSE ====
bIsMovementPaused: FALSE
PauseTimeRemaining: 0.00
DefaultPauseDuration: 1.00
RotationSpeed: 5.00

==== COMBAT ====
bIsEngagedInCombat: TRUE
Enemies Count: 2
Enemies[0]: BP_Player (Dist: 245.00)
Enemies[1]: BP_Friend (Dist: 380.00)

==== ORBIT ====
ShouldOrbit(): TRUE
OrbitDistance: 300.00
OrbitTolerance: 0.00

==== AVOIDANCE ====
AvoidanceRadius: 160.00
AvoidanceStrength: 0.75
ObstacleTraceDistance: 140.00
ObstacleTraceRadius: 28.00
ObstacleTraceHeight: 35.00
ObstacleMaxAttempts: 7
ObstacleAngleStep: 25.00
ObstacleTraceChannel: ECC_Visibility

==== GOTO / SPLINE ====
TargetSpline: None
CurrentPathIndex: 0
GotoAcceptanceRadius: 50.00
```

### Combat Component (`bDebugCombatComponent`)

```
==== COMBAT COMPONENT DEBUG (NEW) ====
bCanAttack (FLAG): TRUE
CanAttack() (GATE): TRUE
Is Attacking: FALSE
Is Getting Hit: FALSE
Is CounterAttack: FALSE
Is Current Finisher: FALSE
Is Aiming: FALSE
Is Dead: FALSE

Can Defend (bCanDeffend): FALSE
Can Be Defended (bCanBeDeffended): FALSE

Health: 85.00 / 100.00
Deffense: 100.00 / 100.00

ComboCounter: 0
CounterAttackCounter: 0
AttackFinisherCounter: 0

TargetDistance: 187.50
DistantAttackThreshold: 120.00
CurrentAttackTarget: BP_Player
CurrentAttacker (SLOT): NONE
CurrentAttackTargets: BP_Player | BP_Friend |

DirToTargetYaw: 12.50 | AttackAnim: Front
DirFromTargetYaw: -167.50 | HitAnim: Back

RotateTowardsTarget: FALSE
FacingRotationSpeed: 12.00

DeathImpulseType: EDeathImpulseType::Backward
DeathImpulseStrength: 500.00
```

### AnimInstance (`bDebugAnimInstance`)

```
==== ANIM INSTANCE DEBUG ====
Character Pointer: BP_Player_C_0
AttachmentState: ECharAttachmentState::None
HandsOccupied: False
```

### Attachments Component (`bDebugAttachmentsComponent`)

```
AttachmentState: 0 | HandsOccupied: NO
Slot: 0 -> Actor: BP_Sword_C_0
Slot: 1 -> Empty
```

### Character Movement (`bDebugCharacterMovementComponent`)

```
Movement Mode: 1
Custom Mode: 0
Velocity: (125.0, 0.0, 0.0)
Max Walk Speed: 600.00
Gravity Scale: 1.00
Ground Friction: 8.00
Braking Decel Walking: 2048.00

Has Anim Root Motion: No
Anim Root Motion Velocity: (0.0, 0.0, 0.0)

Avoidance UID: 0
Avoidance Weight: 0.50
Avoidance Velocity Lock: No

Min Floor Dist: 1.90
Max Floor Dist: 2.40
Sweep Edge Reject Distance: 0.15
```

### Interaction Component (`bDebugInteractionComponent`)

```
────────────────────────────
UCharInteractionComponent: ACTIVE
────────────────────────────
Character: BP_Player
Vehicle: None
CurrentActorToFocus: BP_Door_01
OverlappingItems Count: 2
OverlappingItems[0]: BP_Door_01
OverlappingItems[1]: BP_Chest_01
OverlappingItem: BP_Door_01
```

## Log Category Integration

Other components use conditional macros that check this component's flags:

```cpp
// In UCharAIComponent
#define AI_LOG(Component, Format, ...) \
    if (Component && Component->Character && \
        Component->Character->GetDebugComponent() && \
        Component->Character->GetDebugComponent()->bDebugAIComponent) \
    { \
        UE_LOG(LogAICombat, Warning, Format, ##__VA_ARGS__); \
    }

// In UCharCombatComponent
#define COMBAT_LOG(Component, Format, ...) \
    if (Component && Component->Character && \
        Component->Character->GetDebugComponent() && \
        Component->Character->GetDebugComponent()->bDebugCombatComponent) \
    { \
        UE_LOG(LogCombat, Warning, Format, ##__VA_ARGS__); \
    }
```

This means enabling `bDebugAIComponent` also enables detailed output logs.

## Properties

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `Character` | `APlayerCharacter*` | — | Cached owner reference |
| `bDebugAIComponent` | `bool` | false | Toggle AI debug |
| `bDebugAnimInstance` | `bool` | false | Toggle AnimInstance debug |
| `bDebugAnimComponent` | `bool` | false | Toggle AnimComponent debug |
| `bDebugAttachmentsComponent` | `bool` | false | Toggle Attachments debug |
| `bDebugCombatComponent` | `bool` | false | Toggle Combat debug |
| `bDebugInputComponent` | `bool` | false | Toggle Input debug (reserved) |
| `bDebugCharacterMovementComponent` | `bool` | false | Toggle Movement debug |
| `bDebugInteractionComponent` | `bool` | false | Toggle Interaction debug |
| `bDebugCareer` | `bool` | false | Toggle Career debug (reserved) |
| `bDebugSoundComponent` | `bool` | false | Toggle Sound debug (reserved) |
| `bDebugWidgetComponent` | `bool` | false | Toggle Widget debug (reserved) |

## Lifecycle

```cpp
// Constructor: Enable tick
UCharDebugComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
}

// BeginPlay: Cache owner
void BeginPlay()
{
    Character = Cast<APlayerCharacter>(GetOwner());
}

// Tick: Run active debug functions
void TickComponent(float DeltaTime, ...)
{
    DebugGeneralCharacter(DeltaTime);
}

void DebugGeneralCharacter(float DeltaTime)
{
    if (!GEngine || !Character) return;
    
    if (bDebugAIComponent)          DebugAIComponent();
    if (bDebugAnimInstance)         DebugAnimInstance();
    if (bDebugAnimComponent)        DebugAnimComponent();
    if (bDebugAttachmentsComponent) DebugAttachmentsComponent();
    if (bDebugCombatComponent)      DebugCombatComponent();
    if (bDebugInputComponent)       DebugInputComponent();
    if (bDebugCharacterMovementComponent) DebugCharacterMovementComponent();
    if (bDebugInteractionComponent) DebugInteractionComponent();
}
```

## Quick Start

### 1. Enable debug in editor:
```
Select Character → Details → Debug Component
├── bDebugCombatComponent ✓
└── bDebugAIComponent ✓
```

### 2. Enable debug at runtime:
```cpp
Character->GetDebugComponent()->bDebugCombatComponent = true;
```

### 3. Toggle via console (Blueprint):
```
Create console command that sets:
DebugComp->bDebugAIComponent = !DebugComp->bDebugAIComponent;
```

### 4. Add new debug section:
```cpp
// In header
UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Debug")
bool bDebugMyNewComponent = false;

// In cpp
void UCharDebugComponent::DebugGeneralCharacter(float DeltaTime)
{
    // ...existing...
    if (bDebugMyNewComponent) DebugMyNewComponent();
}

void UCharDebugComponent::DebugMyNewComponent()
{
    if (!GEngine || !Character) return;
    
    UMyComponent* Comp = Character->GetMyComponent();
    if (!Comp) return;
    
    int32 Line = 0;
    GEngine->AddOnScreenDebugMessage(Line++, 0.1f, FColor::Green, 
        FString::Printf(TEXT("MyValue: %d"), Comp->MyValue));
}
```

### 5. Use conditional logging in other components:
```cpp
// Define macro at top of cpp
#define MY_LOG(Component, Format, ...) \
    if (Component && Component->Character && \
        Component->Character->GetDebugComponent() && \
        Component->Character->GetDebugComponent()->bDebugMyNewComponent) \
    { \
        UE_LOG(LogMyComponent, Warning, Format, ##__VA_ARGS__); \
    }

// Use throughout code
MY_LOG(this, TEXT("[MyFunction] Value=%d"), SomeValue);
```

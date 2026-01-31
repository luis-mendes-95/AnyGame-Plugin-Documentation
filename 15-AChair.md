# AChair

> **Module:** AnyGame World | **Header:** `World/Interactables/Chair/Chair.h` | **Parent:** `AActor`, `IInteractable`

## Overview

`AChair` is an **interactable chair** — it implements `IInteractable` to allow characters to sit and stand up with motion-warped animations. Manages occupation state and dynamic interaction prompts.

## Design Principles

**World Reacts:** Chair is a World Actor. It only reacts to character interactions — it never initiates gameplay logic.

**IInteractable Compliant:** Full implementation of the interaction interface. Works seamlessly with `UCharInteractionComponent`.

**Motion Warping:** Uses `UMotionWarpingComponent` to align character position during sit/stand animations.

**Player/AI Parity:** Same interaction flow for players and NPCs. Widget visibility is AI-aware.

## Components

| Component | Type | Purpose |
|-----------|------|---------|
| `ChairMesh` | `UStaticMeshComponent` | Visual mesh (Root) |
| `InteractionSphere` | `USphereComponent` | Overlap detection (radius 100) |
| `InteractionWidget` | `UWidgetComponent` | Interaction prompt UI |
| `WarpTargetSit` | `UArrowComponent` | Motion warp target for sitting |
| `WarpTargetStandUp` | `UArrowComponent` | Motion warp target for standing |

## Properties

### Configuration

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `PlayerAttachmentOffset` | `FVector` | — | Offset for character attachment |
| `PromptSit` | `FText` | "Press E to Sit" | Prompt when chair is free |
| `PromptStandUp` | `FText` | "Press E to Stand Up" | Prompt when sitting |

### State

| Property | Type | Purpose |
|----------|------|---------|
| `CurrentSitter` | `APlayerCharacter*` | Character currently sitting |
| `bIsOccupied` | `bool` | Chair occupation state |

## IInteractable Implementation

### CanInteract

```cpp
bool CanInteract_Implementation(AActor* InteractingActor) const
{
    APlayerCharacter* Character = Cast<APlayerCharacter>(InteractingActor);
    if (!Character) return false;
    
    // Can interact if:
    // - Chair is free, OR
    // - Character is already sitting (to stand up)
    return !bIsOccupied || IsCharacterSitting(Character);
}
```

### Interact

```cpp
bool Interact_Implementation(AActor* InteractingActor)
{
    APlayerCharacter* Character = Cast<APlayerCharacter>(InteractingActor);
    if (!Character) return false;
    
    // If already sitting in THIS chair → stand up
    if (IsCharacterSitting(Character))
    {
        StandUp(Character);
        return true;
    }
    
    // If occupied by someone else → deny
    if (bIsOccupied) return false;
    
    // Sit down
    Sit(Character);
    return true;
}
```

### GetInteractionPrompt

```cpp
FText GetInteractionPrompt_Implementation() const
{
    if (bIsOccupied && CurrentSitter)
        return PromptStandUp;  // "Press E to Stand Up"
    
    return PromptSit;  // "Press E to Sit"
}
```

### OnEnterInteractionRange

```cpp
void OnEnterInteractionRange_Implementation(AActor* InteractingActor)
{
    APlayerCharacter* Character = Cast<APlayerCharacter>(InteractingActor);
    if (!Character) return;
    
    // Hide widget for AI characters
    if (UCharAIComponent* AIComp = Character->GetAIComponent())
    {
        if (AIComp->bIsAIActive)
            return;
    }
    
    InteractionWidget->SetVisibility(true);
}
```

### OnExitInteractionRange

```cpp
void OnExitInteractionRange_Implementation(AActor* InteractingActor)
{
    InteractionWidget->SetVisibility(false);
}
```

## Chair API

| Function | Purpose |
|----------|---------|
| `Sit(Character)` | Make character sit down |
| `StandUp(Character)` | Make character stand up |
| `IsOccupied()` | Check if chair is occupied |
| `GetCurrentSitter()` | Get sitting character |

## Sit Flow

```cpp
void Sit(APlayerCharacter* InteractingCharacter)
{
    if (!InteractingCharacter || bIsOccupied) return;
    
    // 1. Update state
    CurrentSitter = InteractingCharacter;
    bIsOccupied = true;
    
    // 2. Hide widget
    InteractionWidget->SetVisibility(false);
    
    // 3. Setup motion warping
    SetupWarpTarget(InteractingCharacter, WarpTargetSit, FName("WarpTargetSit"));
    
    // 4. Play sit montage
    AnimComp->PlayMontage(FName("ChairMontages"), FName("SitDown"), FName("Default"), true);
    
    // 5. Disable chair collision (prevent push)
    ChairMesh->SetCollisionEnabled(ECollisionEnabled::NoCollision);
    
    // 6. Timer: Update CharacterActivity after 0.5s
    SetTimer(0.5f, [Character]() {
        AnimInstance->CharacterActivity = ECharacterActivity::Sitting;
    });
}
```

## StandUp Flow

```cpp
void StandUp(APlayerCharacter* InteractingCharacter)
{
    if (!InteractingCharacter || InteractingCharacter != CurrentSitter) return;
    
    // 1. Setup motion warping
    SetupWarpTarget(InteractingCharacter, WarpTargetStandUp, FName("WarpTargetStandUp"));
    
    // 2. Clear state
    CurrentSitter = nullptr;
    bIsOccupied = false;
    
    // 3. Play stand montage
    AnimComp->PlayMontage(FName("ChairMontages"), FName("StandUp"), FName("Default"), true);
    
    // 4. Timer: Restore state after 1.5s
    SetTimer(1.5f, [Character, Chair]() {
        AnimInstance->CharacterActivity = ECharacterActivity::None;
        ChairMesh->SetCollisionEnabled(ECollisionEnabled::QueryAndPhysics);
    });
}
```

## Motion Warping

```cpp
void SetupWarpTarget(APlayerCharacter* ThisCharacter, UArrowComponent* WarpTarget, FName WarpName)
{
    if (!ThisCharacter || !WarpTarget) return;
    
    if (UMotionWarpingComponent* MotionWarpingComp = ThisCharacter->FindComponentByClass<UMotionWarpingComponent>())
    {
        MotionWarpingComp->AddOrUpdateWarpTargetFromComponent(
            WarpName,           // "WarpTargetSit" or "WarpTargetStandUp"
            WarpTarget,         // Arrow component
            NAME_None,          // Bone name (none)
            true,               // Follow component
            FVector::ZeroVector,
            FRotator::ZeroRotator
        );
    }
}
```

**Animation Requirement:** Montages in "ChairMontages" collection must have Motion Warping notifies referencing `WarpTargetSit` and `WarpTargetStandUp`.

## Overlap Handlers

### OnOverlapBegin

```cpp
void OnOverlapBegin(...)
{
    APlayerCharacter* OverlapChar = Cast<APlayerCharacter>(OtherActor);
    if (!OverlapChar) return;
    
    // Register with InteractionComponent
    if (UCharInteractionComponent* InteractComp = OverlapChar->GetInteractionComponent())
    {
        InteractComp->SetOverlappingItem(this);
    }
}
```

### OnOverlapEnd

```cpp
void OnOverlapEnd(...)
{
    APlayerCharacter* OverlapChar = Cast<APlayerCharacter>(OtherActor);
    if (!OverlapChar) return;
    
    // Unregister from InteractionComponent
    if (UCharInteractionComponent* InteractComp = OverlapChar->GetInteractionComponent())
    {
        if (InteractComp->GetOverlappingItem() == this)
        {
            InteractComp->SetOverlappingItem(nullptr);
        }
    }
    
    InteractionWidget->SetVisibility(false);
}
```

## IsCharacterSitting Check

```cpp
bool IsCharacterSitting(APlayerCharacter* Character) const
{
    if (!Character) return false;
    
    if (UCharAnimInstance* Anim = Cast<UCharAnimInstance>(Character->GetMesh()->GetAnimInstance()))
    {
        // Both conditions must be true:
        // 1. AnimInstance activity is Sitting
        // 2. This chair's CurrentSitter is this character
        return Anim->CharacterActivity == ECharacterActivity::Sitting 
            && CurrentSitter == Character;
    }
    
    return false;
}
```

## Lifecycle

```cpp
// Constructor: Create all components
AChair()
{
    PrimaryActorTick.bCanEverTick = true;
    
    // ChairMesh as root
    ChairMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("ChairMesh"));
    RootComponent = ChairMesh;
    
    // InteractionSphere with overlap
    InteractionSphere = CreateDefaultSubobject<USphereComponent>(TEXT("InteractionSphere"));
    InteractionSphere->SetSphereRadius(100.f);
    InteractionSphere->SetCollisionResponseToChannel(ECC_Pawn, ECR_Overlap);
    InteractionSphere->OnComponentBeginOverlap.AddDynamic(this, &AChair::OnOverlapBegin);
    InteractionSphere->OnComponentEndOverlap.AddDynamic(this, &AChair::OnOverlapEnd);
    
    // WarpTargets for motion warping
    WarpTargetSit = CreateDefaultSubobject<UArrowComponent>(TEXT("WarpTargetSit"));
    WarpTargetStandUp = CreateDefaultSubobject<UArrowComponent>(TEXT("WarpTargetStandUp"));
}

// BeginPlay: Hide widget
void BeginPlay()
{
    InteractionWidget->SetVisibility(false);
}
```

## Quick Start

### 1. Create Chair Blueprint:

```
BP_Chair (Parent: AChair):
├── ChairMesh → Assign static mesh
├── InteractionSphere → Adjust radius if needed
├── WarpTargetSit → Position where character sits
└── WarpTargetStandUp → Position where character stands
```

### 2. Position warp targets:

```
WarpTargetSit:
├── Location: In front of chair seat
└── Rotation: Facing chair (character will rotate to face away)

WarpTargetStandUp:
├── Location: Where character ends up after standing
└── Rotation: Direction character will face
```

### 3. Create chair montages:

```
AM_SitDown:
├── Montage Section: "Default"
├── Motion Warping Notify:
│   └── WarpTargetName: "WarpTargetSit"
└── Add to collection "ChairMontages" as "SitDown"

AM_StandUp:
├── Montage Section: "Default"
├── Motion Warping Notify:
│   └── WarpTargetName: "WarpTargetStandUp"
└── Add to collection "ChairMontages" as "StandUp"
```

### 4. Drop in level:

```
Place BP_Chair in level → Character walks up → Press E → Sits
Press E again → Stands up
```

### 5. AI interaction:

```cpp
// In UCharAIComponent behavior tree
if (UCharInteractionComponent* InteractComp = Character->GetInteractionComponent())
{
    InteractComp->SetOverlappingItem(Chair);
    InteractComp->TryInteract(Character, false);
}
```

## State Machine

```
[Chair Free]
     │
     ▼ Interact (by Character A)
[Chair Occupied by A]
     │
     ├──► Interact (by Character A) → StandUp → [Chair Free]
     │
     └──► Interact (by Character B) → CanInteract = false → No action
```

## Architecture Compliance

✅ **World Reacts:** Chair only responds to interactions. Never initiates.

✅ **IInteractable:** Full interface implementation. Zero coupling to InteractionComponent internals.

✅ **Player/AI Parity:** Same Sit/StandUp logic for both. Widget visibility adapts to AI.

✅ **Anti-Lock-In:** Remove chair → lose chair feature. Character system unaffected.

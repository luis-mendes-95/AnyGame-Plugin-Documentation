# ADoor

> **Module:** AnyGame World | **Header:** `World/Interactables/Door/Door.h` | **Parent:** `AActor`, `IInteractable`

## Overview

`ADoor` is an **interactable door** — it implements `IInteractable` with a 3-state machine (Closed/Open/Locked), direction-aware animations, motion warping, and story trigger integration. Auto-closes when character exits range.

## Design Principles

**World Reacts:** Door is a World Actor. It only reacts to character interactions — never initiates gameplay logic.

**IInteractable Compliant:** Full implementation of the interaction interface. Works seamlessly with `UCharInteractionComponent`.

**Direction-Aware:** Selects animation based on character position (front/back, left/right) for realistic door opening.

**Story Integration:** Can trigger `AStoryTrigger` scenes for locked, unlocked, and trespass scenarios.

**Skeletal Mesh Driven:** Uses `USkeletalMeshComponent` with `UAnimSequence` for smooth door animations.

## Components

| Component | Type | Purpose |
|-----------|------|---------|
| `DoorMesh` | `USkeletalMeshComponent` | Animated door mesh (Root) |
| `InteractionSphere` | `USphereComponent` | Overlap detection (radius 120) |
| `InteractionWidget` | `UWidgetComponent` | Interaction prompt UI |
| `WarpTargetLeft` | `UArrowComponent` | Motion warp target (left side) |
| `WarpTargetRight` | `UArrowComponent` | Motion warp target (right side) |

## Door State Machine

```
     ┌─────────────────────────────────────┐
     │                                     │
     ▼                                     │
[Closed] ◄──── Close() ◄──── [Open] ──────┘
     │                          ▲    (auto-close on exit)
     │ Interact                 │
     │ (if bInteractionAllowed) │
     ▼                          │
  Open() ───────────────────────┘
     
     
[Locked] ───► Interact ───► bHasKey?
                              │
              ┌───────────────┴───────────────┐
              │ YES                           │ NO
              ▼                               ▼
          Unlock()                    LockedScene
              │                       (StoryTrigger)
              ▼
          [Closed]
```

## Properties

### Configuration

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `Animations` | `FDoorAnimations` | — | 4-directional animation set |
| `bHasKey` | `bool` | false | Can unlock locked doors |
| `bInteractionAllowed` | `bool` | true | Trespass check |

### State

| Property | Type | Purpose |
|----------|------|---------|
| `DoorState` | `EDoorState` | Current state (Closed/Open/Locked) |
| `bIsInteracting` | `bool` | Interaction cooldown flag |
| `CurrentInteractor` | `TWeakObjectPtr<ACharacter>` | Character currently interacting |
| `LastOpenAnimation` | `UAnimSequence*` | Cached for reverse close |

### Story Triggers

| Property | Type | Purpose |
|----------|------|---------|
| `TrespassScene` | `AStoryTrigger*` | Plays when `bInteractionAllowed = false` |
| `LockedScene` | `AStoryTrigger*` | Plays when locked and no key |
| `UnlockedScene` | `AStoryTrigger*` | Plays when successfully unlocked |

### Prompts

| Property | Type | Default |
|----------|------|---------|
| `PromptOpen` | `FText` | "Press E to Open" |
| `PromptClose` | `FText` | "Press E to Close" |
| `PromptLocked` | `FText` | "Door is Locked" |

## EDoorState Enum

```cpp
UENUM(BlueprintType)
enum class EDoorState : uint8
{
    Closed,
    Open,
    Locked
};
```

## FDoorAnimations Struct

```cpp
USTRUCT(BlueprintType)
struct FDoorAnimations
{
    UPROPERTY(EditAnywhere)
    UAnimSequence* OpenForwardLeft;
    
    UPROPERTY(EditAnywhere)
    UAnimSequence* OpenForwardRight;
    
    UPROPERTY(EditAnywhere)
    UAnimSequence* OpenBackwardLeft;
    
    UPROPERTY(EditAnywhere)
    UAnimSequence* OpenBackwardRight;
};
```

## IInteractable Implementation

### CanInteract

```cpp
bool CanInteract_Implementation(AActor* InteractingActor) const
{
    // Can interact if not in cooldown
    return !bIsInteracting;
}
```

### Interact

```cpp
bool Interact_Implementation(AActor* InteractingActor)
{
    APlayerCharacter* Character = Cast<APlayerCharacter>(InteractingActor);
    if (!Character || bIsInteracting) return false;
    
    // Range validation
    if (!InteractionSphere->IsOverlappingActor(Character))
        return false;
    
    bIsInteracting = true;
    CurrentInteractor = Character;
    
    // 1. Trespass check
    if (!bInteractionAllowed)
    {
        if (TrespassScene) TrespassScene->StartStorySequence();
        ReleaseInteractionAfterDelay();
        return false;
    }
    
    // 2. Setup motion warp + play kick animation
    SetupWarpTarget(Character);
    AnimComp->PlayMontage(FName("DoorMontages"), FName("KickDownSuccessful"), ...);
    
    // 3. State machine
    switch (DoorState)
    {
        case EDoorState::Closed:
            Open();
            break;
            
        case EDoorState::Open:
            Close();
            break;
            
        case EDoorState::Locked:
            if (bHasKey)
                Unlock();
            else if (LockedScene)
                LockedScene->StartStorySequence();
            break;
    }
    
    ReleaseInteractionAfterDelay();
    return true;
}
```

### GetInteractionPrompt

```cpp
FText GetInteractionPrompt_Implementation() const
{
    switch (DoorState)
    {
        case EDoorState::Open:   return PromptClose;   // "Press E to Close"
        case EDoorState::Locked: return PromptLocked;  // "Door is Locked"
        default:                 return PromptOpen;    // "Press E to Open"
    }
}
```

## Door API

| Function | Purpose |
|----------|---------|
| `Open()` | Open the door (plays direction-aware animation) |
| `Close()` | Close the door (plays cached animation in reverse) |
| `Lock()` | Set state to Locked |
| `Unlock()` | Set state to Closed, trigger UnlockedScene |
| `GetDoorState()` | Get current EDoorState |
| `IsOpen()` | Check if Open |
| `IsLocked()` | Check if Locked |

## Direction-Aware Animation

### GetOpenAnimation

```cpp
UAnimSequence* GetOpenAnimation(ACharacter* Character) const
{
    FVector DoorForward = GetActorForwardVector();
    FVector DoorRight = GetActorRightVector();
    FVector ToCharacter = (Character->GetActorLocation() - GetActorLocation()).GetSafeNormal();
    
    bool bInFront = FVector::DotProduct(DoorForward, ToCharacter) > 0.f;
    bool bRightSide = FVector::DotProduct(DoorRight, ToCharacter) > 0.f;
    
    if (bInFront)
        return bRightSide ? Animations.OpenForwardRight : Animations.OpenForwardLeft;
    else
        return bRightSide ? Animations.OpenBackwardRight : Animations.OpenBackwardLeft;
}
```

### Animation Matrix

| Character Position | Animation |
|--------------------|-----------|
| Front + Right | `OpenForwardRight` |
| Front + Left | `OpenForwardLeft` |
| Back + Right | `OpenBackwardRight` |
| Back + Left | `OpenBackwardLeft` |

### PlayAnimation

```cpp
void PlayAnimation(UAnimSequence* Anim, bool bReverse)
{
    if (!Anim || !DoorMesh) return;
    
    DoorMesh->PlayAnimation(Anim, false);
    
    if (bReverse)
    {
        // Start at end, play backwards
        DoorMesh->SetPosition(Anim->GetPlayLength(), false);
        DoorMesh->SetPlayRate(-1.f);
    }
    else
    {
        DoorMesh->SetPlayRate(1.f);
    }
}
```

## Motion Warping

```cpp
void SetupWarpTarget(APlayerCharacter* ThisCharacter)
{
    FVector DoorRight = GetActorRightVector();
    FVector ToCharacter = (ThisCharacter->GetActorLocation() - GetActorLocation()).GetSafeNormal();
    
    // Select warp target based on character side
    bool bRightSide = FVector::DotProduct(DoorRight, ToCharacter) > 0.f;
    UArrowComponent* SelectedWarp = bRightSide ? WarpTargetRight : WarpTargetLeft;
    
    if (UMotionWarpingComponent* MotionWarpingComp = ThisCharacter->FindComponentByClass<UMotionWarpingComponent>())
    {
        MotionWarpingComp->AddOrUpdateWarpTargetFromComponent(
            FName("Door"),
            SelectedWarp,
            NAME_None,
            true,
            FVector::ZeroVector,
            FRotator::ZeroRotator
        );
    }
}
```

## Auto-Close on Exit

```cpp
void OnInteractionExit(...)
{
    // ... standard unregister from InteractionComponent ...
    
    // Door-specific: auto-close when interactor leaves
    if (OtherActor == CurrentInteractor.Get())
    {
        CurrentInteractor = nullptr;
        
        if (DoorState == EDoorState::Open)
        {
            Close();
        }
    }
}
```

## Interaction Cooldown

Prevents rapid re-interaction during animations:

```cpp
void ReleaseInteractionAfterDelay()
{
    GetWorld()->GetTimerManager().SetTimer(
        InteractionCooldownHandle,
        this,
        &ADoor::FinishInteraction,
        4.0f,  // 4 second cooldown
        false
    );
}

void FinishInteraction()
{
    bIsInteracting = false;
}
```

## Overlap Handlers

### OnInteractionEnter

```cpp
void OnInteractionEnter(...)
{
    APlayerCharacter* OverlapChar = Cast<APlayerCharacter>(OtherActor);
    if (!OverlapChar) return;
    
    CurrentInteractor = OverlapChar;
    
    // Register with InteractionComponent
    if (UCharInteractionComponent* InteractComp = OverlapChar->GetInteractionComponent())
    {
        InteractComp->SetOverlappingItem(this);
    }
}
```

### OnInteractionExit

```cpp
void OnInteractionExit(...)
{
    APlayerCharacter* OverlapChar = Cast<APlayerCharacter>(OtherActor);
    if (!OverlapChar) return;
    
    // Unregister from InteractionComponent
    if (UCharInteractionComponent* InteractComp = OverlapChar->GetInteractionComponent())
    {
        if (InteractComp->GetOverlappingItem() == this)
            InteractComp->SetOverlappingItem(nullptr);
    }
    
    // Auto-close if interactor left
    if (OtherActor == CurrentInteractor.Get())
    {
        CurrentInteractor = nullptr;
        if (DoorState == EDoorState::Open)
            Close();
    }
}
```

## Quick Start

### 1. Create Door Blueprint:

```
BP_Door (Parent: ADoor):
├── DoorMesh → Assign skeletal mesh
├── InteractionSphere → Adjust radius if needed
├── WarpTargetLeft → Position for left-side interaction
└── WarpTargetRight → Position for right-side interaction
```

### 2. Assign animations:

```
BP_Door → Details → Animations:
├── OpenForwardLeft → AS_Door_OpenFL
├── OpenForwardRight → AS_Door_OpenFR
├── OpenBackwardLeft → AS_Door_OpenBL
└── OpenBackwardRight → AS_Door_OpenBR
```

### 3. Create door kick montage:

```
AM_KickDownSuccessful:
├── Motion Warping Notify:
│   └── WarpTargetName: "Door"
└── Add to collection "DoorMontages" as "KickDownSuccessful"
```

### 4. Setup story triggers (optional):

```
Level:
├── BP_Door
│   ├── TrespassScene → StoryTrigger_Trespass
│   ├── LockedScene → StoryTrigger_Locked
│   └── UnlockedScene → StoryTrigger_Unlocked
```

### 5. Lock a door:

```cpp
// In Blueprint or C++
Door->Lock();
Door->bHasKey = false;  // Requires key to unlock

// When player picks up key
Door->bHasKey = true;
// Next interaction will unlock
```

### 6. Restrict access (trespass):

```cpp
// Prevent interaction
Door->bInteractionAllowed = false;

// Assign trespass scene
Door->TrespassScene = MyTrespassStoryTrigger;
// Will play scene instead of opening
```

## Lifecycle

```cpp
// Constructor: Create all components
ADoor()
{
    PrimaryActorTick.bCanEverTick = true;
    
    // SkeletalMesh for animated door
    DoorMesh = CreateDefaultSubobject<USkeletalMeshComponent>(TEXT("DoorMesh"));
    RootComponent = DoorMesh;
    
    // InteractionSphere with overlap
    InteractionSphere = CreateDefaultSubobject<USphereComponent>(TEXT("InteractionSphere"));
    InteractionSphere->SetSphereRadius(120.f);
    InteractionSphere->OnComponentBeginOverlap.AddDynamic(this, &ADoor::OnInteractionEnter);
    InteractionSphere->OnComponentEndOverlap.AddDynamic(this, &ADoor::OnInteractionExit);
    
    // Warp targets for left/right approach
    WarpTargetLeft = CreateDefaultSubobject<UArrowComponent>(TEXT("WarpTargetLeft"));
    WarpTargetRight = CreateDefaultSubobject<UArrowComponent>(TEXT("WarpTargetRight"));
}

// BeginPlay: Hide widget
void BeginPlay()
{
    HideUI();
}
```

## Architecture Compliance

✅ **World Reacts:** Door only responds to interactions. Never initiates.

✅ **IInteractable:** Full interface implementation. Zero coupling to InteractionComponent internals.

✅ **Story Integration:** Optional `AStoryTrigger` references for narrative moments.

✅ **Anti-Lock-In:** Remove door → lose door feature. Character system unaffected.

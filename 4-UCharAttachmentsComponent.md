# UCharAttachmentsComponent

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Attachments/CharAttachmentsComponent.h` | **Parent:** `UActorComponent`

## Overview

`UCharAttachmentsComponent` is a **slot-based attachment manager** — it attaches/detaches actors to character sockets, tracks occupied hands, and broadcasts state changes for animation and gameplay systems.

## Design Principles

**Slot System:** Attachments are keyed by `ECharAttachmentSlot`, not by actor reference. One slot = one actor. Attaching to an occupied slot auto-detaches the previous actor.

**Name-Based Classification:** Attachment state is inferred from actor names (`Phone`, `Flashlight`, `Melee`, `Range`). No interfaces or tags required — keeps World Actors simple.

**Animation Bridge:** `OnAttachmentStateChanged` broadcasts to `UCharAnimInstance`, which updates hand IK and pose selection without polling.

**No Tick:** State is recalculated only on attach/detach. Zero per-frame cost.

## Slots

```cpp
enum class ECharAttachmentSlot : uint8
{
    RightHand,
    LeftHand,
    Back,
    Hip,
    // ... extensible
};
```

| Slot | Typical Use |
|------|-------------|
| `RightHand` | Weapons, tools, phone |
| `LeftHand` | Flashlight, shield, two-hand support |
| `Back` | Holstered rifle, backpack |
| `Hip` | Holstered pistol, melee |

## States

```cpp
enum class ECharAttachmentState : uint8
{
    None,                 // Hands free
    HoldingPhone,         // Actor name contains "Phone"
    HoldingFlashlight,    // Actor name contains "Flashlight"
    HoldingMeleeWeapon,   // Actor name contains "Melee"
    HoldingRangeWeapon,   // Actor name contains "Range"
    TwoHandsOccupied      // Both hands have actors
};
```

**Resolution Priority:**
1. Both hands occupied → `TwoHandsOccupied`
2. Single hand → classify by actor name
3. No hands → `None`

## Properties

| Property | Type | Purpose |
|----------|------|---------|
| `Character` | `APlayerCharacter*` | Cached owner reference |
| `CurrentAttachmentState` | `ECharAttachmentState` | Current classified state |
| `AttachedActors` | `TMap<ECharAttachmentSlot, AActor*>` | Slot → Actor mapping |

## Key Functions

| Function | Returns | Purpose |
|----------|---------|---------|
| `AttachActor(Actor, Slot, Socket)` | `bool` | Attach actor to socket on BodyMesh |
| `DetachSlot(Slot)` | `bool` | Detach and optionally destroy actor |
| `GetActorInSlot(Slot)` | `AActor*` | Query what's in a slot |
| `GetCurrentAttachmentState()` | `ECharAttachmentState` | Current state |
| `AreHandsOccupied()` | `bool` | Either hand has an actor |
| `GetAttachedActors()` | `const TMap&` | Read-only access to all slots |

## Delegates

| Delegate | Signature | Fires When |
|----------|-----------|------------|
| `OnAttachmentStateChanged` | `FOnAttachmentStateChanged(ECharAttachmentState)` | State changes after attach/detach |

## Attach Flow

```cpp
bool AttachActor(AActor* Actor, ECharAttachmentSlot Slot, FName SocketName)
{
    // 1. Find "BodyMesh" skeletal mesh on owner
    USkeletalMeshComponent* BodyMesh = FindBodyMesh();
    
    // 2. Validate socket exists
    if (!BodyMesh->DoesSocketExist(SocketName)) return false;
    
    // 3. Auto-detach if slot occupied
    if (AttachedActors.Contains(Slot))
        DetachSlot(Slot);
    
    // 4. Attach with snap rules
    Actor->AttachToComponent(BodyMesh, 
        FAttachmentTransformRules(EAttachmentRule::SnapToTarget, true),
        SocketName);
    
    // 5. Register and recalculate
    AttachedActors.Add(Slot, Actor);
    RecalculateAttachmentState();
    
    return true;
}
```

## Detach Flow

```cpp
bool DetachSlot(ECharAttachmentSlot Slot)
{
    AActor* Actor = AttachedActors[Slot];
    
    // 1. Detach from hierarchy
    Actor->DetachFromActor(FDetachmentTransformRules::KeepWorldTransform);
    
    // 2. Special handling (e.g., Phone hides HUD widget)
    if (Actor->GetName().Contains(TEXT("Phone")))
    {
        PlayerHUD->HideMobilePhone();
        Actor->Destroy();
    }
    
    // 3. Remove from map and recalculate
    AttachedActors.Remove(Slot);
    RecalculateAttachmentState();
    
    return true;
}
```

## State Classification

```cpp
void RecalculateAttachmentState()
{
    const AActor* RightHand = GetActorInSlot(ECharAttachmentSlot::RightHand);
    const AActor* LeftHand = GetActorInSlot(ECharAttachmentSlot::LeftHand);
    
    ECharAttachmentState NewState = ECharAttachmentState::None;
    
    if (RightHand && LeftHand)
    {
        NewState = ECharAttachmentState::TwoHandsOccupied;
    }
    else if (RightHand || LeftHand)
    {
        const FString Name = HeldActor->GetName();
        
        if (Name.Contains(TEXT("Phone")))          NewState = HoldingPhone;
        else if (Name.Contains(TEXT("Flashlight"))) NewState = HoldingFlashlight;
        else if (Name.Contains(TEXT("Melee")))      NewState = HoldingMeleeWeapon;
        else if (Name.Contains(TEXT("Range")))      NewState = HoldingRangeWeapon;
    }
    
    if (NewState != CurrentAttachmentState)
    {
        CurrentAttachmentState = NewState;
        OnAttachmentStateChanged.Broadcast(NewState);  // → AnimInstance listens
    }
}
```

## Integration with AnimInstance

```cpp
// In UCharAnimInstance::BindAttachments()
AttachmentsComponent->OnAttachmentStateChanged.AddDynamic(
    this, &UCharAnimInstance::OnAttachmentStateChanged
);

void UCharAnimInstance::OnAttachmentStateChanged(ECharAttachmentState NewState)
{
    AttachmentState = NewState;
    bHandsOccupied = (NewState != ECharAttachmentState::None);
    // AnimBP can now select correct hand pose / disable IK
}
```

## Lifecycle

```cpp
// Constructor: No tick needed
UCharAttachmentsComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

// BeginPlay: Cache owner
void BeginPlay()
{
    Character = Cast<APlayerCharacter>(GetOwner());
}
```

## Quick Start

1. **Attach a weapon:**
```cpp
AActor* Sword = GetWorld()->SpawnActor<AActor>(SwordClass);
AttachmentsComp->AttachActor(Sword, ECharAttachmentSlot::RightHand, TEXT("hand_r_socket"));
// State → HoldingMeleeWeapon (if actor name contains "Melee")
```

2. **Attach phone (with HUD integration):**
```cpp
AActor* Phone = SpawnPhone();
AttachmentsComp->AttachActor(Phone, ECharAttachmentSlot::RightHand, TEXT("hand_r_socket"));
// State → HoldingPhone
// On detach → HUD->HideMobilePhone() + Destroy()
```

3. **Check if can interact:**
```cpp
if (!AttachmentsComp->AreHandsOccupied())
{
    // Can pick up items
}
```

4. **Listen to state changes (Blueprint):**
```
Event: OnAttachmentStateChanged
├── Switch on NewState
│   ├── HoldingPhone → Enable phone UI
│   ├── HoldingMeleeWeapon → Enable combat stance
│   └── None → Reset to idle
```

5. **Query current state:**
```cpp
ECharAttachmentState State = AttachmentsComp->GetCurrentAttachmentState();
if (State == ECharAttachmentState::HoldingRangeWeapon)
{
    // Enable aiming
}
```

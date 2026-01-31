# Animation System

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Animation/` | **Parent:** Multiple

## Overview

The Animation System is a **three-layer architecture**: `AnimFlow` orchestrates what plays, `AnimInstance` provides runtime state for AnimBPs, and `Notifies` trigger gameplay events from montages.

## Design Principles

**Separation of Concerns:** AnimFlow doesn't know how animations blend. AnimInstance doesn't know about story sequences. Notifies don't know about montage selection.

**Dual Mesh Support:** Characters have Body + Face skeletal meshes. Each has its own AnimInstance. AnimFlow coordinates both.

**Data-Driven Notifies:** Combat notifies use enums + structs instead of subclasses. One notify class handles all combat events via `EAnimNotifyCombatType`.

## Folder Structure

```
Animation/
├── AnimFlow/
│   ├── DataAssets/
│   │   ├── MontageCollections/
│   │   │   └── UMontageCollectionsDataAsset
│   │   └── StorySequences/
│   │       └── UStorySequencesDataAsset
│   └── UCharAnimComponent
├── AnimInstance/
│   ├── Body/
│   │   └── UCharAnimInstance
│   └── Face/
│       └── UFaceAnimInstance
└── Notifies/
    ├── Combat/
    │   └── UAnimNotifyCombatBase
    └── UAnimNotifyBase
```

---

# UCharAnimComponent

> **Header:** `Animation/AnimFlow/CharAnimComponent.h` | **Parent:** `UActorComponent`

## Overview

`UCharAnimComponent` is the **animation orchestration layer** — it coordinates montages, story sequences, facial animations, dialogue audio, and motion warping across multiple skeletal meshes.

## Design Principles

**Orchestrator, Not Animator:** Doesn't own AnimInstances. Finds them, triggers montages, manages sync.

**Visual Actor Pattern:** Face montages play on a `ChildActorComponent` named "Visual" attached to the mesh.

**Priority System:** When Body, Face, and Audio play together, determines which one dictates sequence end.

## Data Assets

### UMontageCollectionsDataAsset

```cpp
UCLASS(BlueprintType)
class UMontageCollectionsDataAsset : public UPrimaryDataAsset
{
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TArray<FCustomAnimMontageCollection> MontageCollections;
};
```

| Struct | Purpose |
|--------|---------|
| `FCustomAnimMontageCollection` | Named group: `CollectionName` + `TArray<FCustomAnimMontage>` |
| `FCustomAnimMontage` | Entry: `MontageName` + `UAnimMontage*` |

**Example:**
```
MontageCollections:
├── "Combat" → [LightAttack_01, LightAttack_02, HeavyAttack]
├── "Traversal" → [Vault, Climb]
└── "Emotes" → [Wave, Dance]
```

### UStorySequencesDataAsset

```cpp
UCLASS(BlueprintType)
class UStorySequencesDataAsset : public UPrimaryDataAsset
{
    UPROPERTY(EditAnywhere, BlueprintReadOnly)
    TArray<FStorySequence> StorySequences;
};
```

| Struct | Fields |
|--------|--------|
| `FStorySequence` | `SequenceID`, `LevelSequence`, `WorldManager`, `StartDelay` |
| `FStoryStepParticipant` | `Acting` (Body/Face montages, Audio, Subtitles) |

## End Priority System

```cpp
enum class EStorySequenceEndPriority : uint8
{
    None,   // Only subtitles → timer-based
    Body,   // Body montage ends last
    Face,   // Face montage ends last  
    Audio   // Dialogue audio ends last
};
```

**Resolution:** Face > Audio > Body > None

## Key Functions

| Function | Purpose |
|----------|---------|
| `ExecuteStorySequence()` | Plays full cinematic (Body + Face + Audio + Subtitles) |
| `PlayMontage(Collection, Name, Section, bIsBody)` | Plays montage by Data Asset lookup |
| `SetWarpTarget(Transform)` | Sets motion warp target ("AdjustPosition") |
| `GetVisualActor()` | Finds Face mesh ChildActor |

## Delegates

| Delegate | Fires When |
|----------|------------|
| `OnStorySequenceFinished` | Sequence fully complete |

---

# UCharAnimInstance

> **Header:** `Animation/AnimInstance/Body/CharAnimInstance.h` | **Parent:** `UAnimInstance`

## Overview

`UCharAnimInstance` is the **Body AnimBP runtime** — exposes character state to Animation Blueprints without gameplay logic.

## Design Principles

**State Mirror:** Reflects `CharMovementComponent` and `CharAttachmentsComponent` state. Doesn't decide, just reports.

**Thread-Safe:** Uses `NativeThreadSafeUpdateAnimation` for parallel evaluation. No game thread dependencies.

**Attachment Aware:** Listens to `OnAttachmentStateChanged` to update hand IK and pose selection.

## Properties

| Property | Type | Source | Purpose |
|----------|------|--------|---------|
| `Character` | `APlayerCharacter*` | `TryGetPawnOwner()` | Owner reference |
| `CharacterMovement` | `UCharMovementComponent*` | Character | Movement state access |
| `AnimationType` | `FName` | PlayerAsset | Locomotion set selector |
| `AttachmentState` | `ECharAttachmentState` | AttachmentsComponent | What's equipped |
| `CharacterActivity` | `ECharacterActivity` | External | Sitting, Swimming, etc. |
| `bHandsOccupied` | `bool` | AttachmentsComponent | Blocks hand IK |

## Key Functions

| Function | Purpose |
|----------|---------|
| `ShouldBlendRoot()` | Returns false when Sitting (disables root motion blend) |
| `AreHandsOccupied()` | Check if hands are busy (carrying, weapon) |
| `GetAttachmentStateString()` | Debug: enum to string |

## Lifecycle

```cpp
void NativeInitializeAnimation()
{
    Character = Cast<APlayerCharacter>(TryGetPawnOwner());
    CharacterMovement = Character->GetCharacterMovement();
    BindAttachments();  // Subscribe to OnAttachmentStateChanged
}

void NativeThreadSafeUpdateAnimation(float DeltaSeconds)
{
    // Thread-safe state updates for AnimBP
}
```

## Attachment Binding

```cpp
void BindAttachments()
{
    // Initial state
    AttachmentState = AttachmentsComponent->GetCurrentAttachmentState();
    bHandsOccupied = AttachmentsComponent->AreHandsOccupied();
    
    // Subscribe to changes
    AttachmentsComponent->OnAttachmentStateChanged.AddDynamic(
        this, &UCharAnimInstance::OnAttachmentStateChanged
    );
}

void OnAttachmentStateChanged(ECharAttachmentState NewState)
{
    AttachmentState = NewState;
    bHandsOccupied = (NewState != ECharAttachmentState::None);
}
```

---

# UFaceAnimInstance

> **Header:** `Animation/AnimInstance/Face/FaceAnimInstance.h` | **Parent:** `UAnimInstance`

## Overview

`UFaceAnimInstance` is the **Face AnimBP runtime** — minimal state for facial animations on the Visual Actor's "Face" mesh.

## Design Principles

**Lightweight:** Only caches Character reference. Face animations are montage-driven, not state-driven.

**Visual Actor Resident:** Lives on the "Face" `USkeletalMeshComponent` inside the Visual ChildActor.

## Properties

| Property | Type | Purpose |
|----------|------|---------|
| `Character` | `APlayerCharacter*` | Owner reference (for potential lipsync/emotion state) |

## Lifecycle

```cpp
void NativeInitializeAnimation()
{
    Character = Cast<APlayerCharacter>(TryGetPawnOwner());
}

void NativeThreadSafeUpdateAnimation(float DeltaSeconds)
{
    // Reserved for future facial state (blink, lipsync)
}
```

---

# UAnimNotifyBase

> **Header:** `Animation/Notifies/AnimNotifyBase.h` | **Parent:** `UAnimNotify`

## Overview

`UAnimNotifyBase` is the **generic notify dispatcher** — handles common gameplay events from montages via `EAnimNotifyType` enum.

## Design Principles

**Enum-Driven:** One class, multiple behaviors. No subclass explosion.

**CombatComponent Gateway:** All combat-related events route through `UCharCombatComponent`.

## Notify Types

```cpp
enum class EAnimNotifyType : uint8
{
    EndAttack,           // Attack sequence complete
    ResetAttack,         // Reset attack state + rotation lock
    AddMotionWarpingOffset,  // Reserved
    PlayGetHitMontage,   // Trigger hit reaction on target
    ResetGetHit,         // Clear hit reaction state
    ApplyHitEffect,      // Spawn VFX/SFX, apply damage
    CheckDie,            // Evaluate death condition
    SlowMotionStart,     // Reserved
    SlowMotionEnd        // Reserved
};
```

## Properties

| Property | Type | Purpose |
|----------|------|---------|
| `NotifyType` | `EAnimNotifyType` | Which event to fire |
| `HitEffect` | `FHitEffectInfo` | VFX/SFX params for `ApplyHitEffect` |
| `ImpulseType` | `EDeathImpulseType` | Ragdoll impulse direction |
| `ImpulseStrength` | `float` | Ragdoll impulse force |

## Execution Flow

```cpp
void Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    APlayerCharacter* Character = Cast<APlayerCharacter>(MeshComp->GetOwner());
    UCharCombatComponent* CombatComp = Character->GetCombatComponent();
    
    switch (NotifyType)
    {
        case EndAttack:      CombatComp->EndAttack(); break;
        case ResetAttack:    CombatComp->ResetAttack(); break;
        case PlayGetHitMontage: CombatComp->HandleHitTarget(); break;
        case ApplyHitEffect: CombatComp->ApplyHit(HitInfo); break;
        case CheckDie:       CombatComp->CheckDie(); break;
        // ...
    }
}
```

---

# UAnimNotifyCombatBase

> **Header:** `Animation/Notifies/Combat/AnimNotifyCombatBase.h` | **Parent:** `UAnimNotify`

## Overview

`UAnimNotifyCombatBase` is the **combat-specific notify dispatcher** — extends base notifies with counter windows, motion params, and UI feedback.

## Design Principles

**Counter System Integration:** Opens/closes defend windows with visual feedback (icons above head).

**Motion Params:** Single notify can configure capsule, movement mode, root motion, and warp in one shot.

**NPC Aware:** Counter icons only show on non-player characters.

## Notify Types

```cpp
enum class EAnimNotifyCombatType : uint8
{
    OpenCounterWindow,    // Enable defend + show icon
    CloseCounterWindow,   // Disable defend + hide icon
    PlayGetHitMontage,    // Hit reaction (counter-aware)
    EndAttack,            // Attack complete
    ResetAttack,          // Reset state
    ApplyAnimMotionParams,// Bulk motion configuration
    ResetGetHit,          // Clear hit state
    ApplyHitEffect,       // VFX/SFX/damage
    CheckDie              // Death check
};
```

## Properties

| Property | Type | Purpose |
|----------|------|---------|
| `CombatNotifyType` | `EAnimNotifyCombatType` | Which event to fire |
| `CombatIcon` | `UTexture2D*` | Icon for counter window |
| `AnimMotionParams` | `FAnimMotionParams` | Bulk motion config |
| `HitEffect` | `FHitEffectInfo` | VFX/SFX params |
| `ImpulseType` / `ImpulseStrength` | | Ragdoll params |

## FAnimMotionParams

Single struct to configure multiple systems at once:

| Field | Type | Purpose |
|-------|------|---------|
| `bApplyMotionWarp` | `bool` | Enable warp to target |
| `WarpTargetName` | `FName` | Warp target identifier |
| `bRotateTowardsTarget` | `bool` | Enable facing lock |
| `FacingRotationSpeed` | `float` | Rotation interp speed |
| `bSetCapsuleHalfHeight` | `bool` | Resize capsule |
| `CapsuleHalfHeight` | `float` | New half height |
| `bDisableCapsuleCollision` | `bool` | Disable collision |
| `bSetMovementMode` | `bool` | Change movement mode |
| `MovementMode` | `EMovementMode` | Target mode |
| `bSetRootMotion` | `bool` | Toggle root motion |
| `bUseRootMotion` | `bool` | Enable/disable |
| `RootMotionMultiplier` | `float` | Playrate modifier |

## Counter Window Flow

```cpp
void SetCounterWindow(UCharCombatComponent* CombatComp, bool bEnable, UTexture2D* Icon)
{
    CombatComp->SetCanBeDeffended(bEnable);
    
    // Show icon only on NPCs
    if (!Character->IsPlayerControlled())
    {
        WidgetComp->SetCombatIcon(bEnable ? Icon : nullptr);
    }
}
```

## Counter-Aware Hit

```cpp
void HandleHitTargetNotify(UCharCombatComponent* CombatComp)
{
    if (CombatComp->GetIsCounterAttack())
    {
        CombatComp->HandleCounterGetHit();  // Special counter reaction
        CombatComp->SetIsCounterAttack(false);
        return;
    }
    
    CombatComp->HandleHitTarget();  // Normal hit
}
```

---

## Quick Start

### 1. Play a montage by name:
```cpp
AnimComp->PlayMontage(TEXT("Combat"), TEXT("LightAttack"), NAME_None, true);
```

### 2. Setup combat notify in montage:
```
Add UAnimNotifyCombatBase
├── CombatNotifyType = ApplyAnimMotionParams
└── AnimMotionParams:
    ├── bApplyMotionWarp = true
    ├── WarpTargetName = "AdjustPosition"
    └── bRotateTowardsTarget = true
```

### 3. Counter window sequence:
```
Montage Timeline:
├── [0.0s] OpenCounterWindow (Icon = DefendIcon)
├── [0.3s] ApplyHitEffect
└── [0.5s] CloseCounterWindow
```

### 4. Access AnimInstance state in AnimBP:
```
AnimGraph:
├── Get AttachmentState → Select pose
├── Get CharacterActivity → Branch (Sitting disables locomotion)
└── ShouldBlendRoot() → Enable/disable root blend
```

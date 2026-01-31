# UCharInputComponent

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Input/CharInputComponent.h` | **Parent:** `UActorComponent`

## Overview

`UCharInputComponent` is a **thin input routing layer** — it binds Enhanced Input actions and forwards them to gameplay components. No input logic lives here, only delegation.

## Design Principles

**Enhanced Input Only:** Uses UE5's Enhanced Input System. No legacy `BindAxis`/`BindAction`.

**Router, Not Handler:** Input functions are one-liners that call the appropriate component. Combat logic stays in `CombatComponent`, interaction logic stays in `InteractionComponent`.

**Data Asset Driven:** `UInputMappingContext` and `UInputAction` are assigned in Blueprint/Data Asset, not hardcoded.

**No Tick:** `PrimaryComponentTick.bCanEverTick = false`. Input is event-driven.

## Input Actions

| Action | Trigger | Delegates To |
|--------|---------|--------------|
| `AttackAction` | Started | `CombatComponent->HandleAttack()` |
| `AttackAction` | Completed | `StopAttack()` (reserved) |
| `AimAction` | Started | `CombatComponent->SetAiming(true)` |
| `AimAction` | Completed | `CombatComponent->SetAiming(false)` |
| `InteractAction` | Started | `InteractionComponent->TryInteract()` + Combat Defend |

## Properties

| Property | Type | Purpose |
|----------|------|---------|
| `PlayerCharacter` | `APlayerCharacter*` | Cached owner reference |
| `DefaultMappingContext` | `UInputMappingContext*` | Input context to add on setup |
| `AttackAction` | `UInputAction*` | Attack input action |
| `AimAction` | `UInputAction*` | Aim/ranged mode input action |
| `InteractAction` | `UInputAction*` | Interact/defend input action |
| `MoveValue` | `FVector2D` | Current movement input (reserved) |

## Key Functions

| Function | Purpose |
|----------|---------|
| `SetupInputBindings()` | Adds mapping context, binds actions. Called by `APlayerCharacter::PossessedBy()` |
| `Attack()` | Routes to `CombatComponent->HandleAttack()` |
| `StopAttack()` | Reserved for hold-attack patterns |
| `Aim()` | Routes to `CombatComponent->SetAiming(true)` |
| `StopAim()` | Routes to `CombatComponent->SetAiming(false)` |
| `Interact()` | Routes to `InteractionComponent->TryInteract()` + sets `bCanDeffend` |

## Setup Flow

```cpp
// Called automatically when PlayerController possesses the character
// See APlayerCharacter::PossessedBy()

void SetupInputBindings()
{
    APlayerController* PC = Cast<APlayerController>(PlayerCharacter->GetController());
    
    // 1. Add mapping context
    UEnhancedInputLocalPlayerSubsystem* Subsys = 
        ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PC->GetLocalPlayer());
    Subsys->AddMappingContext(DefaultMappingContext, 0);
    
    // 2. Bind actions
    UEnhancedInputComponent* EIC = Cast<UEnhancedInputComponent>(PC->InputComponent);
    
    EIC->BindAction(AttackAction, ETriggerEvent::Started, this, &Attack);
    EIC->BindAction(AttackAction, ETriggerEvent::Completed, this, &StopAttack);
    
    EIC->BindAction(InteractAction, ETriggerEvent::Started, this, &Interact);
}
```

## Action Implementations

### Attack

```cpp
void Attack()
{
    if (PlayerCharacter->GetCombatComponent())
    {
        PlayerCharacter->GetCombatComponent()->HandleAttack();
    }
}
```

Simple delegation. All gate logic (`bCanAttack`, target validation, etc.) lives in `CombatComponent`.

### Interact (Dual Purpose)

```cpp
void Interact()
{
    // 1. World interaction (doors, chairs, items)
    if (UCharInteractionComponent* IntComp = PlayerCharacter->GetInteractionComponent())
    {
        IntComp->TryInteract(PlayerCharacter);
    }
    
    // 2. Combat defend (if target has open counter window)
    if (UCharCombatComponent* CombatComp = PlayerCharacter->GetCombatComponent())
    {
        if (CombatComp->CurrentAttackTarget != nullptr &&
            CombatComp->CurrentAttackTarget->GetCombatComponent()->GetCanBeDeffended())
        {
            CombatComp->SetCanDeffend(true);
        }
    }
}
```

**Dual behavior:** Same button handles world interaction AND combat defense. If an enemy is attacking with `bCanBeDeffended = true`, pressing Interact enables counter.

### Aim

```cpp
void Aim()
{
    PlayerCharacter->GetCombatComponent()->SetAiming(true);
}

void StopAim()
{
    PlayerCharacter->GetCombatComponent()->SetAiming(false);
}
```

Toggles ranged attack mode. When aiming, `CombatComponent` uses `DistantAttackThreshold` instead of `NearAttackThreshold`.

## Lifecycle

```cpp
// Constructor: No tick needed
UCharInputComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

// BeginPlay: Cache owner (bindings happen later in PossessedBy)
void BeginPlay()
{
    PlayerCharacter = Cast<APlayerCharacter>(GetOwner());
}
```

## Integration with APlayerCharacter

```cpp
// In APlayerCharacter::PossessedBy()
void APlayerCharacter::PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    
    if (APlayerController* PC = Cast<APlayerController>(NewController))
    {
        if (CharInputComp)
        {
            CharInputComp->SetupInputBindings();
        }
    }
}
```

Bindings are set up only when possessed by a `PlayerController`, not for AI-controlled characters.

## Quick Start

### 1. Create Input Actions (in Editor):

```
Content/Input/
├── IA_Attack.uasset      (UInputAction)
├── IA_Aim.uasset         (UInputAction)
├── IA_Interact.uasset    (UInputAction)
└── IMC_Default.uasset    (UInputMappingContext)
```

### 2. Configure Mapping Context:

```
IMC_Default:
├── IA_Attack → Left Mouse Button
├── IA_Aim → Right Mouse Button (Hold)
└── IA_Interact → E Key
```

### 3. Assign to Component (Blueprint):

```
BP_PlayerCharacter → CharInputComponent:
├── DefaultMappingContext → IMC_Default
├── AttackAction → IA_Attack
├── AimAction → IA_Aim
└── InteractAction → IA_Interact
```

### 4. Call Interact from AI (for traversal):

```cpp
// AI can trigger interaction programmatically
if (UCharInputComponent* InputComp = Character->GetCharInputComponent())
{
    InputComp->Interact();
}
```

This is how `UCharAIComponent` handles auto-interact during follow.

### 5. Add new action:

```cpp
// Header
UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Input")
UInputAction* DodgeAction;

void Dodge();

// Cpp - SetupInputBindings
if (DodgeAction)
{
    EIC->BindAction(DodgeAction, ETriggerEvent::Started, this, &UCharInputComponent::Dodge);
}

// Cpp - Implementation
void UCharInputComponent::Dodge()
{
    if (UCharMovementComponent* MoveComp = PlayerCharacter->GetMovementComponent())
    {
        MoveComp->TryDodge();
    }
}
```

# APlayerCharacter

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/PlayerCharacter.h` | **Parent:** `ACharacter`

## Overview

`APlayerCharacter` is a **thin orchestration layer** — it holds no gameplay logic, only hosts and exposes Actor Components. All gameplay lives inside components, never in the Character itself.

## Design Principles

**Why `final`?** The class enforces **composition over inheritance**. Need custom behavior? Create an ActorComponent or use Blueprints — don't subclass.

**Player/AI Parity:** Despite the name, this class serves both players and NPCs. AI uses `UCharAIComponent` with the same component structure.

**No Tick:** `PrimaryActorTick.bCanEverTick = false`. All tick logic belongs in individual components.

## Components

Created automatically in constructor:

| Component | Category | Purpose |
|-----------|----------|---------|
| `UCharInputComponent` | Core | Player input via Enhanced Input |
| `UCharAnimComponent` | Core | Animation Blueprint & states |
| `UCharMovementComponent` | Core | Replaces default CharacterMovementComponent |
| `UCharCombatComponent` | Gameplay | Attacks, defense, targeting |
| `UCharInteractionComponent` | Gameplay | World interaction detection & execution |
| `UCharAIComponent` | Gameplay | NPC behavior trees |
| `UCharCareerComponent` | Gameplay | Progression, stats, inventory |
| `UCharAttachmentsComponent` | Gameplay | Weapons, visual attachments |
| `UCharSoundComponent` | Feedback | Contextual audio |
| `UCharWidgetComponent` | Feedback | Healthbar, nametag UI |
| `UCharDebugComponent` | Debug | Runtime debugging tools |

All components have `BlueprintPure` getters following the pattern `Get[Name]Component()` with categories `Player|Components|[Category]`.

## Lifecycle

```cpp
// Constructor: Replaces default movement, disables tick, creates all subobjects
APlayerCharacter(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer.SetDefaultSubobjectClass<UCharMovementComponent>(
        ACharacter::CharacterMovementComponentName))

// PossessedBy: Auto-configures input when possessed by PlayerController
void PossessedBy(AController* NewController)
{
    Super::PossessedBy(NewController);
    if (APlayerController* PC = Cast<APlayerController>(NewController))
    {
        if (CharInputComp) { CharInputComp->SetupInputBindings(); }
    }
}
```

**Reserved hooks** (TODO): `BeginPlay()`, `Landed()`, `UnPossessed()`

## Quick Start

1. Set your Blueprint Character to inherit from `APlayerCharacter`
2. Drop one in the world → possess as Player 0 → it's a player
3. Drop another → don't possess → it's an NPC

That's it. Components handle the rest.
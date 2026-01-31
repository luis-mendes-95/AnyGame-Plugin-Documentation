# UCharInteractionComponent

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Interaction/CharInteractionComponent.h` | **Parent:** `UActorComponent`

## Overview

`UCharInteractionComponent` is the **interaction router** — it detects interactables via overlap, calls `IInteractable` on them, and broadcasts events for Blueprint/Traversal hooks. Zero coupling to specific interactable types.

## Design Principles

**Interface-Based:** All interactables implement `IInteractable`. Component never casts to `ADoor`, `AChair`, etc. SOLID compliance.

**Player/AI Parity:** Same `TryInteract()` is called by `UCharInputComponent` (player) and `UCharAIComponent` (NPC). Identical behavior.

**Event Chain:** Interaction flows through Core → Delegate → Hook Interface → BP Event. Multiple extension points.

**No Tick:** `PrimaryComponentTick.bCanEverTick = false`. Interaction is event-driven (overlap + input).

## Interaction Flow

```
[Input/AI]
     │
     ▼
TryInteract(Instigator, bJump)
     │
     ├──► IInteractable::Execute_CanInteract()     ← Query
     │         │
     │         ▼ (if true)
     ├──► IInteractable::Execute_Interact()        ← World Actor executes
     │
     ├──► OnInteractAfterCore.Broadcast()          ← Delegate (C++ listeners)
     │
     ├──► ICharInteractHookInterface::Execute_OnAnyGameTryInteractHook()
     │                                             ← Hook Interface (BP Traversal)
     │
     └──► OnTryInteractBP()                        ← BP Implementable Event
```

## IInteractable Interface

World Actors implement this to be interactable:

```cpp
UINTERFACE(BlueprintType)
class UInteractable : public UInterface { ... };

class IInteractable
{
    // Can this actor be interacted with right now?
    UFUNCTION(BlueprintNativeEvent)
    bool CanInteract(APlayerCharacter* Instigator);
    
    // Execute the interaction. Return true if successful.
    UFUNCTION(BlueprintNativeEvent)
    bool Interact(APlayerCharacter* Instigator);
    
    // Called when character enters interaction range
    UFUNCTION(BlueprintNativeEvent)
    void OnEnterInteractionRange(AActor* Character);
    
    // Called when character exits interaction range
    UFUNCTION(BlueprintNativeEvent)
    void OnExitInteractionRange(AActor* Character);
    
    // UI prompt text (e.g., "Press E to Open")
    UFUNCTION(BlueprintNativeEvent)
    FText GetInteractionPrompt();
};
```

## ICharInteractHookInterface

Blueprint hook for post-interaction logic (e.g., Traversal system):

```cpp
UINTERFACE(BlueprintType, meta = (DisplayName = "AnyGame Interact Hook"))
class UCharInteractHookInterface : public UInterface { ... };

class ICharInteractHookInterface
{
    // Called AFTER core interaction executes
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable)
    void OnAnyGameTryInteractHook(
        APlayerCharacter* InstigatorCharacter,
        AActor* OverlappingActor,
        bool bInteractedWithItem,
        bool bJump
    );
};
```

**Usage:** Add interface to `BP_PlayerCharacter`, implement `OnAnyGameTryInteractHook` event.

## Properties

| Property | Type | Purpose |
|----------|------|---------|
| `Character` | `TObjectPtr<APlayerCharacter>` | Owner reference (legacy compat) |
| `CachedOwner` | `TObjectPtr<APlayerCharacter>` | Cached owner (internal) |
| `OverlappingItem` | `TObjectPtr<AActor>` | Current interactable in range |
| `OverlappingItems` | `TArray<TObjectPtr<AActor>>` | All items in overlap (multi) |
| `CurrentActorToFocus` | `TObjectPtr<AActor>` | Actor for visual highlight |
| `Vehicle` | `TObjectPtr<AActor>` | Current vehicle (if in one) |

## Delegates

| Delegate | Signature | Fires When |
|----------|-----------|------------|
| `OnInteractAfterCore` | `(APlayerCharacter*, AActor*, bool)` | After core interaction executes |
| `OnOverlappingItemChanged` | `(AActor*)` | When `OverlappingItem` changes |

## Key Functions

| Function | Purpose |
|----------|---------|
| `TryInteract(Instigator, bJump)` | Main entry point — call from Input or AI |
| `SetOverlappingItem(Actor)` | Set current interactable (call from overlap) |
| `GetOverlappingItem()` | Get current interactable |
| `HasOverlappingItem()` | Check if anything is in range |
| `GetCurrentInteractionPrompt()` | Get UI prompt from current interactable |

## TryInteract Implementation

```cpp
void TryInteract(APlayerCharacter* InstigatorCharacter, bool bJump)
{
    // 1. Resolve instigator (default to owner)
    APlayerCharacter* Instigator = InstigatorCharacter ? InstigatorCharacter : CachedOwner.Get();
    
    bool bInteractedWithItem = false;
    
    // 2. If item in range, try interface call
    if (IsValid(OverlappingItem))
    {
        if (OverlappingItem->Implements<UInteractable>())
        {
            // 3. Check CanInteract
            if (IInteractable::Execute_CanInteract(OverlappingItem, Instigator))
            {
                // 4. Execute interaction
                bInteractedWithItem = IInteractable::Execute_Interact(OverlappingItem, Instigator);
            }
        }
    }
    
    // 5. Broadcast delegate
    OnInteractAfterCore.Broadcast(Instigator, OverlappingItem, bInteractedWithItem);
    
    // 6. Trigger hook interface (BP Traversal)
    TriggerInterfaceHook(Instigator, OverlappingItem, bInteractedWithItem, bJump);
    
    // 7. BP implementable event
    OnTryInteractBP(Instigator, bJump);
}
```

## SetOverlappingItem Implementation

```cpp
void SetOverlappingItem(AActor* NewItem)
{
    if (OverlappingItem == NewItem) return;
    
    // Notify exit on previous
    if (IsValid(OverlappingItem) && OverlappingItem->Implements<UInteractable>())
    {
        IInteractable::Execute_OnExitInteractionRange(OverlappingItem, GetOwner());
    }
    
    OverlappingItem = NewItem;
    CurrentActorToFocus = NewItem;
    
    // Notify enter on new
    if (IsValid(NewItem) && NewItem->Implements<UInteractable>())
    {
        IInteractable::Execute_OnEnterInteractionRange(NewItem, GetOwner());
    }
    
    // Broadcast change
    OnOverlappingItemChanged.Broadcast(NewItem);
}
```

## Lifecycle

```cpp
// Constructor: No tick needed
UCharInteractionComponent()
{
    PrimaryComponentTick.bCanEverTick = false;
}

// BeginPlay: Cache owner
void BeginPlay()
{
    CachedOwner = Cast<APlayerCharacter>(GetOwner());
    Character = CachedOwner; // Legacy compat
}
```

## Quick Start

### 1. Create interactable actor (C++):

```cpp
UCLASS()
class ADoor : public AActor, public IInteractable
{
    // IInteractable implementation
    virtual bool CanInteract_Implementation(APlayerCharacter* Instigator) override
    {
        return !bIsLocked;
    }
    
    virtual bool Interact_Implementation(APlayerCharacter* Instigator) override
    {
        bIsOpen = !bIsOpen;
        PlayOpenAnimation();
        return true;
    }
    
    virtual FText GetInteractionPrompt_Implementation() override
    {
        return bIsOpen ? LOCTEXT("Close", "Close Door") : LOCTEXT("Open", "Open Door");
    }
};
```

### 2. Create interactable actor (Blueprint):

```
BP_Chest:
├── Class Settings → Interfaces → Add "Interactable"
├── Implement "Can Interact" → Return: NOT bIsEmpty
├── Implement "Interact" → Open Chest, Give Items, Return true
└── Implement "Get Interaction Prompt" → Return "Open Chest"
```

### 3. Setup overlap detection (on interactable):

```cpp
// In interactable actor (e.g., ADoor)
void ADoor::OnOverlapBegin(AActor* OtherActor)
{
    if (APlayerCharacter* PC = Cast<APlayerCharacter>(OtherActor))
    {
        if (UCharInteractionComponent* IntComp = PC->GetInteractionComponent())
        {
            IntComp->SetOverlappingItem(this);
        }
    }
}

void ADoor::OnOverlapEnd(AActor* OtherActor)
{
    if (APlayerCharacter* PC = Cast<APlayerCharacter>(OtherActor))
    {
        if (UCharInteractionComponent* IntComp = PC->GetInteractionComponent())
        {
            if (IntComp->GetOverlappingItem() == this)
            {
                IntComp->SetOverlappingItem(nullptr);
            }
        }
    }
}
```

### 4. Setup BP traversal hook:

```
BP_PlayerCharacter:
├── Class Settings → Interfaces → Add "CharInteractHookInterface"
└── Event Graph:
    Event OnAnyGameTryInteractHook
    ├── bJump == true?
    │   └── Yes → Trigger Traversal System
    └── bInteractedWithItem == false?
        └── Check for Mantleable surfaces
```

### 5. Show interaction prompt (UI):

```cpp
// In HUD or Widget
FText Prompt = InteractionComp->GetCurrentInteractionPrompt();
if (!Prompt.IsEmpty())
{
    ShowPromptWidget(Prompt);
}
```

### 6. Listen to interaction events:

```cpp
// C++
InteractionComp->OnInteractAfterCore.AddDynamic(this, &UMyComponent::OnInteraction);

void UMyComponent::OnInteraction(APlayerCharacter* Instigator, AActor* Target, bool bSuccess)
{
    if (bSuccess)
    {
        // Log, achievement, etc.
    }
}
```

```
// Blueprint
Bind Event to OnInteractAfterCore
├── InstigatorCharacter
├── OverlappingActor
└── bInteractedWithItem → Branch → Do something
```

## AI Integration

AI uses the same `TryInteract()` path:

```cpp
// In UCharAIComponent during follow
void UCharAIComponent::TryAutoInteract()
{
    if (UCharInteractionComponent* IntComp = Character->GetInteractionComponent())
    {
        // bJump = true for traversal (vault, climb)
        IntComp->TryInteract(Character, /*bJump=*/ true);
    }
}
```

No special AI code needed — same interface, same flow.

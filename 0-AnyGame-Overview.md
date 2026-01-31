# AnyGame Plugin — Documentation Overview

> **Version:** 1.0 | **Engine:** Unreal Engine 5.7+ | **Architecture:** Player-Centric Composition

## What is AnyGame?

AnyGame is an **opinionated gameplay framework** for Unreal Engine that enforces a clean, scalable architecture through four core principles:

| Principle | Description |
|-----------|-------------|
| **Player-Centric** | All gameplay lives inside the Character via Actor Components. No scattered logic. |
| **World Reacts** | World Actors (doors, chairs, triggers) only respond to character actions — they never initiate. |
| **Player/AI Parity** | Same code path for players and NPCs. No special cases. |
| **Anti-Lock-In** | Modular design. Remove any system without breaking others. |

---

## Architecture at a Glance

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           APlayerCharacter (final)                            │
│                          "Thin Orchestration Layer"                           │
│                                                                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │   Input     │ │  Animation  │ │   Combat    │ │     AI      │            │
│  │  Component  │ │   System    │ │  Component  │ │  Component  │            │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘            │
│         │               │               │               │                    │
│  ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐            │
│  │ Interaction │ │ Attachments │ │   Career    │ │    Sound    │            │
│  │  Component  │ │  Component  │ │  Component  │ │  Component  │            │
│  └──────┬──────┘ └─────────────┘ └─────────────┘ └─────────────┘            │
│         │                                                                    │
│  ┌──────┴──────┐ ┌─────────────┐                                            │
│  │   Widget    │-│    Debug    │                                            │
│  │  Component  │ │  Component  │                                            │
│  └─────────────┘ └─────────────┘                                            │
└──────────────────────────────────────────────────────────────────────────────┘
         │
         │ Interacts with (via IInteractable)
         ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              WORLD MODULE                                     │
│                                                                               │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │
│  │   AChair    │ │   ADoor     │ │ ASoundBox   │ │ ASplinePath │            │
│  │             │ │             │ │             │ │             │            │
│  │ Sit/Stand   │ │ Open/Lock   │ │ Playlist    │ │ AI Paths    │            │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘            │
│                                                                               │
│  ┌────────────────────────────┐ ┌────────────────────────────┐              │
│  │      AStoryTrigger         │ │       ACityManager         │              │
│  │                            │ │                            │              │
│  │ Narrative Orchestrator     │ │ Procedural City + HISM     │              │
│  └────────────────────────────┘ └────────────────────────────┘              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Component Index

### Core Components (Character)

| # | Component | Purpose | Key Features |
|---|-----------|---------|--------------|
| 1 | [APlayerCharacter](1-APlayerCharacter.md) | Orchestration layer | `final` class, hosts all components, no tick |
| 7 | [UCharInputComponent](7-UCharInputComponent.md) | Input routing | Enhanced Input, action delegation |
| 3 | [Animation System](3-AnimationSystem.md) | Animation orchestration | AnimFlow + AnimInstance + Notifies |
| 5 | [UCharCombatComponent](5-UCharCombatComponent.md) | Combat state machine | Attack flow, counters, finishers, LOS gate |
| 2 | [UCharAIComponent](2-UCharAIComponent.md) | AI behavior controller | Follow, combat, trail recorder, tag matrix |
| 8 | [UCharInteractionComponent](8-UCharInteractionComponent.md) | Interaction router | IInteractable, event chain, BP hooks |
| 4 | [UCharAttachmentsComponent](4-UCharAttachmentsComponent.md) | Attachment manager | Slot system, state broadcast |
| 9 | [UCharSoundComponent](9-UCharSoundComponent.md) | Audio manager | Dual channel (Body + Mouth) |
| 10 | [UCharWidgetComponent](10-UCharWidgetComponent.md) | UI manager | HUD, combat icons, tutorials |
| 6 | [UCharDebugComponent](6-UCharDebugComponent.md) | Debug inspector | Flag-per-component, on-screen display |

### UI Widgets

| # | Widget | Purpose | Key Features |
|---|--------|---------|--------------|
| 11 | [UPlayerHUD](11-UPlayerHUD.md) | Main HUD | Health, minimap, subtitles, fade systems |
| 12 | [UPauseMenu](12-UPauseMenu.md) | Pause menu | Input mode switching, level management |
| 13 | [UMainMenu](13-UMainMenu.md) | Main menu | Music playlist, auto-loop |

### World Actors

| # | Actor | Purpose | Key Features |
|---|-------|---------|--------------|
| 15 | [AChair](15-AChair.md) | Interactable chair | Motion warping, sit/stand states |
| 16 | [ADoor](16-ADoor.md) | Interactable door | 3-state machine, direction-aware anims |
| 17 | [ASoundBox](17-ASoundBox.md) | Ambient audio | Playlist, spatialization, no tick |
| 18 | [ASplinePath](18-ASplinePath.md) | Spline wrapper | Editor icons, AI/camera paths |
| 19 | [AStoryTrigger](19-AStoryTrigger.md) | Narrative orchestrator | Multi-step, participants, handlers |
| 14 | [ACityManager](14-ACityManager.md) | Procedural city | Spline-driven, HISM optimization |

---

## Key Systems

### 1. Player/AI Parity

Same character class, same components, same code path:

```cpp
// Player
APlayerCharacter* Player = SpawnCharacter();
PlayerController->Possess(Player);  // → Input drives actions

// NPC
APlayerCharacter* NPC = SpawnCharacter();
NPC->Tags.Add("Enemy");  // → AI drives actions
// Don't possess → AI auto-activates
```

### 2. Tag Matrix (Factions)

Combat and AI use Actor Tags for faction resolution:

```
         │ Player │ Friend │ Enemy
─────────┼────────┼────────┼────────
Player → │   ✗    │   ✗    │   ✓
Friend → │   ✗    │   ✗    │   ✓
Enemy  → │   ✓    │   ✓    │   ✗
```

### 3. IInteractable Interface

All world interactables implement:

```cpp
bool CanInteract(APlayerCharacter* Instigator);
bool Interact(APlayerCharacter* Instigator);
void OnEnterInteractionRange(AActor* Character);
void OnExitInteractionRange(AActor* Character);
FText GetInteractionPrompt();
```

### 4. Combat Flow

```
HandleAttack()
     │
     ├── Gate: bCanAttack, target, LOS, TagMatrix
     │
     ├── InitializeAttack() → Lock gate
     │
     ├── CalculateDirectionAndCombo() → Front/Back/Left/Right
     │
     ├── Play Montage
     │        │
     │        ├── [Notify] ResetAttack → Unlock gate (can queue)
     │        ├── [Notify] OpenCounterWindow → Target can defend
     │        ├── [Notify] ApplyHitEffect → Damage + VFX
     │        ├── [Notify] PlayGetHitMontage → Target reacts
     │        └── [Notify] EndAttack → Full cleanup
```

### 5. AI Trail System

Player records path → NPCs follow exact trail (no NavMesh needed):

```
[Player]                              [NPC Follower]
    │                                       │
    ├── RecordTrailPoint() every 0.1s       │
    │        │                              │
    │        ▼                              │
    │   Trail Spline (128 points max)       │
    │        │                              │
    │        └──────────────────────────────┘
    │                                       │
    │                              FollowTrailSpline()
    │                                       │
    │                              Guaranteed traversal
    │                              (stairs, jumps, doors)
```

### 6. Story Sequence Flow

```
AStoryTrigger (overlap or BeginPlay)
         │
         ▼
    StartStorySequence()
         │
         ▼
    For each step:
         │
         ├── PlayLevelSequence()
         ├── HandleParticipantsForStep()
         │   ├── ExecuteStorySequence() → CharAnimComp
         │   └── ApplyHandlers()
         │       ├── AIHandler (Activate/Deactivate, GoTo)
         │       ├── CareerHandler (Missions, Achievements)
         │       └── AttachmentHandler (Attach/Detach)
         ├── PlayWorldManagerSounds()
         ├── HandleActorSpawns()
         └── HandleActorDestroys()
         │
         ▼
    Wait for all participants → Next step
```

---

## Data Assets

| Asset Type | Purpose | Used By |
|------------|---------|---------|
| `UMontageCollectionsDataAsset` | Named montage groups | UCharAnimComponent |
| `UStorySequencesDataAsset` | Multi-step narratives | AStoryTrigger |
| `UCityBlockAsset` | Road modules + spawn zones | ACityManager |
| `UPlayerAsset` | Character config | Multiple components |

---

## Performance Patterns

| Pattern | Implementation | Benefit |
|---------|----------------|---------|
| **No Tick** | `bCanEverTick = false` | Zero idle cost |
| **Timer-Based** | `FTimerHandle` | Explicit scheduling |
| **Event-Driven** | Delegates + Notifies | Reactive, not polling |
| **HISM Batching** | `UHierarchicalInstancedStaticMeshComponent` | 1 draw call per mesh type |
| **LOS Caching** | Throttled checks + hysteresis | Prevents flicker, saves traces |
| **Thread-Safe Anim** | `NativeThreadSafeUpdateAnimation` | Parallel evaluation |

---

## Quick Start

### Create a Player

```cpp
// 1. Create Blueprint inheriting from APlayerCharacter
// 2. Drop in level
// 3. Possess as Player 0
// Done. All components auto-created.
```

### Create an NPC Follower

```cpp
NPC->Tags.Add("Friend");
NPC->GetAIComponent()->FollowPlayer(PlayerCharacter);
```

### Create an NPC Enemy

```cpp
NPC->Tags.Add("Enemy");
// Combat starts automatically when Player enters range
```

### Create an Interactable

```cpp
UCLASS()
class AMyPickup : public AActor, public IInteractable
{
    virtual bool Interact_Implementation(AActor* Instigator) override
    {
        Destroy();
        return true;
    }
};
```

### Trigger a Story Sequence

```cpp
// 1. Create UStorySequencesDataAsset
// 2. Place AStoryTrigger in level
// 3. Assign data asset + trigger actor
// 4. Player enters → Sequence plays
```

---

## Design Rules

### ✓ DO

- Create Actor Components for new character behaviors
- Implement IInteractable for world interactions
- Use Data Assets for content
- Route through components, never access Character internals
- Same code path for Player and AI

### ✗ DON'T

- Subclass APlayerCharacter (it's `final`)
- Put logic in World Actors (they only react)
- Hardcode references (use interfaces and tags)
- Special-case Player vs AI
- Tick when you can delegate

---

## Extension Points

| What | How |
|------|-----|
| New Character Behavior | Create `UActorComponent`, add to constructor |
| New Interactable | Implement `IInteractable` |
| New Combat Action | Add to `EAnimNotifyCombatType` |
| New AI State | Extend `EAIState` enum |
| New Attachment Slot | Extend `ECharAttachmentSlot` |
| New Debug Section | Add flag to `UCharDebugComponent` |
| New Story Handler | Add struct + apply function |
| New City Block | Create `UCityBlockAsset` |

---

## File Structure

```
AnyGame/
├── Core/
│   └── PlayerCharacter/
│       ├── PlayerCharacter.h/.cpp
│       └── Components/
│           ├── AI/
│           ├── Animation/
│           │   ├── AnimFlow/
│           │   ├── AnimInstance/
│           │   └── Notifies/
│           ├── Attachments/
│           ├── Combat/
│           ├── Debug/
│           ├── Input/
│           ├── Interaction/
│           ├── Career/
│           ├── Sound/
│           └── Widget/
│               └── UI/
│
└── World/
    ├── City/Manager/
    ├── Interactables/
    │   ├── Chair/
    │   ├── Door/
    │   └── SoundBox/
    ├── SplinePath/
    └── StoryTrigger/
```

---

## Documentation Summary

| Category | Documents | Total Lines |
|----------|-----------|-------------|
| Core Components | 10 | ~2,500 |
| UI Widgets | 3 | ~620 |
| World Actors | 6 | ~1,850 |
| **Total** | **19** | **~5,000** |

---

## Philosophy

> *"Opinionated enough to accelerate. Modular enough to not trap."*

AnyGame is not a template — it's an architecture. It tells you **where** things go so you can focus on **what** to build.

- Gameplay logic? → Character Components
- World reactions? → IInteractable implementers
- Narratives? → StoryTrigger + Data Assets
- Procedural content? → CityManager + Data Assets

The framework makes decisions so you don't have to debate architecture. You just build gameplay.

---

**AnyGame Plugin** — Solo developed over 1 year of iteration.

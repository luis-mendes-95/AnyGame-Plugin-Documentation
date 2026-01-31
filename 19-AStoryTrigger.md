# AStoryTrigger

> **Module:** AnyGame World | **Header:** `World/StoryTrigger/StoryTrigger.h` | **Parent:** `AActor`

## Overview

`AStoryTrigger` is the **narrative orchestrator** — it executes multi-step story sequences with participant animations, AI control, career updates, attachments, sounds, actor spawns/destroys, and level sequences. Can be triggered by overlap, BeginPlay, or manual call.

## Design Principles

**Data-Driven:** Story sequences are defined in `UStorySequencesDataAsset`. No hardcoded narrative logic.

**Multi-Participant:** Each step can involve multiple characters with independent handlers.

**Asynchronous Completion:** Steps advance only when all participants finish their sequences.

**World Manager:** Centralized control for sounds, spawns, destroys, and other triggers.

**No Tick:** `PrimaryActorTick.bCanEverTick = false`. Uses timers and delegates.

## Components

| Component | Type | Purpose |
|-----------|------|---------|
| `EditorIcon` | `UBillboardComponent` | Editor visualization (Root) |
| `TriggerBox` | `UBoxComponent` | Overlap trigger (200×200×100) |

## Properties

### Story Configuration

| Property | Type | Purpose |
|----------|------|---------|
| `StorySequencesAsset` | `UStorySequencesDataAsset*` | Data asset with sequence steps |
| `ActorToTriggerStory` | `AActor*` | Actor that triggers on overlap |
| `StartingIndex` | `int32` | Which step to start from |
| `CurrentIndex` | `int32` | Current step (read-only) |

### Trigger Settings

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `bIsRunning` | `bool` | true | Sequence is active |
| `bInteractionIsAllowed` | `bool` | true | Can be triggered |
| `bStartAtBeginPlay` | `bool` | false | Auto-start on BeginPlay |

### Internal State

| Property | Type | Purpose |
|----------|------|---------|
| `CurrentStepIndex` | `int32` | Current step being executed |
| `FinishedParticipants` | `TSet<AActor*>` | Tracks who finished current step |
| `bIsSequenceActive` | `bool` | Prevents re-entry |

## Execution Flow

```
[Trigger]
    │
    ├── OnTriggerBeginOverlap (if ActorToTriggerStory overlaps)
    ├── BeginPlay (if bStartAtBeginPlay)
    └── Manual call (StartStorySequence)
           │
           ▼
    StartStorySequence()
           │
           ▼
    AdvanceStep()
           │
           ├── StartDelay > 0? → Timer → ExecuteStep()
           └── StartDelay == 0 → ExecuteStep() immediately
                    │
                    ▼
           ExecuteStep(StepIndex)
                    │
                    ├── PlayLevelSequence()
                    ├── HandleWorldManager(bAtStart=true)
                    │   ├── HandleParticipantsForStep()
                    │   │   ├── ExecuteParticipantStorySequence()
                    │   │   └── ApplyParticipantHandlers()
                    │   ├── PlayWorldManagerSounds()
                    │   ├── HandleActorSpawns()
                    │   ├── HandleActorDestroys()
                    │   └── HandleStoryTriggers()
                    │
                    └── TryAdvanceStepByWorldManager()
                             │
                             ├── No participants → Auto-advance
                             └── Has participants → Wait for OnStepFinished
                                      │
                                      ▼
                             OnStepFinished(Character)
                                      │
                                      ├── Mark participant finished
                                      ├── All finished? → HandleWorldManager(bAtStart=false)
                                      └── TryAdvanceStepWhenAllParticipantsFinished()
                                               │
                                               └── ++CurrentStepIndex → AdvanceStep()
```

## Data Structures

### FStorySequence

```cpp
USTRUCT()
struct FStorySequence
{
    float StartDelay;                    // Delay before step execution
    ULevelSequence* LevelSequence;       // Cinematic to play
    FWorldManager WorldManager;          // World state changes
};
```

### FWorldManager

```cpp
USTRUCT()
struct FWorldManager
{
    TArray<FStoryStepParticipant> Participants;  // Characters in this step
    FSoundFX SoundFX;                            // Sounds at start/end
    TArray<FActorSpawns> ActorSpawns;            // Actors to spawn
    TArray<FActorDestroys> ActorDestroys;        // Actors to destroy
    TArray<FStoryTriggerManager> StoryTriggersToHandle;  // Other triggers
};
```

### FStoryStepParticipant

```cpp
USTRUCT()
struct FStoryStepParticipant
{
    FName CharacterName;              // Actor name or tag to find
    FAIHandler AIHandler;             // AI control settings
    FCareerHandler CareerHandler;     // Mission/achievement updates
    FAttachmentHandler AttachmentHandler;  // Attach/detach items
    // + Animation settings via ExecuteStorySequence
};
```

## Participant Handlers

### AIHandler

```cpp
struct FAIHandler
{
    EAIControlMode AIControl;    // Activate/Deactivate/None
    EAIStateApplyTime ApplyTime; // Immediate or AtEnd
    EAIState TargetState;        // Goto, Patrol, etc.
    FName PathTag;               // SplinePath tag for Goto
};
```

**Usage:**
```cpp
void ApplyAIHandler(AActor* CharacterActor, const FStoryStepParticipant& Participant, bool bAtStart)
{
    if (AIH.AIControl != None)
    {
        bool bShouldApply = (AIH.ApplyTime == Immediate && bAtStart) ||
                            (AIH.ApplyTime == AtEnd && !bAtStart);
        
        if (bShouldApply)
        {
            AIComp->bIsAIActive = (AIH.AIControl == Activate);
        }
    }
    
    if (AIH.TargetState == Goto && AIH.PathTag != NAME_None)
    {
        ASplinePath* FoundPath = FindActorByTag<ASplinePath>(AIH.PathTag);
        if (FoundPath)
        {
            AIComp->GoToSpline(FoundPath, FName(""));
        }
    }
}
```

### CareerHandler

```cpp
struct FCareerHandler
{
    bool bApplyAtStart;
    bool bSetCurrentMission;
    FText CurrentMission;
    bool bSetDestination;
    FVector Destination;
    bool bAddAvailableMission;
    bool bCompleteMission;
    bool bAddAchievement;
    FText Achievement;
};
```

**Usage:**
```cpp
void ApplyCareerHandler(...)
{
    if (CH.bSetCurrentMission)
    {
        CareerComp->CurrentMission = CH.CurrentMission;
        HUD->ShowHint(CH.CurrentMission.ToString());
        HUD->ShowMinimap();
    }
    
    if (CH.bSetDestination)
    {
        CareerComp->Destination = CH.Destination;
    }
    
    if (CH.bCompleteMission)
    {
        CareerComp->CurrentMission = FText::GetEmpty();
        HUD->HideMinimap();
    }
    
    if (CH.bAddAchievement)
    {
        CareerComp->Achievements.Add(CH.Achievement);
        HUD->ShowHint(CH.Achievement.ToString());
    }
}
```

### AttachmentHandler

```cpp
struct FAttachmentHandler
{
    bool bSetupAttachment;
    bool bApplyAtStart;
    bool bAttach;
    TSubclassOf<AActor> ActorClass;
    ECharAttachmentSlot Slot;
    FName SocketName;
    bool bDetach;
    ECharAttachmentSlot DetachSlot;
};
```

**Usage:**
```cpp
void ApplyAttachmentHandler(...)
{
    if (AH.bAttach && AH.ActorClass)
    {
        AActor* SpawnedActor = World->SpawnActor<AActor>(AH.ActorClass);
        AttachComp->AttachActor(SpawnedActor, AH.Slot, AH.SocketName);
    }
    
    if (AH.bDetach)
    {
        AttachComp->DetachSlot(AH.DetachSlot);
    }
}
```

## World Manager Functions

### HandleActorSpawns

```cpp
void HandleActorSpawns(const TArray<FActorSpawns>& Spawns, bool bAtStart)
{
    for (const FActorSpawns& SpawnInfo : Spawns)
    {
        if (SpawnInfo.bSpawnActorAtStart != bAtStart) continue;
        if (!SpawnInfo.Actors) continue;
        
        World->SpawnActor<AActor>(SpawnInfo.Actors, SpawnInfo.SpawnLocation);
    }
}
```

### HandleActorDestroys

```cpp
void HandleActorDestroys(const TArray<FActorDestroys>& Destroys, bool bAtStart)
{
    for (const FActorDestroys& DestroyInfo : Destroys)
    {
        if (DestroyInfo.bDestroyAtStart != bAtStart) continue;
        
        TArray<AActor*> FoundActors;
        GetAllActorsOfClass(DestroyInfo.Actors, FoundActors);
        
        for (AActor* Actor : FoundActors)
        {
            Actor->Destroy();
        }
    }
}
```

### HandleStoryTriggers

```cpp
void HandleStoryTriggers(const TArray<FStoryTriggerManager>& Managers, bool bAtStart)
{
    for (const FStoryTriggerManager& M : Managers)
    {
        if (M.bHandleAtStart != bAtStart) continue;
        
        switch (M.StoryTriggerAction)
        {
            case EnableInteraction:
                M.StoryTrigger->SetActorEnableCollision(true);
                break;
            case DisableInteraction:
                M.StoryTrigger->SetActorEnableCollision(false);
                break;
        }
    }
}
```

### PlayWorldManagerSounds

```cpp
void PlayWorldManagerSounds(const FWorldManager& WorldManager, bool bAtStart)
{
    const TArray<USoundBase*>& Sounds = bAtStart 
        ? WorldManager.SoundFX.SoundsAtStart 
        : WorldManager.SoundFX.SoundsAtEnd;
    
    for (USoundBase* Sound : Sounds)
    {
        UGameplayStatics::PlaySound2D(this, Sound);
    }
}
```

## Utility Functions

### FindActorByName

Static utility to find actors by name or tag:

```cpp
UFUNCTION(BlueprintCallable, Category = "Utils|Actor")
static AActor* FindActorByName(UObject* WorldContextObject, FName ActorName)
{
    for (TActorIterator<AActor> It(World); It; ++It)
    {
        AActor* Actor = *It;
        
        // Match by name or tag
        bool bMatchesName = Actor->GetFName() == ActorName || 
                            Actor->GetName().StartsWith(ActorName.ToString());
        bool bMatchesTag = Actor->Tags.Contains(ActorName);
        
        if (bMatchesName || bMatchesTag)
            return Actor;
    }
    return nullptr;
}
```

## Lifecycle

```cpp
// Constructor: Setup components
AStoryTrigger()
{
    PrimaryActorTick.bCanEverTick = false;
    SetupEditorIcon();
    SetupTriggerBox();
}

void SetupEditorIcon()
{
    EditorIcon = CreateDefaultSubobject<UBillboardComponent>(TEXT("EditorIcon"));
    RootComponent = EditorIcon;
}

void SetupTriggerBox()
{
    TriggerBox = CreateDefaultSubobject<UBoxComponent>(TEXT("TriggerBox"));
    TriggerBox->SetupAttachment(RootComponent);
    TriggerBox->SetCollisionProfileName(TEXT("Trigger"));
    TriggerBox->SetBoxExtent(FVector(200.f, 200.f, 100.f));
}

// BeginPlay: Bind overlap, optionally auto-start
void BeginPlay()
{
    BindTriggerOverlap();
    TryStartStoryAtBeginPlay();
}

void TryStartStoryAtBeginPlay()
{
    if (!bStartAtBeginPlay) return;
    
    // Small delay to ensure world is ready
    SetTimer(0.1f, StartStorySequence);
}
```

## Quick Start

### 1. Create StorySequencesDataAsset:

```
Content Browser → Right Click → Miscellaneous → Data Asset
Class: UStorySequencesDataAsset

DA_IntroSequence:
├── StorySequences[0]:
│   ├── StartDelay: 0.5
│   ├── LevelSequence: LS_IntroCinematic
│   └── WorldManager:
│       ├── Participants[0]:
│       │   ├── CharacterName: "Player"
│       │   └── AIHandler: (None)
│       ├── SoundFX:
│       │   └── SoundsAtStart: [S_IntroMusic]
│       └── ActorSpawns: []
├── StorySequences[1]:
│   └── ...
```

### 2. Place StoryTrigger in level:

```
World Outliner → Add Actor → AStoryTrigger

BP_IntroTrigger:
├── StorySequencesAsset → DA_IntroSequence
├── ActorToTriggerStory → BP_PlayerCharacter
├── bStartAtBeginPlay → false
└── TriggerBox → Position/Scale to desired area
```

### 3. Trigger on overlap:

Player enters TriggerBox → Story sequence starts automatically.

### 4. Trigger manually:

```cpp
// From Blueprint or C++
IntroTrigger->StartStorySequence();
```

### 5. Trigger from ADoor:

```cpp
// In ADoor - when door is locked
UPROPERTY(EditAnywhere, Category = "Story")
AStoryTrigger* LockedScene;

// In Interact_Implementation
if (DoorState == Locked && !bHasKey)
{
    if (LockedScene)
    {
        LockedScene->StartStorySequence();
    }
}
```

### 6. Multi-character cutscene:

```
DA_Conversation:
├── StorySequences[0]:
│   └── WorldManager:
│       ├── Participants[0]:
│       │   ├── CharacterName: "Player"
│       │   └── (animation settings via CharAnimComponent)
│       └── Participants[1]:
│           ├── CharacterName: "NPC_Guard"
│           └── AIHandler:
│               └── AIControl: Deactivate (stop AI during cutscene)
```

### 7. Mission system integration:

```
DA_MissionStart:
├── StorySequences[0]:
│   └── WorldManager:
│       └── Participants[0]:
│           ├── CharacterName: "Player"
│           └── CareerHandler:
│               ├── bSetCurrentMission: true
│               ├── CurrentMission: "Find the key"
│               ├── bSetDestination: true
│               └── Destination: (500, 1000, 0)
```

### 8. Attachment during cutscene:

```
DA_PickupWeapon:
├── StorySequences[0]:
│   └── WorldManager:
│       └── Participants[0]:
│           ├── CharacterName: "Player"
│           └── AttachmentHandler:
│               ├── bSetupAttachment: true
│               ├── bApplyAtStart: true
│               ├── bAttach: true
│               ├── ActorClass: BP_Pistol
│               ├── Slot: RightHand
│               └── SocketName: "hand_r_socket"
```

### 9. Chain triggers:

Use `StoryTriggersToHandle` to enable other triggers after sequence:

```
DA_UnlockNextArea:
├── StorySequences[0]:
│   └── WorldManager:
│       └── StoryTriggersToHandle[0]:
│           ├── StoryTrigger: ST_NextAreaTrigger
│           ├── StoryTriggerAction: EnableInteraction
│           └── bHandleAtStart: false (at end of step)
```

## Integration with CharAnimComponent

Story sequences delegate animation execution to `UCharAnimComponent`:

```cpp
void ExecuteParticipantStorySequence(AActor* CharacterActor, ...)
{
    UCharAnimComponent* CharAnimComp = CharacterActor->FindComponentByClass<UCharAnimComponent>();
    
    // Bind completion callback
    CharAnimComp->OnStorySequenceFinished.Clear();
    CharAnimComp->OnStorySequenceFinished.AddUObject(this, &AStoryTrigger::OnStepFinished);
    
    // Execute the sequence (plays montages, dialogue, etc.)
    if (bAtStart)
    {
        CharAnimComp->ExecuteStorySequence(Sequence, Participant, 0);
    }
}
```

## Logging

Extensive logging via custom log category:

```cpp
DECLARE_LOG_CATEGORY_EXTERN(LogStory, Log, All);

// Filter in Output Log: LogStory
UE_LOG(LogStory, Warning, TEXT("[ExecuteStep] Step %d START"), StepIndex);
```

## Architecture Compliance

✅ **World Reacts:** StoryTrigger orchestrates but doesn't contain gameplay logic. Characters execute via their own components.

✅ **Player/AI Parity:** Same `ExecuteStorySequence` path for players and NPCs.

✅ **Data-Driven:** All narrative content in `UStorySequencesDataAsset`.

✅ **No Tick:** Timer and delegate-based execution.

✅ **Anti-Lock-In:** Remove StoryTrigger → lose that specific cutscene. Character systems unaffected.

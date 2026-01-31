# ASoundBox

> **Module:** AnyGame World | **Header:** `World/Interactables/SoundBox/SoundBox.h` | **Parent:** `AActor`

## Overview

`ASoundBox` is an **ambient sound emitter** — it plays a playlist of sounds with optional looping, random playback, and spatialization. Used for environmental audio like radios, speakers, or ambient soundscapes.

## Design Principles

**World Reacts:** SoundBox is a World Actor. It emits sound but doesn't contain gameplay logic.

**Playlist-Based:** Supports multiple sounds with sequential, random, or single-play modes.

**Asset-Driven Attenuation:** Uses `USoundAttenuation` assets for consistent audio falloff across instances.

**No Tick:** `PrimaryActorTick.bCanEverTick = false`. Uses timer-based playback scheduling.

## Components

| Component | Type | Purpose |
|-----------|------|---------|
| `Mesh` | `UStaticMeshComponent` | Visual representation (Root) |
| `AudioComponent` | `UAudioComponent` | Sound playback |

## Properties

### Sound Configuration

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `SoundList` | `TArray<USoundBase*>` | — | Playlist of sounds to play |
| `bRandomPlayback` | `bool` | false | Random track selection |
| `bLoopPlayback` | `bool` | true | Loop playlist when finished |
| `EffectPresetChain` | `USoundEffectSourcePresetChain*` | — | Audio effect chain |

### Attenuation

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `AttenuationAsset` | `USoundAttenuation*` | — | Spatialization settings |
| `bAutoActivate` | `bool` | false | Start playing on BeginPlay |
| `bAllowSpatialization` | `bool` | true | Enable 3D audio |

### Internal State

| Property | Type | Purpose |
|----------|------|---------|
| `CurrentSoundIndex` | `int32` | Current track index |
| `PlaybackTimerHandle` | `FTimerHandle` | Timer for track transitions |

## Playback Modes

| Mode | Flags | Behavior |
|------|-------|----------|
| **Sequential Loop** | `bLoopPlayback = true`, `bRandomPlayback = false` | Plays 1→2→3→1→2→3... |
| **Random Loop** | `bLoopPlayback = true`, `bRandomPlayback = true` | Plays random tracks forever |
| **Sequential Once** | `bLoopPlayback = false`, `bRandomPlayback = false` | Plays 1→2→3 then stops |
| **Random Once** | `bLoopPlayback = false`, `bRandomPlayback = true` | Plays one random track then stops |

## Key Functions

| Function | Purpose |
|----------|---------|
| `PlaySound(USoundBase*)` | Play a specific sound |
| `PlayNextSound()` | Advance to next sound based on mode |
| `GetCurrentSoundDuration()` | Get duration of current track |

## PlaySound

```cpp
void PlaySound(USoundBase* Sound)
{
    if (AudioComponent && Sound)
    {
        AudioComponent->SetSound(Sound);
        AudioComponent->Play();
        
        // Schedule next track
        float Duration = Sound->GetDuration();
        if (Duration > 0.0f)
        {
            GetWorldTimerManager().SetTimer(
                PlaybackTimerHandle, 
                this, 
                &ASoundBox::PlayNextSound, 
                Duration, 
                false
            );
        }
    }
}
```

## PlayNextSound

```cpp
void PlayNextSound()
{
    if (SoundList.Num() == 0) return;
    
    if (bRandomPlayback)
    {
        // Random mode: pick any track
        CurrentSoundIndex = FMath::RandRange(0, SoundList.Num() - 1);
    }
    else if (bLoopPlayback)
    {
        // Loop mode: wrap around
        CurrentSoundIndex = (CurrentSoundIndex + 1) % SoundList.Num();
    }
    else if (CurrentSoundIndex >= SoundList.Num() - 1)
    {
        // Sequential once: stop at end
        return;
    }
    else
    {
        // Sequential: advance
        CurrentSoundIndex++;
    }
    
    PlaySound(SoundList[CurrentSoundIndex]);
}
```

## Lifecycle

```cpp
// Constructor: Create components, no tick
ASoundBox()
{
    PrimaryActorTick.bCanEverTick = false;
    
    Mesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Mesh"));
    RootComponent = Mesh;
    
    AudioComponent = CreateDefaultSubobject<UAudioComponent>(TEXT("AudioComponent"));
    AudioComponent->SetupAttachment(RootComponent);
}

// BeginPlay: Configure audio, start playback
void BeginPlay()
{
    Super::BeginPlay();
    
    if (AudioComponent)
    {
        // Apply settings
        AudioComponent->bAutoActivate = bAutoActivate;
        AudioComponent->bAllowSpatialization = bAllowSpatialization;
        
        if (AttenuationAsset)
            AudioComponent->AttenuationSettings = AttenuationAsset;
        
        if (EffectPresetChain)
            AudioComponent->SourceEffectChain = EffectPresetChain;
    }
    
    // Start playlist
    if (SoundList.Num() > 0)
    {
        CurrentSoundIndex = 0;
        PlaySound(SoundList[0]);
    }
}
```

## Quick Start

### 1. Create SoundBox Blueprint:

```
BP_SoundBox (Parent: ASoundBox):
├── Mesh → Assign static mesh (radio, speaker, etc.)
└── AudioComponent → (auto-created)
```

### 2. Configure playlist:

```
BP_SoundBox → Details → Sound Box:
├── SoundList:
│   ├── [0] → S_AmbientTrack1
│   ├── [1] → S_AmbientTrack2
│   └── [2] → S_AmbientTrack3
├── bLoopPlayback → true
└── bRandomPlayback → false
```

### 3. Configure spatialization:

```
BP_SoundBox → Details → Attenuation:
├── AttenuationAsset → SA_RadioAttenuation
├── bAutoActivate → true
└── bAllowSpatialization → true
```

### 4. Create attenuation asset:

```
Content Browser → Right Click → Sounds → Sound Attenuation
SA_RadioAttenuation:
├── Attenuation Shape → Sphere
├── Inner Radius → 200
├── Falloff Distance → 1000
└── Spatialization Algorithm → Default
```

### 5. Place in level:

```
Drop BP_SoundBox in level → Plays automatically on BeginPlay
```

### 6. Use with CityManager:

SoundBox is supported by `ACityManager::SpawnReferenceActorMeshes()`:

```cpp
// In CityManager (already implemented)
ASoundBox* SoundBoxBP = Cast<ASoundBox>(ChildActor);
if (SoundBoxBP)
{
    ASoundBox* SpawnedSoundBox = GetWorld()->SpawnActor<ASoundBox>(
        SoundBoxBP->GetClass(), 
        ChildWorldTransform, 
        SpawnParams
    );
    
    // Auto-discover components
    if (!SpawnedSoundBox->Mesh)
        SpawnedSoundBox->Mesh = SpawnedSoundBox->FindComponentByClass<UStaticMeshComponent>();
    
    if (!SpawnedSoundBox->AudioComponent)
        SpawnedSoundBox->AudioComponent = SpawnedSoundBox->FindComponentByClass<UAudioComponent>();
    
    SpawnedActors.Add(SpawnedSoundBox);
}
```

## Use Cases

### Ambient Radio

```
BP_Radio:
├── Mesh → SM_Radio
├── SoundList → [S_RadioStatic, S_RadioMusic1, S_RadioMusic2]
├── bLoopPlayback → true
├── bRandomPlayback → true
└── AttenuationAsset → SA_SmallSpeaker (200-500 range)
```

### PA System / Announcement

```
BP_PASpeaker:
├── Mesh → SM_Speaker
├── SoundList → [S_Announcement1, S_Announcement2]
├── bLoopPlayback → false (play once)
├── bRandomPlayback → false
└── AttenuationAsset → SA_LargeSpeaker (500-2000 range)
```

### Environmental Ambience

```
BP_AmbientEmitter:
├── Mesh → (invisible or small)
├── SoundList → [S_Wind, S_Birds, S_Traffic]
├── bLoopPlayback → true
├── bRandomPlayback → true
└── AttenuationAsset → SA_LargeAmbience (1000-5000 range)
```

## Architecture Compliance

✅ **World Reacts:** SoundBox only emits sound. No gameplay logic.

✅ **No Tick:** Timer-based playback, zero tick overhead.

✅ **CityManager Compatible:** Fully supported in procedural city generation.

✅ **Anti-Lock-In:** Self-contained audio emitter. Remove → lose ambient sound, nothing else affected.

# UCharSoundComponent

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Sound/CharSoundComponent.h` | **Parent:** `UActorComponent`

## Overview

`UCharSoundComponent` is the **audio manager** — it hosts two audio channels (Body + Mouth), plays contextual sounds, and stores sound asset references. Dialogue and general SFX are kept separate.

## Design Principles

**Dual Channel:** `MouthAudioComp` for dialogue/voice, `BodyAudioComp` for SFX (hits, footsteps, etc.). They can play simultaneously without cutting each other off.

**Runtime Attenuation:** Attenuation settings are created at runtime per-sound. No need for pre-configured `USoundAttenuation` assets.

**Data Asset Driven:** Sound references can be populated from `UPlayerAsset` via `SetupSoundsFromDataAsset()`.

**Lazy Init:** Audio components are created on first use, not in constructor.

## Audio Channels

| Channel | Component | Purpose | Attachment |
|---------|-----------|---------|------------|
| **Mouth** | `MouthAudioComp` | Dialogue, voice lines, grunts | Root |
| **Body** | `BodyAudioComp` | Combat SFX, footsteps, impacts | Root |

Both components are attached to owner's root and follow the character.

## Sound Properties

### Combat Sounds

| Property | Type | Purpose |
|----------|------|---------|
| `BreakBoneSound` | `USoundBase*` | Bone break finisher |
| `BlockHitSound` | `USoundBase*` | Blocked attack impact |
| `BodyPunchGetHitSound` | `USoundBase*` | Melee hit reaction |
| `BodySlashGetHitSound` | `USoundBase*` | Slash hit reaction |
| `LandSound` | `USoundBase*` | Landing after fall/jump |

### HUD Sounds

| Property | Type | Purpose |
|----------|------|---------|
| `PopupSound` | `USoundBase*` | UI notification popup |

## Key Functions

| Function | Returns | Purpose |
|----------|---------|---------|
| `PlayGeneralSound(Sound, Volume, Pitch, Radius)` | `void` | Play SFX on Body channel |
| `PlayDialogue(Sound)` | `UAudioComponent*` | Play voice on Mouth channel |
| `StopDialogue()` | `void` | Stop current dialogue |
| `SetupSoundsFromDataAsset(Char, Asset)` | `void` | Populate sounds from data asset |

## PlayGeneralSound

Plays a sound effect with runtime attenuation:

```cpp
void PlayGeneralSound(
    USoundBase* Sound, 
    float VolumeMultiplier = 1.f, 
    float PitchMultiplier = 1.f, 
    float AttenuationRadius = 2000.f
)
{
    if (!Sound || !GetOwner()) return;
    
    // Lazy init Body channel
    if (!BodyAudioComp)
    {
        BodyAudioComp = NewObject<UAudioComponent>(this);
        BodyAudioComp->RegisterComponent();
        BodyAudioComp->AttachToComponent(GetOwner()->GetRootComponent(), ...);
    }
    
    // Create runtime attenuation
    USoundAttenuation* Attenuation = NewObject<USoundAttenuation>(this);
    Attenuation->Attenuation.bAttenuate = true;
    Attenuation->Attenuation.bSpatialize = true;
    Attenuation->Attenuation.AttenuationShape = EAttenuationShape::Sphere;
    Attenuation->Attenuation.AttenuationShapeExtents = FVector(AttenuationRadius);
    Attenuation->Attenuation.FalloffDistance = AttenuationRadius * 0.5f;
    
    // Play
    BodyAudioComp->SetSound(Sound);
    BodyAudioComp->AttenuationSettings = Attenuation;
    BodyAudioComp->SetVolumeMultiplier(VolumeMultiplier);
    BodyAudioComp->SetPitchMultiplier(PitchMultiplier);
    BodyAudioComp->Play();
}
```

### Attenuation Defaults

| Parameter | Value | Notes |
|-----------|-------|-------|
| `AttenuationRadius` | 2000 | Sphere extent |
| `FalloffDistance` | Radius × 0.5 | 50% of radius |
| `AttenuationShape` | Sphere | Omnidirectional |

## PlayDialogue

Plays voice/dialogue with cinematic-quality attenuation:

```cpp
UAudioComponent* PlayDialogue(USoundBase* Sound)
{
    if (!Sound) return nullptr;
    
    // Lazy init Mouth channel
    if (!MouthAudioComp)
    {
        MouthAudioComp = NewObject<UAudioComponent>(this);
        MouthAudioComp->RegisterComponent();
        MouthAudioComp->AttachToComponent(GetOwner()->GetRootComponent(), ...);
    }
    
    // Dialogue-specific attenuation
    USoundAttenuation* Attenuation = NewObject<USoundAttenuation>(this);
    Attenuation->Attenuation.bAttenuate = true;
    Attenuation->Attenuation.bSpatialize = true;
    Attenuation->Attenuation.SpatializationAlgorithm = SPATIALIZATION_Default;
    Attenuation->Attenuation.AttenuationShape = EAttenuationShape::Sphere;
    Attenuation->Attenuation.FalloffDistance = 3200.f;
    Attenuation->Attenuation.dBAttenuationAtMax = -36.f;
    Attenuation->Attenuation.DistanceAlgorithm = EAttenuationDistanceModel::NaturalSound;
    Attenuation->Attenuation.bEnableReverbSend = true;
    Attenuation->Attenuation.bEnableListenerFocus = true;
    
    MouthAudioComp->SetSound(Sound);
    MouthAudioComp->AttenuationSettings = Attenuation;
    MouthAudioComp->Play();
    
    return MouthAudioComp; // Caller can bind to OnAudioFinished
}
```

### Dialogue Attenuation Settings

| Parameter | Value | Notes |
|-----------|-------|-------|
| `FalloffDistance` | 3200 | Long range for dialogue |
| `dBAttenuationAtMax` | -36 dB | Still audible at max distance |
| `DistanceAlgorithm` | NaturalSound | Realistic falloff |
| `bEnableReverbSend` | true | Environment reverb |
| `bEnableListenerFocus` | true | Louder when facing speaker |

## Data Asset Integration

```cpp
void SetupSoundsFromDataAsset(APlayerCharacter* ThisCharacter, UPlayerAsset* PlayerAsset)
{
    if (!ThisCharacter || !PlayerAsset) return;
    
    // Map sounds from data asset
    ThisCharacter->GetSoundComponent()->PopupSound = PlayerAsset->CharacterSounds.CombatSounds.PopupSound;
    
    // Extend for other sounds...
}
```

Called during character initialization to populate sound references from `UPlayerAsset`.

## Lifecycle

```cpp
// Constructor: Tick enabled (can be disabled if not needed)
UCharSoundComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
}

// BeginPlay: Cache owner, init mouth channel
void BeginPlay()
{
    Character = Cast<APlayerCharacter>(GetOwner());
    InitAudioComponent(); // Creates MouthAudioComp
}

// InitAudioComponent: Lazy init for Mouth channel
void InitAudioComponent()
{
    if (!MouthAudioComp)
    {
        MouthAudioComp = NewObject<UAudioComponent>(GetOwner());
        MouthAudioComp->RegisterComponent();
        MouthAudioComp->AttachToComponent(GetOwner()->GetRootComponent(), ...);
        MouthAudioComp->bAutoActivate = false;
    }
}
```

## Quick Start

### 1. Play combat sound:

```cpp
// From AnimNotify or CombatComponent
Character->GetSoundComponent()->PlayGeneralSound(
    Character->GetSoundComponent()->BodyPunchGetHitSound,
    1.0f,  // Volume
    1.0f,  // Pitch
    1500.f // Attenuation radius
);
```

### 2. Play dialogue with completion callback:

```cpp
UAudioComponent* VoiceComp = Character->GetSoundComponent()->PlayDialogue(DialogueSound);
if (VoiceComp)
{
    VoiceComp->OnAudioFinished.AddDynamic(this, &UMyComponent::OnDialogueFinished);
}
```

### 3. Stop dialogue mid-play:

```cpp
Character->GetSoundComponent()->StopDialogue();
```

### 4. Assign sounds in Blueprint:

```
BP_PlayerCharacter → CharSoundComponent:
├── BreakBoneSound → S_BoneBreak
├── BlockHitSound → S_ShieldBlock
├── BodyPunchGetHitSound → S_PunchImpact
├── BodySlashGetHitSound → S_SlashImpact
├── LandSound → S_HardLanding
└── PopupSound → S_UIPopup
```

### 5. Play from AnimNotify:

```cpp
// In custom AnimNotify
void UAnimNotify_PlayCombatSound::Notify(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    if (APlayerCharacter* Char = Cast<APlayerCharacter>(MeshComp->GetOwner()))
    {
        if (UCharSoundComponent* SoundComp = Char->GetSoundComponent())
        {
            SoundComp->PlayGeneralSound(SoundToPlay, Volume, Pitch, AttenuationRadius);
        }
    }
}
```

### 6. Integration with Story Sequences:

```cpp
// In UCharAnimComponent::ExecuteStorySequence
if (Sequence.AudioDialogue)
{
    UAudioComponent* VoiceComp = Character->GetSoundComponent()->PlayDialogue(Sequence.AudioDialogue);
    
    if (Sequence.EndPriority == EEndPriority::Audio)
    {
        // Sequence ends when audio finishes
        VoiceComp->OnAudioFinished.AddDynamic(this, &OnSequenceAudioFinished);
    }
}
```

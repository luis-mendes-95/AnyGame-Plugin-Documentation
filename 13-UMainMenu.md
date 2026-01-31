# UMainMenu

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Widget/UI/MainMenu/MainMenu.h` | **Parent:** `UUserWidget`

## Overview

`UMainMenu` is the **main menu widget** — it provides a base for main menu UI with built-in background music playlist that loops automatically through tracks.

## Design Principles

**Music Playlist:** Cycles through `MusicList` tracks sequentially, looping back to start when complete.

**Auto-Cleanup:** Stops and releases audio component on widget destruction.

**Blueprint-Extensible:** Base class provides music; buttons and UI are added in Blueprint subclass.

## Properties

| Property | Type | Purpose |
|----------|------|---------|
| `MusicList` | `TArray<USoundBase*>` | Playlist of background music tracks |
| `ActiveAudio` | `UAudioComponent*` | Currently playing audio component (private) |
| `CurrentIndex` | `int32` | Current track index (private) |

## Key Functions

| Function | Purpose |
|----------|---------|
| `PlayTrack(int32 Index)` | Start playing track at index |
| `OnTrackFinished()` | Callback when track ends — plays next |

## Music System

### PlayTrack

```cpp
void PlayTrack(int32 Index)
{
    if (!MusicList.IsValidIndex(Index)) return;
    
    USoundBase* Track = MusicList[Index];
    if (!Track) return;
    
    // Spawn 2D sound (no spatialization)
    ActiveAudio = UGameplayStatics::SpawnSound2D(
        this,
        Track,
        1.0f,   // Volume
        1.0f,   // Pitch
        0.0f,   // Start time
        nullptr, // Concurrency
        true    // Persist across level transition
    );
    
    if (ActiveAudio)
    {
        // Bind completion callback
        ActiveAudio->OnAudioFinished.AddDynamic(this, &UMainMenu::OnTrackFinished);
    }
}
```

### OnTrackFinished

```cpp
void OnTrackFinished()
{
    // Single track: loop it
    if (MusicList.Num() <= 1)
    {
        PlayTrack(0);
        return;
    }
    
    // Multiple tracks: advance and wrap
    CurrentIndex++;
    if (CurrentIndex >= MusicList.Num())
        CurrentIndex = 0;
    
    PlayTrack(CurrentIndex);
}
```

### Playlist Flow

```
NativeConstruct
     │
     ▼
PlayTrack(0)
     │
     ├──► SpawnSound2D(Track[0])
     │
     └──► Bind OnAudioFinished
              │
              ▼ (when track ends)
         OnTrackFinished
              │
              ├──► CurrentIndex++
              │
              └──► PlayTrack(CurrentIndex)
                        │
                        └──► Loop forever
```

## Lifecycle

### NativeConstruct

```cpp
void NativeConstruct()
{
    Super::NativeConstruct();
    
    CurrentIndex = 0;
    
    // Start music if playlist exists
    if (MusicList.Num() > 0)
    {
        PlayTrack(CurrentIndex);
    }
}
```

### NativeDestruct

```cpp
void NativeDestruct()
{
    // Stop and release audio
    if (ActiveAudio)
    {
        ActiveAudio->Stop();
        ActiveAudio = nullptr;
    }
    
    Super::NativeDestruct();
}
```

## Quick Start

### 1. Create Main Menu Blueprint:

```
WBP_MainMenu (Parent: UMainMenu):
├── Canvas Panel
│   ├── Background (Image)
│   ├── Title (TextBlock)
│   ├── PlayButton (Button)
│   ├── OptionsButton (Button)
│   └── QuitButton (Button)
```

### 2. Configure music playlist:

```
WBP_MainMenu → Details:
└── MusicList:
    ├── [0] → S_MainMenu_Track1
    ├── [1] → S_MainMenu_Track2
    └── [2] → S_MainMenu_Track3
```

### 3. Add button functionality in Blueprint:

```
Event Graph:
├── PlayButton.OnClicked
│   └── Open Level (GameLevel)
├── OptionsButton.OnClicked
│   └── Show Options Widget
└── QuitButton.OnClicked
    └── Quit Game
```

### 4. Add to viewport (Level Blueprint or GameMode):

```cpp
// In Level Blueprint BeginPlay or GameMode
APlayerController* PC = UGameplayStatics::GetPlayerController(this, 0);
if (PC && MainMenuWidgetClass)
{
    UMainMenu* MainMenu = CreateWidget<UMainMenu>(PC, MainMenuWidgetClass);
    if (MainMenu)
    {
        MainMenu->AddToViewport();
        PC->SetInputMode(FInputModeUIOnly());
        PC->bShowMouseCursor = true;
    }
}
```

### 5. Extend with C++ buttons (optional):

```cpp
// In subclass header
UPROPERTY(meta = (BindWidget))
UButton* PlayButton;

UPROPERTY(meta = (BindWidget))
UButton* QuitButton;

// In NativeConstruct
void UMyMainMenu::NativeConstruct()
{
    Super::NativeConstruct();  // Starts music
    
    if (PlayButton)
        PlayButton->OnClicked.AddDynamic(this, &UMyMainMenu::OnPlayClicked);
    
    if (QuitButton)
        QuitButton->OnClicked.AddDynamic(this, &UMyMainMenu::OnQuitClicked);
}

void UMyMainMenu::OnPlayClicked()
{
    UGameplayStatics::OpenLevel(this, FName("L_Game"));
}

void UMyMainMenu::OnQuitClicked()
{
    UKismetSystemLibrary::QuitGame(GetWorld(), GetOwningPlayer(), EQuitPreference::Quit, false);
}
```

## Notes

**Sound Persistence:** `SpawnSound2D` with `bPersistAcrossLevelTransition = true` keeps music playing during level loads. Set to `false` if you want music to stop on level change.

**Single Track Loop:** If `MusicList` has only one track, it loops that track forever.

**No Crossfade:** Tracks play sequentially with no crossfade. Add custom fade logic if needed.

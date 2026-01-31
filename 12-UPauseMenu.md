# UPauseMenu

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Widget/UI/PauseMenu/PauseMenu.h` | **Parent:** `UUserWidget`

## Overview

`UPauseMenu` is the **pause menu widget** — it provides Resume, Main Menu, Restart, and Quit buttons with proper input mode switching. Starts hidden and toggles visibility on pause.

## Design Principles

**BindWidget Pattern:** Buttons use `meta = (BindWidget)` for automatic binding from Blueprint.

**Input Mode Handling:** Automatically switches between `GameOnly` and `UIOnly` input modes when showing/hiding.

**Self-Registering:** Registers itself with `UCharWidgetComponent` on construct.

**Level-Agnostic Restart:** Uses `GetCurrentLevelName()` for restart, works on any level.

## Bound Widgets

| Widget | Type | Purpose |
|--------|------|---------|
| `ResumeButton` | `UButton` | Continue playing |
| `MainMenuButton` | `UButton` | Return to main menu level |
| `RestartButton` | `UButton` | Reload current level |
| `QuitButton` | `UButton` | Exit application |

## Properties

| Property | Type | Default | Purpose |
|----------|------|---------|---------|
| `Character` | `APlayerCharacter*` | — | Cached owner reference |
| `MainMenuLevelName` | `FName` | `"MainMenu_Tenebris"` | Level to load on Main Menu click |

## Key Functions

| Function | Purpose |
|----------|---------|
| `ShowPauseMenu()` | Show widget, set UI input mode, show cursor |
| `HidePauseMenu()` | Hide widget, unpause, set game input mode |
| `OnResumeClicked()` | Resume game (calls HidePauseMenu) |
| `OnMainMenuClicked()` | Load main menu level |
| `OnRestartClicked()` | Reload current level |
| `OnQuitClicked()` | Exit application |

## Lifecycle

### NativeConstruct

```cpp
void NativeConstruct()
{
    Super::NativeConstruct();
    
    // Enable keyboard focus
    SetIsFocusable(true);
    
    // Validate buttons
    if (!ResumeButton || !MainMenuButton || !RestartButton || !QuitButton)
        return;
    
    // Bind click events
    ResumeButton->OnClicked.AddDynamic(this, &UPauseMenu::OnResumeClicked);
    MainMenuButton->OnClicked.AddDynamic(this, &UPauseMenu::OnMainMenuClicked);
    RestartButton->OnClicked.AddDynamic(this, &UPauseMenu::OnRestartClicked);
    QuitButton->OnClicked.AddDynamic(this, &UPauseMenu::OnQuitClicked);
    
    // Register with WidgetComponent
    Character = Cast<APlayerCharacter>(GetOwningPlayerPawn());
    if (Character && Character->GetWidgetComponent())
    {
        Character->GetWidgetComponent()->PauseMenuHUDWidget = this;
    }
    
    // Start hidden with game input
    if (APlayerController* PC = GetOwningPlayer())
    {
        PC->SetInputMode(FInputModeGameOnly());
        PC->bShowMouseCursor = false;
        SetVisibility(ESlateVisibility::Hidden);
    }
}
```

## Input Mode Switching

```cpp
void SetupInputMode()
{
    if (APlayerController* PC = GetOwningPlayer())
    {
        PC->SetInputMode(FInputModeUIOnly());
        PC->bShowMouseCursor = true;
    }
}
```

| State | Input Mode | Cursor |
|-------|------------|--------|
| **Menu Visible** | UIOnly | Shown |
| **Menu Hidden** | GameOnly | Hidden |

## Button Handlers

### Resume

```cpp
void OnResumeClicked()
{
    HidePauseMenu();
    
    if (APlayerController* PC = GetOwningPlayer())
    {
        PC->SetInputMode(FInputModeGameOnly());
        PC->bShowMouseCursor = false;
    }
}
```

### Main Menu

```cpp
void OnMainMenuClicked()
{
    if (GetWorld())
    {
        UGameplayStatics::OpenLevel(GetWorld(), MainMenuLevelName);
    }
}
```

### Restart

```cpp
void OnRestartClicked()
{
    FString CurrentLevelName = UGameplayStatics::GetCurrentLevelName(this);
    
    if (GetWorld())
    {
        UGameplayStatics::OpenLevel(GetWorld(), FName(*CurrentLevelName));
    }
}
```

### Quit

```cpp
void OnQuitClicked()
{
    UKismetSystemLibrary::QuitGame(
        GetWorld(),
        GetOwningPlayer(),
        EQuitPreference::Quit,
        false  // bIgnorePlatformRestrictions
    );
}
```

## Show/Hide Flow

```cpp
void ShowPauseMenu()
{
    SetVisibility(ESlateVisibility::Visible);
    SetKeyboardFocus();
    SetupInputMode();  // UI mode + cursor
}

void HidePauseMenu()
{
    SetVisibility(ESlateVisibility::Hidden);
    UGameplayStatics::SetGamePaused(GetWorld(), false);
    
    if (APlayerController* PC = GetOwningPlayer())
    {
        PC->SetInputMode(FInputModeGameOnly());
        PC->bShowMouseCursor = false;
    }
}
```

## Quick Start

### 1. Create Pause Menu Blueprint:

```
WBP_PauseMenu (Parent: UPauseMenu):
├── Canvas Panel
│   ├── Background (Image or Border)
│   ├── ResumeButton (Button)      ← MUST match name
│   ├── MainMenuButton (Button)    ← MUST match name
│   ├── RestartButton (Button)     ← MUST match name
│   └── QuitButton (Button)        ← MUST match name
```

**Critical:** Button names must match property names exactly for `BindWidget` to work.

### 2. Assign to CharWidgetComponent:

```
BP_PlayerCharacter → CharWidgetComponent:
└── PauseMenuWidgetClass → WBP_PauseMenu
```

### 3. Toggle from Input:

```cpp
// In UCharInputComponent or PlayerController
void TogglePauseMenu()
{
    if (!Character || !Character->GetWidgetComponent()) return;
    
    UPauseMenu* PauseMenu = Cast<UPauseMenu>(
        Character->GetWidgetComponent()->PauseMenuHUDWidget
    );
    
    if (!PauseMenu) return;
    
    if (PauseMenu->GetVisibility() == ESlateVisibility::Hidden)
    {
        UGameplayStatics::SetGamePaused(GetWorld(), true);
        PauseMenu->ShowPauseMenu();
    }
    else
    {
        PauseMenu->HidePauseMenu();
    }
}
```

### 4. Bind to Enhanced Input:

```cpp
// In SetupInputBindings
if (PauseAction)
{
    EIC->BindAction(PauseAction, ETriggerEvent::Started, this, &UCharInputComponent::TogglePauseMenu);
}
```

### 5. Change Main Menu level name:

```
BP_PauseMenu → Details:
└── MainMenuLevelName → "YourMainMenuLevel"
```

Or in C++:
```cpp
PauseMenu->MainMenuLevelName = FName("L_MainMenu");
```

## Integration Notes

**Pause State:** `ShowPauseMenu()` does NOT call `SetGamePaused()`. The caller must pause the game before showing the menu. `HidePauseMenu()` DOES unpause automatically.

**Widget Component Registration:** The menu registers itself at `Character->GetWidgetComponent()->PauseMenuHUDWidget` on construct, allowing other systems to access it.

**Focus:** `SetKeyboardFocus()` is called on show to ensure button navigation works immediately.

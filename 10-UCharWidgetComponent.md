# UCharWidgetComponent

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Widget/CharWidgetComponent.h` | **Parent:** `UActorComponent`

## Overview

`UCharWidgetComponent` is the **UI manager** — it owns the Player HUD, combat icons, and tutorial flags. Handles both screen-space widgets (Player HUD) and world-space widgets (combat icon above head).

## Design Principles

**Player-Only HUD:** Player HUD is only created for `IsPlayerControlled()` characters. NPCs don't get viewport widgets.

**World-Space Combat Icon:** Combat icon is a `UWidgetComponent` attached above the character's head, visible in screen space. Used for counter-attack prompts.

**Tutorial Gating:** Boolean flags track which tutorials have been shown. One-shot tutorials check these before displaying.

**Blueprint-Assignable Classes:** Widget classes (`TSubclassOf<UUserWidget>`) are assigned in Blueprint, not hardcoded.

## Widget Types

| Widget | Space | Owner | Purpose |
|--------|-------|-------|---------|
| `PlayerHUDWidget` | Screen | Player only | Health bar, ammo, crosshair, phone |
| `CombatIconWidgetComponent` | World (Screen mode) | All characters | Counter window prompt icon |
| `PauseMenuHUDWidget` | Screen | Player only | Pause menu (reserved) |
| `InfoHUDWidget` | Screen | Player only | Info overlay (reserved) |

## Properties

### Widget Classes (Blueprint-Assignable)

| Property | Type | Purpose |
|----------|------|---------|
| `PlayerHUDWidgetClass` | `TSubclassOf<UUserWidget>` | Main HUD class |
| `PauseMenuWidgetClass` | `TSubclassOf<UUserWidget>` | Pause menu class |
| `InfoHUDWidgetClass` | `TSubclassOf<UUserWidget>` | Info overlay class |
| `CombatIconWidgetClass` | `TSubclassOf<UUserWidget>` | Combat icon class |

### Widget Instances

| Property | Type | Purpose |
|----------|------|---------|
| `PlayerHUDWidget` | `UUserWidget*` | Active HUD instance |
| `PauseMenuHUDWidget` | `UUserWidget*` | Active pause menu instance |
| `InfoHUDWidget` | `UUserWidget*` | Active info overlay instance |
| `CombatIconWidgetComponent` | `UWidgetComponent*` | World-space icon component |

### Tutorial Flags

| Flag | Default | Purpose |
|------|---------|---------|
| `bShowFlashlightTutorial` | true | Show flashlight pickup tutorial |
| `bShowDoorTutorial` | true | Show door interaction tutorial |
| `bShowDoorKeyTutorial` | true | Show locked door tutorial |
| `bShowUnarmedCombatTutorial` | true | Show unarmed combat tutorial |
| `bShowMeleeCombatTutorial` | true | Show melee weapon tutorial |
| `bShowPistolTutorial` | true | Show pistol tutorial |
| `bShowNPCTutorial` | true | Show NPC interaction tutorial |

### Other

| Property | Type | Purpose |
|----------|------|---------|
| `Character` | `APlayerCharacter*` | Cached owner reference |
| `bShowPlayerHUD` | `bool` | HUD visibility flag |

## Key Functions

| Function | Purpose |
|----------|---------|
| `SetupPlayerHUD()` | Creates and adds HUD to viewport (Player only) |
| `SetupCombatIcon()` | Creates world-space combat icon component |
| `SetCombatIcon(Texture)` | Sets icon texture and visibility |

## SetupPlayerHUD

Creates the main HUD for player-controlled characters:

```cpp
void SetupPlayerHUD()
{
    if (!Character) return;
    if (!Character->IsPlayerControlled()) return;  // ← Player only
    
    APlayerController* PC = Cast<APlayerController>(Character->GetController());
    if (!PC) return;
    
    if (PlayerHUDWidgetClass && !PlayerHUDWidget)
    {
        PlayerHUDWidget = CreateWidget<UPlayerHUD>(PC, PlayerHUDWidgetClass);
        if (PlayerHUDWidget)
        {
            PlayerHUDWidget->AddToViewport();
        }
    }
}
```

**Key point:** HUD is only created when `IsPlayerControlled()` returns true. NPCs sharing the same character class won't get a HUD.

## SetupCombatIcon

Creates a world-space widget component for combat prompts:

```cpp
void SetupCombatIcon()
{
    if (!Character || !CombatIconWidgetClass) return;
    
    if (!CombatIconWidgetComponent)
    {
        // Create and attach
        CombatIconWidgetComponent = NewObject<UWidgetComponent>(Character);
        CombatIconWidgetComponent->SetupAttachment(Character->GetRootComponent());
        CombatIconWidgetComponent->RegisterComponent();
        
        // Position above head
        CombatIconWidgetComponent->SetRelativeLocation(FVector(0.f, -100.f, 50.f));
        
        // Screen space (always faces camera)
        CombatIconWidgetComponent->SetWidgetSpace(EWidgetSpace::Screen);
        CombatIconWidgetComponent->SetDrawSize(FVector2D(128.f, 128.f));
        
        // Start hidden
        CombatIconWidgetComponent->SetVisibility(false);
        
        // Create widget instance
        CombatIconWidgetComponent->SetWidget(
            CreateWidget<UUserWidget>(GetWorld(), CombatIconWidgetClass)
        );
    }
}
```

### Combat Icon Configuration

| Parameter | Value | Notes |
|-----------|-------|-------|
| `RelativeLocation` | (0, -100, 50) | Above and slightly behind head |
| `WidgetSpace` | Screen | Always faces camera |
| `DrawSize` | 128×128 | Icon pixel size |
| `Visibility` | false | Hidden by default |

## SetCombatIcon

Updates the combat icon texture and visibility:

```cpp
void SetCombatIcon(UTexture2D* NewIcon)
{
    if (!CombatIconWidgetComponent) return;
    
    UUserWidget* Widget = CombatIconWidgetComponent->GetUserWidgetObject();
    if (!Widget) return;
    
    // Find IconImage widget by name
    UImage* IconImage = Cast<UImage>(Widget->GetWidgetFromName(TEXT("IconImage")));
    if (IconImage && NewIcon)
    {
        IconImage->SetBrushFromTexture(NewIcon);
    }
    
    // Visibility controlled by texture presence
    CombatIconWidgetComponent->SetVisibility(NewIcon != nullptr);
}
```

**Widget Structure Requirement:** The `CombatIconWidgetClass` blueprint must have a `UImage` widget named `IconImage`.

## Lifecycle

```cpp
// Constructor: Tick enabled
UCharWidgetComponent()
{
    PrimaryComponentTick.bCanEverTick = true;
}

// BeginPlay: Setup all widgets
void BeginPlay()
{
    Character = Cast<APlayerCharacter>(GetOwner());
    SetupPlayerHUD();    // Player only
    SetupCombatIcon();   // All characters
}
```

## Quick Start

### 1. Create HUD widget (Blueprint):

```
WBP_PlayerHUD (Parent: UPlayerHUD):
├── Canvas Panel
│   ├── HealthBar (ProgressBar)
│   ├── AmmoCounter (TextBlock)
│   └── Crosshair (Image)
```

### 2. Create combat icon widget (Blueprint):

```
WBP_CombatIcon:
├── Canvas Panel
│   └── IconImage (Image)  ← MUST be named "IconImage"
```

### 3. Assign widget classes:

```
BP_PlayerCharacter → CharWidgetComponent:
├── PlayerHUDWidgetClass → WBP_PlayerHUD
└── CombatIconWidgetClass → WBP_CombatIcon
```

### 4. Show counter-attack prompt (from CombatComponent):

```cpp
// When enemy opens counter window
if (UCharWidgetComponent* WidgetComp = Target->GetWidgetComponent())
{
    WidgetComp->SetCombatIcon(CounterPromptTexture);
}

// When window closes
WidgetComp->SetCombatIcon(nullptr);  // Hides icon
```

### 5. Update HUD health:

```cpp
// From CombatComponent::ApplyHit
if (Character->GetWidgetComponent() && Character->GetWidgetComponent()->PlayerHUDWidget)
{
    UPlayerHUD* HUD = Cast<UPlayerHUD>(Character->GetWidgetComponent()->PlayerHUDWidget);
    if (HUD)
    {
        HUD->UpdateHealth(GetHealth() / GetMaxHealth());
    }
}
```

### 6. One-shot tutorial pattern:

```cpp
// In some interaction code
if (UCharWidgetComponent* WidgetComp = Character->GetWidgetComponent())
{
    if (WidgetComp->bShowFlashlightTutorial)
    {
        ShowTutorialPopup("Press F to toggle flashlight");
        WidgetComp->bShowFlashlightTutorial = false;  // Don't show again
    }
}
```

### 7. Check if HUD exists before updating:

```cpp
void UpdateAmmoDisplay(int32 CurrentAmmo, int32 MaxAmmo)
{
    if (!Character->GetWidgetComponent()) return;
    if (!Character->GetWidgetComponent()->PlayerHUDWidget) return;
    
    UPlayerHUD* HUD = Cast<UPlayerHUD>(Character->GetWidgetComponent()->PlayerHUDWidget);
    if (HUD)
    {
        HUD->UpdateAmmo(CurrentAmmo, MaxAmmo);
    }
}
```

## Integration with CombatComponent

Combat icon visibility is managed by `UCharCombatComponent`:

```cpp
// In CombatComponent
void UpdateCombatIconVisibility()
{
    if (!Character) return;
    
    UCharWidgetComponent* WidgetComp = Character->GetWidgetComponent();
    if (!WidgetComp) return;
    
    const bool bHasThreat = CurrentAttackTarget != nullptr || CurrentAttackTargets.Num() > 0;
    const bool bShouldShow = !bIsDead && !bIsGettingHit && bHasThreat;
    
    // Always hide when conditions not met
    if (!bShouldShow)
    {
        WidgetComp->SetCombatIcon(nullptr);
    }
}
```

The icon is shown when:
1. Enemy opens counter window (`SetCanBeDeffended(true)`)
2. Combat notify calls `SetCombatIcon(CounterTexture)`

And hidden when:
1. Counter window closes
2. Character is hit
3. Character dies
4. No combat targets

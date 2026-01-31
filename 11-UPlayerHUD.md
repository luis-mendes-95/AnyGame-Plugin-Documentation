# UPlayerHUD

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Widget/UI/PlayerHUD/PlayerHUD.h` | **Parent:** `UUserWidget`

## Overview

`UPlayerHUD` is the **main viewport widget** — it displays health, defense, ammo, minimap, subtitles, hints, mobile phone overlay, and personality stats. Created by `UCharWidgetComponent` for player-controlled characters only.

## Design Principles

**BindWidget Pattern:** All UI elements use `meta = (BindWidget)` for automatic binding from Blueprint. No manual widget creation in C++.

**Fade System:** Multiple independent fade timers for hints, subtitles, minimap, and mobile phone. Each fades in/out smoothly.

**Tick-Driven Updates:** Minimap rotation, fades, and time display update every frame via `NativeTick`.

**Modular Sections:** HUD is organized into collapsible containers (Needs, Mood, Traits, Job) that can be shown/hidden independently.

## Widget Sections

| Section | Container | Purpose |
|---------|-----------|---------|
| **Main** | `MainHUDContainer` | Root container for basic HUD |
| **Stats** | `TopStatsBox`, `StatsVerticalBox` | Health, Defense bars |
| **Weapon** | — | Weapon icon, ammo counts |
| **Money** | `MoneyBox` | Currency display |
| **Needs** | `NeedsContainer` | Hunger, Energy, Physical bars |
| **Mood** | `MoodContainer` | Happiness, Stress bars |
| **Traits** | `TraitsContainer` | Ambition, Morality bars |
| **Job** | `JobContainer` | Job profile, salary, schedule |
| **Minimap** | — | Rotating minimap with mission waypoint |
| **Mobile Phone** | — | Video call overlay |
| **Hints** | `HintBackground` | Tutorial/hint popups |
| **Subtitles** | `SubtitlesBackground` | Dialogue subtitles |

## Bound Widgets

### Stats & Combat

| Widget | Type | Purpose |
|--------|------|---------|
| `HealthBar` | `UProgressBar` | Player health (0-1) |
| `DefenseBar` | `UProgressBar` | Player defense (0-1) |
| `WeaponIcon` | `UImage` | Current weapon texture |
| `CurrentAmmoText` | `UTextBlock` | Magazine ammo count |
| `AmmunitionText` | `UTextBlock` | Reserve ammo count |
| `TimeText` | `UTextBlock` | In-game time display |
| `MoneyText` | `UTextBlock` | Currency amount |

### Personality (Needs/Mood/Traits)

| Widget | Type | Purpose |
|--------|------|---------|
| `HungerBar` | `UProgressBar` | Hunger level |
| `EnergyBar` | `UProgressBar` | Energy level |
| `PhysicalConditionBar` | `UProgressBar` | Physical condition |
| `HappinessBar` | `UProgressBar` | Happiness level |
| `StressBar` | `UProgressBar` | Stress level (inverted) |
| `AmbitionBar` | `UProgressBar` | Ambition trait |
| `MoralityBar` | `UProgressBar` | Morality trait |

### Job Profile

| Widget | Type | Purpose |
|--------|------|---------|
| `JobNameText` | `UTextBlock` | Job title |
| `JobIcon` | `UImage` | Job icon |
| `JobLevelText` | `UTextBlock` | Current/Max level |
| `RespectText` | `UTextBlock` | Respect value |
| `BaseSalaryText` | `UTextBlock` | Salary per day |
| `WorkScheduleText` | `UTextBlock` | Work hours |
| `JobSatisfactionBar` | `UProgressBar` | Job satisfaction |

### Minimap

| Widget | Type | Purpose |
|--------|------|---------|
| `MinimapImage` | `UImage` | SceneCapture render target |
| `PlayerArrow` | `UImage` | Player direction indicator |
| `MissionIcon` | `UImage` | Current mission waypoint |

### Mobile Phone

| Widget | Type | Purpose |
|--------|------|---------|
| `MobilePhoneImage` | `UImage` | Phone frame |
| `VideoCallImage` | `UImage` | Video call content |

### Hints & Subtitles

| Widget | Type | Purpose |
|--------|------|---------|
| `HintBackground` | `UBorder` | Hint container |
| `HintText` | `UTextBlock` | Hint message |
| `SubtitlesBackground` | `UBorder` | Subtitle container |
| `SubtitlesText` | `UTextBlock` | Subtitle text |

## Fade Parameters

| System | TimeRemain | CurrentOpacity | FadeSpeed |
|--------|------------|----------------|-----------|
| **Main HUD** | `TimeRemain` | `CurrentOpacity` | `FadeSpeed` (3.0) |
| **Hints** | `HintTimeRemain` | `HintCurrentOpacity` | `HintFadeSpeed` (3.0) |
| **Subtitles** | `SubtitlesTimeRemain` | `SubtitlesCurrentOpacity` | `SubtitlesFadeSpeed` (12.0) |
| **Minimap** | — | `MinimapCurrentOpacity` | `MinimapFadeSpeed` (5.0) |
| **Mobile Phone** | — | `MobilePhoneCurrentOpacity` | `MobilePhoneFadeSpeed` (5.0) |

## Key Functions

### Stats Updates

| Function | Purpose |
|----------|---------|
| `UpdateHealth(float)` | Set health bar (0-1) |
| `UpdateDefense(float)` | Set defense bar with interp |
| `UpdateCurrentAmmo(int32)` | Set magazine ammo text |
| `UpdateAmmunition(int32)` | Set reserve ammo text |
| `UpdateWeaponIcon(UTexture2D*)` | Set weapon image |
| `UpdateMoney(int32)` | Set currency text |
| `UpdateTimeDisplay(float)` | Format and set time (HH:MM) |

### Personality Updates

| Function | Purpose |
|----------|---------|
| `UpdatePersonalitySummaryHUD()` | Refresh all personality panels |
| `UpdateNeedsSummary(Hunger, Energy, Physical)` | Set needs bars |
| `UpdateMoodSummary(Happiness, Stress)` | Set mood bars |
| `UpdateTraitsSummary(Ambition, Morality)` | Set traits bars |
| `UpdateJobProfileHUD()` | Refresh job panel |

### Visibility Control

| Function | Purpose |
|----------|---------|
| `ShowPlayerStats(bool)` | Toggle health/defense visibility |
| `ShowWidget(float)` | Show HUD with optional timer |
| `HideHUD()` | Collapse all HUD elements |
| `ShowHint(Message, Time)` | Display hint popup |
| `HideHint()` | Hide hint immediately |
| `ShowSubtitles(Message, Time)` | Display subtitle |
| `HideSubtitles()` | Hide subtitle immediately |
| `ShowMinimap()` | Fade in minimap |
| `HideMinimap()` | Fade out minimap |
| `ShowMobilePhone()` | Fade in phone overlay |
| `HideMobilePhone()` | Fade out phone overlay |

## Lifecycle

### NativeConstruct

```cpp
void NativeConstruct()
{
    // Initialize bars to full
    if (HealthBar) HealthBar->SetPercent(1.0f);
    if (DefenseBar) DefenseBar->SetPercent(1.0f);
    
    // Link to PlayerCharacter
    Character = Cast<APlayerCharacter>(GetOwningPlayerPawn());
    if (Character && Character->GetWidgetComponent())
    {
        Character->GetWidgetComponent()->PlayerHUDWidget = this;
    }
    
    // Initialize fade state
    TimeRemain = DefaultVisibleTime;
    CurrentOpacity = 1.f;
    
    // Set labels
    if (NeedsLabel) NeedsLabel->SetText(FText::FromString("NEEDS"));
    if (MoodLabel) MoodLabel->SetText(FText::FromString("MOOD"));
    if (TraitsLabel) TraitsLabel->SetText(FText::FromString("TRAITS"));
    
    // Start hidden
    HideHUD();
    HideMobilePhone();
    HideMinimap();
    ShowPlayerStats(true);
}
```

### NativeTick

```cpp
void NativeTick(const FGeometry& MyGeometry, float InDeltaTime)
{
    // 1. Hint fade
    if (HintTimeRemain > 0.f)
    {
        HintTimeRemain -= InDeltaTime;
        HintCurrentOpacity = FInterpTo(HintCurrentOpacity, 1.f, InDeltaTime, HintFadeSpeed);
    }
    else
    {
        HintCurrentOpacity = FInterpTo(HintCurrentOpacity, 0.f, InDeltaTime, HintFadeSpeed);
        if (HintCurrentOpacity < 0.01f)
            HintBackground->SetVisibility(Collapsed);
    }
    
    // 2. Subtitles fade
    // ... similar pattern
    
    // 3. Minimap update (position + rotation)
    if (Character)
    {
        ASceneCapture2D* SceneCapture = GetActorOfClass(ASceneCapture2D);
        SceneCapture->SetActorLocation(Character->GetActorLocation());
        SceneCapture->SetActorRotation(FRotator(-90, Character->GetControlRotation().Yaw, 0));
        SceneCapture->GetCaptureComponent2D()->CaptureScene();
    }
    
    // 4. Mission waypoint positioning
    // ... calculate relative position, rotate by player yaw
    
    // 5. Minimap fade
    // ... similar pattern
    
    // 6. Mobile phone fade
    // ... similar pattern
}
```

## Fade Pattern

All fade systems follow the same pattern:

```cpp
// Show (set flag + visibility)
void ShowX()
{
    bXVisible = true;
    XWidget->SetVisibility(ESlateVisibility::Visible);
}

// Hide (set flag only, tick handles fade)
void HideX()
{
    bXVisible = false;
}

// Tick (interpolate opacity)
void NativeTick(...)
{
    float TargetOpacity = bXVisible ? 1.f : 0.f;
    XCurrentOpacity = FInterpTo(XCurrentOpacity, TargetOpacity, DeltaTime, XFadeSpeed);
    
    XWidget->SetRenderOpacity(XCurrentOpacity);
    
    // Collapse when fully faded out
    if (!bXVisible && XCurrentOpacity <= 0.01f)
    {
        XWidget->SetVisibility(ESlateVisibility::Collapsed);
        XCurrentOpacity = 0.f;
    }
}
```

## Minimap Waypoint Math

```cpp
// 1. Get direction to destination
FVector ToDest = Destination - PlayerLocation;
float Distance = ToDest.Size();

// 2. Normalize to minimap scale
float Scale = FMath::Clamp(Distance / MaxDistanceForWaypoint, 0.f, 1.f);
FVector2D ScaledPos = ToDest2D.GetSafeNormal() * Scale * MinimapRadius;

// 3. Rotate by player yaw (so waypoint is relative to facing)
float Rad = FMath::DegreesToRadians(PlayerYaw + 90.f);
FVector2D RotatedPos;
RotatedPos.X = ScaledPos.X * Cos(Rad) + ScaledPos.Y * Sin(Rad);
RotatedPos.Y = -ScaledPos.X * Sin(Rad) + ScaledPos.Y * Cos(Rad);

// 4. Apply to widget
MissionIcon->SetRenderTranslation(RotatedPos);
```

## Quick Start

### 1. Create HUD Blueprint:

```
WBP_PlayerHUD (Parent: UPlayerHUD):
├── Canvas Panel
│   ├── MainHUDContainer (Border)
│   │   ├── TopStatsBox (HorizontalBox)
│   │   │   ├── HealthBar (ProgressBar)
│   │   │   └── DefenseBar (ProgressBar)
│   │   └── ...
│   ├── HintBackground (Border)
│   │   └── HintText (TextBlock)
│   ├── SubtitlesBackground (Border)
│   │   └── SubtitlesText (TextBlock)
│   ├── MinimapImage (Image)
│   ├── PlayerArrow (Image)
│   ├── MissionIcon (Image)
│   └── ...
```

**Critical:** Widget names must match property names exactly for `BindWidget` to work.

### 2. Update health from CombatComponent:

```cpp
void UCharCombatComponent::ApplyHit(const FHitEffectInfo& HitInfo)
{
    Health -= HitInfo.ImpactAmount;
    
    if (Character->GetWidgetComponent() && Character->GetWidgetComponent()->PlayerHUDWidget)
    {
        UPlayerHUD* HUD = Cast<UPlayerHUD>(Character->GetWidgetComponent()->PlayerHUDWidget);
        if (HUD)
        {
            HUD->UpdateHealth(Health / MaxHealth);
        }
    }
}
```

### 3. Show hint with sound:

```cpp
void ShowHint(const FString& HintMessage, float DisplayTime)
{
    // Play popup sound
    if (Character->GetSoundComponent()->PopupSound)
    {
        UGameplayStatics::PlaySound2D(GetWorld(), Character->GetSoundComponent()->PopupSound);
    }
    
    HintText->SetText(FText::FromString(HintMessage));
    HintBackground->SetVisibility(ESlateVisibility::Visible);
    HintTimeRemain = DisplayTime;
    HintCurrentOpacity = 1.f;
}
```

### 4. Show subtitles from Story Sequence:

```cpp
// In UCharAnimComponent::ExecuteStorySequence
if (!Sequence.Subtitles.IsEmpty())
{
    UPlayerHUD* HUD = Cast<UPlayerHUD>(Character->GetWidgetComponent()->PlayerHUDWidget);
    if (HUD)
    {
        HUD->ShowSubtitles(Sequence.Subtitles, Sequence.SubtitlesDuration);
    }
}
```

### 5. Toggle minimap:

```cpp
// Show on key press
if (UPlayerHUD* HUD = GetPlayerHUD())
{
    HUD->ShowMinimap();
}

// Hide on key release
if (UPlayerHUD* HUD = GetPlayerHUD())
{
    HUD->HideMinimap();
}
```

### 6. Show mobile phone (attachment):

```cpp
// When phone is attached
AttachmentsComp->AttachActor(Phone, ECharAttachmentSlot::RightHand, TEXT("hand_r_socket"));

if (UPlayerHUD* HUD = GetPlayerHUD())
{
    HUD->ShowMobilePhone();
}

// On detach (handled automatically in DetachSlot)
HUD->HideMobilePhone();
```

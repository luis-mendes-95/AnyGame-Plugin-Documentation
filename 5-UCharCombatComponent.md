# UCharCombatComponent

> **Module:** AnyGame Core | **Header:** `PlayerCharacter/Components/Combat/CharCombatComponent.h` | **Parent:** `UActorComponent`

## Overview

`UCharCombatComponent` is the **combat state machine** — it manages attack flow, hit reactions, counters, finishers, combos, death, and target acquisition. Works identically for Player and AI via the same API.

## Design Principles

**Single Gate:** One flag (`bCanAttack`) controls whether attacks can start. No scattered checks.

**Montage-Driven:** Combat actions are montages. Notifies trigger state transitions (`ResetAttack`, `EndAttack`, `ApplyHitEffect`).

**Tag Matrix:** Factions use Actor Tags (`Player`, `Friend`, `Enemy`). No data assets or interfaces needed.

**Attacker Slot:** Only one attacker can hit a target at a time. Others must orbit until the slot opens.

**LOS Gate (NPC):** AI cannot attack, approach, or orbit targets without real line-of-sight. Prevents "telepathic" combat.

## State Flags

| Flag | Type | Purpose |
|------|------|---------|
| `bCanAttack` | `bool` | **THE gate** — can we start an attack? |
| `bIsAttacking` | `bool` | Currently in attack montage |
| `bIsGettingHit` | `bool` | Currently in hit reaction |
| `bIsCounterAttack` | `bool` | Currently executing counter |
| `bIsCurrentAttackFinisher` | `bool` | Current attack is a finisher |
| `bIsDead` | `bool` | Character is dead |
| `bCanBeDeffended` | `bool` | Attack can be countered (set by notify) |
| `bCanDeffend` | `bool` | Character can counter incoming attacks |
| `bIsAiming` | `bool` | Ranged attack mode |

## Stats

| Property | Default | Purpose |
|----------|---------|---------|
| `Health` | 100 | Current health |
| `MaxHealth` | 100 | Maximum health |
| `Deffense` | 100 | Defense value (reserved) |
| `NearAttackThreshold` | 95 | Max distance for near attacks |
| `DistantAttackThreshold` | 120 | Max distance for far attacks |

## Tag Matrix (Factions)

```cpp
// Who can attack whom (enforced in CanAttackByTagMatrix)
Player → Enemy ✓
Friend → Enemy ✓
Enemy  → Player ✓
Enemy  → Friend ✓
Enemy  → Enemy ✗
Friend → Player ✗
Player → Friend ✗
```

Applied automatically in `UpdateBestAttackTarget()` and `HandleAttack()`.

---

## Attack Flow

### Main Sequence

```
Input/AI calls HandleAttack()
├── Gate checks (bCanAttack, bIsGettingHit, target valid, LOS, TagMatrix)
├── InitializeAttack()
│   ├── bIsAttacking = true
│   └── bCanAttack = false  ← LOCKED
├── CalculateDirectionAndCombo()
│   ├── CalcDirToTarget() → Front/Back/Left/Right
│   ├── Select collection: "CombatFront", "NearCombatLeft", etc.
│   └── Increment ComboCounter → "1", "2", "3"...
├── TryHandleFinisher() (if target health ≤ 30)
│   └── Play "FinishAttacks" montage
├── HandleNormalAttack()
│   └── Play directional combo montage
└── CacheActiveAttackMontage() ← For notify validation
```

### Montage Flow

```
Montage plays...
├── [Notify] ResetAttack
│   └── bCanAttack = true  ← UNLOCKED (can queue next attack)
├── [Notify] ApplyHitEffect
│   └── Damage + VFX on target
├── [Notify] PlayGetHitMontage (HandleHitTarget)
│   └── Target plays hit reaction (or counter)
└── [Notify] EndAttack
    ├── bCanAttack = true
    ├── bIsAttacking = false
    ├── ComboCounter = 0
    └── Clear MotionWarp
```

### Gate Logic

```cpp
bool CanAttack() const
{
    if (!bCanAttack) return false;           // Locked by active attack
    if (bIsGettingHit) return false;         // In hit stun
    if (IsDead()) return false;              // Dead
    if (!CurrentAttackTarget) return false;  // No target
    
    // Range check
    float MaxDist = bIsAiming ? DistantAttackThreshold : NearAttackThreshold;
    return CurrentEnemyTargetDistance <= MaxDist;
}
```

---

## Hit System

### HandleHitTarget (Called by Notify)

```
HandleHitTarget()
├── Validate notify (IsNotifyValidForCurrentAttack)
├── Check target not countering (bIsCounterAttack → invulnerable)
├── Check target not in finisher (bIsCurrentAttackFinisher → invulnerable)
├── Register attacker relationships
├── IF target.bCanDeffend == true:
│   └── COUNTER ATTACK BRANCH
│       ├── Target plays "CounterAttack" montage
│       ├── Attacker gets interrupted (bCanAttack = true, abort)
│       └── Target becomes invulnerable during counter
├── ELSE: NORMAL HIT BRANCH
│   ├── Target.bIsGettingHit = true
│   ├── IF finisher: Play "FinishesGetHit" montage
│   └── ELSE: Play "GetHit" montage (combo index)
```

### Counter Attack Flow

```
Attacker attacks...
├── Target has bCanDeffend = true (set by player input or AI)
├── HandleHitTarget detects counter opportunity
├── Target:
│   ├── bIsCounterAttack = true (invulnerable)
│   ├── bCanDeffend = false (no double-counter)
│   ├── Plays "CounterAttack" montage
│   └── CacheActiveAttackMontage()
├── Attacker:
│   ├── bIsAttacking = false
│   ├── bCanAttack = true (unlocked)
│   └── Attack aborted
└── Counter montage hits → HandleCounterGetHit()
    └── Original attacker plays "CounterGetHit" montage
```

### ApplyHit (Damage + Effects)

```cpp
void ApplyHit(const FHitEffectInfo& HitInfo)
{
    // Skip if executing finisher (invulnerable)
    if (bIsCurrentAttackFinisher) return;
    
    // Apply damage
    Health -= HitInfo.ImpactAmount;
    UpdateHUD();
    
    // Spawn VFX at socket
    SpawnNiagara(HitInfo.ImpactVFX, SocketLocation);
    
    // Play sound
    PlaySound(HitInfo.ImpactSound);
    
    // Camera shake
    PlayCameraShake(HitInfo.ImpactCameraShake);
}
```

---

## Combo System

### Direction Calculation

```cpp
void CalcDirToTarget()
{
    FVector ToTarget = (Target - Character).GetSafeNormal();
    FVector Forward = Character->GetActorForwardVector();
    
    float Yaw = (ToTarget.Rotation() - Forward.Rotation()).Yaw;
    Yaw = NormalizeAxis(Yaw);
    
    // -45 to 45 = Front, 45 to 135 = Right, etc.
}
```

### Collection Selection

| Range | Direction | Collection Name |
|-------|-----------|-----------------|
| Near (<95) | Front | `NearCombatFront` |
| Near (<95) | Right | `NearCombatRight` |
| Default | Front | `CombatFront` |
| Default | Back | `CombatBack` |

### Combo Counter

```cpp
// Increments each attack, resets on EndAttack or max reached
ComboCounter++;
if (ComboCounter > MaxCombos) ComboCounter = 1;

// Montage name is just the number
FName MontageName = FName(*FString::FromInt(ComboCounter)); // "1", "2", "3"
```

---

## Finisher System

Triggers when target health ≤ 30:

```cpp
bool TryHandleFinisher(UCharAnimComponent* AnimComp)
{
    if (CurrentAttackTarget->GetCombatComponent()->GetHealth() <= 30.f)
    {
        bIsCurrentAttackFinisher = true;
        AttackFinisherCounter++;
        
        // Play from "FinishAttacks" collection
        AnimComp->PlayMontage("FinishAttacks", FString::FromInt(AttackFinisherCounter));
        return true;
    }
    return false;
}
```

**Invulnerability:** During finisher execution, both attacker (`bIsCurrentAttackFinisher`) and target are protected from damage.

---

## Attacker Slot System

Only one character can actively attack a target at a time:

```cpp
// On target's CombatComponent
AActor* CurrentAttacker = nullptr;

bool CanBeAttackedBy(AActor* Attacker) const 
{ 
    return CurrentAttacker == nullptr || CurrentAttacker == Attacker; 
}

void RegisterAttacker(AActor* Attacker) { CurrentAttacker = Attacker; }
void UnregisterAttacker(AActor* Attacker) 
{ 
    if (CurrentAttacker == Attacker) CurrentAttacker = nullptr; 
}
```

**AI Behavior:** When slot is occupied, AI enters `Orbit` state (circles target) until slot opens.

---

## LOS Gate (NPC Only)

Prevents AI from attacking through walls:

```cpp
// Constants
COMBAT_LOS_CHECK_INTERVAL = 0.10s   // Cache refresh rate
COMBAT_LOS_LOSE_DELAY = 0.25s       // Hysteresis before losing target
COMBAT_LOS_GAIN_DELAY = 0.15s       // Hysteresis before regaining target
COMBAT_LOS_SWEEP_RADIUS = 18.f      // Sphere sweep (not line trace)
COMBAT_LOS_EYE_HEIGHT = 60.f        // Eye offset
COMBAT_LOS_MAX_DISTANCE = 2500.f    // Max check distance
```

### Check Flow

```cpp
void UpdateTargetLOSGate(float DeltaTime)
{
    // Player always has LOS (no gate)
    if (Character->IsPlayerControlled()) return;
    
    // Throttled check
    if (Accumulator < CHECK_INTERVAL) return;
    
    bool bRawHasLOS = HasLineOfSightToTarget_SphereSweep(Target);
    
    // Hysteresis (prevents flicker)
    if (bRawHasLOS)
    {
        HasLOSTime += CHECK_INTERVAL;
        if (!bCachedHasLOS && HasLOSTime >= GAIN_DELAY)
            bCachedHasLOS = true;
    }
    else
    {
        NoLOSTime += CHECK_INTERVAL;
        if (bCachedHasLOS && NoLOSTime >= LOSE_DELAY)
            bCachedHasLOS = false;
    }
}
```

### Lost LOS Handling

```cpp
void HandleLostTargetDueToLOS()
{
    // Hard stop: AI cannot attack "telepathically"
    CurrentAttackTarget = nullptr;
    CurrentAttackTargets.Empty();
    
    // Delayed unregister (2-4s) to avoid slot flicker
    ScheduleDelayedUnregisterAttacker(1.f, 2.f);
}
```

---

## Death System

### CheckDie Flow

```cpp
void CheckDie()
{
    if (Health > 0 || bIsDead) return;
    
    bIsDead = true;
    
    // Cleanup
    UpdateCombatIconVisibility();
    ScheduleDelayedUnregisterAttacker();
    
    // Stop character
    Controller->SetIgnoreMoveInput(true);
    Movement->DisableMovement();
    Capsule->SetCollisionEnabled(NoCollision);
    
    // Schedule ragdoll
    SetTimerForNextTick(FinalizeDeath);
}
```

### FinalizeDeath (Ragdoll)

```cpp
void FinalizeDeath()
{
    Mesh->SetCollisionProfileName("Ragdoll");
    Mesh->SetSimulatePhysics(true);
    
    // Apply impulse based on DeathImpulseType
    FVector ImpulseDir = GetImpulseDirection(DeathImpulseType);
    Mesh->AddImpulseToAllBodiesBelow(ImpulseDir * DeathImpulseStrength, "pelvis");
}
```

| DeathImpulseType | Direction |
|------------------|-----------|
| `Forward` | Character forward |
| `Backward` | Character backward |
| `Left` | Character left |
| `Right` | Character right |
| `Down` | World down |

---

## Montage Safety System

Prevents stale notifies from corrupting state:

```cpp
// Cache the montage when attack starts
void CacheActiveAttackMontage()
{
    ActiveAttackMontage = AnimInstance->GetCurrentActiveMontage();
}

// Notifies check if they're from the current attack
bool IsNotifyValidForCurrentAttack() const
{
    if (!bIsAttacking) return false;
    if (!ActiveAttackMontage) return true; // Permissive fallback
    
    // If cached montage isn't playing, notify is stale
    return AnimInstance->Montage_IsPlaying(ActiveAttackMontage);
}

// Fail-safe: if montage ends without EndAttack notify
void OnOwnerMontageEnded(UAnimMontage* Montage, bool bInterrupted)
{
    if (Montage == ActiveAttackMontage && (!bCanAttack || bIsAttacking))
    {
        EndAttack_Internal(true); // Force unlock
    }
}
```

---

## Target Acquisition

### UpdateBestAttackTarget (Tick)

```cpp
void UpdateBestAttackTarget(float DeltaTime)
{
    // Lock target during attack
    if (bIsAttacking) return;
    
    // Find all PlayerCharacters in range (2000 units)
    for (APlayerCharacter* Other : AllCharacters)
    {
        // Skip self, dead, out of range
        // Skip if TagMatrix rejects
        if (!CanAttackByTagMatrix(Character, Other)) continue;
        
        // NPC: Also check LOS
        if (!IsPlayerControlled() && !HasLineOfSight(Other)) continue;
        
        // Track closest valid target
        if (Distance < BestDistance)
        {
            CurrentAttackTarget = Other;
        }
        
        CurrentAttackTargets.Add(Other); // All valid targets for AI
    }
}
```

---

## Motion Warping

```cpp
void ApplyCombatWarp(FName WarpName, const FAnimMotionParams& Params)
{
    // Calculate final position (offset from target)
    FVector ToTarget = (TargetLoc - MyLoc).GetSafeNormal();
    FVector FinalLoc = TargetLoc - (ToTarget * Params.TargetOffset.X);
    FRotator FinalRot = (TargetLoc - FinalLoc).Rotation();
    
    MotionWarping->AddOrUpdateWarpTargetFromTransform(WarpName, FTransform(FinalRot, FinalLoc));
}
```

Called by `ApplyAnimMotionParams` notify with `WarpTargetName`.

---

## Key Functions

| Function | Category | Purpose |
|----------|----------|---------|
| `HandleAttack()` | Attack | Main attack entry point |
| `ResetAttack()` | Attack | Called by notify — unlocks gate |
| `EndAttack()` | Attack | Called by notify — full cleanup |
| `HandleHitTarget()` | Hit | Triggers hit reaction or counter |
| `HandleCounterGetHit()` | Counter | Counter victim plays reaction |
| `ApplyHit(FHitEffectInfo)` | Damage | Apply damage + VFX |
| `CheckDie()` | Death | Evaluate death condition |
| `SetAiming(bool)` | State | Toggle ranged mode |
| `RegisterAttacker(AActor*)` | Slot | Claim attacker slot |
| `UnregisterAttacker(AActor*)` | Slot | Release attacker slot |

---

## Quick Start

### 1. Basic attack (Player):
```cpp
// Input calls this
if (CombatComp->CanAttack())
{
    CombatComp->HandleAttack();
}
```

### 2. Setup counter window (Notify):
```cpp
// In attack montage
[0.0s] OpenCounterWindow → CombatComp->SetCanBeDeffended(true)
[0.3s] PlayGetHitMontage → HandleHitTarget()
[0.5s] CloseCounterWindow → CombatComp->SetCanBeDeffended(false)
```

### 3. Player blocks (Input):
```cpp
// On block input
CombatComp->SetCanDeffend(true);

// On block release
CombatComp->SetCanDeffend(false);
```

### 4. AI target acquisition:
```cpp
// Automatic via Tick
// CurrentAttackTarget is always the best valid target
// CurrentAttackTargets contains all valid targets in range
```

### 5. Check combat state:
```cpp
if (CombatComp->IsDead()) { /* Handle death */ }
if (CombatComp->GetIsAttacking()) { /* In attack */ }
if (CombatComp->GetIsGettingHit()) { /* In hit stun */ }
```

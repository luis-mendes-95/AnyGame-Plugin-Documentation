# 🎮 AnyGame Plugin

**Opinionated Gameplay Framework for Unreal Engine 5.7+**

> *"All gameplay lives inside the Character. No exceptions."*

[![Unreal Engine](https://img.shields.io/badge/Unreal%20Engine-5.7+-blue?logo=unrealengine)](https://www.unrealengine.com/)
[![Architecture](https://img.shields.io/badge/Architecture-Player--Centric-green)]()
[![License](https://img.shields.io/badge/License-Premium-orange)]()

---

## 🧠 What is AnyGame?

AnyGame is a **production-ready gameplay framework** that enforces clean architecture through **composition over inheritance**. 

Instead of creating dozens of character subclasses, you build gameplay by combining **Actor Components** on a single `APlayerCharacter` class. Players and AI share the same systems — no special cases, no spaghetti.

```cpp
// Same class, same components, same logic
APlayerCharacter* Player = SpawnCharacter();  // Possess → Player
APlayerCharacter* NPC = SpawnCharacter();     // Don't possess → AI
NPC->Tags.Add("Enemy");                       // Tag defines faction
```

---

## ⚡ Core Philosophy

| Principle | Description |
|-----------|-------------|
| **Player-Centric** | All gameplay lives inside the Character via Actor Components |
| **World Reacts** | World Actors only respond to character actions — never initiate |
| **Player/AI Parity** | Same code path for players and NPCs. Zero duplication. |
| **Anti-Lock-In** | Modular by design. Remove any system without breaking others. |

---

## 🔥 Features

### Combat System
- ⚔️ **Combo System** — Direction-aware attacks (Front/Back/Left/Right)
- 🛡️ **Counter-Attack Windows** — Responsive defensive mechanics
- 💀 **Finisher System** — Cinematic kills when target health is low
- 🎯 **Attacker Slot** — One attacker at a time, others orbit
- 👁️ **LOS Gate** — AI can't attack through walls

### AI System
- 🤝 **Companion AI** — Friends follow and fight alongside you
- 🔄 **Orbit Combat** — NPCs dynamically circle targets (no clumping)
- 🛤️ **Trail Spline Follow** — NPCs follow your exact path when LOS breaks
- 🚶 **Auto-Traversal** — AI handles doors, obstacles automatically
- 🏷️ **Tag Matrix** — Factions via Actor Tags (Player/Friend/Enemy)

### Interaction System
- 🚪 **IInteractable Interface** — Zero coupling to specific actors
- 🪑 **Motion Warping** — Sit, open doors with precise alignment
- 🔒 **State Machines** — Doors with Open/Closed/Locked states
- 📡 **Event Chain** — Delegates + BP hooks for extensibility

### Narrative System
- 🎬 **Story Triggers** — Multi-step cinematic sequences
- 👥 **Multi-Participant** — Coordinate multiple characters
- 🎭 **Handler System** — Control AI, Career, Attachments per step
- 🔊 **World Manager** — Sounds, spawns, destroys per sequence

### Procedural City
- 🏙️ **Spline-Driven** — Draw a path, generate a city
- 🔀 **Auto-Crossings** — Overlapping roads become intersections
- 📦 **HISM Optimization** — Thousands of instances, minimal draw calls
- 🎨 **Spawn Zones** — Populate with props, actors, lights

### UI System
- 📊 **Player HUD** — Health, minimap, subtitles, hints with fade
- ⏸️ **Pause Menu** — Input mode handling built-in
- 🎵 **Main Menu** — Music playlist with auto-loop

---

## 🏗️ Architecture

```
APlayerCharacter (final)
├── UCharInputComponent        // Enhanced Input routing
├── UCharAnimComponent         // Montages, Story Sequences
├── UCharCombatComponent       // Attack, Counter, Death
├── UCharAIComponent           // Follow, Combat, Trail Recorder
├── UCharInteractionComponent  // IInteractable routing
├── UCharAttachmentsComponent  // Slot-based equipment
├── UCharCareerComponent       // Missions, Stats, Inventory
├── UCharSoundComponent        // Dual-channel audio
├── UCharWidgetComponent       // HUD, Icons, Tutorials
└── UCharDebugComponent        // Runtime inspection
```

**World Actors:**
- `AChair` — Sit/Stand with motion warping
- `ADoor` — 3-state machine with story integration
- `ASoundBox` — Ambient audio playlist
- `ASplinePath` — AI patrol / camera paths
- `AStoryTrigger` — Narrative orchestrator
- `ACityManager` — Procedural city generator

---

## 📚 Documentation

Full technical documentation covering **19 systems** with ~5,000 lines of detailed specs:

| Category | Systems |
|----------|---------|
| **Core** | PlayerCharacter, Input, Animation, Combat, AI, Interaction, Attachments, Sound, Widget, Debug |
| **UI** | PlayerHUD, PauseMenu, MainMenu |
| **World** | Chair, Door, SoundBox, SplinePath, StoryTrigger, CityManager |

---

## 🚀 Quick Start

**1. Create a Player:**
```cpp
// Blueprint inherits from APlayerCharacter
// Drop in level → Possess → Done
```

**2. Create an NPC Enemy:**
```cpp
NPC->Tags.Add("Enemy");
// Combat auto-activates when in range
```

**3. Create a Companion:**
```cpp
NPC->Tags.Add("Friend");
NPC->GetAIComponent()->FollowPlayer(PlayerCharacter);
```

**4. Create an Interactable:**
```cpp
class AMyPickup : public AActor, public IInteractable
{
    bool Interact_Implementation(AActor* Instigator) override
    {
        // Your logic here
        return true;
    }
};
```

---

## 🎯 Who Is This For?

✅ Developers tired of inheritance hell  
✅ Teams wanting a proven architecture  
✅ Solo devs who need production-ready systems  
✅ Anyone building action/adventure/RPG games in UE5  

---

## 📦 Get Access

AnyGame is available through **Patreon Early Access**:

<p align="center">
  <a href="https://www.patreon.com/c/DevLifeUnreal">
    <img src="https://img.shields.io/badge/Patreon-Get%20Access-FF424D?style=for-the-badge&logo=patreon" alt="Patreon">
  </a>
</p>

**What you get:**
- 📦 Latest plugin build (UE 5.7)
- 📚 Full technical documentation
- 💬 Discord community access
- 🔄 Regular updates & new features
- 🎯 Direct developer support

---

## 🔗 Links

| Platform | Link |
|----------|------|
| 🎬 **YouTube** | [DevLifeUnreal](https://www.youtube.com/channel/UC7FXcCmIPaoQUpkQCPFf15g) |
| 🔴 **Patreon** | [DevLifeUnreal](https://www.patreon.com/c/DevLifeUnreal) |
| 💼 **LinkedIn** | [[Luis Mendes](https://www.linkedin.com/in/luis-mendes-aab672239)](https://www.linkedin.com/in/luis-mendes-aab672239)) |
| 💬 **Discord** | [AnyGame Community](https://discord.gg/bg7cukuc) |

---

## 🛠️ Tech Stack

- **Engine:** Unreal Engine 5.7+
- **Language:** C++ with full Blueprint exposure
- **Architecture:** Composition over Inheritance
- **Optimization:** HISM, No-Tick philosophy, Event-driven

---

## 📄 License

AnyGame Plugin is **premium software**. Access available through [Patreon](https://www.patreon.com/c/DevLifeUnreal).

---

<p align="center">
  <i>"Opinionated enough to accelerate. Modular enough to not trap."</i>
</p>

<p align="center">
  <b>Solo developed over 1 year of iteration.</b>
</p>

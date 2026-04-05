# VR Hazard Lab

A VR hazard simulation and training framework built on Unity and SteamVR. Users explore interactive hazard scenarios in virtual reality using a modular timeline-driven architecture.

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Architecture](#architecture)
  - [Scriptable Object Event System](#scriptable-object-event-system)
  - [Timeline Action System](#timeline-action-system)
  - [Hazard Deployment System](#hazard-deployment-system)
  - [Animation Integration](#animation-integration)
- [VR Controller Support](#vr-controller-support)
- [Creating a New Hazard Scenario](#creating-a-new-hazard-scenario)
- [Third-Party Assets](#third-party-assets)

---

## Overview

VR Hazard Lab is a hackathon project that lets users experience different hazard scenarios in virtual reality. A physical "hazard chip" mechanic lets the user select a scenario by inserting a chip into a slot, which then loads the corresponding hazard room. Each room runs a scripted sequence of events—explosions, flying debris, ragdolls, particle effects—controlled by a data-driven timeline system.

---

## Requirements

| Requirement | Version |
|---|---|
| Unity | 2018.2.0f2 |
| SteamVR Plugin | 2.0.1 (included) |
| Steam / SteamVR runtime | Latest |

**Supported VR headsets (via SteamVR):**
- HTC Vive / Vive Pro
- Valve Index
- Meta Quest / Rift (via Oculus Link or Steam)
- Windows Mixed Reality headsets

---

## Getting Started

1. **Clone the repository**

   ```bash
   git clone https://github.com/omar92/VR-Hackathon.git
   ```

2. **Open in Unity**

   - Launch Unity Hub.
   - Click **Add** and select the cloned folder.
   - Open the project with **Unity 2018.2.0f2** (the project may show warnings in newer versions).

3. **Install SteamVR runtime**

   - Install [Steam](https://store.steampowered.com/) and [SteamVR](https://store.steampowered.com/app/250820/SteamVR/).
   - Connect and set up your VR headset before entering Play mode.

4. **Open a scene**

   Navigate to `Assets/Hazards Lab/Scenes/` and open one of:
   - `SampleScene.unity` — base sample level
   - `TestLabTest.unity` — primary test environment
   - `TestLabTest2.unity` — alternate test environment

5. **Press Play**

   Put on your headset and press **Play** in the Unity Editor. The SteamVR overlay will connect automatically.

---

## Project Structure

```
VR-Hackathon/
├── Assets/
│   ├── Hazards Lab/           # Main game content
│   │   ├── Animations/        # Animation clips
│   │   ├── Data/Timeline/     # Timeline ScriptableObject assets
│   │   ├── Material/          # Materials
│   │   ├── Models/            # 3D models
│   │   ├── Prefabs/           # All prefabs (player, rooms, props, hazards)
│   │   ├── Scenes/            # Unity scenes
│   │   └── Scripts/           # Core game scripts
│   │       ├── TimeslotAction/    # Individual timeline actions
│   │       ├── HazardShipScript.cs
│   │       ├── HazardShipSlotScript.cs
│   │       ├── LabTestInstance.cs
│   │       ├── ObjectTimeline.cs
│   │       ├── TimeslotAction.cs
│   │       ├── UpdateTransformField.cs
│   │       └── FollowTransformAction.cs
│   ├── Scriptables/           # Reusable event/field framework
│   ├── SteamVR/               # SteamVR plugin (v2.0.1)
│   ├── SteamVR_Input/         # Generated SteamVR input bindings
│   ├── JMO Assets/            # WarFX & Cartoon FX visual effects
│   └── FbxExporters/          # FBX export tooling
├── Packages/
│   └── manifest.json          # Unity package dependencies
├── ProjectSettings/           # Unity project configuration
├── actions.json               # SteamVR action definitions
├── bindings_vive_controller.json
├── bindings_oculus_touch.json
├── bindings_knuckles.json
├── bindings_holographic_controller.json
└── unityProject.vrmanifest    # SteamVR app manifest
```

---

## Architecture

### Scriptable Object Event System

Located in `Assets/Scriptables/`, this lightweight framework decouples components using ScriptableObject-based events and data fields.

**`GameEvent`** — A ScriptableObject that acts as a broadcast channel.

```csharp
// Raise an event from any script
[SerializeField] GameEvent onLoad;
onLoad.Raise();
```

**`GameEventListener`** — A MonoBehaviour that subscribes to a `GameEvent` and invokes a UnityEvent in response.

**Typed data fields** — `GameObjectField`, `TransformField`, `IntField`, `FloatField`, `BoolField`, `StringField` — each extends `AbstractField<T>` and fires an `onValueChanged` UnityEvent whenever its value is set.

```csharp
// React to a field value change
selectedHazard.onValueChanged.AddListener(OnSelectedHazardChanged);

// Set a field value from any script
selectedHazard.value = newHazardPrefab;
```

---

### Timeline Action System

The core simulation logic uses a data-driven timeline to sequence actions on GameObjects.

#### Key classes

| Class | Type | Role |
|---|---|---|
| `TimeslotAction` | Abstract ScriptableObject | Base class for all actions |
| `ObjectTimeline` | ScriptableObject | Manages a list of actions for one object |
| `LabTestInstance` | MonoBehaviour | Runs multiple `ObjectTimeline`s simultaneously |

#### `TimeslotAction` (abstract base)

```csharp
public float startTime;   // When the action begins (seconds from scene start)
public float duration;    // How long the action runs

void Initialize(GameObject target);   // Called once when the timeline begins
void TakeAction(float currentTime);   // Called every frame while active
void Exit();                          // Called when duration elapses
```

#### Built-in action types

| Script | Effect |
|---|---|
| `MoveAction` | Lerps the object from its start position by a displacement vector |
| `RotateAction` | Rotates the object continuously over its duration |
| `ExplodeAction` | Applies an explosion force to all nearby rigidbodies |
| `PlayAnimationAction` | Triggers an Animator state by name |
| `PlayParticleSystemAction` | Plays a particle system |
| `EnableAction` | Enables or disables a GameObject |
| `EnableGravityAction` | Turns on rigidbody gravity |
| `ReplaceWithRaggedDollAction` | Swaps the object with a ragdoll prefab |
| `FollowTransformAction` | Continuously syncs position/rotation to another transform |

#### Execution flow

```
LabTestInstance.Begin()
  ↓
For each ObjectTimeline:
  - Instantiate the target GameObject
  - Run any "init actions" immediately
  - Schedule remaining actions

Each Update():
  - Advance global timer
  - For each pending action:
      if timer >= action.startTime → call Initialize() + TakeAction()
      if timer >= startTime + duration → call Exit()
  - When all timelines finish → raise onExit GameEvent
```

---

### Hazard Deployment System

**`HazardRoomDeployScript`** listens to the `selectedHazard` `GameObjectField`. When a new hazard is selected it:

1. Destroys the currently loaded hazard instance (raises `onDestroy`).
2. Instantiates the new hazard prefab at `SpawnPosition`.
3. Raises the `onLoad` event for any listening components.

**`HazardShipSlotScript`** manages the physical docking slot for hazard chips:

- **Chip inserted** (trigger enter) → locks chip in slot, raises `ChipInserted` event.
- **Chip removed** (trigger exit) → unlocks chip, raises `ChipRemoved` event.
- `EjectChip()` → applies an outward force to eject the chip.

**`HazardShipScript`** is attached to each hazard chip prefab and holds a reference to the corresponding hazard room prefab. When the chip is inserted, the slot script reads this reference and updates the `selectedHazard` field.

---

### Animation Integration

Three `StateMachineBehaviour` scripts bridge Unity's Animator with the event system:

| Script | Trigger | Event raised |
|---|---|---|
| `LapReadyHandler` | `OnStateEnter` | `OnStart` |
| `LabExitHandler` | `OnStateEnter` | `onExit` |
| `animationEndHandler` | `OnStateEnter` | `onAnimationEnd` |

Attach these behaviours to Animator states to fire GameEvents at specific points in an animation.

---

## VR Controller Support

SteamVR input bindings are included for all major controller types. The binding files are in the project root and are automatically loaded by SteamVR.

| File | Controller |
|---|---|
| `bindings_vive_controller.json` | HTC Vive wand |
| `bindings_oculus_touch.json` | Meta / Oculus Touch |
| `bindings_knuckles.json` | Valve Index |
| `bindings_holographic_controller.json` | Windows Mixed Reality |

Default actions defined in `actions.json`:

| Action | Description |
|---|---|
| `GrabPinch` / `GrabGrip` | Pick up objects |
| `Teleport` | Locomotion |
| `InteractUI` | UI interaction |
| `SkeletonLeftHand` / `SkeletonRightHand` | Hand tracking |
| `Haptic` | Controller vibration |

---

## Creating a New Hazard Scenario

1. **Create a hazard room prefab**
   - Build your environment under `Assets/Hazards Lab/Prefabs/Main Scene prefs/HazardRooms/`.

2. **Create ObjectTimeline assets**
   - Right-click in the Project window → **Create → ObjectTimeline**.
   - Assign the target prefab and add `TimeslotAction` ScriptableObjects to the `actions` list.

3. **Add a LabTestInstance**
   - Add a `LabTestInstance` component to a GameObject in your hazard room prefab.
   - Assign your `ObjectTimeline` assets to its `timelines` list.
   - Wire up the `onExit` GameEvent to control room logic or scene transitions.

4. **Create a hazard chip prefab**
   - Duplicate an existing chip prefab (e.g. `ExplosiveHazardChip.prefab`).
   - Assign your new hazard room prefab to the `HazardShipScript.hazardRoom` field.

5. **Test**
   - Open `TestLabTest.unity`, press Play, and insert your chip into the slot.

---

## Third-Party Assets

| Asset | Location | License |
|---|---|---|
| SteamVR Plugin 2.0.1 | `Assets/SteamVR/` | Valve BSD-style (see `SteamVR/readme.txt`) |
| JMO Assets – WarFX | `Assets/JMO Assets/WarFX/` | See `!JMO Assets Readme.txt` |
| Cartoon FX Easy Editor | `Assets/JMO Assets/Cartoon FX Easy Editor/` | See `!JMO Assets Readme.txt` |
| Unity FBX Exporter | `Assets/FbxExporters/` | See `FbxExporters/LICENSE.txt` |

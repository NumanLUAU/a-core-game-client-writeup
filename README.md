# SERVER RECONSTRUCTION GUIDE
> Every remote, every expected data shape, every stub, and a build order.  

---

## TABLE OF CONTENTS
1. [ReplicatedStorage Layout](#replicatedstorage-layout)
2. [Remote Catalogue](#remote-catalogue)
3. [Server Script Build Order](#server-script-build-order)
4. [Server Script Stubs](#server-script-stubs)

---

## REPLICATEDSTORAGE LAYOUT

The client code implies this exact folder/object structure must exist in `ReplicatedStorage` before the server scripts run.

```
ReplicatedStorage/
├── DialogGui                     (ScreenGui — cloned by dialog system)
├── DialogUnit                    (ModuleScript — required by dialog client)
├── ac_robert_dialog              (RemoteEvent)
├── ClientsideAnimations          (RemoteEvent)
├── ClientTweening                (RemoteEvent — ChocolaBase tween library)
├── LocalTweening                 (RemoteEvent — _CB_CLIENT tween network)
├── LocalModelTweening            (RemoteEvent — ModelTweenArray network)
├── CameraShake/                  (Folder — NOT a remote, acts as a value container)
│   ├── LinearIntensity           (NumberValue, default 0)
│   ├── AngularIntensity          (NumberValue, default 0)
│   ├── LinearMulti               (Vector3Value, default 0,0,0)
│   ├── AngularMulti              (Vector3Value, default 0,0,0)
│   ├── Roughness                 (NumberValue, default 0)
│   └── Value                    (BoolValue  whether shake is active, used by CamShake:Activate)
├── UNPLANNEDOVERLOAD/            (Folder)
│   ├── Shakelol                  (RemoteEvent)
│   ├── CAMANTIHECC               (RemoteEvent)
│   ├── Funny                     (RemoteEvent)
│   ├── FovSwap                   (RemoteEvent)
│   ├── Cinematic                 (RemoteEvent)
│   ├── SongShow                  (RemoteEvent)
│   └── passiveOverload           (RemoteEvent)
├── Events/                       (Folder)
│   ├── ChatWarn                  (RemoteEvent)
│   └── FXHandler                 (RemoteEvent)
├── Settings/                     (Folder)
│   └── Shake                    (StringValue  "on" | "reduced" | "off")
└── PioTweenHookEvent             (RemoteEvent  PioTween library)
```

> **Note on `CameraShake`:** The client code (`CamShake.lua`) treats `ReplicatedStorage.CameraShake` as a **Folder** with child `NumberValue`/`Vector3Value` objects. It reads `.Value` on each child every `RenderStepped`. The server drives shake by tweening these values (e.g. tweening `LinearIntensity.Value` to `0.06`). This is **not** a RemoteEvent itself  it is a replicated state container.

---

## REMOTE CATALOGUE

Each remote is documented with: **Type**, **Direction**, **What the client sends**, **What the client receives**, **Purpose**, and a **Server stub**.

---

### 1. `ac_robert_dialog`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.ac_robert_dialog` |
| **Direction** | **Server → Client** (FireClient) |
| **Client sends** | Nothing. Client does not `FireServer` on this remote. |
| **Client receives** | `(dialogNode: table)` — A dialog tree node. The root node is the starting dialog. |

**Dialog Node Structure** (what the server fires):
```lua
{
    name    = "Robert",               -- string: speaker name shown in NameTag
    image   = "rbxassetid://XXXXX",   -- string: portrait image asset id
    content = "Hello, %s!",           -- string: dialog body. %s is replaced with player.DisplayName
    sound   = "rbxassetid://XXXXX",   -- string (optional): blip sound override
    reserved = any,                   -- any (optional): value passed back on [close]
    paths   = {                       -- table | true | nil
        -- If table: keys are response button labels, values are child dialog nodes
        ["Tell me more"] = {
            name = "Robert",
            image = "rbxassetid://XXXXX",
            content = "Of course! ...",
            paths = true   -- 'true' means show a [return] button that goes back to parent
        },
        ["Goodbye"] = {
            name = "Robert",
            content = "See you around, %s.",
            paths = nil    -- nil means only [close] is shown
        }
    }
}
```

**How the flow works (important for server):**
1. Server calls `ac_robert_dialog:FireClient(player, rootNode)`
2. The client presents the dialog tree completely locally — it navigates the `paths` table client-side with no further server calls
3. When the player presses `[close]`, the dialog ends. The `ProximityPrompt` on Robert re-enables itself client-side.
4. The server should listen to `ProximityPrompt.Triggered` server-side to know when to fire the event, OR fire it after some condition is met.

**Purpose:** Triggers and supplies a complete branching dialog conversation tree to a specific player. The NPC is named "Robert" and is located at `workspace.Misc.Robert`. A camera position part exists at `workspace.Misc.RobertDialogCamPos`.

---

### 2. `ClientsideAnimations`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.ClientsideAnimations` |
| **Direction** | **Server → Client** (FireClient or FireAllClients) |
| **Client sends** | Nothing |
| **Client receives** | `(state: boolean)` |

**What the client does with it:**
- The `LocalAnimations` script listens and either enables/disables its parent LocalScript (the FX/animation loop script) or fires an `AnimationDisabling` BindableEvent to stop internal loops.
- The `CamShake` (ChatTags script) also listens: `true` calls `setupShake()`, `false` tears down all shake connections and deactivates the shake controller.

**Purpose:** Master on/off switch for all client-side visual animations and camera shake. Fire `false` before a cutscene or explosion sequence. Fire `true` to restore normal operation. Typically fired to all clients simultaneously.

---

### 3. `ClientTweening`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.ClientTweening` |
| **Direction** | **Server → Client** (FireClient / FireAllClients) |
| **Client sends** | Nothing |
| **Client receives** | `(mode: string, ...)` |

**Modes the client handles:**
```lua
-- Mode 1: Tween a model's pivot
"TweenModel", model: Instance, info: table, targetCFrame: CFrame
-- info table shape:
-- { Time, EasingStyle (Enum), EasingDirection (Enum), RepeatCount, Reverses, DelayTime }

-- Mode 2: Tween any instance property
"TweenInstance", instance: Instance, info: table, goalTable: table
-- info: same shape as above
-- goalTable: e.g. { Transparency = 0, Size = Vector3.new(4,4,4) }
```

**Purpose:** Server-driven tweening that plays on all clients. Used to animate world models and parts from the server without running a tween on every client independently. Part of the "PioTween" library (author: @PioTheDeveloper). The server counterpart creates tweens server-side and also fires this remote so clients see smooth animation.

---

### 4. `LocalTweening`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.LocalTweening` |
| **Direction** | **Server → All Clients** (FireAllClients) |
| **Client sends** | Nothing (client-side only: `OnServerEvent` exists in `TweenArray` — see note) |
| **Client receives** | `(mode: string, data: table)` |

**Modes the client handles (from `Networker` module):**
```lua
"RUN"     -- play a tween: data = { id, instance, tweenInfo, goal, ... }
"REMOVE"  -- cancel/remove a tween by id: data = tween id string
"M_RUN"   -- play a model tween
"M_REMOVE"-- remove a model tween
```

**Important note — `OnServerEvent` in `TweenArray`:**  
The `TweenArray` module (part of the `_CB_CLIENT` ChocolaBase tween library) registers `LocalTweening.OnServerEvent`. This means this remote is also a **Client → Server** route for the tween system internally. When a client-side tween is created via `_CB_CLIENT`, it fires the server so the server can re-broadcast it to all clients. On the server, `LocalTweening.OnServerEvent` receives the same tween data and should call `FireAllClients` to replicate to every other client.

**Purpose:** Drives the ChocolaBase (`_CB_CLIENT`) client-side tween system. All instance tweens (non-model) are networked through this remote.

---

### 5. `LocalModelTweening`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.LocalModelTweening` |
| **Direction** | **Server → All Clients** (FireAllClients) |
| **Client sends** | Nothing (listen only) |
| **Client receives** | `(mode: string, model, tweenInfo, goalCFrame, ...) ` |

**Modes the client handles (from `ModelTweenArray`):**
```lua
"Part"   -- tween a single part's CFrame
         -- args: (model: BasePart, tweenInfo, goalCFrame, ..., extras)
"Model"  -- tween a full model's pivot
         -- args: (model: Model, tweenInfo, goalCFrame, ..., extras)
```

**Purpose:** Same as `LocalTweening` but specifically for model/pivot tweens. Separate remote so model and instance tweens can be managed independently. Server fires this when it wants to smoothly move world models on all clients.

---

### 6. `Events.ChatWarn`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.Events.ChatWarn` |
| **Direction** | **Client → Server** (FireServer) |
| **Client sends** | `(warnLevel: number)` — a float between 0 and 20 |
| **Client receives** | Nothing |

**Context:**  
The `radiomanager` LocalScript monitors the local player's chat messages. If the player's chat contains begging/requesting language mentioning staff, events, blackholes, "Project Boron", or the Unplanned Overload, a `WarnLevel` float is incremented and `ChatWarn:FireServer(WarnLevel)` is called. The level decays over time.

**What the server should do:**  
Log or act upon elevated warn levels. Possible responses:
- At low level (>2): log to a moderation record
- At medium level (>5): send a warning message or kick notification
- At high level (>10): notify online staff via server messaging or auto-mute

**Purpose:** Anti-begging/harassment detection telemetry. Client sends a continuous threat level score to the server.

---

### 7. `Events.FXHandler`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.Events.FXHandler` |
| **Direction** | **Server → Client** (FireClient or FireAllClients) |
| **Client sends** | Nothing |
| **Client receives** | `(effectName: string, data: any?)` |

**Currently registered effects on the client (from `LocalMasterScript`/`FXHandler` module):**
```lua
"meltShieldExplosion"
-- data: nil (no extra data needed)
-- Client effect: raycasts from camera toward the Core model (tagged "Models.Core").
-- If the Core is visible and on screen:
--   - Briefly flashes white (DOF + color correction brightness spike)
--   - Fades over 10 seconds with DOF expanding
-- Only triggers the visual if the client can actually see the Core.
```

**Purpose:** Server-triggered cinematic/visual effect system. Server sends an effect name and optional payload. Client runs the corresponding local visual. The architecture supports adding more named effects to the `FX` table.

**How to add more effects:**  
Add a new key to the `FX` table in `LocalMasterScript`'s `FXHandler` module. The server fires `FXHandler:FireAllClients("EffectName", optionalData)`.

---

### 8. `UNPLANNEDOVERLOAD.Shakelol`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.UNPLANNEDOVERLOAD.Shakelol` |
| **Direction** | **Server → Client** (FireClient / FireAllClients) |
| **Client sends** | Nothing |
| **Client receives** | `()` — no arguments |

**Client effect:**  
Tweens `CameraShake.AngularIntensity`, `CameraShake.LinearIntensity`, and `CameraShake.Roughness` values up over 4 seconds with `Enum.EasingStyle.Linear + Enum.EasingDirection.In + Reverses=true`. This creates a 4-second shake-in-and-out pulse. The camera shake system reads these values every frame via the `CameraShake` folder in ReplicatedStorage.

**Purpose:** Fire a dramatic camera shake event. Used during the Unplanned Overload sequence.

---

### 9. `UNPLANNEDOVERLOAD.CAMANTIHECC`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.UNPLANNEDOVERLOAD.CAMANTIHECC` |
| **Direction** | **Server → Client** |
| **Client sends** | Nothing |
| **Client receives** | `()` — no arguments |

**Client effect (delayed 8.1 seconds after receiving):**
- Tweens camera FOV back to 70 over 1.5 seconds
- Tweens `game.Lighting.a` (ColorCorrection) Brightness to `-0.04` over 10 seconds with Bounce easing
- Tweens `game.SoundService.Main.FlangeSoundEffect` (Depth, Mix, Rate) to 0 over 10 seconds
- Tweens `game.Lighting.coolantcupblur` (BlurEffect) Size to 0 over 10 seconds with Sine easing

**Purpose:** "Camera anti-hecc" — the cleanup/recovery remote. Fired after the Unplanned Overload's chaotic camera effects to smoothly restore the player's camera and audio to normal. Fire this 8 seconds after the overload sequence begins.

---

### 10. `UNPLANNEDOVERLOAD.Funny`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.UNPLANNEDOVERLOAD.Funny` |
| **Direction** | **Server → Client** |
| **Client sends** | Nothing |
| **Client receives** | `()` — no arguments |

**Client effect:**  
Picks a random scary image from a hardcoded list of 6 asset IDs and flashes it on screen (ImageTransparency 0 → 1 over 1 second with Sine easing). The image is stored as `UOUI.scary` in `PlayerGui.UOUI`.

**Purpose:** Jumpscares the player with a random creepy image. Part of the Unplanned Overload horror sequence.

---

### 11. `UNPLANNEDOVERLOAD.FovSwap`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.UNPLANNEDOVERLOAD.FovSwap` |
| **Direction** | **Server → Client** |
| **Client sends** | Nothing |
| **Client receives** | `(fov: number, time: number, style: Enum.EasingStyle, direction: Enum.EasingDirection)` |

**Client effect:**  
Tweens `workspace.CurrentCamera.FieldOfView` to `fov` over `time` seconds using the provided easing style and direction.

**Purpose:** Dramatically changes the player's FOV during the Unplanned Overload sequence. Can be used for zoom effects, disorientation, etc.

**Example server call:**
```lua
FovSwap:FireAllClients(120, 2, Enum.EasingStyle.Sine, Enum.EasingDirection.Out)
-- shoots FOV to 120 over 2 seconds
```

---

### 12. `UNPLANNEDOVERLOAD.Cinematic`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.UNPLANNEDOVERLOAD.Cinematic` |
| **Direction** | **Server → Client** |
| **Client sends** | Nothing |
| **Client receives** | `(enable: boolean, size: number?, time: number?, style: Enum.EasingStyle?, direction: Enum.EasingDirection?)` |

**Client effect:**  
Shows or hides cinematic letterbox bars (top and bottom bars in `PlayerGui.UOUI.Cinematics`).
- `enable = true` + `size = 0.1`: grows bars to 10% of screen height
- `enable = false`: collapses bars to 0

All tween parameters default to `Exponential/Out, 0.5s` if not provided.

**Purpose:** Adds cinematic black bars to the screen for cutscene-like moments.

**Example server call:**
```lua
-- Show bars:
Cinematic:FireAllClients(true, 0.1, 1, Enum.EasingStyle.Exponential, Enum.EasingDirection.Out)
-- Hide bars:
Cinematic:FireAllClients(false, nil, 0.5)
```

---

### 13. `UNPLANNEDOVERLOAD.SongShow`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.UNPLANNEDOVERLOAD.SongShow` |
| **Direction** | **Server → Client** |
| **Client sends** | Nothing |
| **Client receives** | `(title: string, artist: string)` |

**Client effect:**  
Sets `PlayerGui.Main.songname.Text` to `"title - artist"` and tweens its `TextTransparency` to `0.6` over 0.3 seconds, then after 6 seconds fades it back to `1` over 2 seconds. This is a non-intrusive "now playing" HUD element.

**Purpose:** Displays the currently playing song name to the player. Fire whenever a new track begins.

**Example server call:**
```lua
SongShow:FireAllClients("Cascade Failure", "FACILITY OST")
```

---

### 14. `UNPLANNEDOVERLOAD.passiveOverload`
| Field | Value |
|---|---|
| **Type** | RemoteEvent |
| **Location** | `ReplicatedStorage.UNPLANNEDOVERLOAD.passiveOverload` |
| **Direction** | **Server → Client** |
| **Client sends** | Nothing |
| **Client receives** | `(state: boolean)` |

**Client effect:**
- `state = true`: Expands `UOUI.WARNING` image to full screen size and starts a looping pulse tween (magenta color oscillation every 1 second)
- `state = false`: Expands warning image outward (size 1.5x) and pauses the pulse tween

**Purpose:** The passive overload warning indicator. Shows a pulsing warning overlay when the reactor/system is approaching a passive overload state. `true` = warning active, `false` = warning resolving/dismissed.

---

### 15. `CameraShake` (Folder — NOT a RemoteEvent)
| Field | Value |
|---|---|
| **Type** | Folder with Value children |
| **Location** | `ReplicatedStorage.CameraShake` |
| **Direction** | Server writes → client reads every frame |
| **Client sends** | Nothing |
| **Client reads** | All child values every RenderStepped |

**Children and types:**
```
CameraShake/
├── LinearIntensity    NumberValue  (default 0) — overall linear shake magnitude
├── AngularIntensity   NumberValue  (default 0) — overall angular shake magnitude
├── LinearMulti        Vector3Value (default 0,0,0) — per-axis linear multipliers
├── AngularMulti       Vector3Value (default 0,0,0) — per-axis angular multipliers
├── Roughness          NumberValue  (default 0) — shake frequency
└── Value              BoolValue    (default false) — master enable
```

**How to drive shake from the server:**
```lua
-- Turn on moderate shake:
local cs = game.ReplicatedStorage.CameraShake
cs.Value.Value = true              -- activates the per-client shake loop
TweenService:Create(cs.LinearIntensity, TweenInfo.new(1), {Value = 0.04}):Play()
TweenService:Create(cs.AngularIntensity, TweenInfo.new(1), {Value = 0.04}):Play()
TweenService:Create(cs.Roughness, TweenInfo.new(1), {Value = 15}):Play()
cs.LinearMulti.Value = Vector3.new(1, 0.5, 1)
cs.AngularMulti.Value = Vector3.new(1, 0.3, 0.5)

-- Turn off shake:
cs.Value.Value = false
TweenService:Create(cs.LinearIntensity, TweenInfo.new(2), {Value = 0}):Play()
TweenService:Create(cs.AngularIntensity, TweenInfo.new(2), {Value = 0}):Play()
```

**Purpose:** The camera shake system is data-driven via replicated values. The server does NOT use a RemoteEvent to trigger shake; instead it tweens these NumberValues/Vector3Values and they replicate to all clients automatically. The `Shakelol` remote is a shortcut that makes the client tween these itself.

---

### 16. `Settings.Shake`
| Field | Value |
|---|---|
| **Type** | StringValue |
| **Location** | `ReplicatedStorage.Settings.Shake` |
| **Direction** | Server writes, client reads |
| **Valid values** | `"on"`, `"reduced"`, `"off"` |

**Purpose:** Per-server (or server-set) camera shake accessibility setting. The client's `CamShake` system checks this value on every frame tick. If `"off"`, shake is disabled regardless of intensity values. If `"reduced"`, `GlobalMulti` is set to `0.42`. If `"on"`, full intensity. The server can also let players change this themselves via a GUI that fires a remote to set their own instance.

---

## SERVER SCRIPT BUILD ORDER

Build these **in order**. Each step depends on the previous one existing.

### Step 1 — Create ReplicatedStorage Structure (No code, just instances)
Create every folder, RemoteEvent, RemoteFunction, and Value instance listed in the [ReplicatedStorage Layout](#replicatedstorage-layout) section above. No scripts yet. Just the instance tree.

Specifically create:
- `ReplicatedStorage.Events` (Folder)
- `ReplicatedStorage.Events.ChatWarn` (RemoteEvent)
- `ReplicatedStorage.Events.FXHandler` (RemoteEvent)
- `ReplicatedStorage.UNPLANNEDOVERLOAD` (Folder) + all 7 child RemoteEvents
- `ReplicatedStorage.CameraShake` (Folder) + all 6 child value objects
- `ReplicatedStorage.Settings` (Folder) + `Shake` StringValue set to `"on"`
- `ReplicatedStorage.ClientsideAnimations` (RemoteEvent)
- `ReplicatedStorage.ac_robert_dialog` (RemoteEvent)
- `ReplicatedStorage.ClientTweening` (RemoteEvent)
- `ReplicatedStorage.LocalTweening` (RemoteEvent)
- `ReplicatedStorage.LocalModelTweening` (RemoteEvent)
- `ReplicatedStorage.PioTweenHookEvent` (RemoteEvent)

### Step 2 — Create Workspace Structure
The client expects certain world objects. Make sure these exist:
- `workspace.Misc` (Folder or Model)
  - `workspace.Misc.Robert` (Model with `HumanoidRootPart` containing a `ProximityPrompt`)
  - `workspace.Misc.RobertDialogCamPos` (Part — camera anchor for dialog)
- `workspace.AlarmBeacons` (Folder/Model) with a `Status` (BoolValue) and `Speed` (NumberValue) child
- `workspace.AmbientSound` (Folder containing Sound instances named after ambient zones e.g. `"Facility"`)
- `workspace.Radios` (Folder containing Radio Models, each with a `ToggleIndicator` BasePart)
- `workspace.CoolantRoom.ScriptableStuff.CoolantPool.Pool` (Part with ClickDetector)

### Step 3 — Lighting Setup
Client references these directly by name:
- `game.Lighting.CoolantCC` (ColorCorrectionEffect)
- `game.Lighting.CoolantBlur` (BlurEffect)
- `game.Lighting.a` (ColorCorrectionEffect — used in CAMANTIHECC and meltShieldExplosion)
- `game.Lighting.coolantcupblur` (BlurEffect)
- `game.Lighting.DepthOfField` (DepthOfFieldEffect)
- `game.SoundService.Main.FlangeSoundEffect` (FlangeSoundEffect inside a Sound named "Main")

### Step 4 — Write `DialogModule` (ModuleScript in ReplicatedStorage)
The client does `require(game.ReplicatedStorage:WaitForChild("DialogUnit"))` at the top of the dialog script. This module needs to exist. Its exact API is not used in the client code shown — it's required for its side effects. Create it as a minimal module:
```lua
-- ReplicatedStorage/DialogUnit (ModuleScript)
-- Handles server-side dialog tree logic if needed.
-- Client requires this for side effects (e.g. setting up shared dialog data).
return {}
```

### Step 5 — Write `RobertDialogServer` (Script in ServerScriptService)
This is the most important original server script. It:
1. Listens for the Robert ProximityPrompt trigger
2. Fires `ac_robert_dialog` with a full dialog tree to the triggering player
3. Re-enables the ProximityPrompt after the dialog ends (the client handles disabling it locally; you may want server-side tracking too)

See stub in [Server Script Stubs](#server-script-stubs).

### Step 6 — Write `ChatWarnHandler` (Script in ServerScriptService)
Listens to `Events.ChatWarn.OnServerEvent`. Logs warn levels and takes action at thresholds.

### Step 7 — Write `FXHandlerServer` (Script in ServerScriptService)
Exposes a server-side API/module for firing named FX events to all clients or specific clients. Other server scripts import this to trigger effects.

### Step 8 — Write `UnplannedOverloadServer` (Script in ServerScriptService)
Manages the Unplanned Overload sequence. Fires all the `UNPLANNEDOVERLOAD.*` remotes in the correct order with the correct timing. Uses `ClientsideAnimations` to disable client animations before the sequence.

### Step 9 — Write `TweenReplicationServer` (Script in ServerScriptService)
Handles `LocalTweening.OnServerEvent` — when the ChocolaBase tween library fires from a client, the server re-broadcasts it to all other clients.

### Step 10 — Write `AlarmBeaconsServer` (Script in ServerScriptService)
Sets `workspace.AlarmBeacons.Status.Value` to true/false based on game events. The client `AlarmBeacons` LocalScript reads this value and animates the rotating beacons.

### Step 11 — Write `RadioServer` (Script in ServerScriptService)
Manages radio enable/disable state for `workspace.Radios`. The `radiomanager` client script reads BrickColor of `ToggleIndicator` and volume values — these can be driven by server-set Attributes or Values on each Radio model.

### Step 12 — Write `CoolantQuestServer` (Script in ServerScriptService)
Manages the Coolant Quest — the server-side logic for giving/taking the "Cup Of Coolant" tool. The client `coolantquest` LocalScript listens to the player's Backpack and adjusts the ClickDetector's `MaxActivationDistance` based on whether they have the cup. The server grants/removes the tool.

### Step 13 — Write `AmbientZoneServer` (optional, Script in ServerScriptService)
The ambient system is entirely client-side (touches on `SPart` parts with an `Ambient` StringValue child). The server only needs to ensure those parts exist in the world with the correct `Ambient` StringValue children. No server script is strictly needed unless you want server-side zone tracking.

---

## SERVER SCRIPT STUBS

Full stub code for every server script. Replace `-- TODO` comments with actual logic lol.

---

### `RobertDialogServer`
```lua
-- ServerScriptService/RobertDialogServer
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local DialogRemote = ReplicatedStorage:WaitForChild("ac_robert_dialog")
local Robert = workspace:WaitForChild("Misc"):WaitForChild("Robert")
local Prompt = Robert:WaitForChild("HumanoidRootPart"):WaitForChild("ProximityPrompt")

-- Define the full dialog tree here.
-- The structure must match exactly what the client expects (see Remote #1 docs above).
local DIALOG_TREE = {
    name    = "Robert",
    image   = "rbxassetid://REPLACE_WITH_PORTRAIT_ID",
    content = "Hello, %s. Welcome to the facility.",
    sound   = nil, -- optional blip sound id, or nil for default
    paths   = {
        ["What is this place?"] = {
            name    = "Robert",
            image   = "rbxassetid://REPLACE_WITH_PORTRAIT_ID",
            content = "This is a research facility. Best not to ask too many questions.",
            paths   = true, -- shows [return] button
        },
        ["Who are you?"] = {
            name    = "Robert",
            image   = "rbxassetid://REPLACE_WITH_PORTRAIT_ID",
            content = "I'm Robert. I keep things running around here.",
            paths   = {
                ["Interesting."] = {
                    name    = "Robert",
                    image   = "rbxassetid://REPLACE_WITH_PORTRAIT_ID",
                    content = "Indeed. Now, is there anything else, %s?",
                    paths   = true,
                }
            }
        },
        ["Nothing, goodbye."] = {
            name    = "Robert",
            image   = "rbxassetid://REPLACE_WITH_PORTRAIT_ID",
            content = "Take care, %s.",
            paths   = nil, -- only [close] will appear
        }
    }
}

-- Track players currently in dialog to prevent spam
local InDialog = {}

Prompt.Triggered:Connect(function(player)
    if InDialog[player] then return end
    InDialog[player] = true

    -- Fire the dialog tree to this specific player
    DialogRemote:FireClient(player, DIALOG_TREE)

    -- Wait a moment then clean up the in-dialog flag
    -- The client re-enables the prompt itself, but we track server-side too
    task.delay(2, function()
        InDialog[player] = nil
    end)
end)

Players.PlayerRemoving:Connect(function(player)
    InDialog[player] = nil
end)
```

---

### `ChatWarnHandler`
```lua
-- ServerScriptService/ChatWarnHandler
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local ChatWarn = ReplicatedStorage:WaitForChild("Events"):WaitForChild("ChatWarn")

-- Thresholds
local THRESHOLD_LOG    = 3   -- just log
local THRESHOLD_WARN   = 7   -- warn staff
local THRESHOLD_ACTION = 14  -- take automatic action

-- Per-player warn tracking (server-side mirror)
local PlayerWarnLevels = {}

Players.PlayerAdded:Connect(function(player)
    PlayerWarnLevels[player.UserId] = 0
end)

Players.PlayerRemoving:Connect(function(player)
    PlayerWarnLevels[player.UserId] = nil
end)

ChatWarn.OnServerEvent:Connect(function(player, warnLevel)
    -- Validate input
    if typeof(warnLevel) ~= "number" then return end
    warnLevel = math.clamp(warnLevel, 0, 20)

    PlayerWarnLevels[player.UserId] = warnLevel

    if warnLevel >= THRESHOLD_ACTION then
        warn(("[ChatWarn] HIGH WARN: %s | Level: %.2f — Consider action"):format(player.Name, warnLevel))
        -- TODO: auto-mute, kick, or notify staff via MessagingService
    elseif warnLevel >= THRESHOLD_WARN then
        warn(("[ChatWarn] ELEVATED: %s | Level: %.2f"):format(player.Name, warnLevel))
        -- TODO: notify online staff
    elseif warnLevel >= THRESHOLD_LOG then
        print(("[ChatWarn] LOG: %s | Level: %.2f"):format(player.Name, warnLevel))
    end
end)
```

---

### `FXHandlerServer`
```lua
-- ServerScriptService/FXHandlerServer
-- This script exposes a module-like interface for other server scripts to fire FX.
-- It also acts as the central firing point.

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local FXRemote = ReplicatedStorage:WaitForChild("Events"):WaitForChild("FXHandler")

-- Public API for other server scripts to use:
-- require this script via a ModuleScript wrapper if needed,
-- or use the remote directly.

-- Example of firing an effect to all clients:
local FXServer = {}

function FXServer.FireAll(effectName, data)
    FXRemote:FireAllClients(effectName, data)
end

function FXServer.FirePlayer(player, effectName, data)
    FXRemote:FireClient(player, effectName, data)
end

-- Currently supported effect names (defined in the client's FX table):
-- "meltShieldExplosion" — fires with no data, triggers DOF + color correction FX
--                         only plays if Core model is visible to the client

-- Usage example from another script:
-- local FXServer = require(game.ServerScriptService.FXHandlerServer)
-- FXServer.FireAll("meltShieldExplosion")

return FXServer
```

---

### `UnplannedOverloadServer`
```lua
-- ServerScriptService/UnplannedOverloadServer
-- Manages the Unplanned Overload (UO) event sequence.
-- Call triggerOverload() from game logic when conditions are met.

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")

local UO = ReplicatedStorage:WaitForChild("UNPLANNEDOVERLOAD")
local ClientsideAnimations = ReplicatedStorage:WaitForChild("ClientsideAnimations")
local CameraShake = ReplicatedStorage:WaitForChild("CameraShake")
local Settings = ReplicatedStorage:WaitForChild("Settings")

local IsOverloadActive = false

local function triggerOverload()
    if IsOverloadActive then return end
    IsOverloadActive = true

    -- Step 1: Disable client-side ambient animations
    ClientsideAnimations:FireAllClients(false)

    task.wait(0.5)

    -- Step 2: Enable cinematic bars
    UO.Cinematic:FireAllClients(true, 0.08, 1.5, Enum.EasingStyle.Exponential, Enum.EasingDirection.Out)

    -- Step 3: Show song title (optional — replace with actual event music name)
    UO.SongShow:FireAllClients("CASCADE FAILURE", "PROJECT BORON OST")

    -- Step 4: Show passive overload warning
    UO.passiveOverload:FireAllClients(true)

    task.wait(2)

    -- Step 5: Camera shake
    UO.Shakelol:FireAllClients()

    -- Step 6: Warp FOV outward
    UO.FovSwap:FireAllClients(100, 3, Enum.EasingStyle.Sine, Enum.EasingDirection.Out)

    task.wait(3)

    -- Step 7: Jumpscare flash
    UO.Funny:FireAllClients()

    task.wait(5)

    -- Step 8: Camera recovery
    UO.CAMANTIHECC:FireAllClients()
    -- Note: client waits 8.1 seconds internally before actually starting the recovery tween.
    -- This is intentional — it fires this early so the 8.1s delay lands at the right moment.

    task.wait(10)

    -- Step 9: Resolve passive overload warning
    UO.passiveOverload:FireAllClients(false)

    -- Step 10: Collapse cinematic bars
    UO.Cinematic:FireAllClients(false, nil, 2, Enum.EasingStyle.Exponential, Enum.EasingDirection.InOut)

    task.wait(3)

    -- Step 11: Re-enable client animations
    ClientsideAnimations:FireAllClients(true)

    IsOverloadActive = false
end

-- TODO: Connect triggerOverload() to the game's actual trigger condition.
-- Examples:
-- workspace.SomeButton.ClickDetector.MouseClick:Connect(triggerOverload)
-- game.ReplicatedStorage.Events.SomeServerEvent.OnServerEvent:Connect(triggerOverload)
```

---

### `TweenReplicationServer`
```lua
-- ServerScriptService/TweenReplicationServer
-- Rebroadcasts LocalTweening events from any client to all other clients.
-- This is required by the _CB_CLIENT (ChocolaBase) tween system.

local ReplicatedStorage = game:GetService("ReplicatedStorage")

local LocalTweening = ReplicatedStorage:WaitForChild("LocalTweening")

LocalTweening.OnServerEvent:Connect(function(sender, mode, ...)
    -- Rebroadcast to all other clients (not the sender, to avoid double-play)
    -- If you want sender to also receive, use FireAllClients instead
    for _, player in game.Players:GetPlayers() do
        if player ~= sender then
            LocalTweening:FireClient(player, mode, ...)
        end
    end
end)

-- Note: LocalModelTweening works the same way IF the client fires it.
-- From the code, LocalModelTweening is currently only server→client.
-- Add an OnServerEvent handler here if you add client→server model tweens later.
```

---

### `AlarmBeaconsServer`
```lua
-- ServerScriptService/AlarmBeaconsServer
-- Controls the alarm beacon rotation state.
-- Set AlarmBeacons.Status.Value = true to activate, false to deactivate.
-- The client LocalScript reads this value and speeds up/slows down the beacons.

local AlarmBeacons = workspace:WaitForChild("AlarmBeacons")
local Status = AlarmBeacons:WaitForChild("Status") -- BoolValue

-- Example: activate alarms
local function activateAlarms()
    Status.Value = true
end

local function deactivateAlarms()
    Status.Value = false
end

-- TODO: Connect activateAlarms/deactivateAlarms to the game logic.
-- Example: tie to reactor overload state, game phase, etc.

-- Initial state
Status.Value = false
```

---

### `CoolantQuestServer`
```lua
-- ServerScriptService/CoolantQuestServer
-- Manages the Coolant Cup tool for the coolant quest.
-- The client's coolantquest LocalScript handles the click detector radius client-side.
-- The server must grant/remove the tool and manage quest state.

local Players = game:GetService("Players")
local ServerStorage = game:GetService("ServerStorage")

-- Assume tools live in ServerStorage
-- local EmptyCup = ServerStorage:WaitForChild("Empty Cup")
-- local CupOfCoolant = ServerStorage:WaitForChild("Cup Of Coolant")

local function giveTool(player, toolName)
    local tool = ServerStorage:FindFirstChild(toolName)
    if not tool then
        warn("Tool not found in ServerStorage: " .. toolName)
        return
    end
    local clone = tool:Clone()
    clone.Parent = player.Backpack
end

local function removeTool(player, toolName)
    local backpack = player.Backpack
    local char = player.Character
    local tool = backpack:FindFirstChild(toolName) or (char and char:FindFirstChild(toolName))
    if tool then
        tool:Destroy()
    end
end

-- TODO: Connect to game events.
-- Example sequence:
-- 1. Player starts quest → giveTool(player, "Empty Cup")
-- 2. Player fills cup at coolant pool → ClickDetector.MouseClick fires
--    → removeTool(player, "Empty Cup") → giveTool(player, "Cup Of Coolant")
-- 3. Player delivers coolant → removeTool(player, "Cup Of Coolant") → reward
```

---

## QUICK REFERENCE — ALL REMOTES AT A GLANCE

| Remote Name | Type | Direction | Fires When |
|---|---|---|---|
| `ac_robert_dialog` | RemoteEvent | Server → Client | Player triggers Robert's ProximityPrompt |
| `ClientsideAnimations` | RemoteEvent | Server → All Clients | Before/after cutscenes or overload sequences |
| `ClientTweening` | RemoteEvent | Server → All Clients | Server wants to tween world instances on clients |
| `LocalTweening` | RemoteEvent | Both directions | Tween system sync (client → server → all clients) |
| `LocalModelTweening` | RemoteEvent | Server → All Clients | Server wants to tween model pivots on clients |
| `Events.ChatWarn` | RemoteEvent | Client → Server | Player says something suspicious in chat |
| `Events.FXHandler` | RemoteEvent | Server → Client(s) | Named visual effect needs to play |
| `UNPLANNEDOVERLOAD.Shakelol` | RemoteEvent | Server → All Clients | Shake pulse during overload |
| `UNPLANNEDOVERLOAD.CAMANTIHECC` | RemoteEvent | Server → All Clients | Camera recovery after overload |
| `UNPLANNEDOVERLOAD.Funny` | RemoteEvent | Server → All Clients | Jumpscare flash during overload |
| `UNPLANNEDOVERLOAD.FovSwap` | RemoteEvent | Server → All Clients | FOV warp during overload |
| `UNPLANNEDOVERLOAD.Cinematic` | RemoteEvent | Server → All Clients | Letterbox bars on/off |
| `UNPLANNEDOVERLOAD.SongShow` | RemoteEvent | Server → All Clients | Display "now playing" HUD |
| `UNPLANNEDOVERLOAD.passiveOverload` | RemoteEvent | Server → All Clients | Warning overlay on/off |
| `CameraShake` (Folder) | Values | Server writes | Continuous: server tweens values, clients read per-frame |
| `Settings.Shake` | StringValue | Server writes | Accessibility: "on" / "reduced" / "off" |

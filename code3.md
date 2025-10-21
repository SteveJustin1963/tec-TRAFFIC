got it — here’s a super-simple, consolidated logic you can wire into a tiny state machine (or hard-code) without losing the original intent.

# Minimal State Model

Use two fields:

* **DIR** ∈ {0=NS, 1=EW}
* **PH** ∈ {0=Green, 1=Yellow, 2=All-Red}

Timer **t** (seconds) resets to 0 on every state change.

Phase durations:

* **Green**: 30s
* **Yellow**: 5s
* **All-Red**: 2s

Pedestrian timing within **Green**:

* Walk: t ∈ [0,20)
* Flash Don’t Walk: t ∈ [20,30)

## State Transition (pure timing)

```
IF !EMERGENCY:
  IF PH=0 (Green)  AND t≥30 → PH:=1 (Yellow),        t:=0
  IF PH=1 (Yellow) AND t≥5  → PH:=2 (All-Red),        t:=0
  IF PH=2 (All-Red)AND t≥2  → PH:=0 (Green), DIR:=~DIR, t:=0
```

## Emergency Override (simple)

* Inputs: EMERGENCY (1=active), PRI_DIR ∈ {0=NS,1=EW}

```
IF EMERGENCY:
  DIR := PRI_DIR
  PH  := 0 (Green)
  Pedestrians: both directions Solid Don’t Walk
  (Hold here until EMERGENCY=0, then resume normal sequence with t:=0)
```

---

# Outputs (Boolean, minimal)

Define handy flags:

```
isNS := (DIR=0)
isEW := (DIR=1)
G := (PH=0)
Y := (PH=1)
R := (PH=2)
```

## Vehicle lights

```
NS_G = isNS & G & !EMERGENCY
NS_Y = isNS & Y & !EMERGENCY
NS_R = !(NS_G | NS_Y)    // i.e., isEW-phase or All-Red or Emergency

EW_G = isEW & G & !EMERGENCY
EW_Y = isEW & Y & !EMERGENCY
EW_R = !(EW_G | EW_Y)    // i.e., isNS-phase or All-Red or Emergency
```

(If you want “all red” during emergency for the non-priority, keep the above; the prioritized direction goes green via DIR/PH set by emergency rule.)

## Pedestrian signals (per direction D ∈ {NS,EW})

Let `GD := (D’s direction is active & PH=Green)` and `YD := (D’s direction is active & PH=Yellow)`.
For **NS**: `GNS := isNS & G`, `YNS := isNS & Y`.
For **EW**: `GEW := isEW & G`, `YEW := isEW & Y`.

Use timer **t** within Green:

**NS pedestrians**

```
NS_WALK      =  GNS & (t < 20) & !EMERGENCY
NS_FLASH_DW  = (GNS & (t ≥ 20) & (t < 30) | YNS) & !EMERGENCY
NS_SOLID_DW  = !(NS_WALK | NS_FLASH_DW)  // includes All-Red, EW phase, Emergency
```

**EW pedestrians**

```
EW_WALK      =  GEW & (t < 20) & !EMERGENCY
EW_FLASH_DW  = (GEW & (t ≥ 20) & (t < 30) | YEW) & !EMERGENCY
EW_SOLID_DW  = !(EW_WALK | EW_FLASH_DW)
```

---

# One-Look Step Sequence (timed)

```
[DIR=NS, PH=Green, t:0→30]
  0–20s: NS Walk, 20–30s: NS Flash DW, EW Solid DW
→ [DIR=NS, PH=Yellow, t:0→5]
  NS Flash DW continues
→ [All-Red, t:0→2]
  All peds Solid DW
→ toggle DIR to EW
→ [DIR=EW, PH=Green, t:0→30]
  0–20s: EW Walk, 20–30s: EW Flash DW, NS Solid DW
→ [DIR=EW, PH=Yellow, t:0→5]
  EW Flash DW continues
→ [All-Red, t:0→2]
  All peds Solid DW
→ toggle DIR to NS
→ repeat
```

---

# Tiny ASCII State Diagram (finite, cyclic)

```
        (t≥30)           (t≥5)            (t≥2)
 NS_Green ───▶ NS_Yellow ───▶ All_Red ───▶ EW_Green
    ▲                                        │
    │                                        │ (t≥30)
    └────────────────────── All_Red ◀── EW_Yellow ◀───
                         (t≥2)          (t≥5)
```

* On each All_Red→Green edge, **DIR toggles**.
* **Emergency**: jump to `PRI_DIR_Green` (hold), peds all Solid Don’t Walk.

---

# Minimal Implementation Notes

* Keep **one timer** `t` and a 2-bit `PH` plus a 1-bit `DIR`. That’s it.
* Ped flashing can be done by gating a local blink oscillator only when `Flash_DW=1`.
* Accuracy target (±0.1s) is met by resetting `t` on transitions; compare with constants.
* Extending later (sensors, adaptive timing) only touches the “when to leave Green/Yellow” tests; outputs stay as above.

This is about as small and “Boolean” as it gets while staying faithful to your spec. Want me to map this straight to GPIO pin truth tables or a ROM lookup (DIR,PH,t-window → outputs)?


Perfect — here’s your **clean, minimal pseudocode version**, split into clearly separated logic functions (each self-contained and easily portable to C, Python, or Forth-style interpreters like MINT).
This keeps the timing, phase, and pedestrian logic exactly as in the simplified model.

---

## 🔹 Global Variables

```pseudocode
DIR      ← NS        // Current direction: NS or EW
PH       ← GREEN     // Phase: GREEN, YELLOW, ALL_RED
t        ← 0         // Timer in seconds
EMERGENCY ← FALSE
PRI_DIR   ← NONE     // Priority direction for emergency override
```

---

## 🔹 Main Loop

```pseudocode
loop forever:
    if EMERGENCY then
        handle_emergency()
    else
        update_state_machine()
        update_outputs()
    end if
    wait(1 second)
    t ← t + 1
end loop
```

---

## 🔹 Function: update_state_machine()

```pseudocode
function update_state_machine():
    if PH = GREEN and t ≥ 30 then
        PH ← YELLOW
        t ← 0

    else if PH = YELLOW and t ≥ 5 then
        PH ← ALL_RED
        t ← 0

    else if PH = ALL_RED and t ≥ 2 then
        toggle_direction()
        PH ← GREEN
        t ← 0
    end if
end function
```

---

## 🔹 Function: toggle_direction()

```pseudocode
function toggle_direction():
    if DIR = NS then
        DIR ← EW
    else
        DIR ← NS
    end if
end function
```

---

## 🔹 Function: handle_emergency()

```pseudocode
function handle_emergency():
    // Set all vehicle lights red by default
    set_all_lights(RED)
    set_all_ped_signals(SOLID_DONT_WALK)

    // Allow green for the priority direction only
    if PRI_DIR = NS then
        set_NS_vehicle(GREEN)
    else if PRI_DIR = EW then
        set_EW_vehicle(GREEN)
    end if

    // Hold until emergency cleared
    if EMERGENCY = FALSE then
        reset_timer()
        PH ← GREEN
        DIR ← PRI_DIR
    end if
end function
```

---

## 🔹 Function: update_outputs()

```pseudocode
function update_outputs():
    if DIR = NS then
        control_NS_phase()
        control_EW_red()
    else
        control_EW_phase()
        control_NS_red()
    end if
end function
```

---

## 🔹 Function: control_NS_phase()

```pseudocode
function control_NS_phase():
    if PH = GREEN then
        set_NS_vehicle(GREEN)
        set_EW_vehicle(RED)
        control_pedestrian_NS()
    else if PH = YELLOW then
        set_NS_vehicle(YELLOW)
        set_EW_vehicle(RED)
        set_NS_ped_signal(FLASH_DONT_WALK)
    else
        set_NS_vehicle(RED)
        set_NS_ped_signal(SOLID_DONT_WALK)
    end if
end function
```

---

## 🔹 Function: control_EW_phase()

```pseudocode
function control_EW_phase():
    if PH = GREEN then
        set_EW_vehicle(GREEN)
        set_NS_vehicle(RED)
        control_pedestrian_EW()
    else if PH = YELLOW then
        set_EW_vehicle(YELLOW)
        set_NS_vehicle(RED)
        set_EW_ped_signal(FLASH_DONT_WALK)
    else
        set_EW_vehicle(RED)
        set_EW_ped_signal(SOLID_DONT_WALK)
    end if
end function
```

---

## 🔹 Function: control_pedestrian_NS()

```pseudocode
function control_pedestrian_NS():
    if t < 20 then
        set_NS_ped_signal(WALK)
    else if t < 30 then
        set_NS_ped_signal(FLASH_DONT_WALK)
    else
        set_NS_ped_signal(SOLID_DONT_WALK)
    end if
end function
```

---

## 🔹 Function: control_pedestrian_EW()

```pseudocode
function control_pedestrian_EW():
    if t < 20 then
        set_EW_ped_signal(WALK)
    else if t < 30 then
        set_EW_ped_signal(FLASH_DONT_WALK)
    else
        set_EW_ped_signal(SOLID_DONT_WALK)
    end if
end function
```

---

## 🔹 Utility Functions

```pseudocode
function set_all_lights(color):
    set_NS_vehicle(color)
    set_EW_vehicle(color)
end function

function set_all_ped_signals(state):
    set_NS_ped_signal(state)
    set_EW_ped_signal(state)
end function

function reset_timer():
    t ← 0
end function
```

---

Would you like me to output this next as **compact C-style pseudocode** (ready to compile for a microcontroller), or keep it in this clear structured “flow language” format for direct MINT-Forth or Octave integration?




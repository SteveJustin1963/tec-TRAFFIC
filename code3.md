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


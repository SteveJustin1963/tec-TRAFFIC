# tec-TRAFFIC
TEC-1 traffic controller

```
// Traffic Light Controller - Correct Mint2 Syntax
// Uses only integer operations and stack manipulation

// Variables (single letters only in Mint2)
// s = state (0-5)
// t = timer counter
// l = lights output pattern
// e = emergency flag

// Initialize system
:I
  2 s!          // Start at NS_RED state
  0 t!          // Clear timer
  9 l!          // All red pattern (1001 binary)
  l 1 /O        // Output l to port 1
;

// Timer tick increment
:T 
  t 1 + t!      // Get t, add 1, store back to t
;

// Timer reset  
:R 
  0 t!          // Store 0 in timer
;

// Output lights (pattern on stack)
:L 
  l!            // Store pattern in l
  l 1 /O        // Output l to port 1
;

// State 0: NS Green
:A
  s 0 = (       // If state equals 0
    4 L;        // Call output function for NS green, EW red
    t 300 > (   // If timer > 300
      1 s! R;   // Set state to 1 and call reset
    )
  )
;

// State 1: NS Yellow  
:B
  s 1 = (       // If state equals 1
    2 L;        // Call output function for NS yellow, EW red
    t 50 > (    // If timer > 50
      2 s! R;   // Set state to 2 and call reset
    )
  )
;

// State 2: NS Red (clearance)
:C
  s 2 = (       // If state equals 2
    9 L;        // Call output function for all red
    t 20 > (    // If timer > 20
      3 s! R;   // Set state to 3 and call reset
    )
  )
;

// State 3: EW Green
:E
  s 3 = (       // If state equals 3
    1 L;        // Call output function for EW green, NS red
    t 300 > (   // If timer > 300
      4 s! R;   // Set state to 4 and call reset
    )
  )
;

// State 4: EW Yellow
:F
  s 4 = (       // If state equals 4
    8 L;        // Call output function for EW yellow, NS red
    t 50 > (    // If timer > 50
      5 s! R;   // Set state to 5 and call reset
    )
  )
;

// State 5: EW Red (clearance)
:H
  s 5 = (       // If state equals 5
    9 L;        // Call output function for all red
    t 20 > (    // If timer > 20
      0 s! R;   // Set state to 0 and call reset
    )
  )
;

// Process all states
:P A; B; C; E; F; H; ;

// Emergency override (dir--)
// dir: 0=give NS green, 1=give EW green
:O
  1 e!          // Set emergency flag
  9 L           // All red first
  
  0 = (         // If dir=0 (NS emergency)
    4 L         // NS green pattern
  ) /E (        // Else (EW emergency)
    1 L         // EW green pattern
  )
  
  // Wait for manual clear
  [ e 0 = /U ]  // Loop until emergency cleared
  
  2 s! R        // Return to NS_RED state
;

// Clear emergency
:X 0 e! ;

// Main program loop
:M
  I             // Initialize
  
  [             // Main loop
    T;          // Call timer increment function
    
    e 0 = (     // If not emergency
      P;        // Call process states function
    )
    
    1           // Continue loop
  ]
;

// Debug: show state and timer
:S
  s . 32 /C     // Print state and space
  t . 10 /C     // Print timer and newline
;

// Run the controller
M
```





1. **Main State Flow**: The complete cycle through all 6 states with timing and light conditions
2. **Emergency Override Flow**: How the emergency system interrupts normal operation
3. **State Transitions Diagram**: A compact view of the state sequence
4. **Light Output Patterns**: Binary patterns for each state
5. **Mint2 Function Map**: How the functions relate to the flowchart

The flowchart clearly shows:
- The cyclic nature of the traffic controller
- All red clearance periods between direction changes (safety feature)
- Timer-based state transitions
- Pedestrian signal coordination with traffic lights
- Emergency override capability that can prioritize either direction

Each state box shows what lights are active and the timer duration, making it easy to follow the logic flow and understand how the Mint2 implementation maps to the actual traffic control sequence.


```
Traffic Light Controller State Machine
                    ========================================

                              ┌─────────────┐
                              │   START     │
                              │ Initialize  │
                              └──────┬──────┘
                                     │
                                     ▼
                         ┌──────────────────────┐
                         │   NS_RED State       │
                         │   ================   │
                         │   NS: RED            │
                         │   EW: RED            │
                         │   All Pedestrian:    │
                         │      DON'T WALK      │
                         │   Timer: 2 sec       │
                         └──────────┬───────────┘
                                    │
                                    │ Timer >= 2s
                                    ▼
                         ┌──────────────────────┐
                         │   NS_GREEN State     │
                         │   ================   │
                         │   NS: GREEN          │
                         │   EW: RED            │
                         │   NS Ped: WALK      │
                         │   EW Ped: DON'T WALK│
                         │   Timer: 30 sec      │
                         └──────────┬───────────┘
                                    │
                                    │ Timer >= 30s
                                    ▼
                         ┌──────────────────────┐
                         │   NS_YELLOW State    │
                         │   ================   │
                         │   NS: YELLOW         │
                         │   EW: RED            │
                         │   NS Ped: FLASHING  │
                         │   EW Ped: DON'T WALK│
                         │   Timer: 5 sec       │
                         └──────────┬───────────┘
                                    │
                                    │ Timer >= 5s
                                    ▼
                         ┌──────────────────────┐
                         │   NS_RED State       │
                         │   ================   │
                         │   NS: RED            │
                         │   EW: RED            │
                         │   All Pedestrian:    │
                         │      DON'T WALK      │
                         │   Timer: 2 sec       │
                         └──────────┬───────────┘
                                    │
                                    │ Timer >= 2s
                                    ▼
                         ┌──────────────────────┐
                         │   EW_GREEN State     │
                         │   ================   │
                         │   NS: RED            │
                         │   EW: GREEN          │
                         │   NS Ped: DON'T WALK│
                         │   EW Ped: WALK      │
                         │   Timer: 30 sec      │
                         └──────────┬───────────┘
                                    │
                                    │ Timer >= 30s
                                    ▼
                         ┌──────────────────────┐
                         │   EW_YELLOW State    │
                         │   ================   │
                         │   NS: RED            │
                         │   EW: YELLOW         │
                         │   NS Ped: DON'T WALK│
                         │   EW Ped: FLASHING  │
                         │   Timer: 5 sec       │
                         └──────────┬───────────┘
                                    │
                                    │ Timer >= 5s
                                    ▼
                         ┌──────────────────────┐
                         │   EW_RED State       │
                         │   ================   │
                         │   NS: RED            │
                         │   EW: RED            │
                         │   All Pedestrian:    │
                         │      DON'T WALK      │
                         │   Timer: 2 sec       │
                         └──────────┬───────────┘
                                    │
                                    │ Timer >= 2s
                                    │
                                    └────────────┐
                                                 │
                    ┌────────────────────────────┘
                    │
                    ▼
            [Loop back to NS_GREEN]


═══════════════════════════════════════════════════════════════════════

                        EMERGENCY OVERRIDE FLOW
                        =====================

    ┌─────────────────┐
    │  EMERGENCY      │
    │  TRIGGERED      │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │  Set ALL RED    │
    │  All lights: RED│
    │  All ped: DON'T │
    └────────┬────────┘
             │
             ▼
        ┌────┴────┐
        │Direction│
        │   NS?   │
        └─┬────┬──┘
          │    │
      Yes │    │ No
          │    │
          ▼    ▼
    ┌──────┐  ┌──────┐
    │NS    │  │EW    │
    │GREEN │  │GREEN │
    └──┬───┘  └───┬──┘
       │          │
       └────┬─────┘
            │
            ▼
    ┌─────────────────┐
    │  Wait for Reset │
    │  or Timeout     │
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │  Resume Normal  │
    │  Return to      │
    │  NS_RED State   │
    └─────────────────┘

═══════════════════════════════════════════════════════════════════════

                           STATE TRANSITIONS
                           ================

    State Sequence (Normal Operation):
    ────────────────────────────────────

    NS_RED(2s) ──► NS_GREEN(30s) ──► NS_YELLOW(5s) ──┐
         ▲                                            │
         │                                            ▼
    EW_RED(2s) ◄── EW_YELLOW(5s) ◄── EW_GREEN(30s) ◄─ NS_RED(2s)


    Light Output Patterns (Binary):
    ──────────────────────────────────
    
    State        NS Lights  EW Lights  NS Ped   EW Ped
    ─────────    ─────────  ─────────  ───────  ───────
    NS_GREEN     G (100)    R (001)    Walk     Don't
    NS_YELLOW    Y (010)    R (001)    Flash    Don't
    NS_RED       R (001)    R (001)    Don't    Don't
    EW_GREEN     R (001)    G (100)    Don't    Walk
    EW_YELLOW    R (001)    Y (010)    Don't    Flash
    EW_RED       R (001)    R (001)    Don't    Don't

    Timer Events:
    ─────────────
    • Each state monitors timer
    • State change resets timer
    • Timer increments each tick
    • Transitions occur when timer >= threshold

═══════════════════════════════════════════════════════════════════════

                          MINT2 FUNCTION MAP
                          ==================

    Function  Description              Trigger
    ────────  ──────────────────────  ─────────────
    :M        Main loop                Entry point
    :I        Initialize controller    Called by :M
    :T        Timer tick               Called each loop
    :P        Process states           Called by :M
    :A        Handle NS_GREEN state    Called by :P
    :B        Handle NS_YELLOW state   Called by :P
    :C        Handle NS_RED state      Called by :P
    :E        Handle EW_GREEN state    Called by :P
    :F        Handle EW_YELLOW state   Called by :P
    :H        Handle EW_RED state      Called by :P
    :O        Emergency override       Manual trigger
    :X        Clear emergency          Manual trigger
    :R        Reset timer              State changes
    :L        Set light output         Each state
    :S        Show debug info          Manual trigger
```



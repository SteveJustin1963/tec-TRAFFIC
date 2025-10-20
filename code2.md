# tec-TRAFFIC
TEC-1 traffic controller

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


```
// Traffic Light Controller - Proper Mint2 Implementation
// Uses only integer operations and stack manipulation

// Variables (allocated at start)
0 s!              // state
0 t!              // timer  
0 l!              // lights output
0 e!              // emergency flag

// Constants using immediate values
// States: 0=NS_G, 1=NS_Y, 2=NS_R, 3=EW_G, 4=EW_Y, 5=EW_R

// Initialize system
:I
  2 s!            // Start at NS_RED
  0 t!            // Clear timer
  9 l!            // All red pattern (1001 binary)
  l@ 1 /O         // Output to port 1
;

// Timer tick increment
:T t@ 1+ t! ;

// Timer reset  
:R 0 t! ;

// Output lights (pattern--)
:L l! l@ 1 /O ;

// State transitions based on timer
// Green time check (--flag)
:G t@ 300 > ;

// Yellow time check (--flag)  
:Y t@ 50 > ;

// Red clearance time check (--flag)
:D t@ 20 > ;

// State 0: NS Green
:A
  s@ 0 = (         // If state is NS_GREEN
    4 L            // Output NS green, EW red (0100)
    G (            // If green time exceeded
      1 s! R       // Go to NS_YELLOW and reset
    )
  )
;

// State 1: NS Yellow  
:B
  s@ 1 = (         // If state is NS_YELLOW
    2 L            // Output NS yellow, EW red (0010)
    Y (            // If yellow time exceeded
      2 s! R       // Go to NS_RED and reset
    )
  )
;

// State 2: NS Red (clearance)
:C
  s@ 2 = (         // If state is NS_RED
    9 L            // Output all red (1001)
    D (            // If clearance time exceeded
      3 s! R       // Go to EW_GREEN and reset
    )
  )
;

// State 3: EW Green
:E
  s@ 3 = (         // If state is EW_GREEN
    1 L            // Output EW green, NS red (0001)
    G (            // If green time exceeded
      4 s! R       // Go to EW_YELLOW and reset
    )
  )
;

// State 4: EW Yellow
:F
  s@ 4 = (         // If state is EW_YELLOW  
    8 L            // Output EW yellow, NS red (1000)
    Y (            // If yellow time exceeded
      5 s! R       // Go to EW_RED and reset
    )
  )
;

// State 5: EW Red (clearance)
:H
  s@ 5 = (         // If state is EW_RED
    9 L            // Output all red (1001)
    D (            // If clearance time exceeded
      0 s! R       // Go to NS_GREEN and reset
    )
  )
;

// Process all states
:P A B C E F H ;

// Emergency override (dir--)
// dir: 0=give NS green, 1=give EW green
:O
  1 e!             // Set emergency flag
  9 L              // All red first
  
  0 = (            // If dir=0 (NS emergency)
    4 L            // NS green pattern
  ) /E (           // Else (EW emergency)
    1 L            // EW green pattern
  )
  
  // Wait for manual clear
  [ e@ 0 = /U ]    // Loop until emergency cleared
  
  2 s! R           // Return to NS_RED state
;

// Clear emergency
:X 0 e! ;

// Main program loop
:M
  I                // Initialize
  
  [                // Main loop
    T              // Increment timer
    
    e@ 0 = (       // If not emergency
      P            // Process states
    )
    
    1              // Continue loop
  ]
;

// Debug: show state and timer
:S
  s@ . 32 /C       // Print state and space
  t@ . 10 /C       // Print timer and newline
;

// Run the controller
M



```



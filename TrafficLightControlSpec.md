# Traffic Light Intersection Control Specification

## 1. Overview
This specification defines the requirements and logic for a software system to control a four-way traffic light intersection. The system manages vehicle traffic lights (red, yellow, green) and pedestrian signals (walk, don't walk) for each direction, ensuring safe and efficient traffic flow.

## 2. Requirements
### 2.1 Functional Requirements
- **Intersection Layout**: Four-way intersection with directions labeled North-South (NS) and East-West (EW).
- **Traffic Lights**: Each direction has three vehicle lights (red, yellow, green) and two pedestrian signals (walk, don't walk).
- **Timing Control**:
  - Green light duration: 30 seconds per direction.
  - Yellow light duration: 5 seconds per direction.
  - Red light duration: Sufficient to allow the opposing direction’s green and yellow phases (minimum 35 seconds).
  - Pedestrian walk signal: Active for 20 seconds during the corresponding green phase, followed by a 10-second flashing "don't walk" before transitioning to solid "don't walk."
- **Safety**:
  - Only one direction (NS or EW) can have a green or yellow light at a time.
  - All lights in a direction must be red before the opposing direction transitions to green.
  - Pedestrian signals must align with vehicle light states (e.g., walk signal active only during green light in the same direction).
- **State Transitions**:
  - Sequence: Green → Yellow → Red → Green (for opposing direction).
  - Pedestrian sequence: Walk → Flashing Don't Walk → Solid Don't Walk.
- **Emergency Override**: Allow manual override or priority mode (e.g., for emergency vehicles) to set all directions to red except the prioritized direction.

### 2.2 Non-Functional Requirements
- **Reliability**: The system must run continuously without downtime, handling state transitions accurately.
- **Timing Accuracy**: Light and pedestrian signal timings must be precise within ±0.1 seconds.
- **Extensibility**: The system should support future additions, such as sensor inputs (e.g., vehicle detection) or adaptive timing.
- **Hardware Interface**: The software must interface with physical traffic light hardware via a defined API or GPIO pins (implementation-specific).

## 3. Assumptions
- The intersection is a standard four-way cross with no left-turn signals or special lanes.
- No external sensors (e.g., vehicle or pedestrian detection) are required for the base implementation.
- The system operates on a real-time clock for timing control.
- Hardware supports immediate state changes for lights and signals.

## 4. System Design
### 4.1 Components
- **TrafficLightController**: Manages the overall state machine and coordinates light transitions.
- **Light**: Represents a single light (red, yellow, green) with on/off methods.
- **PedestrianSignal**: Manages walk/don’t walk signals, including flashing behavior.
- **Timer**: Tracks elapsed time for each phase and triggers state changes.
- **EmergencyHandler**: Processes override requests for emergency vehicle priority.

### 4.2 State Machine
The system uses a finite state machine with the following states:
1. **NS_Green**: NS green light, EW red light, NS pedestrian walk, EW pedestrian don’t walk.
2. **NS_Yellow**: NS yellow light, EW red light, NS pedestrian flashing don’t walk, EW pedestrian don’t walk.
3. **NS_Red**: NS red light, EW red light, all pedestrian signals don’t walk (all-red phase for safety).
4. **EW_Green**: EW green light, NS red light, EW pedestrian walk, NS pedestrian don’t walk.
5. **EW_Yellow**: EW yellow light, NS red light, EW pedestrian flashing don’t walk, NS pedestrian don’t walk.
6. **EW_Red**: EW red light, NS red light, all pedestrian signals don’t walk (all-red phase for safety).

### 4.3 Timing Configuration
| State        | Duration (seconds) | Vehicle Lights        | Pedestrian Signals       |
|--------------|--------------------|-----------------------|--------------------------|
| NS_Green     | 30                 | NS: Green, EW: Red    | NS: Walk, EW: Don’t Walk |
| NS_Yellow    | 5                  | NS: Yellow, EW: Red   | NS: Flashing, EW: Don’t  |
| NS_Red       | 2                  | NS: Red, EW: Red      | All: Don’t Walk          |
| EW_Green     | 30                 | EW: Green, NS: Red    | EW: Walk, NS: Don’t Walk |
| EW_Yellow    | 5                  | EW: Yellow, NS: Red   | EW: Flashing, NS: Don’t  |
| EW_Red       | 2                  | EW: Red, NS: Red      | All: Don’t Walk          |

### 4.4 Pseudocode
```pseudocode
initialize TrafficLightController:
  set NS lights: red
  set EW lights: red
  set all pedestrian signals: don’t walk
  set state: NS_Red
  start Timer

main loop:
  switch (current_state):
    case NS_Green:
      set NS lights: green
      set EW lights: red
      set NS pedestrian: walk
      set EW pedestrian: don’t walk
      if Timer.elapsed >= 30 seconds:
        change_state(NS_Yellow)
        reset Timer
    case NS_Yellow:
      set NS lights: yellow
      set EW lights: red
      set NS pedestrian: flashing don’t walk
      set EW pedestrian: don’t walk
      if Timer.elapsed >= 5 seconds:
        change_state(NS_Red)
        reset Timer
    case NS_Red:
      set NS lights: red
      set EW lights: red
      set all pedestrian signals: don’t walk
      if Timer.elapsed >= 2 seconds:
        change_state(EW_Green)
        reset Timer
    case EW_Green:
      set EW lights: green
      set NS lights: red
      set EW pedestrian: walk
      set NS pedestrian: don’t walk
      if Timer.elapsed >= 30 seconds:
        change_state(EW_Yellow)
        reset Timer
    case EW_Yellow:
      set EW lights: yellow
      set NS lights: red
      set EW pedestrian: flashing don’t walk
      set NS pedestrian: don’t walk
      if Timer.elapsed >= 5 seconds:
        change_state(EW_Red)
        reset Timer
    case EW_Red:
      set EW lights: red
      set NS lights: red
      set all pedestrian signals: don’t walk
      if Timer.elapsed >= 2 seconds:
        change_state(NS_Green)
        reset Timer

function change_state(new_state):
  current_state = new_state
  reset Timer

function emergency_override(direction):
  set all lights: red
  set all pedestrian signals: don’t walk
  set direction lights: green
  wait for manual reset or timeout
  resume normal state machine
```

## 5. Interface Specifications
- **Light Control API**:
  - `set_light(direction, color, state)`: Sets the specified light (red, yellow, green) in the given direction to on/off.
  - Example: `set_light("NS", "green", true)`
- **Pedestrian Signal API**:
  - `set_pedestrian_signal(direction, state)`: Sets the pedestrian signal to walk, flashing don’t walk, or solid don’t walk.
  - Example: `set_pedestrian_signal("EW", "walk")`
- **Timer API**:
  - `start_timer()`: Starts the timer.
  - `reset_timer()`: Resets the timer to zero.
  - `get_elapsed_time()`: Returns elapsed time in seconds.
- **Emergency API**:
  - `trigger_emergency(direction)`: Activates emergency mode for the specified direction.
  - `reset_emergency()`: Returns to normal operation.

## 6. Error Handling
- If a light fails to change state, log the error and attempt to retry once before setting all lights to red and halting.
- If the timer fails, log the error and set all lights to red until the timer is restored.
- Invalid state transitions (e.g., green to green) should be rejected with an error log.

## 7. Testing Considerations
- **Unit Tests**:
  - Verify each light and pedestrian signal can be set to all valid states.
  - Test timer accuracy for all phase durations.
- **Integration Tests**:
  - Simulate a full cycle of state transitions (NS_Green → EW_Red).
  - Test emergency override and recovery.
- **Edge Cases**:
  - Handle power interruptions by resuming in a safe state (all red).
  - Test rapid emergency override requests.

## 8. Future Enhancements
- Add sensor inputs for adaptive timing based on traffic density.
- Support left-turn signals or additional phases.
- Integrate with a central traffic management system for coordinated control.

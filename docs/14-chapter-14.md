# Chapter 14: Implementing a `HealthComponent` with Signals

## Goal

The goal of this chapter is to finalize and solidify our **`HealthComponent`** implementation. We will ensure it robustly manages an entity's health state, correctly emits `health_changed` and `died` signals for other components to react to, and adheres fully to our `Component` base class and data-driven principles. This reinforces how components encapsulate specific behaviors and communicate effectively in a decoupled manner.

## Concept Explanation: The `HealthComponent`'s Responsibility

A `HealthComponent` has a very clear and singular responsibility: to manage the health attribute of its owning `Entity`. This includes:

*   **Storing Health Data**: Keeping track of current health and maximum health.
*   **Modifying Health**: Providing methods to `take_damage()` and `heal()`.
*   **Emitting Events**: Notifying other systems when health changes or when the entity dies, using Godot signals.
*   **Utilizing Data**: Potentially reading initial max health from a `HealthData` resource (which we'll introduce in the next chapter).

It's crucial that the `HealthComponent` **does not** contain logic for *what happens* when an entity dies (e.g., playing an animation, showing a game over screen, despawning). Its only concern is *that* the entity died. Other components or systems (like a `PlayerController`, `EnemyAI`, or `GameManager`) will *listen* for the `died` signal and react accordingly. This is the essence of loose coupling.

### Reviewing Our Existing `HealthComponent`

We already created a `HealthComponent` in Chapter 6 and refactored it in Chapter 12 to extend our `Component` base. In this chapter, we'll review it, ensure its `_on_entity_ready()` is correctly handling initialization, and confirm its signal emission logic is robust. We'll also briefly discuss how it will eventually integrate with a `HealthData` resource.

## Architectural Reasoning: Event-Driven State Management

The `HealthComponent` is a prime example of **event-driven state management**. Its internal state (current health) changes, and it broadcasts these changes as events (signals) without knowing or caring who is listening.

*   **Producer-Consumer Model**: The `HealthComponent` is a "producer" of health-related events. Any other component or system can be a "consumer" by connecting to its signals.
*   **Observability**: Any part of the game that needs to know about an entity's health can "observe" the `HealthComponent`'s signals. This is far superior to constantly polling for health values.
*   **Decoupled Reactions**: Player UI, enemy AI, particle effects, sound effects, and game over logic can all react to health changes independently, without the `HealthComponent` needing direct references to any of them. This is vital for moddability, as modders can easily add new reactions to existing health events.

## Production Mindset Notes: Robustness and Clarity

*   **Clamping Values**: Always ensure health values are clamped within valid ranges (0 to `max_health`).
*   **Clear Signal Semantics**: The names of your signals and their parameters should clearly convey the event and its relevant data.
*   **Idempotence (for `died` signal)**: The `died` signal should ideally only be emitted once when health transitions from >0 to 0, not repeatedly while health is at 0. Our current implementation handles this.
*   **Editor Configurability**: Using `@export` for `max_health` allows designers to configure health values directly in the editor.

## Step-by-Step Instructions: Finalizing the `HealthComponent`

Our `HealthComponent` is already quite solid from previous chapters. This chapter will focus on reviewing it, making minor refinements, and ensuring its integration into our current architecture is perfect. We'll also prepare it for a `HealthData` resource.

### 1. Review and Refine `HealthComponent.gd`

Open `res://scripts/components/HealthComponent.gd`.

Ensure it matches this code, paying close attention to the `extends Component`, `_on_entity_ready()`, and the `set` methods for `max_health` and `_health`.

```gdscript
# HealthComponent.gd
extends Component
class_name HealthComponent # Ensure class_name is set

signal health_changed(current_health: int, max_health: int)
signal died()

# @export var health_data: HealthData # Placeholder for future data-driven health

@export var max_health: int = 100: # Initial max health, can be overridden by HealthData
    set(value):
        max_health = max(0, value) # Ensure max_health is never negative
        # If the component is already active, adjust current health and emit signal
        if is_on_ready() and _health != null: # Check if _health has been initialized
            _health = min(_health, max_health) # Don't exceed new max_health
            health_changed.emit(_health, max_health)

var _health: int = 0: # Initialize to 0, will be set in _on_entity_ready
    set(value):
        var old_health = _health
        _health = clampi(value, 0, max_health) # Clamp health between 0 and max_health

        if _health != old_health: # Only emit if health actually changed
            health_changed.emit(_health, max_health)
            if _health == 0 and old_health > 0: # Only emit died once when health hits 0
                died.emit()
                # Use get_owner_entity().name for better context in print
                print(get_owner_entity().name + " has died!")

func _on_entity_ready():
    super._on_entity_ready() # Call the base Component's _on_entity_ready
    if not is_instance_valid(get_owner_entity()): return # Safety check

    # Initialize current health.
    # If a HealthData resource were present, we'd use health_data.max_health here.
    # For now, it defaults to the @export var max_health.
    self.health = max_health # Use the setter to ensure _health is clamped and signal is emitted

    print("HealthComponent on '" + get_owner_entity().name + "' ready. Max health: " + str(max_health))

func take_damage(amount: int):
    if amount <= 0: return # No negative or zero damage
    if _health > 0: # Only take damage if not already dead
        self.health -= amount
        print(get_owner_entity().name + " took " + str(amount) + " damage. Current health: " + str(_health))

func heal(amount: int):
    if amount <= 0: return # No negative or zero healing
    if _health < max_health: # Only heal if not at max health
        self.health += amount
        print(get_owner_entity().name + " healed " + str(amount) + ". Current health: " + str(_health))

func get_health() -> int:
    return _health

func get_max_health() -> int:
    return max_health
```

*   **`_health: int = 0`**: Explicitly initialize `_health` to `0` to ensure the setter logic for `_health` and `max_health` works correctly on first assignment in `_on_entity_ready()`.
*   **`self.health = max_health` in `_on_entity_ready()`**: This ensures that when the component is ready, its current health is set to the maximum, and critically, the `health_changed` signal is emitted for initial UI setup or other systems that need to know the starting health.
*   **`get_owner_entity().name`**: Consistent use of our `Component` base helper for better print output.
*   **Placeholder for `health_data`**: The commented-out `@export var health_data: HealthData` shows where we would integrate a data resource for health, which we'll do in the next chapter.

### 2. Verify `Player.tscn` Setup

1.  Open `res://scenes/entities/Player.tscn`.
2.  Select the `HealthComponent` node (child of `Player`).
3.  In the Inspector, ensure its `Max Health` is set to `100` (or your desired default). This value will be used as the initial `max_health` for the component.

### 3. Verify `PlayerController.gd` Connections

Open `res://scripts/components/PlayerController.gd`.

Ensure that the `_on_entity_ready()` method correctly finds the `HealthComponent` and connects to its `health_changed` and `died` signals:

```gdscript
# PlayerController.gd (Relevant excerpt)
# ...

func _on_entity_ready():
    super._on_entity_ready()
    # ... other component finding ...

    _health_component = get_sibling_component(HealthComponent) # Use get_sibling_component
    if _health_component:
        _health_component.health_changed.connect(_on_health_changed)
        _health_component.died.connect(_on_died)
        print("PlayerController on '" + get_owner_entity().name + "': Connected to HealthComponent signals.")
    else:
        push_error("PlayerController on '" + get_owner_entity().name + "': No HealthComponent sibling found.")

    # ... rest of _on_entity_ready ...

# ... _physics_process() ...

func _on_health_changed(current_health: int, max_health: int):
    print("PlayerController: Player Health: " + str(current_health) + "/" + str(max_health))

func _on_died():
    print("PlayerController: Player received 'died' signal. Handling player death!")
    set_process(false)
    set_physics_process(false)
    set_process_input(false)
    # ...
```

*   **`get_sibling_component(HealthComponent)`**: This is the correct and robust way for `PlayerController` to find its sibling `HealthComponent` on the same `Entity`.

### 4. Test the System

1.  Run `res://scenes/core/GameContext.tscn` (F5).
2.  Navigate to the gameplay state (press Enter).
3.  Observe the "Output" panel:
    *   You should see "HealthComponent on 'Player_...' ready. Max health: 100".
    *   You should also see "PlayerController: Connected to HealthComponent signals." and "PlayerController: Player Health: 100/100". This confirms the initial state and connections.
4.  Press `Spacebar` repeatedly to make the player take damage.
    *   Observe the `print` statements confirming damage taken, health changes, and eventually the "died" messages from both `HealthComponent` and `PlayerController`.
    *   Player movement should stop upon death.

## Consistency Check

You have successfully reviewed and finalized the `HealthComponent`, ensuring it correctly manages health, uses our `Component` base, and effectively communicates state changes via signals. The `PlayerController` demonstrates how to cleanly connect to and react to these signals without tight coupling. This robust `HealthComponent` is now a fully functional and reusable building block in our compositional architecture, ready for data-driven configuration.

In the next chapter, we will implement this data-driven aspect by creating `ComponentData` resources to configure various components, starting with our `HealthComponent`.
# Chapter 15: Data-Driven Components: `ComponentData` Resources

## Goal

The goal of this chapter is to fully embrace data-driven design by configuring component behavior and properties using external **`ComponentData` Resource** files instead of hardcoding values or relying solely on `@export` variables within the component script. We will create custom `Resource` types for `HealthData` and `MovementData` (revisiting our existing one), assign these data resources to our `HealthComponent` and `MovementComponent` in the editor, and highlight the benefits for iteration, content creation, and modding.

## Concept Explanation: `ComponentData` Resources

In Chapter 7, we introduced custom resources and created `MovementData.gd`. Now, we're formalizing this concept for *all* components that have configurable properties. A `ComponentData` resource is a `Resource` script specifically designed to hold configuration data for a particular type of component.

For example:

*   **`HealthData.gd`**: Holds `max_health`, `regen_rate`, `invulnerability_duration` for a `HealthComponent`.
*   **`MovementData.gd`**: Holds `speed`, `acceleration`, `friction`, `jump_force` for a `MovementComponent`.
*   **`WeaponData.gd`**: Holds `damage`, `fire_rate`, `ammo_type` for a `WeaponComponent`.

Each component will have an `@export var` property that expects an instance of its corresponding `ComponentData` resource. This allows designers to easily swap out different data sets for the same component, effectively changing its behavior without touching any code.

### Why Data-Driven Components are Crucial for AAA Design:

*   **Content Creation Empowerment**: Designers, not just programmers, can create new enemy types, item variations, or player archetypes by simply creating new data resources and assigning them.
*   **Rapid Prototyping & Iteration**: Quickly test different values for balance, difficulty, or feel without recompiling.
*   **Reduced Code Maintenance**: Component scripts become simpler, focusing purely on *how* to use data, not *what* the data is.
*   **Moddability**: Modders can create entirely new `ComponentData` resources (e.g., a "SuperSpeedMovementData" or "EliteEnemyHealthData") and load them into existing components without modifying the core game logic. This is a powerful extension point.
*   **Scalability**: Adding 100 new enemy types means creating 100 new data resources, not 100 new enemy classes.

## Architectural Reasoning: Externalizing All Configuration

By making components data-driven, we push configuration data out of the script and into external `.tres` files. This creates a clear boundary:

*   **Component Script (`.gd`)**: Defines the *logic* and *interface* (what properties it expects).
*   **Component Data (`.tres`)**: Defines the *values* for those properties (the actual configuration).

The component loads its assigned `ComponentData` resource and uses its values. If no data resource is assigned, the component should ideally fall back to sensible defaults or gracefully disable itself with an error message. This architecture ensures that components are highly reusable and independent of specific content values.

## Production Mindset Notes: Resource Management

*   **Default Values**: Provide sensible default values in your `ComponentData.gd` scripts. This ensures a component still functions if no specific `.tres` file is assigned.
*   **Validation**: Components should validate that their `ComponentData` resource is not `null` and that its properties are within expected ranges.
*   **Read-Only**: `ComponentData` resources should generally be treated as read-only by game logic. If a component needs to modify its own health, that's internal state, not configuration. If you need to modify a resource instance, `.duplicate()` it first.
*   **Type Hinting**: Use type hinting for your `@export var` properties (e.g., `@export var health_data: HealthData`) to leverage Godot's editor features and GDScript's static analysis.

## Step-by-Step Instructions: Data-Driving `HealthComponent` and `MovementComponent`

We've already created `MovementData.gd` in Chapter 7. Now we'll create `HealthData.gd`, integrate it, and ensure both components are fully data-driven.

### 1. Create the `HealthData` Custom Resource Script

1.  In `res://scripts/data_types/`, create a new script named `HealthData.gd`.
2.  Ensure it `extends Resource`.
3.  Add the `class_name` and `@export` properties:

    ```gdscript
    # HealthData.gd
    extends Resource
    class_name HealthData

    @export var max_health: int = 100 # Maximum health for the entity
    @export var invulnerability_duration: float = 0.5 # Duration (seconds) of invulnerability after taking damage
    # Add more health-related properties as needed (e.g., regen_rate, resistance_to_fire)
    ```

    *   **`class_name HealthData`**: Makes it recognizable by name in the editor.
    *   **`@export var max_health`**: Our primary configurable health property.

### 2. Create an Instance of `HealthData` for the Player

1.  In the `FileSystem` dock, navigate to `res://data/game_data/`.
2.  Right-click and select "New Resource...".
3.  In the "Create New Resource" dialog, search for `HealthData`. Select it and click "Create".
4.  Name the new resource file `PlayerHealthData.tres`.
5.  Save it in `res://data/game_data/`.
6.  Select `PlayerHealthData.tres` in the `FileSystem` dock.
7.  In the Inspector dock, set `Max Health` to `100` and `Invulnerability Duration` to `0.5` (or your preferred values).

### 3. Integrate `HealthData` into `HealthComponent.gd`

Now, `HealthComponent` will get its `max_health` from the `HealthData` resource.

1.  Open `res://scripts/components/HealthComponent.gd`.
2.  Modify the script to `@export` a `HealthData` resource and use its `max_health` property. Remove the old `@export var max_health`.

    ```gdscript
    # HealthComponent.gd
    extends Component
    class_name HealthComponent

    signal health_changed(current_health: int, max_health: int)
    signal died()

    @export var health_data: HealthData # NEW: Reference to the HealthData resource

    var _health: int = 0:
        set(value):
            var old_health = _health
            # Use health_data.max_health for clamping
            _health = clampi(value, 0, health_data.max_health if health_data else 1) # Fallback to 1 if no data

            if _health != old_health:
                health_changed.emit(_health, health_data.max_health if health_data else 1)
                if _health == 0 and old_health > 0:
                    died.emit()
                    print(get_owner_entity().name + " has died!")

    # Internal timer for invulnerability
    var _invulnerability_timer: Timer = null
    var _is_invulnerable: bool = false

    func _on_entity_ready():
        super._on_entity_ready()
        if not is_instance_valid(get_owner_entity()): return

        if health_data == null:
            push_error("HealthComponent on '" + get_owner_entity().name + "': HealthData resource not assigned. Disabling component.")
            set_process(false)
            return
        
        # Initialize invulnerability timer
        _invulnerability_timer = Timer.new()
        add_child(_invulnerability_timer)
        _invulnerability_timer.one_shot = true
        _invulnerability_timer.timeout.connect(_on_invulnerability_timeout)

        # Initialize current health using health_data.max_health
        self.health = health_data.max_health # Use the setter to ensure _health is clamped and signal is emitted

        print("HealthComponent on '" + get_owner_entity().name + "' ready. Max health: " + str(health_data.max_health))

    func take_damage(amount: int):
        if amount <= 0: return
        if _health > 0 and not _is_invulnerable: # Only take damage if not dead and not invulnerable
            self.health -= amount
            print(get_owner_entity().name + " took " + str(amount) + " damage. Current health: " + str(_health))
            
            # Start invulnerability timer if duration is set
            if health_data.invulnerability_duration > 0:
                _is_invulnerable = true
                _invulnerability_timer.start(health_data.invulnerability_duration)
                print(get_owner_entity().name + " is now invulnerable for " + str(health_data.invulnerability_duration) + " seconds.")

    func heal(amount: int):
        if amount <= 0: return
        if _health < health_data.max_health: # Use health_data.max_health
            self.health += amount
            print(get_owner_entity().name + " healed " + str(amount) + ". Current health: " + str(_health))

    func get_health() -> int:
        return _health

    func get_max_health() -> int:
        return health_data.max_health if health_data else 1 # Return max_health from data

    func _on_invulnerability_timeout():
        _is_invulnerable = false
        print(get_owner_entity().name + " is no longer invulnerable.")
    ```

    *   **`@export var health_data: HealthData`**: New export variable to assign the resource.
    *   **Removed `@export var max_health`**: The `max_health` property is now entirely managed by the `HealthData` resource.
    *   **`health_data.max_health`**: Used throughout the script for clamping and signal emission. Added a fallback `1` for safety if `health_data` is `null`.
    *   **Invulnerability**: Added basic invulnerability logic using a `Timer` and `health_data.invulnerability_duration`, demonstrating another data-driven property.

### 4. Integrate `MovementData` into `MovementComponent.gd` (Review/Refine)

We already set up `MovementData` in Chapter 7 and integrated it with `MovementComponent` in Chapter 13. Let's just review the `MovementComponent.gd` to ensure it's robust with its `movement_data` reference.

1.  Open `res://scripts/components/MovementComponent.gd`.

    ```gdscript
    # MovementComponent.gd
    extends Component
    class_name MovementComponent

    @export var movement_data: MovementData # Already exists

    var _character_body: CharacterBody2D = null
    var _current_velocity: Vector2 = Vector2.ZERO

    func _on_entity_ready():
        super._on_entity_ready()
        if not is_instance_valid(get_owner_entity()): return

        _character_body = get_owner_entity().find_child("CharacterBody2D") as CharacterBody2D
        if not _character_body:
            push_error("MovementComponent on '" + get_owner_entity().name + "': No CharacterBody2D child found. Disabling component.")
            set_physics_process(false)
            return
        
        if movement_data == null: # Crucial check for data-driven components
            push_error("MovementComponent on '" + get_owner_entity().name + "': movement_data resource not assigned. Disabling component.")
            set_physics_process(false)
            return

        set_physics_process(true)
        print("MovementComponent on '" + get_owner_entity().name + "' ready. CharacterBody2D found.")

    func set_direction(direction: Vector2):
        if not is_instance_valid(movement_data): return # Safety check
        _current_velocity = direction * movement_data.speed

    func _physics_process(delta: float):
        if not is_instance_valid(_character_body): return

        _character_body.velocity = _current_velocity
        _character_body.move_and_slide()
        
        _current_velocity = Vector2.ZERO 
    ```

    *   The `if movement_data == null:` check in `_on_entity_ready()` is essential for data-driven components.
    *   The `if not is_instance_valid(movement_data): return` in `set_direction()` is also a good safety measure.

### 5. Assign `ComponentData` Resources in `Player.tscn`

1.  Open `res://scenes/entities/Player.tscn`.

2.  **Assign `PlayerHealthData.tres` to `HealthComponent`**:
    *   Select the `HealthComponent` node (child of `Player`).
    *   In the Inspector, locate the `Health Data` property.
    *   Drag and drop `res://data/game_data/PlayerHealthData.tres` into this slot.

3.  **Verify `PlayerMovementData.tres` for `MovementComponent`**:
    *   Select the `MovementComponent` node (child of `Player`).
    *   In the Inspector, verify that `res://data/game_data/PlayerMovementData.tres` is assigned to the `Movement Data` property. If not, assign it.

### 6. Test the Data-Driven Components

1.  Run `res://scenes/core/GameContext.tscn` (F5).
2.  Navigate to the gameplay state (press Enter).
3.  Observe the "Output" panel. You should see "HealthComponent on 'Player_...' ready. Max health: 100".
4.  Press `Spacebar` to take damage.
    *   The player should now become invulnerable for `0.5` seconds after each hit. If you spam `Spacebar` quickly, you'll notice damage is only applied after the invulnerability period.
    *   The print statements will indicate when the player is invulnerable and when invulnerability ends.
5.  Now, without changing any code:
    *   Open `res://data/game_data/PlayerHealthData.tres`. Change `Max Health` to `50` and `Invulnerability Duration` to `2.0`. Save.
    *   Open `res://data/game_data/PlayerMovementData.tres`. Change `Speed` to `250.0`. Save.
    *   Run the game again.
    *   The player should start with 50 health, move much faster, and have a longer invulnerability period after taking damage. This demonstrates the power of data-driven design!

## Consistency Check

You have successfully implemented data-driven components by creating `HealthData` and integrating it with `HealthComponent`, alongside confirming the `MovementComponent`'s use of `MovementData`. Your `HealthComponent` now also includes robust invulnerability logic, fully configured by its data resource. This chapter solidifies the pattern of separating configuration data from component logic, making your game highly configurable, extensible, and moddable without touching a single line of code for content changes.

In the next chapter, we will design more comprehensive external game data structures and explore how to load them, further preparing our project for user-generated content and modding.
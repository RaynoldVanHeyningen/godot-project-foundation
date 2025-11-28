# Chapter 13: Implementing a `MovementComponent`

## Goal

The goal of this chapter is to implement a dedicated **`MovementComponent`** that encapsulates all movement-related logic for an `Entity`. This component will extend our generic `Component` base script, interact with the `Entity`'s `CharacterBody2D` node, and utilize a `MovementData` resource for configuration. By doing so, we will formalize our movement system into a reusable, data-driven component, demonstrating how to build specific functionality with our compositional architecture.

## Concept Explanation: The `MovementComponent`'s Role

A `MovementComponent` is a prime example of a specialized component. Its single responsibility is to provide movement capabilities to its `Entity`. This means it will:

*   **Receive Input/Direction**: Get a desired movement direction (e.g., from an `InputManager` for player control, or from an AI script for enemy movement).
*   **Apply Physics**: Interact with the `Entity`'s physics body (like `CharacterBody2D` for 2D platformers/top-down, or `CharacterBody3D` for 3D) to apply velocity and handle collisions.
*   **Utilize Data**: Read movement parameters (speed, acceleration, friction) from a `MovementData` resource.

By isolating movement logic into this component, we achieve:

*   **Reusability**: The same `MovementComponent` script can be used for players, enemies, or even movable platforms, simply by attaching it to different `Entity` scenes and providing different `MovementData` resources.
*   **Swappability**: You could easily swap out a `GroundMovementComponent` for a `FlyingMovementComponent` without affecting other parts of the `Entity`.
*   **Clear Responsibility**: All movement logic is in one place, making it easier to debug and modify.

## Architectural Reasoning: Interacting with the `Entity`'s Children

The `MovementComponent` will demonstrate how components interact with other child nodes of their `Entity` owner. Specifically:

*   It needs to find and control the `CharacterBody2D` node that is also a child of the `Entity`.
*   It will read configuration from a `MovementData` resource, reinforcing data-driven design.
*   It will receive input, potentially from our global `InputManager` (for player movement), but could also be driven by AI.

This interaction pattern (`Component` -> `Entity` -> `Sibling Physics Node` -> `Resource Data`) is a core aspect of our compositional approach. The `Component` base script's `get_owner_entity()` and `get_sibling_component()` helpers will be invaluable here.

## Production Mindset Notes: Physics Integration

*   **`_physics_process()`**: Movement logic that interacts with Godot's physics engine (`CharacterBody2D`, `RigidBody2D`, etc.) should always be placed in `_physics_process(delta)`. This ensures consistent behavior regardless of frame rate.
*   **`move_and_slide()`**: For `CharacterBody2D`, `move_and_slide()` is the recommended method for movement, as it handles collisions, sliding along surfaces, and returning collision information.
*   **Separation**: The `MovementComponent` *controls* the `CharacterBody2D` but does not *own* its physics properties (like mass, gravity scale, collision layers). Those are configured directly on the `CharacterBody2D` node or through its physics material resources.

## Step-by-Step Instructions: Creating and Integrating the `MovementComponent`

We will replace our temporary `PlayerMovementTest.gd` with a proper `MovementComponent.gd` attached directly to the `Player` `Entity`. This `MovementComponent` will then find the `CharacterBody2D` sibling and control it.

### 1. Create the `MovementComponent` Script

1.  In `res://scripts/components/`, create a new script named `MovementComponent.gd`.
2.  Ensure it `extends Component`.
3.  Add the following code:

    ```gdscript
    # MovementComponent.gd
    extends Component
    class_name MovementComponent

    @export var movement_data: MovementData # Reference to the MovementData resource

    var _character_body: CharacterBody2D = null
    var _current_velocity: Vector2 = Vector2.ZERO

    func _on_entity_ready():
        super._on_entity_ready() # Call base Component's _on_entity_ready
        if not is_instance_valid(get_owner_entity()): return

        # Find the CharacterBody2D sibling node
        _character_body = get_owner_entity().find_child("CharacterBody2D") as CharacterBody2D
        if not _character_body:
            push_error("MovementComponent on '" + get_owner_entity().name + "': No CharacterBody2D child found. Disabling component.")
            set_physics_process(false) # Disable physics processing if no body to move
            return
        
        if movement_data == null:
            push_error("MovementComponent on '" + get_owner_entity().name + "': movement_data resource not assigned. Disabling component.")
            set_physics_process(false)
            return

        set_physics_process(true) # Enable physics processing once ready
        print("MovementComponent on '" + get_owner_entity().name + "' ready. CharacterBody2D found.")

    # Public method to set the desired movement direction
    func set_direction(direction: Vector2):
        if not is_instance_valid(movement_data): return
        _current_velocity = direction * movement_data.speed

    func _physics_process(delta: float):
        if not is_instance_valid(_character_body): return

        # Apply movement
        _character_body.velocity = _current_velocity
        _character_body.move_and_slide()
        
        # Reset current velocity if not actively being set by input (for next frame)
        _current_velocity = Vector2.ZERO 
        # Note: For more complex movement (e.g., acceleration, friction), 
        # _current_velocity would be integrated over time, not reset to ZERO.
        # We'll keep it simple for now, but the data is there for future expansion.
    ```

    *   **`@export var movement_data: MovementData`**: Allows us to assign the data resource in the editor.
    *   **`_character_body`**: A cached reference to the `CharacterBody2D` node.
    *   **`_on_entity_ready()`**: Finds and caches the `CharacterBody2D`. It also performs validation checks.
    *   **`set_direction()`**: A public method for other scripts (like our `PlayerController` below) to tell the `MovementComponent` *where* to move.
    *   **`_physics_process()`**: This is where `_character_body.velocity` is set and `move_and_slide()` is called.

### 2. Create a `PlayerController` Component (New)

Since `MovementComponent` only *moves* the `CharacterBody2D` based on a `direction`, we need a separate component that *generates* that direction (e.g., from player input). This is another great example of separation of concerns.

1.  In `res://scripts/components/`, create a new script named `PlayerController.gd`.
2.  Ensure it `extends Component`.
3.  Add the following code:

    ```gdscript
    # PlayerController.gd
    extends Component
    class_name PlayerController

    var _movement_component: MovementComponent = null
    var _health_component: HealthComponent = null # For damage test

    func _on_entity_ready():
        super._on_entity_ready()
        if not is_instance_valid(get_owner_entity()): return

        # Find the MovementComponent sibling
        _movement_component = get_sibling_component(MovementComponent)
        if not _movement_component:
            push_error("PlayerController on '" + get_owner_entity().name + "': No MovementComponent sibling found. Disabling component.")
            set_process(false)
            set_physics_process(false)
            return

        # Also get HealthComponent for damage testing
        _health_component = get_sibling_component(HealthComponent)
        if _health_component:
            _health_component.health_changed.connect(_on_health_changed)
            _health_component.died.connect(_on_died)
        else:
            push_error("PlayerController on '" + get_owner_entity().name + "': No HealthComponent sibling found.")

        set_process_input(true) # Enable input processing for this component
        print("PlayerController on '" + get_owner_entity().name + "' ready.")

    func _physics_process(delta: float):
        # Get movement direction from the global InputManager
        var direction = InputManager.get_movement_direction()
        _movement_component.set_direction(direction)

    func _input(event: InputEvent):
        # Temporary: Take damage on "ui_accept" (Spacebar/Enter)
        if event.is_action_just_pressed("ui_accept"):
            if _health_component:
                _health_component.take_damage(10)
            else:
                print("PlayerController: Cannot take damage, HealthComponent not found.")
        
        # Temporary: Switch skin on "ui_text_next" (E)
        if event.is_action_just_pressed("ui_text_next"):
            var skin_component = get_sibling_component(SkinComponent) as SkinComponent
            if skin_component:
                var red_skin_data: PlayerSkinData = preload("res://data/game_data/PlayerSkinRed.tres")
                var blue_skin_data: PlayerSkinData = preload("res://data/game_data/PlayerSkinBlue.tres")
                
                if skin_component.get_current_skin_id() == blue_skin_data.id:
                    skin_component.apply_skin_from_data(red_skin_data)
                else:
                    skin_component.apply_skin_from_data(blue_skin_data)
            else:
                print("PlayerController: Cannot switch skin, SkinComponent not found.")

    func _on_health_changed(current_health: int, max_health: int):
        print("PlayerController: Player Health: " + str(current_health) + "/" + str(max_health))

    func _on_died():
        print("PlayerController: Player received 'died' signal. Handling player death!")
        # Stop processing input/movement when dead
        set_process(false)
        set_physics_process(false)
        set_process_input(false)
        # You might also want to hide the player or play a death animation
    ```

    *   **`get_sibling_component(MovementComponent)`**: This is where our `Component` base script's utility method shines. The `PlayerController` easily finds its `MovementComponent` sibling.
    *   **`_physics_process()`**: Gets input and delegates the movement command to the `_movement_component`.
    *   **`_input()`**: Handles damage and skin switching for testing, again by finding sibling components.

### 3. Refactor `Player.tscn` to Use New Components

We will now remove the old `PlayerMovementTest.gd` and add the new `MovementComponent` and `PlayerController`.

1.  Open `res://scenes/entities/Player.tscn`.
2.  Select the `CharacterBody2D` node.
3.  In the Inspector, remove the `PlayerMovementTest.gd` script by clicking the small "x" icon next to its name.
    *   *Note*: The `CharacterBody2D` node itself does not `extends Component`. Its script (if it had one) would. But our `MovementComponent` will control it.

4.  **Add `MovementComponent`**:
    *   Select the root `Player` node (instance of `Entity.tscn`).
    *   Click the "+" icon to add a new child `Node`.
    *   Rename it to `MovementComponent`.
    *   Attach `MovementComponent.gd` (`res://scripts/components/MovementComponent.gd`) to it.
    *   In the Inspector for `MovementComponent`, drag `res://data/game_data/PlayerMovementData.tres` into the `Movement Data` slot.

5.  **Add `PlayerController` Component**:
    *   Select the root `Player` node.
    *   Click the "+" icon to add a new child `Node`.
    *   Rename it to `PlayerController`.
    *   Attach `PlayerController.gd` (`res://scripts/components/PlayerController.gd`) to it.

Your `Player.tscn` scene tree should now look like this:

```
Player (Instance of Entity.tscn, with Entity.gd script)
├── Sprite (Sprite2D)
├── CharacterBody2D
│   └── CollisionShape2D
├── HealthComponent (Node with HealthComponent.gd script)
├── SkinComponent (Node with SkinComponent.gd script)
├── MovementComponent (Node with MovementComponent.gd script)
└── PlayerController (Node with PlayerController.gd script)
```

6.  Save `Player.tscn`.

### 4. Test the New Movement System

1.  Run `res://scenes/core/GameContext.tscn` (F5).
2.  Navigate to the gameplay state (press Enter).
3.  Observe the "Output" panel. You should see `MovementComponent` and `PlayerController` initializing messages.
4.  Verify that player movement, damage (Spacebar), and skin switching (E) still work exactly as before. The external behavior is the same, but the internal architecture is now much more modular and robust.

## Consistency Check

You have successfully implemented a dedicated `MovementComponent` that extends our `Component` base and uses a `MovementData` resource. You also created a `PlayerController` component that handles input and delegates movement commands to the `MovementComponent`. This demonstrates a clear separation of concerns: input generation is separate from movement application, and both are separate from raw data. Your player's movement system is now fully compositional, data-driven, and highly reusable.

In the next chapter, we will further enhance our `HealthComponent` by solidifying its implementation and ensuring it communicates effectively via signals.
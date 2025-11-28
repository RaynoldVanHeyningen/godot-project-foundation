# Chapter 6: Event-Driven Communication with Godot Signals

## Goal

The goal of this chapter is to master Godot's built-in **signal mechanism** for event-driven communication. We will learn how to declare custom signals, emit them from one node, and connect them to methods in other nodes, both programmatically in GDScript and visually in the editor. This is a fundamental pattern for achieving loose coupling between disparate systems and components, a cornerstone of our AAA compositional architecture.

## Concept Explanation: What are Signals?

Imagine a light switch and a light bulb. The switch doesn't directly turn on the bulb by physically manipulating it. Instead, it *emits a signal* (an electrical impulse) when flipped. The light bulb *listens* for this signal and, when it receives it, performs its action (turns on). The switch doesn't need to know *what* listens or *how* the light bulb works; it just emits its event.

In Godot, **signals** work exactly like this event-driven pattern. A node (the "emitter") can declare and emit a signal when something important happens (e.g., a button is pressed, a player dies, health changes). Other nodes (the "receivers") can connect to this signal and execute a method in response.

### Key Characteristics of Signals:

*   **Loose Coupling**: The emitter doesn't need to know anything about the receiver, and vice versa. They only need to agree on the signal's name and any parameters it sends. This makes systems independent and easier to swap or modify.
*   **Event-Driven**: Code execution is triggered by events, making it reactive and efficient.
*   **Built-in**: Godot's Node system is inherently signal-based. UI elements, physics bodies, and many other nodes come with powerful built-in signals.
*   **Customizable**: You can define your own custom signals in your scripts.

### Why Signals are Crucial for Compositional Design:

When building with composition, you have many small, specialized nodes (components) that need to interact. Direct references between them can quickly lead to a tangled mess. Signals provide a clean, one-way communication channel:

*   A `HealthComponent` emits a `died` signal. An `EnemyAI` component listens to this signal to despawn the enemy. A `UIController` listens to update the health bar.
*   An `InventoryComponent` emits an `item_added` signal. A `SoundManager` plays a sound. A `NotificationSystem` displays a message.

Without signals, the `HealthComponent` would need direct references to the `EnemyAI`, `UIController`, etc., becoming tightly coupled and difficult to maintain.

## Architectural Reasoning: Decoupling and Modularity

Signals are the primary mechanism for **decoupling** systems in Godot. They allow us to uphold the "separation of concerns" principle by ensuring that:

*   **Emitters focus on their own state**: A `HealthComponent` only cares about its health value and when it changes. It doesn't care *who* needs to know about the change.
*   **Receivers focus on their reactions**: A `UIController` only cares about *reacting* to a health change, not *how* the health changed.

This makes individual components truly modular. You can drop a `HealthComponent` onto any entity, and as long as other systems are configured to listen to its signals, it will integrate seamlessly. This is vital for scalability and moddability, as new systems can easily plug into existing event streams without modifying core logic.

## Production Mindset Notes: Best Practices for Signals

*   **Descriptive Names**: Name your signals clearly to indicate *what* event occurred (e.g., `health_changed`, `player_died`, `item_collected`).
*   **Meaningful Parameters**: Pass relevant data with your signals (e.g., `health_changed.emit(current_health, max_health)`).
*   **Connect Where Appropriate**:
    *   **In Editor**: For simple, fixed connections between nodes in the same scene (e.g., a `Button` to a `LevelManager`).
    *   **In Script (`_ready()` or `_init()`)**: For dynamic connections, or when connecting to `AutoLoad` singletons, or when the connection depends on runtime logic.
*   **Disconnect When Not Needed**: For temporary connections, remember to `disconnect()` to prevent memory leaks or unwanted calls, especially with dynamically created nodes. (Though for `_ready()` connections that live for the scene, this is less critical).
*   **Avoid Signal Spam**: Don't emit signals excessively (e.g., every frame) if a simpler direct method call would suffice for tight internal component communication. Signals are for *events* that others might want to react to.

## Step-by-Step Instructions: Implementing a `HealthComponent` with Signals

Let's refactor our Player to include a proper `HealthComponent` that emits signals when its health changes or when the entity dies.

### 1. Create the `HealthComponent` Script

1.  In the `FileSystem` dock, navigate to `res://scripts/components/`.
2.  Right-click and select "New Script...".
3.  Name the script `HealthComponent.gd`.
4.  Ensure `extends Node` (as it's a general component, not necessarily a `CharacterBody2D`).
5.  Click "Create".
6.  Open `HealthComponent.gd` and add the following code:

    ```gdscript
    # HealthComponent.gd
    extends Node

    # Declare custom signals
    signal health_changed(current_health: int, max_health: int)
    signal died()

    @export var max_health: int = 100:
        set(value):
            max_health = max(0, value) # Ensure max_health is never negative
            if is_on_ready(): # Only update if _ready has been called
                _health = min(_health, max_health) # Don't exceed new max_health
                health_changed.emit(_health, max_health)

    var _health: int:
        set(value):
            var old_health = _health
            _health = clampi(value, 0, max_health) # Clamp health between 0 and max_health

            if _health != old_health: # Only emit if health actually changed
                health_changed.emit(_health, max_health)
                if _health == 0 and old_health > 0: # Only emit died once when health hits 0
                    died.emit()
                    print(owner.name + " has died!") # For debugging

    func _ready():
        _health = max_health # Initialize current health to max_health
        health_changed.emit(_health, max_health) # Emit initial health state

    func take_damage(amount: int):
        if _health > 0: # Only take damage if not already dead
            self.health -= amount
            print(owner.name + " took " + str(amount) + " damage. Current health: " + str(_health))

    func heal(amount: int):
        if _health < max_health: # Only heal if not at max health
            self.health += amount
            print(owner.name + " healed " + str(amount) + ". Current health: " + str(_health))

    # Helper to get the current health
    func get_health() -> int:
        return _health
    ```

    *   **`signal health_changed(current_health: int, max_health: int)`**: Declares a signal that will be emitted whenever health changes, passing the current and max health values.
    *   **`signal died()`**: Declares a signal emitted when health drops to 0.
    *   **`@export var max_health`**: Allows setting max health from the editor.
    *   **`_health` (private setter)**: The `set` method for `_health` ensures it's clamped, and crucially, `emits` the `health_changed` and `died` signals when appropriate.
    *   **`take_damage()` / `heal()`**: Public methods to modify health.

### 2. Add `HealthComponent` to the Player Scene

1.  Open `res://scenes/entities/Player.tscn`.
2.  Select the root `Player` node.
3.  Click the "+" icon to add a new child node.
4.  Search for `Node` and create it. This will be our container for the `HealthComponent` script.
5.  Rename this new `Node` to `HealthComponent`.
6.  Attach the `HealthComponent.gd` script (`res://scripts/components/HealthComponent.gd`) to this `HealthComponent` node.
7.  In the Inspector for the `HealthComponent` node, you can set its `Max Health` (e.g., to 100).

Your `Player.tscn` scene tree should now look like this:

```
Player (Node2D, root of the scene)
├── Sprite (Sprite2D)
├── CharacterBody2D
│   └── CollisionShape2D
└── HealthComponent (Node with HealthComponent.gd script)
```

### 3. Connect to `HealthComponent` Signals (Programmatically)

Let's modify our temporary `PlayerMovementTest.gd` script to react to the `HealthComponent`'s signals. We'll add a simple way to take damage and print messages.

1.  Open `res://scripts/components/PlayerMovementTest.gd`.
2.  Add a reference to the `HealthComponent` and connect to its signals in `_ready()`:

    ```gdscript
    # PlayerMovementTest.gd
    extends CharacterBody2D

    @export var speed: float = 100.0
    var health_component: HealthComponent

    func _ready():
        # Get a reference to the HealthComponent child node
        health_component = get_node("../HealthComponent") # Assumes HealthComponent is a sibling of CharacterBody2D

        # Connect to its signals
        if health_component:
            health_component.health_changed.connect(_on_health_changed)
            health_component.died.connect(_on_died)
            print("PlayerMovementTest connected to HealthComponent signals.")
        else:
            push_error("HealthComponent not found as sibling of CharacterBody2D!")

    func _physics_process(delta: float):
        var direction = InputManager.get_movement_direction()
        if direction:
            velocity = direction * speed
        else:
            velocity = Vector2.ZERO
        move_and_slide()

        # Temporary: Take damage on Spacebar press
        if Input.is_action_just_pressed("ui_accept"): # Default action for Spacebar/Enter
            if health_component:
                health_component.take_damage(10)
            else:
                print("Cannot take damage, HealthComponent not found.")

    func _on_health_changed(current_health: int, max_health: int):
        print("Player Health: " + str(current_health) + "/" + str(max_health))

    func _on_died():
        print("PlayerMovementTest received 'died' signal. Handling player death!")
        # In a real game, this might trigger game over, animation, etc.
        set_process(false) # Stop processing input/movement
        set_physics_process(false)
        # You might also want to hide the player or play a death animation
    ```

    *   **`get_node("../HealthComponent")`**: This line accesses the `HealthComponent` which is a sibling of the `CharacterBody2D` (the parent of this script). The `..` means "go up one level to the parent node, then find a child named 'HealthComponent'".
    *   **`health_component.health_changed.connect(_on_health_changed)`**: This is the core of signal connection. It connects the `health_changed` signal from `health_component` to the `_on_health_changed` method in *this* script.
    *   **`ui_accept`**: This is a default Godot input action, typically bound to Spacebar or Enter. We're using it for a quick damage test.

4.  Save `PlayerMovementTest.gd`.

### 5. Test the Signals

1.  Run `res://scenes/levels/Main.tscn` (F5).
2.  Observe the "Output" panel at the bottom of the Godot editor.
3.  You should see "Player Health: 100/100" printed immediately (from `_ready()` in `HealthComponent` and `_on_health_changed` in `PlayerMovementTest`).
4.  Press the `Spacebar` repeatedly. You should see "Player took 10 damage..." messages and "Player Health: X/100" updates.
5.  When health reaches 0, you'll see "Player has died!" and "PlayerMovementTest received 'died' signal. Handling player death!". The player will also stop moving.

### 6. Connect to `HealthComponent` Signals (Via Editor - Optional)

For simple, direct connections within a scene, the editor can be very convenient.

1.  Open `res://scenes/entities/Player.tscn`.
2.  Select the `HealthComponent` node in the Scene dock.
3.  Go to the "Node" tab next to the "Inspector" dock (usually on the right).
4.  You will see "Signals" listed under `HealthComponent`.
5.  Double-click `health_changed(current_health: int, max_health: int)`.
6.  A "Connect a Signal" window will appear. Select the `Player` root node (or any other node in the scene you want to connect to).
7.  Click "Connect". Godot will automatically generate a new method in the script attached to the selected node (e.g., `_on_HealthComponent_health_changed`). You can rename it if you wish.
8.  This creates a connection directly in the scene file, visually represented in the Node dock. This is useful for UI elements or fixed scene interactions.

## Consistency Check

You've successfully implemented a `HealthComponent` that uses custom signals to notify other parts of the game about its state changes. By connecting to these signals, our `PlayerMovementTest` script could react to damage and death without directly querying the `HealthComponent`'s internal state or needing a direct reference to it beyond the initial `_ready()` setup. This loose coupling is foundational for building scalable and maintainable game architectures in Godot.

In the next chapter, we will explore another crucial architectural pattern: Data-Driven Design using Godot Resources, which allows us to separate configuration data from our game logic.
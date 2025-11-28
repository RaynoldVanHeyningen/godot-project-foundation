# Chapter 7: Data-Driven Design with Godot Resources

## Goal

The goal of this chapter is to introduce and implement **Data-Driven Design** using Godot's powerful **Custom Resource** types. We will learn how to create custom `Resource` scripts (`.gd`), instance them as `.tres` files, and use them to separate configuration data from game logic. This approach is fundamental for improving iteration speed, making content creation easier for designers, and inherently supporting modding.

## Concept Explanation: Data-Driven Design

In traditional game development, game designers often rely on programmers to hardcode values like player speed, enemy health, item stats, or level configurations directly into scripts. This creates a bottleneck: any design change requires a code change, recompilation, and often redeployment.

**Data-Driven Design** (DDD) is an architectural approach where as much game data as possible is stored in external, configurable data files, separate from the game's core logic. The game logic then *reads* and *uses* this data at runtime.

### Why Data-Driven Design is Powerful:

*   **Faster Iteration**: Designers can tweak values (e.g., character speed, weapon damage) directly in editor-friendly files without needing a programmer or recompiling code.
*   **Reduced Risk**: Changes to data are less likely to introduce bugs into core game logic.
*   **Scalability**: Adding new content (e.g., a new enemy type, a new item) often means just creating a new data file, not writing new code.
*   **Moddability**: External data files are much easier for modders to create, modify, and integrate into the game without touching the game's source code. This is a *critical* component of our modding-ready architecture.
*   **Collaboration**: Programmers focus on systems, designers focus on content, artists focus on assets. Clear separation of concerns.

### Godot's Solution: Custom Resources

Godot provides a first-class solution for Data-Driven Design through its **`Resource`** system. A `Resource` is a data container that can be saved to disk as a `.tres` (text resource) or `.res` (binary resource) file.

You can create **Custom Resources** by extending the `Resource` class with your own GDScript. These custom resource scripts define the properties (variables) that your data will contain. Once defined, you can create multiple instances of these resources as `.tres` files, each holding different values for those properties.

For example, instead of hardcoding `player_speed = 100` in a `PlayerMovement.gd` script, you would:

1.  Create a `MovementData.gd` script that `extends Resource` and has an `@export var speed: float`.
2.  Create a `PlayerMovementData.tres` file based on `MovementData.gd`.
3.  In `PlayerMovementData.tres`, set the `speed` property to `100`.
4.  In your `PlayerMovement.gd` script, load the `PlayerMovementData.tres` and read its `speed` property.

## Architectural Reasoning: Decoupling Logic from Configuration

Using custom resources enforces a strong separation between:

*   **What** a game object *is* (its data/configuration).
*   **How** a game object *behaves* (its logic/code).

A `MovementComponent` script defines *how* an entity moves. A `MovementData` resource defines *how fast* it moves, *how high* it can jump, etc. The component uses the data, but doesn't own it. This means:

*   You can have multiple entities using the *same* `MovementComponent` script but with *different* `MovementData` resources, resulting in varied behaviors (e.g., a fast player, a slow enemy).
*   Updating the `MovementComponent` script doesn't require touching any specific entity's data.
*   Changing a `MovementData.tres` file doesn't require modifying or recompiling any script.

This clear division of labor is a hallmark of scalable, maintainable, and moddable architectures.

## Production Mindset Notes: Resource Best Practices

*   **Granularity**: Create specific resources for specific data sets (e.g., `WeaponData`, `EnemyStats`, `DialogueLine`). Avoid a single "GameData" resource that holds everything.
*   **Folder Structure**: Store your `.gd` resource definitions in `res://scripts/data_types/` (or similar) and your `.tres` instances in `res://data/game_data/` or `res://data/configs/`.
*   **`@export` for Editor Usability**: Use `@export` for properties in your custom resource scripts. This makes them editable directly in the Godot editor when you create a `.tres` file.
*   **Read-Only Resources**: Once loaded, consider treating resources as read-only in your game logic to prevent accidental modifications that might affect other instances or lead to unexpected behavior. If you need to modify data per instance, you might duplicate the resource (`.duplicate()`) or load it as a scene. For pure configuration, read-only is best.

## Step-by-Step Instructions: Creating and Using a `MovementData` Resource

Let's create a `MovementData` custom resource to configure the speed of our player's `PlayerMovementTest` script.

### 1. Create the `MovementData` Custom Resource Script

1.  In the `FileSystem` dock, navigate to `res://scripts/data_types/`. (If you don't have this folder, create it under `scripts`).
2.  Right-click and select "New Script...".
3.  Name the script `MovementData.gd`.
4.  Ensure `extends Resource` is selected.
5.  Click "Create".
6.  Open `MovementData.gd` and add the following code:

    ```gdscript
    # MovementData.gd
    extends Resource
    class_name MovementData # Make it recognizable by name in editor

    @export var speed: float = 100.0
    @export var acceleration: float = 500.0
    @export var friction: float = 800.0
    @export var jump_velocity: float = -400.0 # For platformers, if applicable
    ```

    *   **`class_name MovementData`**: This line is important. It registers `MovementData` as a global class name. This allows Godot's editor to recognize it by name when creating new resources or setting `@export` types.
    *   **`@export var ...`**: These properties will appear in the Inspector when we create an instance of this resource.

### 2. Create an Instance of `MovementData` for the Player

1.  In the `FileSystem` dock, navigate to `res://data/game_data/`.
2.  Right-click and select "New Resource...".
3.  In the "Create New Resource" dialog, search for `MovementData`. Select it and click "Create".
4.  Name the new resource file `PlayerMovementData.tres`.
5.  Save it in `res://data/game_data/`.
6.  Select `PlayerMovementData.tres` in the `FileSystem` dock.
7.  In the Inspector dock, you will now see the `speed`, `acceleration`, `friction`, and `jump_velocity` properties.
8.  Set `Speed` to `150.0` (or any value you prefer, different from the default).

### 3. Integrate `MovementData` into `PlayerMovementTest.gd`

Now, let's modify our `PlayerMovementTest.gd` to load and use this resource.

1.  Open `res://scripts/components/PlayerMovementTest.gd`.
2.  Modify the script to `@export` a `MovementData` resource and use its `speed` property:

    ```gdscript
    # PlayerMovementTest.gd
    extends CharacterBody2D

    # We now @export a MovementData resource instead of a raw speed float.
    @export var movement_data: MovementData # Type hint for editor and static analysis

    # Removed: @export var speed: float = 100.0 (as it's now in the resource)

    var health_component: HealthComponent

    func _ready():
        if movement_data == null:
            push_error("MovementData resource not assigned to PlayerMovementTest!")
            set_process(false) # Disable if no data
            return

        health_component = get_node("../HealthComponent")
        if health_component:
            health_component.health_changed.connect(_on_health_changed)
            health_component.died.connect(_on_died)
            print("PlayerMovementTest connected to HealthComponent signals.")
        else:
            push_error("HealthComponent not found as sibling of CharacterBody2D!")

    func _physics_process(delta: float):
        if movement_data == null: return # Prevent errors if data is missing

        var direction = InputManager.get_movement_direction()

        if direction:
            # Use speed from the MovementData resource
            velocity = direction * movement_data.speed
        else:
            velocity = Vector2.ZERO

        move_and_slide()

        if Input.is_action_just_pressed("ui_accept"):
            if health_component:
                health_component.take_damage(10)
            else:
                print("Cannot take damage, HealthComponent not found.")

    func _on_health_changed(current_health: int, max_health: int):
        print("Player Health: " + str(current_health) + "/" + str(max_health))

    func _on_died():
        print("PlayerMovementTest received 'died' signal. Handling player death!")
        set_process(false)
        set_physics_process(false)
    ```

3.  Save `PlayerMovementTest.gd`.

### 4. Assign the Resource to the Player's Movement Script

1.  Open `res://scenes/entities/Player.tscn`.
2.  Select the `CharacterBody2D` node (which has the `PlayerMovementTest.gd` script attached).
3.  In the Inspector dock, you will now see an `Export` property named `Movement Data`.
4.  Drag and drop `res://data/game_data/PlayerMovementData.tres` from the `FileSystem` dock into the "Movement Data" slot in the Inspector.

### 5. Test with Data-Driven Values

1.  Run `res://scenes/levels/Main.tscn` (F5).
2.  Move the player using `W`, `A`, `S`, `D`. You should observe that the player moves at the `speed` you set in `PlayerMovementData.tres` (e.g., `150.0`).
3.  Now, without changing any code, *open `PlayerMovementData.tres`*, change its `Speed` property to `50.0`, save it, and run the game again. You'll immediately notice the player moves much slower.

## Consistency Check

You have successfully implemented a data-driven approach for configuring player movement speed using a custom `MovementData` resource. This demonstrates how to separate configuration data from game logic, making your project more flexible, easier to iterate on, and inherently more moddable. Designers can now adjust player speeds without touching a single line of code.

In the next chapter, we will build upon this by exploring how to dynamically load and unload resources at runtime, which is crucial for efficient memory management and advanced modding scenarios.
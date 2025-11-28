# Chapter 5: Global Access with AutoLoad (Singletons)

## Goal

The goal of this chapter is to introduce Godot's `AutoLoad` feature, also known as **Singletons**, and demonstrate how to use it to implement globally accessible systems. We will learn the appropriate use cases for `AutoLoad`s, understand their benefits for core game systems, and discuss strategies to avoid common anti-patterns like creating "God objects" that violate the principles of modularity and separation of concerns.

## Concept Explanation: AutoLoad (Singletons)

In game development, some systems need to be accessible from almost anywhere in the game, throughout its entire lifecycle. Examples include:

*   An `InputManager` that centralizes input processing.
*   A `AudioManager` that plays sounds and music.
*   A `GameManager` that handles overall game state or score.
*   A `Logger` that outputs debug information.

Godot provides the `AutoLoad` feature (often referred to as Singletons in other engines) specifically for this purpose. An `AutoLoad` script or scene is automatically loaded into the scene tree *before* any other scene, and remains active for the entire duration of the game. It is accessible globally by its name, making it very convenient for core systems.

### How AutoLoad Works:

When you register a script or scene as an `AutoLoad`:

1.  Godot instances it automatically at game start.
2.  It's added as a child of the `root` node (the `Viewport` or `SceneTree` itself), making it globally accessible.
3.  You can access it from any other script using its registered name, e.g., `InputManager.is_action_pressed("move_left")`.

### Benefits of Using AutoLoad for Global Systems:

*   **Global Accessibility**: Easily call methods or access properties from anywhere without passing references around.
*   **Guaranteed Instantiation**: Always available from the very beginning of the game.
*   **Single Instance**: Ensures there's only one instance of a given system, which is crucial for managers like `AudioManager` or `GameManager`.
*   **Persistence**: `AutoLoad` nodes are not removed when scenes change, making them ideal for persistent data or services.

### Avoiding the "God Object" Anti-Pattern

While powerful, `AutoLoad`s can be misused, leading to a "God object" – a single class that knows or controls too much about other parts of the system. This violates separation of concerns and leads to tightly coupled, unmaintainable code.

To avoid this:

*   **Single Responsibility Principle**: Each `AutoLoad` should have one clear, well-defined responsibility (e.g., `AudioManager` only manages audio, `InputManager` only manages input).
*   **Avoid Direct References**: `AutoLoad`s should generally *not* directly reference or control specific game entities (like the Player). Instead, they should emit signals (which we'll cover in the next chapter) or provide services that other objects can use.
*   **Focus on Services**: Think of `AutoLoad`s as providing *services* to the rest of the game, not as controllers of the entire game.

## Architectural Reasoning: Centralized Services, Decoupled Logic

From an architectural standpoint, `AutoLoad`s allow us to centralize core services while maintaining loose coupling for the rest of the game.

*   **Core Systems**: `AutoLoad`s house the foundational, often engine-level, services that every part of the game might need (e.g., logging, input, audio).
*   **Decoupled Consumers**: Game entities and components (like our `Player` scene) don't need to know *how* `InputManager` works, only that they can call `InputManager.get_direction()`. This keeps the entity logic clean and independent.
*   **Clear Boundaries**: By strictly defining the responsibilities of each `AutoLoad`, we create clear boundaries between different parts of our game, making it easier to scale and maintain.

## Production Mindset Notes: Naming and Consistency

*   **Consistent Naming**: Name your `AutoLoad`s clearly and descriptively (e.g., `AudioManager`, `InputManager`).
*   **Global Scope**: Remember that these are globally accessible. This means their names should be unique and not conflict with other global identifiers. Godot automatically handles this by making the `AutoLoad` name the global variable name.
*   **Start Small**: Don't make everything an `AutoLoad`. Only systems that truly require global, persistent access throughout the game's lifetime are good candidates.

## Step-by-Step Instructions: Implementing an `InputManager` AutoLoad

Let's create a simple `InputManager` as an `AutoLoad` to centralize our input processing. This will provide a clean API for other nodes to query input without directly interacting with Godot's `Input` singleton.

### 1. Create the `InputManager` Script

1.  In the `FileSystem` dock, navigate to `res://scripts/managers/`.
2.  Right-click and select "New Script...".
3.  Name the script `InputManager.gd`.
4.  Ensure `extends Node` is selected.
5.  Click "Create".
6.  Open `InputManager.gd` and add the following basic structure. For now, it will just contain a placeholder for future input logic.

    ```gdscript
    # InputManager.gd
    extends Node

    # This AutoLoad will centralize input handling.
    # Other scripts will query this manager instead of Godot's Input singleton directly.

    func _ready():
        # This _ready runs once when the AutoLoad is initialized.
        print("InputManager ready.")

    func get_movement_direction() -> Vector2:
        # Placeholder for complex input logic (e.g., mapping multiple keys, gamepad input)
        var direction = Vector2.ZERO

        if Input.is_action_pressed("move_right"):
            direction.x += 1
        if Input.is_action_pressed("move_left"):
            direction.x -= 1
        if Input.is_action_pressed("move_down"):
            direction.y += 1
        if Input.is_action_pressed("move_up"):
            direction.y -= 1

        return direction.normalized() # Normalize to prevent faster diagonal movement
    ```

### 2. Configure Godot's Input Map

Before we use our `InputManager`, we need to define the input actions in Godot's Project Settings.

1.  Go to `Project` -> `Project Settings...`.
2.  Select the "Input Map" tab.
3.  In the "Actions" section, type `move_up` in the "Action" field and click "Add".
4.  Repeat for `move_down`, `move_left`, `move_right`.
5.  For each action, click the `+` button next to it to assign a key:
    *   `move_up`: `W` key (or `Up Arrow`)
    *   `move_down`: `S` key (or `Down Arrow`)
    *   `move_left`: `A` key (or `Left Arrow`)
    *   `move_right`: `D` key (or `Right Arrow`)
    *   *Optional*: Add gamepad axis inputs if you plan to support gamepads. For `move_right`, for example, click `+` again and then press the right direction on your gamepad's left stick.
6.  Close Project Settings.

### 3. Register `InputManager` as an AutoLoad

Now, let's make our `InputManager` globally accessible.

1.  Go to `Project` -> `Project Settings...`.
2.  Select the "AutoLoad" tab.
3.  Click the folder icon next to the "Path" field.
4.  Navigate to `res://scripts/managers/InputManager.gd` and select it.
5.  For "Node Name", Godot will auto-fill `InputManager`. This is the global name you'll use to access it. Keep it as `InputManager`.
6.  Ensure "Enable" is checked.
7.  Click "Add".
8.  Close Project Settings.

### 4. Use the `InputManager` in our Player Scene (for demonstration)

Let's modify our `Player.tscn` to use the new `InputManager`. This will be a temporary script to demonstrate `AutoLoad` usage; we'll create proper components in later chapters.

1.  Open `res://scenes/entities/Player.tscn`.
2.  Select the root `Player` node.
3.  Attach a new script to it:
    *   Click the "Attach Script" icon (the scroll icon).
    *   For "Path", navigate to `res://scripts/` and name the script `PlayerTestMovement.gd`.
    *   Ensure `extends CharacterBody2D` (or `Node2D` if your root `Player` is `Node2D` and `CharacterBody2D` is a child. If your `Player` root is already a `CharacterBody2D`, extend `CharacterBody2D`. For this example, let's assume `Player`'s root node is a `CharacterBody2D` for simplicity in movement). *Correction from Chapter 4*: The `Player` root was `Node2D` and `CharacterBody2D` was a child. For this temporary script, let's attach it to the `CharacterBody2D` child node instead of the root `Player` node to directly access its movement capabilities.
        *   Select the `CharacterBody2D` node under `Player`.
        *   Attach a new script to it.
        *   Path: `res://scripts/components/PlayerMovementTest.gd`
        *   Extends: `CharacterBody2D`
        *   Click "Create".

4.  Add the following code to `PlayerMovementTest.gd`:

    ```gdscript
    # PlayerMovementTest.gd
    extends CharacterBody2D

    @export var speed: float = 100.0

    func _physics_process(delta: float):
        # Access the global InputManager singleton
        var direction = InputManager.get_movement_direction()

        if direction:
            velocity = direction * speed
        else:
            velocity = Vector2.ZERO

        move_and_slide()
    ```

5.  Save `PlayerMovementTest.gd` and `Player.tscn`.
6.  Run `res://scenes/levels/Main.tscn` (F5).
7.  Now, try moving your player using `W`, `A`, `S`, `D` keys. You should see it respond to input!

## Consistency Check

You have successfully created an `InputManager` `AutoLoad` and used it to control your `Player`'s movement. This demonstrates how a globally accessible system can provide services to game entities without tight coupling. Our `PlayerMovementTest` script doesn't need to know *how* input is processed, only that it can ask `InputManager` for a movement direction. This is a powerful step towards building a modular and maintainable project.

In the next chapter, we will delve into Godot's powerful signal system for robust, event-driven communication, further enhancing loose coupling between our systems.
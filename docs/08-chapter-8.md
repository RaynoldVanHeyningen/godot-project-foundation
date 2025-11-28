# Chapter 8: Dynamic Resource Loading & Unloading

## Goal

The goal of this chapter is to understand and implement efficient strategies for **dynamic resource loading and unloading** in Godot. We will learn when and how to use `load()`, `preload()`, and `ResourceLoader` for synchronous and asynchronous loading, and discuss memory management considerations. This is crucial for optimizing game performance, reducing startup times, and enabling advanced features like loading user-generated content (modding).

## Concept Explanation: Static vs. Dynamic Loading

So far, we've used `@export` to assign resources in the editor, and `preload()` implicitly when attaching scripts (which are also resources). This is often called **static loading** because the resources are loaded into memory as soon as the scene or script that references them is loaded.

While convenient, static loading has limitations:

*   **Memory Bloat**: If you have many large assets (textures, audio, complex scenes) that aren't immediately needed, they will still be loaded into memory, consuming valuable RAM.
*   **Slow Startup Times**: Loading all assets at once can significantly increase the game's initial startup time, leading to a poor user experience.
*   **Limited Modding**: Modders often need to load custom content at runtime, which static loading cannot directly support.

**Dynamic Loading** is the process of loading resources into memory only when they are actually needed, and potentially unloading them when they are no longer required. This allows for:

*   **Optimized Memory Usage**: Only necessary assets are in RAM, freeing up memory for other parts of the game.
*   **Faster Initial Load Times**: The game starts quicker, as only core assets are loaded initially.
*   **Streaming Content**: Enables loading levels, characters, or items as the player progresses, reducing hitches.
*   **Modding Support**: Critical for loading user-generated content (UGC) that isn't known at compile time.

### Godot's Loading Mechanisms:

1.  **`preload("res://path/to/resource.tres")`**:
    *   **Synchronous**: The game pauses until the resource is loaded.
    *   **Static**: Loaded at script parsing time.
    *   **Best for**: Small, frequently used resources that are always needed (e.g., a common UI sound, a small particle effect, a script). Efficient because it's compiled into the script.
    *   **Caveat**: Cannot load paths determined at runtime.

2.  **`load("res://path/to/resource.tres")`**:
    *   **Synchronous**: The game pauses until the resource is loaded.
    *   **Dynamic**: Loaded when the `load()` function is called.
    *   **Best for**: Resources whose paths are determined at runtime (e.g., loading an enemy based on an ID, or a specific level scene). Still blocks the main thread, so use sparingly for large assets.
    *   **Returns**: The loaded `Resource` object.

3.  **`ResourceLoader.load_threaded_request("res://path/to/resource.tres")`**:
    *   **Asynchronous**: The resource is loaded on a separate thread, not blocking the main game thread.
    *   **Dynamic**: Initiated when the function is called.
    *   **Best for**: Large assets or scenes that would cause a noticeable hitch if loaded synchronously (e.g., next level scene, boss character assets). Requires polling for completion.
    *   **Returns**: An error code, but the resource itself is retrieved later.

### Unloading Resources:

Godot uses **reference counting** for resources. A resource is only unloaded from memory when nothing else is referencing it.

*   When you `load()` a resource, its reference count increases.
*   When a node using a resource is freed, its reference count decreases.
*   When the reference count reaches zero, Godot automatically unloads the resource.
*   You can manually `queue_free()` nodes or clear references to resources to help Godot garbage collect.

## Architectural Reasoning: Resource Management System

For a AAA project, you often need a dedicated system to manage resource loading and unloading. This system would:

*   **Centralize Loading Logic**: All dynamic loading goes through this manager.
*   **Handle Asynchronous Operations**: Provide a clean API for requesting assets and getting callbacks when they're ready.
*   **Cache Resources**: Store frequently used resources to avoid reloading them.
*   **Track References**: Help ensure resources are properly unloaded.
*   **Support Modding**: Provide a pathway for loading resources from non-`res://` paths (like `user://`).

While we won't build a full-blown resource manager in this chapter, understanding the `load()` and `ResourceLoader` functions is the first step towards creating such a system.

## Production Mindset Notes: When to Use What

*   **`preload()`**: For small, essential scripts, sprites, or resources directly used by a scene/script and always present.
*   **`load()`**: For dynamically chosen small resources, or when a temporary hitch is acceptable (e.g., a very small sound effect, a simple icon). Avoid for large assets.
*   **`ResourceLoader.load_threaded_request()`**: For large scenes, complex characters, or significant audio/texture packs that would cause a visible stutter if loaded synchronously. Pair this with loading screens or progress bars.

Always prioritize `preload()` for static, small assets as it's efficient. Use dynamic loading when flexibility or performance demands it.

## Step-by-Step Instructions: Dynamic Loading for Player Skins

Let's enhance our Player by allowing it to dynamically load different "skins" (sprites) based on a `PlayerSkinData` resource. This demonstrates dynamic loading and prepares us for modding where new skins could be added.

### 1. Create `PlayerSkinData` Custom Resource

1.  In `res://scripts/data_types/`, create a new script named `PlayerSkinData.gd`.
2.  Ensure it `extends Resource`.
3.  Add the `class_name` and `@export` properties:

    ```gdscript
    # PlayerSkinData.gd
    extends Resource
    class_name PlayerSkinData

    @export var id: String = "" # Unique identifier for the skin
    @export var texture_path: String # Path to the skin's Sprite2D texture (e.g., "res://assets/graphics/player_skin_red.png")
    @export var description: String = ""
    ```

### 2. Create Some Skin Data Instances and Textures

1.  **Textures**: For this example, you can use Godot's default `icon.svg`. Duplicate it a few times and rename them to `player_skin_blue.svg`, `player_skin_red.svg`. Change their colors slightly in an image editor if you wish, or just use the same icon for now. Place them in `res://assets/graphics/`.
2.  **Skin Resources**:
    *   In `res://data/game_data/`, create a "New Resource..."
    *   Search for `PlayerSkinData`.
    *   Create two instances:
        *   `PlayerSkinBlue.tres`:
            *   `ID`: `blue_player`
            *   `Texture Path`: `res://assets/graphics/player_skin_blue.svg`
            *   `Description`: `A cool blue player skin.`
        *   `PlayerSkinRed.tres`:
            *   `ID`: `red_player`
            *   `Texture Path`: `res://assets/graphics/player_skin_red.svg`
            *   `Description`: `A fiery red player skin.`

### 3. Implement a `SkinComponent`

This component will be responsible for loading and applying the correct skin texture to the `Sprite2D` node.

1.  In `res://scripts/components/`, create a new script named `SkinComponent.gd`.
2.  Ensure it `extends Node`.
3.  Add the following code:

    ```gdscript
    # SkinComponent.gd
    extends Node
    class_name SkinComponent

    @export var default_skin_data: PlayerSkinData # Assign a default skin in editor

    var _current_skin_id: String = ""
    var _sprite_node: Sprite2D

    func _ready():
        # Get reference to the Sprite2D node, assuming it's a sibling or child
        _sprite_node = owner.find_child("Sprite") # Assumes Sprite is a child of the owner (Player)
        if not _sprite_node:
            push_error("SkinComponent: Could not find Sprite2D child in owner.")
            set_process(false)
            return

        if default_skin_data:
            apply_skin_from_data(default_skin_data)
        else:
            push_error("SkinComponent: No default_skin_data assigned.")

    func apply_skin_from_data(skin_data: PlayerSkinData):
        if not skin_data:
            push_error("Attempted to apply null skin data.")
            return

        if _current_skin_id == skin_data.id:
            return # Skin already applied

        print("Applying skin: " + skin_data.id + " from " + skin_data.texture_path)

        # Dynamic synchronous loading: Use load() as texture paths are dynamic
        var texture: Texture2D = load(skin_data.texture_path)
        if texture:
            _sprite_node.texture = texture
            _current_skin_id = skin_data.id
        else:
            push_error("Failed to load texture for skin: " + skin_data.texture_path)

    func get_current_skin_id() -> String:
        return _current_skin_id
    ```

    *   **`@export var default_skin_data: PlayerSkinData`**: Allows us to assign a default skin resource in the editor.
    *   **`owner.find_child("Sprite")`**: Safely finds the `Sprite2D` node.
    *   **`load(skin_data.texture_path)`**: This is the dynamic loading in action. It loads the texture based on the path stored in the `PlayerSkinData` resource.

### 4. Add `SkinComponent` to the Player Scene

1.  Open `res://scenes/entities/Player.tscn`.
2.  Select the root `Player` node.
3.  Add a new `Node` as a child and rename it to `SkinComponent`.
4.  Attach `SkinComponent.gd` (`res://scripts/components/SkinComponent.gd`) to it.
5.  In the Inspector for the `SkinComponent` node, drag `res://data/game_data/PlayerSkinBlue.tres` into the "Default Skin Data" slot.

### 5. Add a Test Functionality to Change Skins (Optional)

Let's quickly add a way to switch skins in our `PlayerMovementTest.gd` for demonstration purposes.

1.  Open `res://scripts/components/PlayerMovementTest.gd`.
2.  Add a reference to the `SkinComponent` and modify the `_physics_process` (or `_input`) to switch skins.

    ```gdscript
    # PlayerMovementTest.gd
    extends CharacterBody2D

    @export var movement_data: MovementData
    var health_component: HealthComponent
    var skin_component: SkinComponent # New reference

    # Preload skin data for quick switching demonstration
    # Note: For many skins, you'd use a DataManager to get them, not preload all.
    const RED_SKIN_DATA: PlayerSkinData = preload("res://data/game_data/PlayerSkinRed.tres")
    const BLUE_SKIN_DATA: PlayerSkinData = preload("res://data/game_data/PlayerSkinBlue.tres")

    func _ready():
        if movement_data == null:
            push_error("MovementData resource not assigned to PlayerMovementTest!")
            set_process(false)
            return

        health_component = get_node("../HealthComponent")
        if health_component:
            health_component.health_changed.connect(_on_health_changed)
            health_component.died.connect(_on_died)
            print("PlayerMovementTest connected to HealthComponent signals.")
        else:
            push_error("HealthComponent not found as sibling of CharacterBody2D!")

        # Get reference to SkinComponent
        skin_component = get_node("../SkinComponent") # Assumes SkinComponent is a sibling
        if not skin_component:
            push_error("SkinComponent not found as sibling of CharacterBody2D!")
            set_process(false) # Or handle gracefully

    func _physics_process(delta: float):
        if movement_data == null: return

        var direction = InputManager.get_movement_direction()
        if direction:
            velocity = direction * movement_data.speed
        else:
            velocity = Vector2.ZERO
        move_and_slide()

        if Input.is_action_just_pressed("ui_accept"):
            if health_component:
                health_component.take_damage(10)
            else:
                print("Cannot take damage, HealthComponent not found.")

        # Test skin switching on 'E' key
        if Input.is_action_just_pressed("ui_text_next"): # Default action for E
            if skin_component:
                if skin_component.get_current_skin_id() == BLUE_SKIN_DATA.id:
                    skin_component.apply_skin_from_data(RED_SKIN_DATA)
                else:
                    skin_component.apply_skin_from_data(BLUE_SKIN_DATA)
            else:
                print("Cannot switch skin, SkinComponent not found.")
    
    # ... _on_health_changed and _on_died methods remain the same ...
    ```

    *   **`preload("res://...")`**: We're using `preload` here for the *data resources* themselves, as they are small and we know their paths. The *texture* inside them is loaded dynamically using `load()`.
    *   **`ui_text_next`**: Another default Godot input action, typically bound to `E`. Add `E` to this action in `Project Settings -> Input Map` if it's not already there.

6.  Save `PlayerMovementTest.gd` and `Player.tscn`.

### 6. Test Dynamic Skin Loading

1.  Run `res://scenes/levels/Main.tscn` (F5).
2.  Your player should start with the blue skin (or whatever you set as default).
3.  Press the `E` key. The player's sprite should immediately change to the red skin. Press `E` again to switch back.
4.  Observe the "Output" panel for "Applying skin..." messages.

## Consistency Check

You've successfully implemented dynamic resource loading using `load()` within a `SkinComponent`. This allows your player to change appearances at runtime based on data resources, a flexible approach that's vital for modding and content expansion. We've also touched on the difference between `preload()` and `load()`, and the importance of `ResourceLoader` for asynchronous operations (which we'll explore more in future modding chapters).

In the next chapter, we will design the `GameContext`, a central but decoupled point to manage the lifecycle and dependencies of our core game systems, bringing together our `AutoLoad`s and other foundational elements.
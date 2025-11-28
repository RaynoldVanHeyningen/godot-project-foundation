# Chapter 16: Externalizing Game Data with Custom Data Resources

## Goal

The goal of this chapter is to further expand our data-driven design by externalizing more comprehensive game data structures using **Custom Godot Resources**. We will move beyond component-specific configuration to define broader game elements like `ItemData`, `EnemyData`, or `AbilityData`. This involves abstracting core game definitions from logic, making it easier to create vast amounts of content, balance the game, and provide clear extension points for modding.

## Concept Explanation: Game Data Resources

Just as we used `ComponentData` resources to configure individual components, we can use broader "Game Data Resources" to define entire game concepts. These resources typically don't directly attach to a `Node` or `Component`, but are instead loaded and managed by dedicated systems (e.g., an `InventorySystem` loads `ItemData` resources, an `EnemySpawnSystem` loads `EnemyData` resources).

Examples:

*   **`ItemData.gd`**: Defines properties for all items in the game (name, description, icon, weight, stack size, effects, etc.).
*   **`EnemyData.gd`**: Defines properties for different enemy types (base `HealthData`, `MovementData`, attack patterns, loot tables, visual assets, etc.).
*   **`AbilityData.gd`**: Defines properties for player or enemy abilities (cooldown, cost, animation, effects, target type, etc.).

These resources form the "database" of your game's content.

### Why Externalizing Game Data is Crucial:

*   **Content Volume**: Enables the creation of hundreds or thousands of distinct items, enemies, or abilities without writing unique code for each.
*   **Balance & Tuning**: Game designers can adjust stats and properties across the entire game's content by modifying `.tres` files, facilitating rapid iteration and balancing.
*   **Referential Integrity**: Resources can reference other resources (e.g., `EnemyData` references `HealthData`, `ItemData` references `Texture2D`), creating a powerful, interconnected data graph.
*   **Modding Foundation**: Modders can easily add new items, enemies, or abilities by simply creating new `.tres` files that conform to your defined `Resource` schemas. The game's logic will then automatically recognize and use this new content.
*   **Clear Definitions**: Each resource defines a specific game concept, making the game's design clear and well-documented.

## Architectural Reasoning: The Data Layer

These `Game Data Resources` represent the **data layer** of your game. They sit above the individual components and provide the overarching definitions for game objects.

*   **`GameContext` / `GameStateManager`**: Might load a central `DataManager` `AutoLoad`.
*   **`DataManager` (Future)**: Responsible for loading, caching, and providing access to all these `Game Data Resources`.
*   **Game Systems (e.g., `InventorySystem`, `EnemySpawnSystem`)**: Query the `DataManager` for specific `ItemData` or `EnemyData` based on IDs.
*   **Components (e.g., `HealthComponent`)**: Receive their specific `ComponentData` (like `HealthData`) either directly assigned in the editor or provided by a game system at runtime.

This layered approach ensures that game logic is entirely separated from content definitions.

## Production Mindset Notes: Structuring Resources

*   **Resource References**: Use `@export var` with type hinting to allow resources to reference other resources directly in the editor (e.g., `EnemyData` has an `@export var health_data: HealthData`). This is incredibly powerful.
*   **Unique IDs**: Every `Game Data Resource` should have a unique string `id` (`@export var id: String`) property. This ID is how your game systems will request and identify specific data instances.
*   **Descriptive Properties**: Define clear and descriptive `@export` properties in your `Resource` scripts, making them intuitive for content creators.
*   **Folder Organization**: Maintain a clear folder structure for your `.tres` files (e.g., `res://data/game_data/items/`, `res://data/game_data/enemies/`).

## Step-by-Step Instructions: Creating `ItemData` and `EnemyData` Resources

We will create `ItemData` and `EnemyData` custom resources, demonstrating how they can reference other resources and define complex game objects.

### 1. Create `ItemData` Custom Resource

1.  In `res://scripts/data_types/`, create a new script named `ItemData.gd`.
2.  Ensure it `extends Resource`.
3.  Add the `class_name` and `@export` properties:

    ```gdscript
    # ItemData.gd
    extends Resource
    class_name ItemData

    enum ItemType {GENERIC, WEAPON, ARMOR, CONSUMABLE, QUEST}

    @export var id: String = "" # Unique identifier (e.g., "sword_iron", "potion_health")
    @export var item_name: String = "New Item"
    @export var description: String = "A generic item."
    @export var item_type: ItemType = ItemType.GENERIC
    @export var icon_texture: Texture2D # Visual representation in UI
    @export var stackable: bool = false
    @export var max_stack_size: int = 1
    @export var weight: float = 0.0
    @export var value: int = 0 # Gold value

    # Example: If this is a weapon, it might reference a WeaponStatsData
    # @export var weapon_stats: WeaponStatsData
    # Example: If this is a consumable, it might reference a ConsumableEffectData
    # @export var consumable_effect: ConsumableEffectData
    ```

    *   **`ItemType` enum**: Demonstrates how enums can be used for categorization.
    *   **`id`**: Essential for programmatic lookup.
    *   **`icon_texture: Texture2D`**: A resource referencing another resource (a texture).

### 2. Create `EnemyData` Custom Resource

This resource will demonstrate how to reference `ComponentData` resources (like `HealthData` and `MovementData`) to define an enemy's base stats.

1.  In `res://scripts/data_types/`, create a new script named `EnemyData.gd`.
2.  Ensure it `extends Resource`.
3.  Add the `class_name` and `@export` properties:

    ```gdscript
    # EnemyData.gd
    extends Resource
    class_name EnemyData

    @export var id: String = "" # Unique identifier (e.g., "goblin", "orc_grunt")
    @export var enemy_name: String = "New Enemy"
    @export var visual_scene: PackedScene # Reference to the enemy's visual/base scene (e.g., a Goblin.tscn)
    @export var base_health_data: HealthData # References our HealthData resource
    @export var base_movement_data: MovementData # References our MovementData resource
    @export var experience_on_death: int = 10
    @export var loot_table_id: String = "" # Reference to a future loot table resource
    # Add more enemy-specific properties as needed
    ```

    *   **`visual_scene: PackedScene`**: References a scene, which could be an `Entity.tscn` with specific visuals.
    *   **`base_health_data: HealthData`**: This is a key example of a `Game Data Resource` (EnemyData) referencing a `ComponentData` resource (HealthData).
    *   **`base_movement_data: MovementData`**: Similar reference for movement.

### 3. Create Instances of `ItemData` and `EnemyData`

Let's create some example data files.

1.  **Create Item Folders**: In `res://data/game_data/`, create a new folder named `items/`.
2.  **Create `HealthPotionData.tres`**:
    *   In `res://data/game_data/items/`, create a "New Resource..." -> `ItemData`.
    *   Name it `HealthPotionData.tres`.
    *   Set properties:
        *   `ID`: `potion_health`
        *   `Item Name`: `Health Potion`
        *   `Description`: `Restores a small amount of health.`
        *   `Item Type`: `CONSUMABLE`
        *   `Icon Texture`: (Drag a small image, e.g., `icon.svg` or create a new `potion_icon.png` in `res://assets/graphics/`)
        *   `Stackable`: `true`
        *   `Max Stack Size`: `5`
        *   `Value`: `20`
3.  **Create Enemy Folders**: In `res://data/game_data/`, create a new folder named `enemies/`.
4.  **Create `GoblinHealthData.tres` and `GoblinMovementData.tres`**:
    *   In `res://data/game_data/enemies/`, create a "New Resource..." -> `HealthData`.
    *   Name it `GoblinHealthData.tres`. Set `Max Health` to `30`, `Invulnerability Duration` to `0.0` (goblins are squishy).
    *   In `res://data/game_data/enemies/`, create a "New Resource..." -> `MovementData`.
    *   Name it `GoblinMovementData.tres`. Set `Speed` to `70.0`.
5.  **Create `GoblinData.tres`**:
    *   In `res://data/game_data/enemies/`, create a "New Resource..." -> `EnemyData`.
    *   Name it `GoblinData.tres`.
    *   Set properties:
        *   `ID`: `goblin`
        *   `Enemy Name`: `Goblin Grunt`
        *   `Visual Scene`: (Leave empty for now, we don't have a Goblin.tscn yet)
        *   `Base Health Data`: Drag `GoblinHealthData.tres` here.
        *   `Base Movement Data`: Drag `GoblinMovementData.tres` here.
        *   `Experience On Death`: `10`

### 4. Demonstrate Data Retrieval (No Code Change, Conceptual)

At this point, we don't need to change any existing game logic. The purpose of this chapter is to create the *data*. In a future `DataManager` (Chapter 18), we would load these resources.

For now, imagine a script that wants to create a goblin:

```gdscript
# Conceptual code for a future EnemySpawner or DataManager
func spawn_goblin():
    var goblin_data: EnemyData = load("res://data/game_data/enemies/GoblinData.tres")
    if goblin_data:
        print("Spawning " + goblin_data.enemy_name + " with ID: " + goblin_data.id)
        print("  Base Health: " + str(goblin_data.base_health_data.max_health))
        print("  Base Speed: " + str(goblin_data.base_movement_data.speed))
        
        # In a real scenario, you'd instance goblin_data.visual_scene,
        # and then assign goblin_data.base_health_data to its HealthComponent, etc.
        # var goblin_scene_instance = goblin_data.visual_scene.instantiate()
        # var health_component = goblin_scene_instance.find_child("HealthComponent") as HealthComponent
        # if health_component:
        #     health_component.health_data = goblin_data.base_health_data
        #     health_component._on_entity_ready() # Manually call if needed for dynamic setup
    else:
        push_error("Failed to load GoblinData!")
```

This conceptual code highlights how a system would load a top-level `EnemyData` resource, and then access its nested `ComponentData` resources (`HealthData`, `MovementData`) to configure an enemy `Entity`.

## Consistency Check

You have successfully created new custom `ItemData` and `EnemyData` resources, demonstrating how to define complex game entities and items purely through data. Crucially, you've seen how these `Game Data Resources` can reference `ComponentData` resources, building a powerful, interconnected data graph. This robust data layer is a cornerstone of a scalable, maintainable, and highly moddable AAA project.

In the next chapter, we will implement the actual systems to load these external configuration and mod data files, moving beyond `res://` paths to support user-generated content directly.
# Chapter 17: Loading External Configuration & Mod Data (JSON/CSV)

## Goal

The goal of this chapter is to implement systems to **read game data from external files**, specifically JSON or CSV formats. This goes beyond Godot's native `.tres` resources to support easily modifiable and extensible content. We will learn how to use Godot's `FileAccess` to read structured data, parse it into usable formats, and understand how the `user://` directory is leveraged for user-generated content, integrating external data into our game systems.

## Concept Explanation: External Data Files (JSON/CSV)

While Godot's `.tres` resources are excellent for data-driven design *within* the editor and for packaged games, they have some limitations for pure external configuration and modding:

*   **Human Readability/Editability**: `.tres` files, especially in text format, are human-readable, but they are Godot-specific. Non-Godot users (e.g., modders who don't have the Godot editor) might find them less intuitive to create or modify than standard formats like JSON or CSV.
*   **External Tooling**: Many external tools and pipelines generate data in JSON, XML, or CSV formats.
*   **Mod Distribution**: For modders, distributing a simple JSON file is often easier than requiring them to understand Godot's `Resource` system.
*   **Runtime Creation**: It's easier to dynamically generate and save JSON/CSV files at runtime (e.g., for save games or user preferences) than `.tres` files.

**JSON (JavaScript Object Notation)** and **CSV (Comma Separated Values)** are popular plain-text formats for structured data:

*   **JSON**: Ideal for hierarchical, nested data (e.g., an item with multiple properties, some of which are objects or arrays).
*   **CSV**: Best for tabular data (e.g., a list of enemy stats where each row is an enemy and each column is a property).

### Why External Data Files are Crucial for Modding:

*   **Standard Formats**: Modders can use any text editor or spreadsheet software to create and modify content.
*   **Loose Coupling**: The game logic reads these files, so modders don't touch any game code.
*   **`user://` Directory**: Godot provides a special `user://` path, which resolves to a persistent, user-specific directory on the player's machine. This is the *ideal* location for save games, user settings, and, critically, **user-generated mods**. Data placed here by modders can be loaded by the game.

## Architectural Reasoning: The `DataManager` AutoLoad

To manage the loading of these external data files (and eventually our `.tres` files), we'll introduce a **`DataManager`** `AutoLoad`. This `DataManager` will be responsible for:

*   **Centralized Data Access**: Provide a single point for any part of the game to request data (e.g., `DataManager.get_item_data("potion_health")`).
*   **Loading Logic**: Encapsulate the details of reading from JSON/CSV/`.tres` files.
*   **Caching**: Store loaded data in memory to avoid redundant file reads.
*   **Mod Data Integration**: Prioritize loading data from `user://` (mod directory) over `res://` (game's default resources).

This `DataManager` acts as a crucial layer between our game logic and our diverse data sources.

## Production Mindset Notes: Error Handling & Robustness

*   **File Existence Checks**: Always check if a file exists before trying to open it.
*   **Parsing Errors**: Be prepared for malformed JSON/CSV and handle parsing errors gracefully.
*   **Default Values**: If a data file is missing or malformed, have fallback default values.
*   **Mod Prioritization**: When loading data, establish a clear priority: `user://` (mod data) > `res://` (game data).
*   **Directory Structure**: Encourage modders to follow a specific directory structure within `user://mods/` (e.g., `user://mods/my_mod/data/items.json`).

## Step-by-Step Instructions: Implementing a `DataManager` for JSON/CSV

We will create a `DataManager` `AutoLoad` that can load `ItemData` from a JSON file. This demonstrates the core principles for external data loading.

### 1. Create a Sample JSON Data File

Let's create a `items.json` file that defines some items, similar to our `ItemData.tres`.

1.  In `res://data/game_data/items/`, create a new file named `items.json`. You can do this with a text editor or directly in Godot by right-clicking, "New Resource...", then choosing "TextFile" and renaming it to `items.json`.
2.  Add the following JSON content:

    ```json
    [
        {
            "id": "potion_health_small",
            "item_name": "Small Health Potion",
            "description": "Restores a small amount of health.",
            "item_type": "CONSUMABLE",
            "icon_texture_path": "res://assets/graphics/potion_icon_small.png",
            "stackable": true,
            "max_stack_size": 5,
            "weight": 0.1,
            "value": 10
        },
        {
            "id": "iron_sword",
            "item_name": "Iron Sword",
            "description": "A basic iron sword.",
            "item_type": "WEAPON",
            "icon_texture_path": "res://assets/graphics/sword_icon_iron.png",
            "stackable": false,
            "max_stack_size": 1,
            "weight": 2.5,
            "value": 50
        }
    ]
    ```

    *   **`icon_texture_path`**: Note that we're storing the *path* to the texture, not the `Texture2D` resource itself, as JSON can't directly embed Godot resources. The `DataManager` will load this path.
    *   **Placeholder Textures**: Create `potion_icon_small.png` and `sword_icon_iron.png` (can be duplicates of `icon.svg` for now) in `res://assets/graphics/`.

### 2. Create the `DataManager` AutoLoad Script

This script will load and manage our JSON data.

1.  In `res://scripts/managers/`, create a new script named `DataManager.gd`.
2.  Ensure it `extends Node`.
3.  Add the following code:

    ```gdscript
    # DataManager.gd
    extends Node
    class_name DataManager # Make it globally recognizable

    # AutoLoad singleton responsible for loading and managing all game data (Resources, JSON, CSV).
    # Prioritizes mod data from user:// over res:// game data.

    var _item_data_cache: Dictionary = {}
    var _enemy_data_cache: Dictionary = {}
    # Add caches for other data types as needed

    func _ready():
        print("DataManager ready. Loading game data...")
        _load_all_game_data()

    func _load_all_game_data():
        # Clear caches before reloading
        _item_data_cache.clear()
        _enemy_data_cache.clear()

        # Load default game data from res://
        _load_items_from_json("res://data/game_data/items/items.json")
        # _load_enemies_from_json("res://data/game_data/enemies/enemies.json") # Example for enemies

        # Load mod data from user:// (will override default data if IDs match)
        _load_mod_data()

    func _load_mod_data():
        var mod_dir = "user://mods/"
        var dir = DirAccess.open(mod_dir)

        if dir:
            dir.list_dir_begin()
            var file_name = dir.get_next()
            while file_name != "":
                if dir.current_is_dir() and file_name != "." and file_name != "..":
                    print("DataManager: Found mod directory: " + file_name)
                    # Attempt to load items.json from this mod
                    _load_items_from_json(mod_dir + file_name + "/data/items.json", true)
                    # Add calls for other mod data files (enemies.json, etc.)
                file_name = dir.get_next()
            dir.list_dir_end()
        else:
            print("DataManager: No 'mods' directory found in user://, or unable to open.")

    func _load_items_from_json(path: String, is_mod_data: bool = false):
        var file = FileAccess.open(path, FileAccess.READ)
        if not file:
            if is_mod_data:
                print("DataManager: No mod items data found at: " + path)
            else:
                push_warning("DataManager: Default items data not found at: " + path)
            return

        var content = file.get_as_text()
        file.close()

        var parse_result = JSON.parse_string(content)
        if parse_result is Array:
            for item_data_dict in parse_result:
                if item_data_dict is Dictionary:
                    var id = item_data_dict.get("id", "")
                    if id.is_empty():
                        push_warning("DataManager: Item data entry in " + path + " missing 'id'. Skipping.")
                        continue
                    
                    # Create a new ItemData resource instance for this entry
                    var new_item_data = ItemData.new()
                    new_item_data.id = id
                    new_item_data.item_name = item_data_dict.get("item_name", "Unknown Item")
                    new_item_data.description = item_data_dict.get("description", "")
                    
                    # Convert string to ItemType enum
                    var item_type_str = item_data_dict.get("item_type", "GENERIC").to_upper()
                    new_item_data.item_type = ItemData.ItemType.get(item_type_str, ItemData.ItemType.GENERIC)
                    
                    # Load texture dynamically (important for modding textures)
                    var texture_path = item_data_dict.get("icon_texture_path", "")
                    if not texture_path.is_empty():
                        new_item_data.icon_texture = load(texture_path) # Synchronous load for now
                    
                    new_item_data.stackable = item_data_dict.get("stackable", false)
                    new_item_data.max_stack_size = item_data_dict.get("max_stack_size", 1)
                    new_item_data.weight = float(item_data_dict.get("weight", 0.0))
                    new_item_data.value = int(item_data_dict.get("value", 0))

                    _item_data_cache[id] = new_item_data
                    if is_mod_data:
                        print("DataManager: Loaded/Overrode mod item: " + id + " from " + path)
                    else:
                        print("DataManager: Loaded default item: " + id + " from " + path)
                else:
                    push_warning("DataManager: Invalid item data entry in " + path + ". Expected Dictionary.")
        else:
            push_error("DataManager: Failed to parse JSON from " + path + ". Content: " + content)

    # Public method to get item data by ID
    func get_item_data(id: String) -> ItemData:
        if _item_data_cache.has(id):
            return _item_data_cache[id]
        push_warning("DataManager: Item data with ID '" + id + "' not found.")
        return null # Or return a default "missing item" data
    
    # Add similar functions for enemy data, ability data, etc.
    # func _load_enemies_from_json(path: String, is_mod_data: bool = false): ...
    # func get_enemy_data(id: String) -> EnemyData: ...
    ```

    *   **`_item_data_cache`**: A dictionary to store `ItemData` resources, keyed by their `id`.
    *   **`_load_all_game_data()`**: Orchestrates loading default data and then mod data.
    *   **`_load_mod_data()`**: Iterates through subdirectories in `user://mods/` to find mod data files.
    *   **`_load_items_from_json()`**:
        *   Uses `FileAccess.open()` to read the JSON file.
        *   `JSON.parse_string()` to parse the content.
        *   Iterates through the parsed `Array` of item dictionaries.
        *   **Crucially**: For each item, it creates a new `ItemData` resource (`ItemData.new()`) and populates its properties from the JSON dictionary. This dynamically creates our Godot `Resource` objects from plain text data.
        *   `load(texture_path)`: Dynamically loads the icon texture.
        *   Stores the created `ItemData` resource in the cache.
    *   **`get_item_data()`**: Provides a public API for other scripts to retrieve item data.

### 3. Register `DataManager` as an AutoLoad

1.  Go to `Project` -> `Project Settings...`.
2.  Select the "AutoLoad" tab.
3.  Click the folder icon next to "Path".
4.  Navigate to `res://scripts/managers/DataManager.gd` and select it.
5.  For "Node Name", enter `DataManager`.
6.  Ensure "Enable" is checked.
7.  Click "Add".
8.  Close Project Settings.

### 4. Update `GameContext` to Verify `DataManager`

1.  Open `res://scripts/core/GameContext.gd`.
2.  Modify `_initialize_singletons()` to verify `DataManager`:

    ```gdscript
    # GameContext.gd (Relevant excerpt)
    func _initialize_singletons():
        if not is_instance_valid(InputManager):
            push_error("GameContext: InputManager AutoLoad not found or not initialized!")
            get_tree().quit()
        print("GameContext: InputManager verified.")
        
        if not is_instance_valid(GameStateManager):
            push_error("GameContext: GameStateManager AutoLoad not found or not initialized!")
            get_tree().quit()
        print("GameContext: GameStateManager verified.")

        # NEW: Verify DataManager is loaded
        if not is_instance_valid(DataManager):
            push_error("GameContext: DataManager AutoLoad not found or not initialized!")
            get_tree().quit()
        print("GameContext: DataManager verified.")
    ```

3.  Save `GameContext.gd`.

### 5. Test Data Loading and Modding (Conceptual)

1.  **Run the game (F5)**:
    *   Navigate to the gameplay state.
    *   Observe the "Output" panel. You should see "DataManager ready. Loading game data...", "Loaded default item: potion_health_small...", and "Loaded default item: iron_sword...".
    *   You should also see "DataManager: No 'mods' directory found in user://, or unable to open." because we haven't created one yet.

2.  **Access Data (Conceptual in `PlayerController.gd`)**:
    *   Temporarily add some code to `PlayerController.gd` (or any script in `Main.tscn`) to retrieve data from `DataManager` in `_ready()`:

    ```gdscript
    # PlayerController.gd (Temporary addition for testing)
    func _on_entity_ready():
        super._on_entity_ready()
        # ... existing code ...

        # Test retrieving data from DataManager
        var small_potion = DataManager.get_item_data("potion_health_small")
        if small_potion:
            print("Retrieved item: " + small_potion.item_name + ", type: " + str(small_potion.item_type))
        
        var missing_item = DataManager.get_item_data("non_existent_item")
        if missing_item == null:
            print("Correctly identified 'non_existent_item' as missing.")
    ```
    *   Run the game again and verify these `print` statements. Remove this temporary code after testing.

3.  **Simulate Mod Data**:
    *   **Locate `user://` directory**: In Godot, go to `Project -> Open Project Data Folder` (or `Project -> Open Project Data Folder` in the editor menu, usually at the top right). This opens the `res://` directory. The `user://` path is typically in a separate location, specific to your OS:
        *   **Windows**: `%APPDATA%\Godot\app_userdata\AAA_Blueprint_Course`
        *   **Linux**: `~/.config/godot/app_userdata/AAA_Blueprint_Course`
        *   **macOS**: `~/Library/Application Support/Godot/app_userdata/AAA_Blueprint_Course`
    *   **Create Mod Structure**: Inside the `user://` directory (the app_userdata folder), create the following folders: `mods/my_first_mod/data/`.
    *   **Create Modded `items.json`**: Inside `user://mods/my_first_mod/data/`, create a new `items.json` file.
    *   Paste the following content, which includes a new item and an override for an existing item:

        ```json
        [
            {
                "id": "potion_health_small",
                "item_name": "Modded Small Health Potion",
                "description": "A super potent modded health potion!",
                "item_type": "CONSUMABLE",
                "icon_texture_path": "res://assets/graphics/player_skin_red.svg", # Using existing texture for demo
                "stackable": true,
                "max_stack_size": 10,
                "weight": 0.2,
                "value": 50
            },
            {
                "id": "new_mod_item",
                "item_name": "Mysterious Orb",
                "description": "An orb from another dimension.",
                "item_type": "QUEST",
                "icon_texture_path": "res://assets/graphics/player_skin_blue.svg",
                "stackable": false,
                "max_stack_size": 1,
                "weight": 1.0,
                "value": 1000
            }
        ]
        ```
    *   **Run the game (F5)**:
        *   Observe the "Output" panel. You should now see messages like "DataManager: Found mod directory: my_first_mod".
        *   Then: "DataManager: Loaded/Overrode mod item: potion_health_small from user://mods/my_first_mod/data/items.json".
        *   And: "DataManager: Loaded/Overrode mod item: new_mod_item from user://mods/my_first_mod/data/items.json".
        *   If you re-add the temporary `PlayerController` test code to get `potion_health_small`, it should now print "Modded Small Health Potion" and its modded values.

## Consistency Check

You have successfully implemented a `DataManager` `AutoLoad` capable of loading game data from external JSON files. You've learned how to parse this data and dynamically create Godot `Resource` objects from it. Crucially, you've set up a system to prioritize loading mod data from the `user://` directory, providing a robust foundation for user-generated content and modding. This is a significant step towards a truly extensible AAA project.

In the next chapter, we will introduce the Service Locator pattern, providing a flexible way for components and systems to find required services without hard dependencies, which is particularly beneficial for moddable systems.
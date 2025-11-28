# Chapter 18: Service Locator Pattern for Moddable Systems

## Goal

The goal of this chapter is to implement the **Service Locator pattern**, providing a flexible and decoupled way for components and systems to find and utilize required services (like our `DataManager`, `AudioManager`, or even custom mod services) without hard dependencies. This pattern is crucial for creating highly moddable projects, as it allows modders to replace core services or register their own without modifying existing game code.

## Concept Explanation: The Service Locator Pattern

Imagine a busy office. Instead of every employee knowing exactly where every other employee sits and calling them directly, they have a central "receptionist" or "directory." An employee needs a service (e.g., "send a package"), they ask the receptionist, who then connects them to the Mail Room. The employee doesn't need to know *who* works in the Mail Room or *how* they send packages, just that the service exists.

The **Service Locator** pattern is a design pattern that provides a centralized registry (the "receptionist") where services (like our `DataManager`, `AudioManager`, `Logger`, `InputManager`) can be registered and then looked up by their consumers.

### Key Elements of a Service Locator:

*   **Service Locator (The Registry)**: A central object that holds references to all available services.
*   **Services**: The objects providing specific functionalities (e.g., `DataManager` provides data access, `AudioManager` provides sound playback).
*   **Registration**: Services register themselves (or are registered by the `GameContext`) with the Service Locator.
*   **Lookup**: Consumers request a service from the Service Locator, typically by its type or a unique identifier.

### How Service Locator Differs from Direct AutoLoad Access:

We've been using Godot's `AutoLoad` feature, which acts somewhat like a Service Locator by making services globally accessible by name (e.g., `DataManager.get_item_data()`). So why introduce a separate `ServiceLocator`?

*   **Dynamic Registration/Replacement**: A custom `ServiceLocator` allows services to be registered and *replaced* at runtime. This is incredibly powerful for modding, as a mod could register its own `ModdedDataManager` that overrides the game's default `DataManager`. With `AutoLoad`s, once they're loaded, they're fixed.
*   **Abstraction**: Consumers can request an *interface* or *base type* of a service, rather than a concrete implementation. This makes it easier to swap out different versions of a service.
*   **Testability**: Makes it easier to inject mock services for unit testing.
*   **Controlled Access**: Can provide more controlled access to services than direct global variables.

### Why Service Locator is Crucial for Modding:

*   **Override Core Systems**: Modders can write their own versions of core systems (e.g., a custom `InventoryManager` or `CombatSystem`) and register them with the Service Locator, effectively replacing the game's default implementation.
*   **Add New Services**: Mods can register entirely new services that other mods or even the core game can then discover and use.
*   **Forward Compatibility**: If the game's internal implementation of a service changes, as long as the service's interface (methods) remains compatible, mods don't need to change.

## Architectural Reasoning: Decoupled Service Discovery

The Service Locator pattern promotes **decoupled service discovery**. Instead of a component having a hardcoded reference to `DataManager`, it asks the `ServiceLocator` for "the Data Manager." This means:

*   The component doesn't care *which* Data Manager it gets (the default one, a modded one, a test one).
*   The `GameContext` (or a mod loader) can decide *which* implementation of a service to register.
*   The overall architecture becomes more flexible and adaptable to changes and extensions.

## Production Mindset Notes: Trade-offs and Best Practices

*   **Overuse**: Don't use Service Locator for *every* interaction. For simple parent-child communication or highly localized dependencies, direct references or signals are often clearer. Use it for truly global, interchangeable services.
*   **Initialization Order**: The `ServiceLocator` itself must be initialized very early (as an `AutoLoad`). Services must be registered *before* anything tries to look them up. The `GameContext` is the ideal place for this.
*   **Interfaces/Base Classes**: For maximum flexibility, services should ideally implement a common interface or extend a base class. Consumers then request the service by this base type.
*   **Performance**: Lookups are generally fast, but if hundreds of lookups happen per frame, consider caching the result locally if the service won't change.

## Step-by-Step Instructions: Implementing a `ServiceLocator` AutoLoad

We will create a `ServiceLocator` `AutoLoad` that can register and retrieve services. We'll then register our `DataManager` with it and demonstrate how components can access services through this locator.

### 1. Create the `ServiceLocator` Script

1.  In `res://scripts/core/`, create a new script named `ServiceLocator.gd`. (We place it in `core` as it's a fundamental architectural element).
2.  Ensure it `extends Node`.
3.  Add the following code:

    ```gdscript
    # ServiceLocator.gd
    extends Node
    class_name ServiceLocator # Make it globally recognizable

    # AutoLoad singleton that provides a central registry for game services.
    # Allows for dynamic registration and lookup of services, facilitating modding.

    var _services: Dictionary = {}

    func _ready():
        print("ServiceLocator ready.")

    # Registers a service with the locator.
    # 'service_type' can be a String (unique ID) or a GDScript class (for type-based lookup).
    # If a service with the same type/ID already exists, it will be replaced (useful for mods).
    func register_service(service_type: Variant, service_instance: Node):
        if not service_instance:
            push_error("ServiceLocator: Attempted to register a null service for type: " + str(service_type))
            return
        
        # Use class name if GDScript class is provided
        var key = service_type
        if service_type is GDScript:
            key = service_type.resource_name.get_file().replace(".gd", "") # Get script name without .gd
        elif service_type is String:
            key = service_type # Use provided string as key
        else:
            push_error("ServiceLocator: Invalid service_type provided for registration. Must be String or GDScript.")
            return

        if _services.has(key):
            print("ServiceLocator: Service '" + str(key) + "' already registered. Overriding with new instance.")
        
        _services[key] = service_instance
        print("ServiceLocator: Registered service: " + str(key))

    # Retrieves a service by its type or unique ID.
    func get_service(service_type: Variant) -> Node:
        var key = service_type
        if service_type is GDScript:
            key = service_type.resource_name.get_file().replace(".gd", "")
        elif service_type is String:
            key = service_type
        else:
            push_error("ServiceLocator: Invalid service_type provided for lookup. Must be String or GDScript.")
            return null

        if _services.has(key):
            return _services[key]
        
        push_warning("ServiceLocator: Service '" + str(key) + "' not found.")
        return null
    
    # Optional: Unregister a service
    func unregister_service(service_type: Variant):
        var key = service_type
        if service_type is GDScript:
            key = service_type.resource_name.get_file().replace(".gd", "")
        elif service_type is String:
            key = service_type
        else:
            push_error("ServiceLocator: Invalid service_type provided for unregistration. Must be String or GDScript.")
            return

        if _services.has(key):
            _services.erase(key)
            print("ServiceLocator: Unregistered service: " + str(key))
        else:
            push_warning("ServiceLocator: Attempted to unregister non-existent service: " + str(key))
    ```

    *   **`_services`**: A dictionary to store registered services, keyed by their type/ID.
    *   **`register_service()`**: Takes a `service_type` (can be a `String` ID or a `GDScript` class for type-based lookup) and the `service_instance` (the `Node` object). It handles overriding existing services.
    *   **`get_service()`**: Retrieves a service based on its type/ID.
    *   **`resource_name.get_file().replace(".gd", "")`**: A helper to get the script's class name from its path, which we'll use as the key for `GDScript` types.

### 2. Register `ServiceLocator` as an AutoLoad

The `ServiceLocator` itself needs to be an `AutoLoad` so it's always available from the start.

1.  Go to `Project` -> `Project Settings...`.
2.  Select the "AutoLoad" tab.
3.  Click the folder icon next to "Path".
4.  Navigate to `res://scripts/core/ServiceLocator.gd` and select it.
5.  For "Node Name", enter `ServiceLocator`.
6.  Ensure "Enable" is checked.
7.  Click "Add".
8.  **Order Matters**: Make sure `ServiceLocator` is *above* `DataManager` and `GameStateManager` in the AutoLoad list. It needs to be ready before other systems try to register/lookup services. You can drag and drop entries to reorder them.
9.  Close Project Settings.

### 3. Update `GameContext` to Register Services

The `GameContext` will now be responsible for registering our `AutoLoad` singletons with the `ServiceLocator`.

1.  Open `res://scripts/core/GameContext.gd`.
2.  Modify `_initialize_singletons()` to use `ServiceLocator.register_service()`:

    ```gdscript
    # GameContext.gd (Relevant excerpt)
    func _initialize_singletons():
        # Verify ServiceLocator is loaded first
        if not is_instance_valid(ServiceLocator):
            push_error("GameContext: ServiceLocator AutoLoad not found or not initialized!")
            get_tree().quit()
        print("GameContext: ServiceLocator verified.")

        # Register core AutoLoad singletons with the ServiceLocator
        if is_instance_valid(InputManager):
            ServiceLocator.register_service(InputManager.get_script(), InputManager) # Register by script type
            ServiceLocator.register_service("InputManager", InputManager) # Also register by string ID
            print("GameContext: InputManager registered with ServiceLocator.")
        else:
            push_error("GameContext: InputManager AutoLoad not found!")
        
        if is_instance_valid(GameStateManager):
            ServiceLocator.register_service(GameStateManager.get_script(), GameStateManager)
            ServiceLocator.register_service("GameStateManager", GameStateManager)
            print("GameContext: GameStateManager registered with ServiceLocator.")
        else:
            push_error("GameContext: GameStateManager AutoLoad not found!")

        if is_instance_valid(DataManager):
            ServiceLocator.register_service(DataManager.get_script(), DataManager)
            ServiceLocator.register_service("DataManager", DataManager)
            print("GameContext: DataManager registered with ServiceLocator.")
        else:
            push_error("GameContext: DataManager AutoLoad not found!")
    ```

    *   We now register each `AutoLoad` twice: once by its `GDScript` class (e.g., `InputManager.get_script()`) and once by a `String` ID (e.g., `"InputManager"`). This provides flexibility for consumers.

### 4. Update `PlayerController` to Use `ServiceLocator` for `DataManager`

Now, instead of directly calling `DataManager.get_item_data()`, our `PlayerController` will ask the `ServiceLocator` for the `DataManager`.

1.  Open `res://scripts/components/PlayerController.gd`.
2.  Modify the temporary test code in `_on_entity_ready()`:

    ```gdscript
    # PlayerController.gd (Relevant excerpt)
    func _on_entity_ready():
        super._on_entity_ready()
        # ... existing code ...

        # Test retrieving data from DataManager via ServiceLocator
        var data_manager = ServiceLocator.get_service(DataManager.get_script()) as DataManager # Lookup by script type
        # Or: var data_manager = ServiceLocator.get_service("DataManager") as DataManager # Lookup by string ID

        if data_manager:
            print("PlayerController: DataManager retrieved via ServiceLocator.")
            var small_potion = data_manager.get_item_data("potion_health_small")
            if small_potion:
                print("PlayerController: Retrieved item via ServiceLocator: " + small_potion.item_name)
            
            var new_mod_item = data_manager.get_item_data("new_mod_item") # Test mod item
            if new_mod_item:
                print("PlayerController: Retrieved mod item via ServiceLocator: " + new_mod_item.item_name)

        else:
            push_error("PlayerController: DataManager not found via ServiceLocator!")
    ```

### 5. Test the Service Locator

1.  **Ensure Mod Data is Present**: If you deleted the mod data from the previous chapter, recreate the `user://mods/my_first_mod/data/items.json` file.
2.  **Run the project (F5)**:
    *   Observe the "Output" panel.
    *   You should see `ServiceLocator ready.` first.
    *   Then `GameContext` verifying and registering each `AutoLoad` with the `ServiceLocator`.
    *   Then `DataManager` loading default and modded items.
    *   Finally, `PlayerController` will print its messages, demonstrating it successfully retrieved the `DataManager` via the `ServiceLocator` and accessed both default and modded item data.

3.  **Conceptual Modding (Override `DataManager`)**:
    *   Imagine a mod wants to entirely replace the `DataManager` with its own version that loads data from a custom format.
    *   A mod script (loaded by a future `ModLoader`) could simply do:
        ```gdscript
        # Example ModLoader script
        # var my_mod_data_manager = MyModDataManager.new() # MyModDataManager extends Node
        # ServiceLocator.register_service(DataManager.get_script(), my_mod_data_manager)
        # ServiceLocator.register_service("DataManager", my_mod_data_manager)
        # my_mod_data_manager.load_my_mod_data()
        ```
    *   If this mod script runs *after* the default `DataManager` is registered but *before* `PlayerController` tries to get the service, `PlayerController` would then receive `my_mod_data_manager` instead of the default one, without *PlayerController* needing any code changes. This is the power of Service Locator.

## Consistency Check

You have successfully implemented the Service Locator pattern, creating a central registry for your game's core services. The `GameContext` now registers `AutoLoad` singletons with the `ServiceLocator`, and components like `PlayerController` retrieve these services through the locator. This significantly enhances the moddability of your project, allowing for dynamic replacement and extension of core systems by user-generated content without altering game code.

In the next chapter, we will lay the foundation for Localization and Internationalization, preparing our project to support multiple languages from the very beginning.
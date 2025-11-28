# Chapter 20: Robust Logging & Debugging Utilities

## Goal

The goal of this chapter is to implement a flexible and configurable **logging system** for our Godot project. This system will be essential for development, debugging, and even post-release issue diagnosis. We will create a custom `Logger` singleton (AutoLoad), define different log levels (info, warn, error, debug), and implement conditional logging, with the ability to output messages to both the console and a file.

## Concept Explanation: Why a Custom Logger?

Godot's built-in `print()` and `push_error()`/`push_warning()` functions are useful, but they have limitations for a professional project:

*   **Lack of Control**: You can't easily turn off specific types of messages (e.g., all debug messages) without commenting out code.
*   **No Log Levels**: All messages are treated equally, making it hard to filter important information from noise.
*   **No File Output**: `print()` only goes to the console. For shipped games, you need logs saved to a file for bug reports.
*   **No Context**: `print()` doesn't automatically include timestamps, source file, or line numbers.
*   **Performance**: Excessive `print()` calls can impact performance, especially on release builds.

A **custom logging system** addresses these issues:

*   **Log Levels**: Categorize messages by severity (e.g., `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`).
*   **Filtering**: Configure the logger to only show messages above a certain level (e.g., only `WARNING` and `ERROR` in release builds).
*   **Output Targets**: Send logs to multiple destinations (console, file, network, in-game UI).
*   **Contextual Information**: Automatically add timestamps, source script, line number, or even custom tags.
*   **Performance Control**: In production builds, most debug/info logging can be compiled out or disabled, improving performance.

### Log Levels Explained:

*   **`DEBUG`**: Detailed information, typically only of interest to developers diagnosing problems.
*   **`INFO`**: General information about the program's progress or state.
*   **`WARNING`**: An indication that something unexpected happened, or a problem in the near future (e.g., "resource not found"), but the software is still working as expected.
*   **`ERROR`**: Due to a more serious problem, the software has not been able to perform some function.
*   **`CRITICAL`**: A serious error, indicating that the program itself may be unable to continue running.

## Architectural Reasoning: The `Logger` AutoLoad

Our `Logger` will be implemented as an `AutoLoad` singleton. This makes it globally accessible from any script, ensuring a consistent logging API across the entire project.

*   **Centralized Logging**: All log messages go through this single point, allowing for easy configuration and modification of logging behavior.
*   **Decoupled from Game Logic**: Game logic simply calls `Logger.info("...")`, `Logger.error("...")`. It doesn't need to know *how* or *where* the message is logged.
*   **Service Locator Integration**: We will register our `Logger` with the `ServiceLocator`, allowing for potential modding or replacement of the logging system itself.

## Production Mindset Notes: Release Builds and Performance

*   **Conditional Compilation**: For performance, `DEBUG` level logging should ideally be compiled out or completely disabled in release builds. Godot doesn't have a direct preprocessor for GDScript like C#, but we can achieve similar results with runtime checks or by having different log level configurations.
*   **Asynchronous File Writing**: For very high-volume logging, writing to a file synchronously can cause performance hitches. A more advanced logger might buffer logs and write them to a file on a separate thread or at intervals. For most games, synchronous writing is sufficient.
*   **Log File Rollover**: In long-running games, log files can become huge. A professional logger often includes features to "roll over" log files (e.g., start a new log file daily, or when a certain size is reached).

## Step-by-Step Instructions: Implementing a Custom `Logger`

### 1. Create the `Logger` AutoLoad Script

1.  In `res://scripts/managers/`, create a new script named `Logger.gd`.
2.  Ensure it `extends Node`.
3.  Add the following code:

    ```gdscript
    # Logger.gd
    extends Node
    class_name Logger # Make it globally recognizable

    # AutoLoad singleton for robust logging with different levels and output targets.

    enum LogLevel {
        DEBUG,
        INFO,
        WARNING,
        ERROR,
        CRITICAL
    }

    # @export allows setting in Project Settings -> AutoLoad or in editor if attached to a scene
    @export var current_log_level: LogLevel = LogLevel.DEBUG
    @export var enable_file_logging: bool = true
    @export var log_file_path: String = "user://game.log"
    @export var max_log_file_size_mb: int = 5 # Max size before rotating logs

    var _log_file: FileAccess = null

    func _ready():
        print("Logger ready. Current level: " + LogLevel.keys()[current_log_level])
        if enable_file_logging:
            _initialize_file_logging()
        
        # Register with ServiceLocator
        ServiceLocator.register_service(Logger.get_script(), self)
        ServiceLocator.register_service("Logger", self)
        print("Logger registered with ServiceLocator.")

    func _notification(what: int):
        if what == NOTIFICATION_WM_CLOSE_REQUEST:
            _close_file_logging()

    func _initialize_file_logging():
        _close_file_logging() # Close any existing file access

        # Check for log file size and rotate if necessary
        if FileAccess.file_exists(log_file_path):
            var file_size = FileAccess.get_file_length(log_file_path)
            if file_size > max_log_file_size_mb * 1024 * 1024:
                _rotate_log_file()

        _log_file = FileAccess.open(log_file_path, FileAccess.WRITE_READ) # Open for append
        if _log_file:
            _log_file.seek_end() # Go to end of file to append new logs
            _log_file.store_line("--- Log Session Started: " + Time.get_datetime_string_from_system() + " ---")
            _log_file.flush() # Ensure it's written immediately
            print("Logger: File logging enabled to: " + log_file_path)
        else:
            push_error("Logger: Failed to open log file: " + log_file_path)
            enable_file_logging = false # Disable if we can't open

    func _rotate_log_file():
        var old_path = log_file_path
        var timestamp = Time.get_datetime_string_from_system().replace(":", "-").replace("T", "_")
        var new_path = old_path.replace(".log", "_" + timestamp + ".log")
        
        print("Logger: Rotating log file from " + old_path + " to " + new_path)
        DirAccess.rename_absolute(old_path, new_path)


    func _close_file_logging():
        if _log_file and _log_file.is_open():
            _log_file.store_line("--- Log Session Ended: " + Time.get_datetime_string_from_system() + " ---")
            _log_file.close()
            _log_file = null

    # Public logging methods
    func debug(message: String, context: String = ""):
        _log(LogLevel.DEBUG, message, context)

    func info(message: String, context: String = ""):
        _log(LogLevel.INFO, message, context)

    func warn(message: String, context: String = ""):
        _log(LogLevel.WARNING, message, context)

    func error(message: String, context: String = ""):
        _log(LogLevel.ERROR, message, context)

    func critical(message: String, context: String = ""):
        _log(LogLevel.CRITICAL, message, context)

    # Internal logging function
    func _log(level: LogLevel, message: String, context: String = ""):
        if level < current_log_level: # Filter by current_log_level
            return

        var timestamp = Time.get_datetime_string_from_system()
        var level_str = LogLevel.keys()[level]
        var formatted_message = "[%s][%s]%s %s" % [timestamp, level_str, ("[" + context + "]") if not context.is_empty() else "", message]

        # Output to console
        match level:
            LogLevel.ERROR, LogLevel.CRITICAL:
                push_error(formatted_message)
            LogLevel.WARNING:
                push_warning(formatted_message)
            _:
                print(formatted_message)
        
        # Output to file
        if enable_file_logging and _log_file and _log_file.is_open():
            _log_file.store_line(formatted_message)
            _log_file.flush() # Ensure it's written immediately
    ```

    *   **`LogLevel` enum**: Defines our log severity levels.
    *   **`@export var current_log_level`**: Allows setting the minimum log level directly in the editor or Project Settings.
    *   **`enable_file_logging`, `log_file_path`, `max_log_file_size_mb`**: Configurable properties for file logging.
    *   **`_initialize_file_logging()`**: Handles opening the log file in `user://` and basic log rotation.
    *   **`_rotate_log_file()`**: Renames the old log file with a timestamp.
    *   **`_close_file_logging()`**: Ensures the log file is properly closed on application exit.
    *   **Public methods (`debug`, `info`, `warn`, `error`, `critical`)**: The API for other scripts to use.
    *   **`_log()`**: The internal core logic. It filters messages based on `current_log_level`, formats them with a timestamp and level, and outputs them to both console and file.
    *   **Service Locator**: Registers itself with the `ServiceLocator`.

### 2. Register `Logger` as an AutoLoad

1.  Go to `Project` -> `Project Settings...`.
2.  Select the "AutoLoad" tab.
3.  Click the folder icon next to "Path".
4.  Navigate to `res://scripts/managers/Logger.gd` and select it.
5.  For "Node Name", enter `Logger`.
6.  Ensure "Enable" is checked.
7.  Click "Add".
8.  **Order Matters**: Place `Logger` *above* `ServiceLocator` in the AutoLoad list. `Logger` needs to be ready first so `ServiceLocator` can use it for its own `print` statements during registration, and so `GameContext` can verify it.
9.  In the `Logger` entry in the AutoLoad list, you can click its "Config" icon (wrench) to expose the `@export` variables in the Inspector. Set `Current Log Level` to `DEBUG` for now.
10. Close Project Settings.

### 3. Update `GameContext` to Verify `Logger`

1.  Open `res://scripts/core/GameContext.gd`.
2.  Modify `_initialize_singletons()` to verify `Logger` (it should be the very first `AutoLoad` verified after `ServiceLocator` itself).

    ```gdscript
    # GameContext.gd (Relevant excerpt)
    func _initialize_singletons():
        # Verify ServiceLocator is loaded first (as Logger registers with it)
        if not is_instance_valid(ServiceLocator):
            push_error("GameContext: ServiceLocator AutoLoad not found or not initialized!")
            get_tree().quit()
        # print("GameContext: ServiceLocator verified.") # Logger will now print this
        Logger.info("ServiceLocator verified.", "GameContext") # Use Logger

        # Verify Logger is loaded
        if not is_instance_valid(Logger):
            push_error("GameContext: Logger AutoLoad not found or not initialized!")
            get_tree().quit()
        Logger.info("Logger verified and registered with ServiceLocator.", "GameContext") # Use Logger
        
        # Now register other services, using Logger for output
        if is_instance_valid(InputManager):
            ServiceLocator.register_service(InputManager.get_script(), InputManager)
            ServiceLocator.register_service("InputManager", InputManager)
            Logger.info("InputManager registered with ServiceLocator.", "GameContext")
        else:
            Logger.error("InputManager AutoLoad not found!", "GameContext")
        
        if is_instance_valid(GameStateManager):
            ServiceLocator.register_service(GameStateManager.get_script(), GameStateManager)
            ServiceLocator.register_service("GameStateManager", GameStateManager)
            Logger.info("GameStateManager registered with ServiceLocator.", "GameContext")
        else:
            Logger.error("GameStateManager AutoLoad not found!", "GameContext")

        if is_instance_valid(DataManager):
            ServiceLocator.register_service(DataManager.get_script(), DataManager)
            ServiceLocator.register_service("DataManager", DataManager)
            Logger.info("DataManager registered with ServiceLocator.", "GameContext")
        else:
            Logger.error("DataManager AutoLoad not found!", "GameContext")
    ```

    *   Notice how `print()` calls are replaced with `Logger.info()` or `Logger.error()`. The second argument is a "context" string, useful for filtering.

### 4. Replace `print()` Calls with `Logger` Calls

Now, go through your other scripts and replace `print()` calls with appropriate `Logger` calls. This is a crucial step for consistency.

1.  **`HealthComponent.gd`**:
    *   Replace `print()` with `Logger.info()` and `Logger.warn()`/`Logger.error()`
    *   Use `get_owner_entity().name` as context.

    ```gdscript
    # HealthComponent.gd (Relevant excerpt)
    # ...
    var _health: int = 0:
        set(value):
            var old_health = _health
            _health = clampi(value, 0, health_data.max_health if health_data else 1)

            if _health != old_health:
                health_changed.emit(_health, health_data.max_health if health_data else 1)
                if _health == 0 and old_health > 0:
                    died.emit()
                    Logger.warn(tr("Player has died!"), get_owner_entity().name) # Use Logger.warn

    # ... _on_entity_ready() ...
        Logger.info("HealthComponent ready. Max health: " + str(health_data.max_health), get_owner_entity().name)

    func take_damage(amount: int):
        if amount <= 0: return
        if _health > 0 and not _is_invulnerable:
            self.health -= amount
            Logger.info(tr("Player took {amount} damage. Current health: {health}").format({
                "amount": amount,
                "health": _health
            }), get_owner_entity().name)
            
            if health_data.invulnerability_duration > 0:
                _is_invulnerable = true
                _invulnerability_timer.start(health_data.invulnerability_duration)
                Logger.info(tr("Player is now invulnerable for {duration} seconds.").format({
                    "duration": health_data.invulnerability_duration
                }), get_owner_entity().name)

    func heal(amount: int):
        if amount <= 0: return
        if _health < health_data.max_health:
            self.health += amount
            Logger.info(tr("Player healed {amount}. Current health: {health}").format({
                "amount": amount,
                "health": _health
            }), get_owner_entity().name)

    func _on_invulnerability_timeout():
        _is_invulnerable = false
        Logger.info(tr("Player is no longer invulnerable."), get_owner_entity().name)
    ```

2.  **`PlayerController.gd`**:
    *   Replace `print()` with `Logger.info()` and `Logger.error()`
    *   Use `get_owner_entity().name` as context.

    ```gdscript
    # PlayerController.gd (Relevant excerpt)
    # ...
    func _on_entity_ready():
        super._on_entity_ready()
        if not is_instance_valid(get_owner_entity()): return

        _movement_component = get_sibling_component(MovementComponent)
        if not _movement_component:
            Logger.error("No MovementComponent sibling found. Disabling.", get_owner_entity().name + "/PlayerController")
            set_process(false)
            set_physics_process(false)
            return

        _health_component = get_sibling_component(HealthComponent)
        if _health_component:
            _health_component.health_changed.connect(_on_health_changed)
            _health_component.died.connect(_on_died)
            Logger.info("Connected to HealthComponent signals.", get_owner_entity().name + "/PlayerController")
        else:
            Logger.error("No HealthComponent sibling found.", get_owner_entity().name + "/PlayerController")

        set_process_input(true)
        Logger.info("PlayerController ready.", get_owner_entity().name)

    # ... _input() ...
        if event.is_action_just_pressed("ui_accept"):
            if _health_component:
                _health_component.take_damage(10)
            else:
                Logger.warn("Cannot take damage, HealthComponent not found.", get_owner_entity().name + "/PlayerController")
        
        if event.is_action_just_pressed("ui_text_next"):
            var skin_component = get_sibling_component(SkinComponent) as SkinComponent
            if skin_component:
                # ... skin switching logic ...
            else:
                Logger.warn("Cannot switch skin, SkinComponent not found.", get_owner_entity().name + "/PlayerController")
    
    func _on_health_changed(current_health: int, max_health: int):
        Logger.info(tr("Player Health: {current}/{max}").format({
            "current": current_health,
            "max": max_health
        }), get_owner_entity().name + "/PlayerController")

    func _on_died():
        Logger.warn(tr("Player received 'died' signal. Handling player death!"), get_owner_entity().name + "/PlayerController")
        # ...
    ```

3.  **`Component.gd`**:
    *   Replace `print()` with `Logger.info()` and `push_error()`/`push_warning()` with `Logger.error()`/`Logger.warn()`.

    ```gdscript
    # Component.gd (Relevant excerpt)
    # ...
    func _on_entity_ready():
        # ...
        if owner is Entity:
            _owner_entity = owner
        else:
            Logger.error("Owner of '" + name + "' is not an Entity. Expected 'Entity' but got '" + owner.get_class() + "'.", "Component")
            # ...
            return

        Logger.info("Component '" + name + "' on Entity '" + _owner_entity.name + "' is ready.", "Component")

    func get_sibling_component(component_type: GDScript) -> Component:
        if not is_instance_valid(_owner_entity):
            Logger.error("Cannot get sibling component, owner entity is invalid.", "Component")
            return null
        
        # ...
        Logger.warn("Sibling component of type " + component_type.get_class() + " not found on " + _owner_entity.name, "Component")
        return null
    ```
    *   Do similar replacements for `MovementComponent.gd`, `SkinComponent.gd`, `GameState.gd`, `MainMenuState.gd`, `GameplayState.gd`, `GameStateManager.gd`, `DataManager.gd`, `InputManager.gd`, `ServiceLocator.gd`.
    *   **Important**: For `ServiceLocator.gd` and `Logger.gd` themselves, be careful. `Logger` should use `print()` for its *own* initialization messages *before* it's fully ready and registered. Once ready, it can use `Logger.info()`. `ServiceLocator` can use `Logger.info()` after `Logger` is registered.

### 5. Test the Logging System

1.  **Run the project (F5)**.
2.  Observe the "Output" panel. All messages should now be formatted with timestamps, log levels, and contexts.
3.  **Check the log file**:
    *   Go to your `user://` directory (as found in Chapter 17).
    *   You should find a `game.log` file. Open it with a text editor.
    *   Verify that all the formatted log messages are present in the file.
4.  **Test Log Level Filtering**:
    *   Go to `Project -> Project Settings -> AutoLoad`.
    *   Click the wrench icon next to the `Logger` entry.
    *   In the Inspector, change `Current Log Level` to `LogLevel.WARNING`.
    *   Run the game again. You should now only see `WARNING`, `ERROR`, and `CRITICAL` messages in the console and log file, not `INFO` or `DEBUG`.
    *   Change it back to `LogLevel.DEBUG` for development.

## Consistency Check

You have successfully implemented a robust, configurable logging system using a `Logger` `AutoLoad`. This system provides different log levels, outputs to both console and file, and integrates with the `ServiceLocator`. You've also refactored existing `print()` calls to use this new `Logger`, establishing a professional and consistent approach to debugging and information output throughout your project. This is invaluable for maintaining a complex game and diagnosing issues in production.

In the next chapter, we will discuss performance considerations and basic profiling techniques to ensure our AAA foundation remains optimized.
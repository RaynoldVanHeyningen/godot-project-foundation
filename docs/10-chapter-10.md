# Chapter 10: Implementing a Robust Game State Machine

## Goal

The goal of this chapter is to design and implement a robust **Game State Machine (FSM)** to manage the distinct phases and states of our overall game (e.g., Main Menu, Gameplay, Pause, Game Over). We will create a `GameState` base script, implement a central `GameStateManager` `AutoLoad`, and learn how the `GameContext` orchestrates these states to control the flow of our application.

## Concept Explanation: Finite State Machine (FSM)

A **Finite State Machine (FSM)** is a mathematical model of computation. It is an abstract machine that can be in exactly one of a finite number of states at any given time. The FSM can change from one state to another in response to some inputs or events; this is called a **transition**.

In game development, FSMs are incredibly useful for managing complex behaviors or overall game flow. For a game, an FSM helps us define:

*   **States**: Distinct phases of the game where different rules apply (e.g., `MainMenuState`, `GameplayState`, `PauseState`, `GameOverState`).
*   **Transitions**: How the game moves from one state to another (e.g., from `MainMenuState` to `GameplayState` when "Start Game" is pressed).
*   **Actions**: What happens when entering a state (`_on_enter()`), while in a state (`_process()`, `_physics_process()`), and when exiting a state (`_on_exit()`).

### Why a Game State Machine is Essential for AAA Projects:

*   **Clear Flow**: Provides a structured way to manage the entire game's lifecycle, making complex game flows understandable.
*   **Separation of Concerns**: Each state encapsulates the logic relevant to that specific phase, preventing a monolithic `GameManager` script.
*   **Modularity**: New states can be added easily without altering existing state logic.
*   **Maintainability**: Debugging issues related to game flow becomes much simpler as you can pinpoint which state the game is in.
*   **Scalability**: As games grow, the number of distinct phases increases (e.g., cutscene states, tutorial states, multiplayer lobby states). An FSM handles this gracefully.

## Architectural Reasoning: GameState and GameStateManager

Our FSM implementation will consist of two main parts:

1.  **`GameState` (Base Script)**: An abstract base class (script) that defines the interface for all specific game states. Each specific game state (e.g., `MainMenuState`) will extend this base class and implement its `_on_enter()`, `_on_exit()`, and potentially `_process()` methods. This ensures consistency across all states.
2.  **`GameStateManager` (AutoLoad Singleton)**: A central `AutoLoad` responsible for holding the current game state, handling transitions between states, and notifying the active state of engine callbacks (`_process`, `_physics_process`, `_input`). It acts as the "brain" of the FSM.

The `GameContext` will primarily interact with the `GameStateManager` by telling it which initial state to enter. After that, the `GameStateManager` takes over, delegating control to the active `GameState` object. This maintains separation: `GameContext` orchestrates *systems*, `GameStateManager` orchestrates *states*.

## Production Mindset Notes: State Management Best Practices

*   **Explicit Transitions**: All state changes should go through the `GameStateManager`'s `transition_to()` method. Avoid direct state changes from within individual state scripts.
*   **Clean Entry/Exit**: Ensure `_on_enter()` and `_on_exit()` methods are used to set up and tear down resources/nodes specific to that state (e.g., adding/removing UI scenes, pausing/unpausing game logic).
*   **State-Specific Logic**: Keep logic within state scripts focused on *that state*. If logic is shared across states, it might belong in a common utility or a dedicated manager.
*   **Debug Logging**: Log state transitions to aid in debugging game flow.

## Step-by-Step Instructions: Building the Game State Machine

### 1. Create the `GameState` Base Script

This script defines the interface for all our game states.

1.  In `res://scripts/core/`, create a new script named `GameState.gd`.
2.  Ensure it `extends Node`.
3.  Add the following code:

    ```gdscript
    # GameState.gd
    extends Node
    class_name GameState # Make it globally recognizable

    # Base class for all game states.
    # Specific states will inherit from this and implement the methods.

    # Reference to the GameStateManager
    var state_manager: Node

    func _init(_state_manager: Node):
        state_manager = _state_manager

    # Called when entering this state
    func _on_enter():
        print("Entering state: " + name)
        set_process(true)
        set_physics_process(true)
        set_process_input(true)

    # Called when exiting this state
    func _on_exit():
        print("Exiting state: " + name)
        set_process(false)
        set_physics_process(false)
        set_process_input(false)

    # These methods are called by the GameStateManager when this state is active.
    # Override them in child classes to implement state-specific logic.
    func _process(delta: float):
        pass

    func _physics_process(delta: float):
        pass

    func _input(event: InputEvent):
        pass

    # Helper to transition to another state
    func transition_to(new_state_class: GDScript):
        if state_manager:
            state_manager.transition_to(new_state_class)
        else:
            push_error("GameState: Cannot transition, state_manager is null.")
    ```

    *   **`class_name GameState`**: Makes it a global type.
    *   **`_init()`**: Passes a reference to the `GameStateManager` for easy transitions.
    *   **`_on_enter()` / `_on_exit()`**: Lifecycle methods for states. They enable/disable `_process` callbacks by default.
    *   **`transition_to()`**: A helper for states to request a change from the manager.

### 2. Create Specific Game States

Let's create two basic states: `MainMenuState` and `GameplayState`.

1.  In `res://scripts/core/`, create `MainMenuState.gd`:

    ```gdscript
    # MainMenuState.gd
    extends GameState
    class_name MainMenuState

    # Preload the Main Menu scene (we'll create this scene later)
    const MAIN_MENU_SCENE: PackedScene = preload("res://scenes/ui/MainMenu.tscn")
    var _main_menu_instance: CanvasLayer

    func _on_enter():
        super._on_enter()
        print("MainMenuState: Loading main menu UI...")
        _main_menu_instance = MAIN_MENU_SCENE.instantiate()
        get_tree().root.add_child(_main_menu_instance)
        
        # Connect to a hypothetical "Start Game" signal from the Main Menu UI
        # For now, we'll just simulate it with a key press.
        print("MainMenuState: Press 'Start' (Enter) to begin game.")

    func _on_exit():
        super._on_exit()
        print("MainMenuState: Unloading main menu UI...")
        if is_instance_valid(_main_menu_instance):
            _main_menu_instance.queue_free()
            _main_menu_instance = null

    func _input(event: InputEvent):
        if event.is_action_pressed("ui_accept"): # Press Enter to start game
            print("MainMenuState: 'Start Game' pressed. Transitioning to GameplayState.")
            transition_to(GameplayState)
    ```

2.  In `res://scripts/core/`, create `GameplayState.gd`:

    ```gdscript
    # GameplayState.gd
    extends GameState
    class_name GameplayState

    # Preload the main game level scene
    const GAME_LEVEL_SCENE: PackedScene = preload("res://scenes/levels/Main.tscn")
    var _game_level_instance: Node

    func _on_enter():
        super._on_enter()
        print("GameplayState: Loading game level...")
        _game_level_instance = GAME_LEVEL_SCENE.instantiate()
        get_tree().root.add_child(_game_level_instance)

        # Optional: Set up player input, etc. (already handled by PlayerMovementTest)

    func _on_exit():
        super._on_exit()
        print("GameplayState: Unloading game level...")
        if is_instance_valid(_game_level_instance):
            _game_level_instance.queue_free()
            _game_level_instance = null
        # Reset game state, scores, etc.
    ```

### 3. Create a Placeholder `MainMenu.tscn`

To prevent errors when `MainMenuState` tries to load its scene:

1.  In `res://scenes/ui/`, create a new scene named `MainMenu.tscn`.
2.  Add a `CanvasLayer` as the root node.
3.  Add a `Label` as a child of `CanvasLayer`.
4.  Set the `Label`'s `Text` property to "Main Menu - Press Enter to Start".
5.  Position the label centrally.
6.  Save the scene.

### 4. Create the `GameStateManager` AutoLoad

This will be our central FSM controller.

1.  In `res://scripts/managers/`, create a new script named `GameStateManager.gd`.
2.  Ensure it `extends Node`.
3.  Add the following code:

    ```gdscript
    # GameStateManager.gd
    extends Node
    class_name GameStateManager # Make it globally recognizable

    # AutoLoad singleton responsible for managing game states.

    var _current_state: GameState = null

    func _ready():
        print("GameStateManager ready.")
        set_process(true) # Enable _process, _physics_process, _input
        set_physics_process(true)
        set_process_input(true)

    func _notification(what: int):
        if what == NOTIFICATION_WM_CLOSE_REQUEST:
            print("Game window close requested. Exiting current state.")
            if _current_state:
                _current_state._on_exit()
            get_tree().quit()

    # Public method to transition to a new state
    func transition_to(new_state_class: GDScript):
        if not new_state_class is GDScript:
            push_error("GameStateManager: Invalid new_state_class provided for transition.")
            return

        # Exit current state
        if _current_state:
            _current_state._on_exit()
            _current_state.queue_free() # Free the old state node

        # Enter new state
        _current_state = new_state_class.new(self) # Pass self (manager) to the new state
        add_child(_current_state) # Add the state node as a child of the manager
        _current_state._on_enter()
        print("GameStateManager: Transitioned to " + _current_state.name)

    # Delegate engine callbacks to the current active state
    func _process(delta: float):
        if _current_state:
            _current_state._process(delta)

    func _physics_process(delta: float):
        if _current_state:
            _current_state._physics_process(delta)

    func _input(event: InputEvent):
        if _current_state:
            _current_state._input(event)
    ```

    *   **`_current_state`**: Holds the currently active `GameState` object.
    *   **`transition_to()`**: The core logic for changing states. It calls `_on_exit()` on the old state, frees it, creates the new state, adds it as a child (important for `get_tree()` context), and calls `_on_enter()` on the new state.
    *   **`_process`, `_physics_process`, `_input`**: These methods in the `GameStateManager` delegate their calls to the currently active `_current_state`, effectively giving control to the state.
    *   **`_notification(NOTIFICATION_WM_CLOSE_REQUEST)`**: A good practice to ensure the current state exits cleanly when the game window is closed.

### 5. Register `GameStateManager` as an AutoLoad

1.  Go to `Project` -> `Project Settings...`.
2.  Select the "AutoLoad" tab.
3.  Click the folder icon next to "Path".
4.  Navigate to `res://scripts/managers/GameStateManager.gd` and select it.
5.  For "Node Name", enter `GameStateManager`.
6.  Ensure "Enable" is checked.
7.  Click "Add".
8.  Close Project Settings.

### 6. Update `GameContext` to Initialize the Game State Machine

Now, `GameContext` will tell `GameStateManager` which state to start with.

1.  Open `res://scripts/core/GameContext.gd`.
2.  Modify the `_initialize_game_state()` and `_load_initial_scene()` methods:

    ```gdscript
    # GameContext.gd
    extends Node
    class_name GameContext

    func _ready():
        print("GameContext: Initializing core systems...")
        
        _initialize_singletons()
        _initialize_game_state()
        # _load_initial_scene() # No longer needed here, GameStateManager handles scene loading per state

        print("GameContext: All core systems initialized.")

    func _initialize_singletons():
        if not is_instance_valid(InputManager):
            push_error("GameContext: InputManager AutoLoad not found or not initialized!")
            get_tree().quit()
        print("GameContext: InputManager verified.")
        
        # Verify GameStateManager is also loaded
        if not is_instance_valid(GameStateManager):
            push_error("GameContext: GameStateManager AutoLoad not found or not initialized!")
            get_tree().quit()
        print("GameContext: GameStateManager verified.")

    func _initialize_game_state():
        # Tell the GameStateManager to transition to the initial state (e.g., MainMenuState)
        print("GameContext: Initializing Game State Machine...")
        GameStateManager.transition_to(MainMenuState) # Start with the main menu

    func _load_initial_scene():
        # This method is now obsolete as states handle their own scene loading.
        # It's good practice to remove it or ensure it does nothing if no longer needed.
        # For this course, we'll simply comment it out or remove its call from _ready().
        pass 
    ```

3.  Save `GameContext.gd`.

### 7. Run and Verify the Game State Flow

1.  Run the project (F5).
2.  Observe the "Output" panel.
    *   You should see `GameContext` and `GameStateManager` initializing.
    *   Then `GameStateManager: Transitioned to MainMenuState`.
    *   Then `MainMenuState: Loading main menu UI...`
    *   And your `MainMenu.tscn` scene should be displayed, showing "Main Menu - Press Enter to Start".
3.  Press the `Enter` key.
    *   You should see `MainMenuState: 'Start Game' pressed. Transitioning to GameplayState.`
    *   Then `MainMenuState: Unloading main menu UI...`
    *   Then `GameStateManager: Transitioned to GameplayState`.
    *   Then `GameplayState: Loading game level...`
    *   Your `Main.tscn` (with the player) should now be visible, and you can move the player as before.

## Consistency Check

You have successfully implemented a robust Game State Machine using a `GameStateManager` `AutoLoad` and a `GameState` base script. The `GameContext` now gracefully hands over control to the `GameStateManager` to manage the game's overall flow, demonstrating a highly decoupled and scalable architectural pattern. Each state encapsulates its own scene loading, input handling, and lifecycle, making it easy to add more complex game phases in the future.

In the next chapter, we will return to building compositional game elements, starting with a foundational `Entity` base scene that will serve as a flexible container for all dynamic game objects.
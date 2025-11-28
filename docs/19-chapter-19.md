# Chapter 19: Localization & Internationalization Foundation

## Goal

The goal of this chapter is to lay the foundation for **Localization (L10n)** and **Internationalization (I18n)** in our Godot project. We will prepare the project to support multiple languages from the outset by understanding Godot's built-in translation system, using the `tr()` function, and managing translation CSV files. This is a crucial step for reaching a broader audience and aligns with the professional mindset of a AAA project.

## Concept Explanation: L10n and I18n

*   **Internationalization (I18n)**: The process of designing and developing a product in such a way that it can be adapted to different languages, regional differences, and technical requirements without engineering changes. It's about making your software *ready* for localization.
*   **Localization (L10n)**: The process of adapting an internationalized product to a specific locale or market. This includes translating text, adapting images, adjusting date/time formats, currencies, and cultural nuances.

In game development, this primarily means:

1.  **Translating Text**: All user-facing text (UI, dialogue, item descriptions) needs to be translatable.
2.  **Handling Different Assets**: Sometimes, images or audio need to change for different languages.
3.  **Formatting**: Numbers, dates, and times might need different formats.

### Why L10n/I18n is Crucial for a AAA Mindset Project:

*   **Wider Audience**: Games translated into multiple languages can reach a significantly larger global player base, increasing sales and community engagement.
*   **Professionalism**: A game that supports multiple languages feels more polished and accessible.
*   **Scalability**: Integrating translation early avoids costly and complex refactoring later when the project is much larger.
*   **Moddability (for text)**: While less about modding *gameplay*, having a robust translation system allows modders to easily provide translations for their own content, or for existing game content if they wish to improve or add languages.

### Godot's Translation System

Godot provides a robust and easy-to-use translation system:

*   **`tr()` Function**: This global function is the core of Godot's translation. When you call `tr("My Text")`, Godot looks up "My Text" in its loaded translation catalogs for the current locale and returns the translated string. If no translation is found, it returns the original string.
*   **Translation Files (`.po` / `.csv`)**: Godot uses Gettext Portable Object (`.po`) files or simpler CSV files to store translations. `.po` files are standard in localization workflows, while `.csv` are simpler for small projects or direct editing.
*   **Project Settings**: You configure the translation files and default locale in Project Settings.
*   **Editor Support**: Godot's editor can scan your project for translatable strings and generate translation templates.

## Architectural Reasoning: Centralized Text Management

By using `tr()`, we centralize all text lookup. This means:

*   **No Hardcoded Text**: All user-facing strings are identified as translatable.
*   **Dynamic Language Switching**: The game can switch languages at runtime by simply changing the locale.
*   **Decoupled UI**: UI elements don't need to know *how* to translate; they just ask `tr()` for the localized string.
*   **Data-Driven Text**: Even text within our `ItemData` or `EnemyData` resources can be made translatable by storing translation keys (e.g., `item_name_key: String`) instead of raw text, and then calling `tr(item_name_key)` at display time.

## Production Mindset Notes: Best Practices for L10n

*   **Translate Early and Often**: Don't wait until the end of development. Integrate `tr()` from the start.
*   **Context is King**: Provide context for translators. Sometimes a word has different meanings depending on usage. Godot's `.po` files support context.
*   **Placeholder Strings**: For dynamic text (e.g., "Killed 5 enemies"), use placeholders (`tr("Killed {count} enemies").format({"count": 5})`).
*   **Avoid String Concatenation**: Prefer using `String.format()` for dynamic text to allow translators to reorder words if needed for grammatical correctness in their language.
*   **Review Translations**: Machine translation is a start, but always get native speakers to review.
*   **Font Support**: Ensure your chosen fonts support all the character sets of your target languages.

## Step-by-Step Instructions: Implementing Basic Localization

We will set up Godot's translation system, create a CSV translation file, and make some of our UI text translatable.

### 1. Configure Project Settings for Localization

1.  Go to `Project` -> `Project Settings...`.
2.  Select the "Localization" tab.

### 2. Add a New Locale

Let's add English (our default) and Spanish as target languages.

1.  In the "Localization" tab, find the "Translations" section.
2.  For "Add Locale", select `en` (English) and click "Add".
3.  For "Add Locale", select `es` (Spanish) and click "Add".

### 3. Generate a Translation Template

Godot can scan your project for strings wrapped in `tr()` and generate a template.

1.  Still in "Localization" -> "Translations" tab, click "Generate Ts/Po...".
2.  For "Locale", select `en`.
3.  For "Output Path", browse to `res://data/localization/` (create this folder if it doesn't exist).
4.  Name the file `messages.pot` (Portable Object Template). This will be our base template.
5.  Click "Generate".

### 4. Create Translation CSV Files

For simplicity in this course, we'll use CSV files. You can convert `.po` files to `.csv` or create them manually.

1.  **Create `en.csv`**:
    *   In `res://data/localization/`, create a new file named `en.csv`.
    *   Add the following content (header for context, then our strings):
        ```csv
        key,text,context
        Main Menu - Press Enter to Start,Main Menu - Press Enter to Start,
        Player Health: {current}/{max},Player Health: {current}/{max},
        Player took {amount} damage. Current health: {health},Player took {amount} damage. Current health: {health},
        Player is now invulnerable for {duration} seconds.,Player is now invulnerable for {duration} seconds.,
        Player is no longer invulnerable.,Player is no longer invulnerable.,
        Player received 'died' signal. Handling player death!,Player received 'died' signal. Handling player death!,
        Player has died!,Player has died!,
        Retrieved item: {item_name},Retrieved item: {item_name},
        Retrieved mod item via ServiceLocator: {item_name},Retrieved mod item via ServiceLocator: {item_name},
        ```
    *   Save `en.csv`.

2.  **Create `es.csv`**:
    *   In `res://data/localization/`, create a new file named `es.csv`.
    *   Add the following content (translated to Spanish):
        ```csv
        key,text,context
        Main Menu - Press Enter to Start,Menú Principal - Presiona Enter para Iniciar,
        Player Health: {current}/{max},Salud del Jugador: {current}/{max},
        Player took {amount} damage. Current health: {health},El jugador recibió {amount} de daño. Salud actual: {health},
        Player is now invulnerable for {duration} seconds.,El jugador es ahora invulnerable por {duration} segundos.,
        Player is no longer invulnerable.,El jugador ya no es invulnerable.,
        Player received 'died' signal. Handling player death!,El jugador recibió la señal 'muerto'. ¡Manejando la muerte del jugador!,
        Player has died!,¡El jugador ha muerto!,
        Retrieved item: {item_name},Objeto recuperado: {item_name},
        Retrieved mod item via ServiceLocator: {item_name},Objeto de mod recuperado vía ServiceLocator: {item_name},
        ```
    *   Save `es.csv`.

### 5. Add Translation Files to Project Settings

1.  Go back to `Project` -> `Project Settings...` -> "Localization" tab.
2.  In the "Translations" section, for "Add", click the folder icon.
3.  Select `res://data/localization/en.csv` and click "Add".
4.  Repeat for `res://data/localization/es.csv`.
5.  Close Project Settings.

### 6. Make UI and Code Strings Translatable

Now, let's wrap our visible strings in `tr()`.

1.  **`MainMenu.tscn` Text**:
    *   Open `res://scenes/ui/MainMenu.tscn`.
    *   Select the `Label` node.
    *   In the Inspector, for the `Text` property, change it to `tr("Main Menu - Press Enter to Start")`.
    *   Save the scene.

2.  **`HealthComponent.gd` Prints**:
    *   Open `res://scripts/components/HealthComponent.gd`.
    *   Modify the `print` statements to use `tr()` and `format()`:

    ```gdscript
    # HealthComponent.gd (Relevant excerpt)
    # ...
    func _health_set_logic(value: int): # Helper function for cleaner setter
        var old_health = _health
        _health = clampi(value, 0, health_data.max_health if health_data else 1)

        if _health != old_health:
            health_changed.emit(_health, health_data.max_health if health_data else 1)
            if _health == 0 and old_health > 0:
                died.emit()
                print(tr("Player has died!")) # Use tr()
    
    # ... take_damage() ...
    func take_damage(amount: int):
        if amount <= 0: return
        if _health > 0 and not _is_invulnerable:
            self.health -= amount
            print(tr("Player took {amount} damage. Current health: {health}").format({
                "amount": amount,
                "health": _health
            }))
            
            if health_data.invulnerability_duration > 0:
                _is_invulnerable = true
                _invulnerability_timer.start(health_data.invulnerability_duration)
                print(tr("Player is now invulnerable for {duration} seconds.").format({
                    "duration": health_data.invulnerability_duration
                }))

    func _on_invulnerability_timeout():
        _is_invulnerable = false
        print(tr("Player is no longer invulnerable."))
    ```
    *   *Self-correction*: The `_health` setter logic was getting complex with `tr()` calls. It's better to put `tr()` in the `take_damage` method directly or in an event handler. For the setter, we only update `_health` and emit signals. The `print` statements in the setter should be avoided, or moved to a dedicated debug `Logger` (Chapter 20). For now, let's simplify the `_health` setter and move the prints to `take_damage` and `_on_invulnerability_timeout`.

    **Revised `HealthComponent.gd` (more robust)**:

    ```gdscript
    # HealthComponent.gd
    extends Component
    class_name HealthComponent

    signal health_changed(current_health: int, max_health: int)
    signal died()

    @export var health_data: HealthData

    var _health: int = 0:
        set(value):
            var old_health = _health
            # Use health_data.max_health for clamping, fallback to 1 if no data
            _health = clampi(value, 0, health_data.max_health if health_data else 1)

            if _health != old_health: # Only emit if health actually changed
                health_changed.emit(_health, health_data.max_health if health_data else 1)
                if _health == 0 and old_health > 0: # Only emit died once when health hits 0
                    died.emit()
                    # Print here if needed, but event handlers are better for reactions
                    print(tr("Player has died!")) # Example for immediate feedback

    var _invulnerability_timer: Timer = null
    var _is_invulnerable: bool = false

    func _on_entity_ready():
        super._on_entity_ready()
        if not is_instance_valid(get_owner_entity()): return

        if health_data == null:
            push_error("HealthComponent on '" + get_owner_entity().name + "': HealthData resource not assigned. Disabling component.")
            set_process(false)
            return
        
        _invulnerability_timer = Timer.new()
        add_child(_invulnerability_timer)
        _invulnerability_timer.one_shot = true
        _invulnerability_timer.timeout.connect(_on_invulnerability_timeout)

        self.health = health_data.max_health # Initialize current health
        print("HealthComponent on '" + get_owner_entity().name + "' ready. Max health: " + str(health_data.max_health))

    func take_damage(amount: int):
        if amount <= 0: return
        if _health > 0 and not _is_invulnerable:
            self.health -= amount # Use the setter
            print(tr("Player took {amount} damage. Current health: {health}").format({
                "amount": amount,
                "health": _health
            }))
            
            if health_data.invulnerability_duration > 0:
                _is_invulnerable = true
                _invulnerability_timer.start(health_data.invulnerability_duration)
                print(tr("Player is now invulnerable for {duration} seconds.").format({
                    "duration": health_data.invulnerability_duration
                }))

    func heal(amount: int):
        if amount <= 0: return
        if _health < health_data.max_health:
            self.health += amount # Use the setter
            print(tr("Player healed {amount}. Current health: {health}").format({
                "amount": amount,
                "health": _health
            }))

    func get_health() -> int:
        return _health

    func get_max_health() -> int:
        return health_data.max_health if health_data else 1

    func _on_invulnerability_timeout():
        _is_invulnerable = false
        print(tr("Player is no longer invulnerable."))
    ```

3.  **`PlayerController.gd` Prints**:
    *   Open `res://scripts/components/PlayerController.gd`.
    *   Modify `_on_health_changed`, `_on_died`, and the temporary DataManager prints to use `tr()` and `format()`:

    ```gdscript
    # PlayerController.gd (Relevant excerpt)
    # ...
    func _on_health_changed(current_health: int, max_health: int):
        print(tr("Player Health: {current}/{max}").format({
            "current": current_health,
            "max": max_health
        }))

    func _on_died():
        print(tr("Player received 'died' signal. Handling player death!"))
        # ...

    # Temporary DataManager test in _on_entity_ready()
    # ...
        var data_manager = ServiceLocator.get_service(DataManager.get_script()) as DataManager
        if data_manager:
            # ...
            var small_potion = data_manager.get_item_data("potion_health_small")
            if small_potion:
                print(tr("Retrieved item: {item_name}").format({
                    "item_name": small_potion.item_name
                }))
            
            var new_mod_item = data_manager.get_item_data("new_mod_item")
            if new_mod_item:
                print(tr("Retrieved mod item via ServiceLocator: {item_name}").format({
                    "item_name": new_mod_item.item_name
                }))
    # ...
    ```

### 7. Test Localization

1.  **Run the project (F5)**:
    *   The game should start in English (default locale). All `print` statements and the Main Menu label should be in English.
    *   Proceed to gameplay, take damage, observe English messages.

2.  **Change Locale to Spanish**:
    *   Go to `Project` -> `Project Settings...`.
    *   Select the "Localization" tab.
    *   In the "Translations" section, for "Locale", select `es` (Spanish).
    *   Close Project Settings.
    *   **Run the project again (F5)**:
        *   The Main Menu label should now display "Menú Principal - Presiona Enter para Iniciar".
        *   All `print` statements from `HealthComponent` and `PlayerController` should now be in Spanish.

3.  **Change Locale back to English**: Repeat step 2, but select `en`. Run the game to confirm it switches back.

## Consistency Check

You have successfully established a foundational localization system for your Godot project. You've configured translation files, used the `tr()` function to mark strings for translation, and demonstrated how to switch locales at runtime. This prepares your project for a global audience and aligns with the professional standards of a AAA game, ensuring all user-facing text can be easily adapted to different languages.

In the next chapter, we will implement a robust logging and debugging utility, which is essential for diagnosing issues in complex, professional projects.
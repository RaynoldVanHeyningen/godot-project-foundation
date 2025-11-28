# Chapter 2: Setting Up a Professional Godot Project Structure

## Goal

The goal of this chapter is to establish a clear, organized, and standardized file and folder structure within your Godot project. This foundation is critical for project clarity, maintainability, scalability, and efficient collaboration, aligning directly with our AAA mindset. A well-organized project makes it easier for you (and any future teammates or modders) to find assets, scripts, and scenes, reducing cognitive load and preventing errors.

## Concept Explanation: Standardized Folder Hierarchy

Imagine walking into a new office where every desk, file cabinet, and supply closet is meticulously labeled and organized. You'd quickly find what you need. A game project is no different. A standardized folder hierarchy acts as a map, guiding developers to the correct location for every asset, script, and scene.

This approach is crucial because:

*   **Discoverability**: New team members (or your future self) can quickly understand where everything lives.
*   **Consistency**: Reduces confusion and errors when multiple people work on the same project.
*   **Maintainability**: Updates and refactors are easier when you know exactly where relevant files are located.
*   **Scalability**: As your project grows, a strong structure prevents it from becoming a "wild west" of files.
*   **Moddability**: Modders will have an easier time understanding your project's layout, making it simpler for them to integrate their content.

Our proposed structure is designed to separate concerns at the highest level, grouping similar types of assets and logic together.

## Architectural Reasoning: Separation of Concerns at the File System Level

At its core, a good folder structure is about applying the "separation of concerns" principle to your file system. Different types of files have different responsibilities:

*   **`assets/`**: Raw art, audio, and effect files. Their concern is purely content.
*   **`scenes/`**: Compositions of nodes into reusable game objects or levels. Their concern is scene graph organization.
*   **`scripts/`**: GDScript files defining logic and behavior. Their concern is code.
*   **`data/`**: Godot Resources or external configuration files. Their concern is configuration.

By separating these concerns into distinct top-level folders, we ensure that changes in one area (e.g., updating an art asset) are less likely to accidentally affect another (e.g., a core script), and vice-versa.

## Production Mindset Notes: Naming Conventions

Beyond folder structure, consistent naming conventions are vital for a professional project. They provide immediate context about a file's purpose and type. While Godot doesn't enforce strict rules, these are widely adopted best practices:

*   **Scenes (`.tscn`)**: `Player.tscn`, `Level01.tscn`, `MainMenu.tscn`. Use PascalCase (or CamelCase) for scene names.
*   **Scripts (`.gd`)**: `PlayerController.gd`, `HealthComponent.gd`, `GameManager.gd`. Use PascalCase for class names and filenames.
*   **Resources (`.tres`, `.res`)**: `PlayerStats.tres`, `EnemyAIConfig.tres`, `ItemData.tres`. Use PascalCase.
*   **Images (`.png`, `.jpg`)**: `player_idle.png`, `button_hover.png`. Use snake_case or kebab-case.
*   **Audio (`.ogg`, `.wav`)**: `music_main_theme.ogg`, `sfx_gun_shot.wav`. Use snake_case, often with a `music_` or `sfx_` prefix.

Consistency is key. Choose a convention and stick to it.

## Step-by-Step Instructions: Creating the Project Structure

Let's create a new Godot project and set up our foundational folder structure.

1.  **Open Godot Engine**: Launch Godot Engine 4.5.
2.  **Create a New Project**:
    *   Click "New Project".
    *   For "Project Name", enter `AAA_Blueprint_Course`.
    *   Choose a suitable "Project Path" on your system.
    *   Ensure "Renderer" is set to "Forward+" (or "Mobile" if targeting mobile, but Forward+ is a good default for general learning).
    *   Click "Create & Edit".
3.  **Create Core Folders**:
    *   In the Godot editor, locate the "FileSystem" dock (usually on the bottom-left).
    *   Right-click on `res://` (the root of your project).
    *   Select "New Folder...".
    *   Create the following folders, one by one, by typing the name and pressing Enter:
        *   `addons` (for third-party Godot plugins/modules)
        *   `assets` (raw art, audio, fonts, shaders, particles, etc.)
            *   Inside `assets`, create subfolders: `audio`, `fonts`, `graphics`, `materials`, `particles`, `shaders`
        *   `data` (Godot Resources, JSON, CSV, configuration files)
            *   Inside `data`, create subfolders: `configs`, `game_data`
        *   `scenes` (all game scenes: levels, UI, reusable prefabs)
            *   Inside `scenes`, create subfolders: `entities`, `levels`, `ui`
        *   `scripts` (all GDScript files)
            *   Inside `scripts`, create subfolders: `components`, `core`, `managers`, `systems`, `utils`
        *   `temp` (for temporary files, scratchpad scenes, or experimental assets)

Your `FileSystem` dock should now look something like this (collapsed for brevity):

```
res://
├── addons/
├── assets/
│   ├── audio/
│   ├── fonts/
│   ├── graphics/
│   ├── materials/
│   ├── particles/
│   └── shaders/
├── data/
│   ├── configs/
│   └── game_data/
├── scenes/
│   ├── entities/
│   ├── levels/
│   └── ui/
├── scripts/
│   ├── components/
│   ├── core/
│   ├── managers/
│   ├── systems/
│   └── utils/
└── temp/
```

4.  **Create a Main Scene Placeholder**:
    *   Right-click on the `scenes/levels` folder.
    *   Select "New Scene".
    *   Choose "2D Scene" or "3D Scene" depending on your preference (for this course, 2D will be sufficient for most examples, but the principles apply to both). Let's go with "2D Scene".
    *   Rename the root node to `Main`.
    *   Save the scene as `Main.tscn` inside `res://scenes/levels/`.
    *   Go to Project -> Project Settings -> Application -> Run.
    *   Click the folder icon next to "Main Scene" and select `res://scenes/levels/Main.tscn`. This sets our main entry point for the game.

## Consistency Check

At this point, you have a clean, logical project structure that provides a home for every type of file we'll create. This structure will serve as our backbone throughout the course, ensuring that as we add more complex systems and assets, our project remains organized and manageable.

In the next chapter, we'll integrate Git for version control, a crucial step for any professional project, especially when working in a team.
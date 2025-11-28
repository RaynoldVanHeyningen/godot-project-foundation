# Chapter 23: Exporting Your AAA Foundation

## Goal

The goal of this chapter is to guide you through the process of **exporting your Godot project**, preparing your AAA foundation for distribution to various platforms. We will cover configuring export presets, understanding build options, asset packaging, and platform-specific considerations. Successfully exporting your game is the final step in making your project playable by others, and a key part of any professional game development pipeline.

## Concept Explanation: Exporting a Game

Exporting a game means compiling and packaging your project (code, scenes, assets, data) into a standalone executable or package that can be run on target platforms (Windows, macOS, Linux, Android, iOS, Web, etc.). This process transforms your development project into a shippable product.

### Why Exporting is More Than Just "Copying Files":

*   **Runtime Environment**: The exported game includes a stripped-down Godot engine executable, optimized for runtime.
*   **Asset Packaging**: All your `res://` assets are bundled into a single `.pck` (or `.zip`) file, which is efficient and protects your game's content.
*   **Platform-Specific Optimizations**: Export templates are tailored for each platform, often including specific drivers, libraries, or configurations.
*   **Security/Protection**: Packaging assets makes it harder (though not impossible) for users to easily browse or modify your game's raw files.
*   **Modding Implications**: We need to understand how exported games handle external files (like our JSON mod data in `user://`).

## Architectural Reasoning: Production Builds

From an architectural perspective, exporting involves creating a "production build" of your game. This build should be:

*   **Optimized**: Debug features (like our `Logger`'s `DEBUG` level) should be configurable to be disabled or removed.
*   **Self-Contained**: All necessary files should be bundled.
*   **Platform-Specific**: The build must run correctly on its target platform.

Our `DataManager`'s ability to load from `user://` is especially important here. An exported game will still look for `user://` files in the correct persistent data location on the player's system, allowing modders to simply drop their mod files there.

## Production Mindset Notes: Release Checklist

Before exporting a final release build, consider these points:

*   **Testing**: Thoroughly test the exported build on the target platform(s), not just in the editor.
*   **Performance**: Ensure the game runs smoothly on target hardware. Use the profiler on exported builds.
*   **Localization**: Verify all languages work correctly.
*   **Bug Fixing**: Address all known critical bugs.
*   **Configuration**: Set appropriate `Project Settings` for the target (e.g., full screen, resolution, V-sync).
*   **Branding**: Include your game's icon, splash screen, and necessary legal information.
*   **Documentation**: Provide instructions for players, especially regarding modding if supported.
*   **Version Control**: Tag your Git repository with the release version (e.g., `v1.0.0`) so you can always go back to that exact state.

## Step-by-Step Instructions: Configuring and Exporting Your Godot Project

### 1. Download Export Templates

Godot requires "export templates" for each platform.

1.  In the Godot editor, go to `Project` -> `Install Android Build Template` (if targeting Android) or `Project` -> `Export` (if not targeting Android, this opens the Export dialog).
2.  In the "Export" dialog, click "Manage Export Templates...".
3.  Click "Download and Install". Godot will download the official templates. This might take a few minutes.
4.  Once installed, close the "Manage Export Templates" dialog.

### 2. Configure Export Presets

We'll set up a preset for Windows, but the process is similar for other platforms.

1.  In the "Export" dialog, click "Add..." and choose "Windows Desktop".
2.  **Rename**: You can rename the preset (e.g., "Windows Release").
3.  **Required Settings**:
    *   **`Export Presets` -> `Windows`**:
        *   `Custom Template`: Leave empty unless you have custom engine builds.
        *   `Binary`: Leave empty.
    *   **`Resources`**:
        *   `Export Mode`:
            *   `Export all resources in the project`: Default, packages everything.
            *   `Export selected scenes (and their dependencies)`: For smaller, more targeted builds.
            *   `Export selected resources (and their dependencies)`: Similar, for specific assets.
            *   We will stick with `Export all resources in the project` for now.
        *   `Export With Debug`: **Uncheck this for release builds!** This disables editor features, debug symbols, and `_debug_` methods, making the build smaller and faster.
        *   `Embed Pck`: Check this to embed the `.pck` file directly into the executable, resulting in a single `.exe` file. Uncheck if you want a separate `.pck` (e.g., for easier modding if mods replace the entire `.pck`). For our modding, a separate `.pck` can be easier to manage, but an embedded one is simpler for distribution. Let's keep it embedded for simplicity.
        *   `Exclude Files`: Use patterns to exclude files (e.g., `*.gdignore`, `tests/*`, `temp/*`).
            *   Add `tests/*` to `Exclude Files` to ensure our test scripts are not bundled in the final game.
            *   Add `temp/*` to exclude temporary files.
    *   **`Application`**:
        *   `Icon`: Click the folder icon and select your game's icon (e.g., `res://icon.svg`).
        *   `Name`: This is the name that appears in the taskbar/process list.
        *   `Product Name`: Name used by the OS for various purposes.
        *   `Copyright/Company/Description/Version`: Fill these out for professional builds.
    *   **`Boot Splash`**:
        *   `Image`: You can set a custom splash image that appears while your game loads.

4.  **Other Platforms**: Repeat this process for other platforms you target (e.g., `Linux/X11`, `macOS`, `Web`). Each platform will have its own specific settings (e.g., `Architecture` for macOS, `User Permissions` for Android).

### 3. Configure `Logger` for Exported Builds

For release builds, we typically want to limit logging to `WARNING` or `ERROR` level, and ensure file logging is active.

1.  Go to `Project` -> `Project Settings...` -> "AutoLoad" tab.
2.  Select the `Logger` entry.
3.  In the Inspector (which appears when `Logger` is selected in AutoLoad), find the "Export Presets" section.
4.  Click "Add" and select "Windows Release" (or whatever you named your export preset).
5.  Now, under "Windows Release overrides", you can set specific values for `Logger`'s `@export` variables *only for this export preset*:
    *   Set `Current Log Level` to `LogLevel.WARNING`.
    *   Ensure `Enable File Logging` is `true`.
6.  This ensures that when you export using the "Windows Release" preset, your logger will automatically switch to a less verbose mode.

### 4. Perform the Export

1.  In the "Export" dialog, select your "Windows Release" preset.
2.  Click "Export Project...".
3.  Choose a destination folder for your exported game (e.g., `res://build/Windows/`).
4.  Specify the executable name (e.g., `AAA_Blueprint_Course.exe`).
5.  Click "Save".

Godot will now package your game. This might take some time depending on your project size.

### 5. Test the Exported Game

1.  Navigate to the folder where you exported your game (e.g., `res://build/Windows/`).
2.  Run the executable (e.g., `AAA_Blueprint_Course.exe`).
3.  **Verify Functionality**:
    *   Does the game launch?
    *   Does the Main Menu appear?
    *   Can you play the game (move, take damage, switch skins)?
    *   Check the log file: Go to the `user://` directory for your exported game. You should find `game.log` (or whatever you named it). Open it and verify that only `WARNING` and `ERROR` messages (and `INFO` from `GameContext`/`DataManager` if you didn't change them) are present, not `DEBUG` messages.
4.  **Test Modding (if applicable)**:
    *   Copy your `user://mods/my_first_mod/` folder (from Chapter 17) into the `user://` data folder of your *exported game*.
    *   Run the exported game again.
    *   Verify that modded items and overridden data are loaded (check logs).

## Consistency Check

You have successfully configured export presets and exported your Godot project for a target platform. You've learned about essential build options, including disabling debug features and excluding test files. Crucially, you've seen how to configure your `Logger` for release builds and understand how mod data in `user://` will persist across exported versions. This chapter completes the process of taking your AAA architectural foundation from development to a shippable product.

In the final chapter, we will summarize the key learnings of this course and provide guidance for your continued growth as a game developer, looking beyond this foundation.
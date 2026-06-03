# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Automated tests for a Unity mobile game (TrashCat) using **AltTester** on the **TestMu AI** (formerly LambdaTest) real device cloud. The stack is:

- **AltTester SDK** — embedded in the Unity APK/IPA; exposes a WebSocket server that lets `AltDriver` inspect and manipulate Unity game objects at runtime
- **AltDriver** — Python client that connects to the in-app AltTester Server and drives the game
- **Appium** — manages the remote Android/iOS device session on TestMu AI
- **LT Tunnel** (`tunnel/LT`) — a binary that creates a secure WebSocket bridge between localhost (where AltDriver listens) and the cloud device
- **pytest** — test runner with session/class/function-scoped fixtures

## Required Environment Variables

Set these in a `.env` file in the project root (gitignored):

```ini
LT_USERNAME=your_testmuai_username
LT_ACCESS_KEY=your_testmuai_access_key
LT_APP_URL=lt://your_uploaded_app_url
```

## Commands

```bash
# Install dependencies
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt

# Run all tests
pytest

# Run a single test file
pytest tests/test_main_menu.py

# Run a single test by name
pytest tests/test_main_menu.py::TestMainMenu::test_main_menu_page_loaded_correctly

# Run a specific test class
pytest tests/test_game_play.py::TestGamePlay -v

# Run with extra verbosity / short tracebacks (already in pytest.ini defaults)
pytest -v --tb=short
```

## Architecture

### Fixture lifecycle (`tests/conftest.py`)

Three nested fixture scopes control the session:

1. **`lt_tunnel` (session)** — starts `tunnel/LT` binary, polls `http://127.0.0.1:8000/api/v1.0/info` until ready, tears it down after all tests.
2. **`setup` (class)** — creates an Appium `Remote` driver pointing at `mobile-hub.lambdatest.com`, waits 30 s for the app to launch, then instantiates `AltDriver()` which connects to the AltTester Server running inside the app over the tunnel. Wires `BasePage.annotate_callback` so every page object can send step-context annotations to TestMu AI. Yields `alt_driver`; teardown marks the session passed/failed and quits both drivers.
3. **`per_test_annotation` (autouse function)** — annotates start/end of every individual test and pings Appium (`get_display_density`) between tests to prevent idle-timeout disconnection.

### Page Object Model (`pages/`)

All game screens inherit from `BasePage`, which holds the `AltDriver` reference and provides `log()` (delegates to `BasePage.annotate_callback` → TestMu AI annotations).

Pages expose game objects as `@property` attributes using `AltDriver.wait_for_object(By.NAME/PATH/ID, ...)`, and methods for interactions (`.tap()`, `.call_component_method(...)`, `.set_text(...)`, etc.).

Key pages and their Unity scenes:

| Page class | Unity scene | Loaded via |
|---|---|---|
| `StartPage` | `Start` | `driver.load_scene("Start")` |
| `MainMenuPage` | `Main` | `driver.load_scene("Main")` |
| `GamePlay` | (in-game) | `MainMenuPage.press_run()` |
| `StorePage` | `Shop` | `MainMenuPage.press_store()` |
| `SettingsPage` | overlay | `MainMenuPage.press_settings()` |
| `PauseOverlayPage` | overlay | `GamePlay.press_pause()` |
| `GetAnotherChancePage` | overlay | automatic on death |
| `GameOverScreen` | overlay | `GetAnotherChancePage.press_game_over()` |

### How `AltDriver` interacts with Unity

- **Finding objects**: `By.NAME`, `By.PATH` (Unity hierarchy path), `By.ID`
- **Calling C# methods**: `alt_object.call_component_method("ComponentName", "MethodName", "AssemblyName", [args])`
- **Reading C# properties/fields**: `alt_object.get_component_property("ComponentName", "propertyName", "AssemblyName")`
- **Static methods**: `driver.call_static_method("UnityEngine.Screen", "SetResolution", "UnityEngine.CoreModule", [x, y, fullscreen], [types])`
- **Player prefs**: `driver.set_key_player_pref / get_string_key_player_pref / delete_player_pref`
- **Scene control**: `driver.load_scene / get_current_scene / unload_scene`

### Test structure

Each test file has one class (`TestStartPage`, `TestMainMenu`, `TestGamePlay`, `TestStore`, `TestUserJourney`). `setup_method` instantiates page objects and navigates to the starting scene for that suite. `teardown_method` resets state (usually `load_scene("Main")` + `settings_page.delete_data()` + `time.sleep(1)`).

The `setup` fixture injects `alt_driver` and `appium_driver` onto `request.cls`, making them accessible as `self.alt_driver` / `self.appium_driver` inside test classes.

## Tunnel Binary

`tunnel/LT` is the macOS TestMu AI tunnel binary (already committed). It must be executable (`chmod +x tunnel/LT`). The tunnel name is hardcoded to `"alttester-tunnel"` in `conftest.py` and must match the `tunnelName` capability passed to Appium.

## Device Configuration

The `setup` fixture targets `Pixel.*` Android 14 by default. iOS config is commented out. To switch platforms, update the `lt_options` dict in `conftest.py` and change the `platformName` capability accordingly.

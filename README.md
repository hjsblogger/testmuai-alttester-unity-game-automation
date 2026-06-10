# AltTester Unity Game Automation on TestMu AI Real Device Cloud

<p align="center">
<img width="604" height="393" alt="GitHub-Banner-Image" src="https://github.com/user-attachments/assets/616bb2c5-9674-44c5-a618-8576f5328f5d" />
</p>

Automated tests for the demonstrating Unity Game Testing with [AltTester](https://alttester.com/alttester/) and [TestMu AI](https://www.testmuai.com/) (formerly LambdaTest) real device cloud.

> This repo draws inspiration from the [AltTester TestMu AI Python Example)](https://github.com/alttester/EXAMPLES-CSharp-Cloud-Services-AltTrashCat/tree/testmu-ai-python-example) repository, with minimal changes made for clarity purposes.

For more informaration, check out my detailed blog on [Automated Unity Game Testing with AltTester and TestMu AI (Formerly LambdaTest](https://www.testmuai.com/blog/automated-unity-game-testing/)

## Overview

This project demonstrates how to use **AltTester** to automate a Unity-based mobile game on **real Android/iOS devices** in the cloud. It combines:

- **AltTester SDK** — integrates into the Unity application and starts the AltTester Server at runtime
- **AltDriver** - connects to the AltTester Server and automates Unity game objects
- **Appium** — provisions and manages the remote device session on TestMu AI
- **LT Tunnel** — creates a secure WebSocket bridge between your machine and the cloud device
- **pytest** —  provides the test runner with session, class, and function scoped fixtures

Tests use the **Page Object Model (POM)** pattern to keep game-screen interactions separate from test logic.

## Prerequisites

- Python 3.9+
- [TestMu AI account](https://www.testmuai.com/) (free tier works)
- A TrashCat `.apk` (Android) or `.ipa` (iOS) **instrumented with AltTester**, uploaded to TestMu AI App Automation — note the `lt://` app URL it gives you. The instrumented app can be downloaded from [this download link](https://drive.google.com/file/d/1A1cB6KtaeTCY6XRfq148OtP_DPqhHEi5/view?usp=sharing)
- The `LT` tunnel binary placed in `tunnel/LT` (already included in this repo for macOS)

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/hjsblogger/testmuai-alttester-unity-game-automation.git
cd testmuai-alttester-unity-game-automation
```

### 2. Create a virtual environment and install dependencies

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Configure environment variables

Create a `.env` file in the project root (it is gitignored):

```bash
touch .env
```

Add your TestMu AI credentials:

```ini
LT_USERNAME=your_testmu_ai_username
LT_ACCESS_KEY=your_testmu_ai_access_key
LT_APP_URL=lt://your_app_url
```

| Variable | Where to find it |
|---|---|
| `LT_USERNAME` | TestMu AI dashboard → Profile |
| `LT_ACCESS_KEY` | TestMu AI dashboard → Profile |
| `LT_APP_URL` | App Automation → uploaded app's `lt://` URL |

## Dependencies

| Package | Version | Purpose |
|---|---|---|
| `alttester-driver` | ≥ 2.2.5 | AltTester Python SDK |
| `Appium-Python-Client` | ≥ 4.0.0 | Appium session management |
| `pytest` | ≥ 8.0.0 | Test runner |
| `python-dotenv` | ≥ 1.0.0 | Load `.env` credentials |

## Project Structure

```
├── pages/                          # Page Object Model
│   ├── base_page.py                # Base class with annotation helper
│   ├── start_page.py               # Splash/start screen
│   ├── main_menu_page.py           # Main menu (run, store, leaderboard, settings)
│   ├── game_play_page.py           # In-game screen (obstacles, pause, power-ups)
│   ├── pause_overlay_page.py       # Pause overlay
│   ├── get_another_chance_page.py  # Revive / second-chance popup
│   ├── settings_page.py            # Settings popup (sliders, data reset)
│   ├── store_page.py               # In-game shop
│   └── game_over_screen_page.py    # Game over screen
├── tests/
│   ├── conftest.py                 # Session/class fixtures: tunnel, Appium, AltDriver
│   ├── test_start_page.py          # Start screen tests
│   ├── test_main_menu.py           # Main menu tests
│   ├── test_game_play.py           # Gameplay tests
│   ├── test_store.py               # Store tests
│   └── test_user_journey.py        # End-to-end user journey tests
├── tunnel/
│   └── LT                          # TestMu AI tunnel binary (macOS)
├── requirements.txt
├── pytest.ini
└── .env                            # Your credentials (gitignored — create this yourself)
```

## How It Works

<p align="center">
  <img width="625" height="345" alt="AltTester-Working" src="https://github.com/user-attachments/assets/100e2565-4bd3-4dd0-b00e-01cf31a75294" />
</p>

1. **Session start** — `conftest.py` launches the `LT` tunnel binary and waits for it to be ready by polling its local info API.
2. **Appium session** — An Appium `Remote` driver connects to `mobile-hub.lambdatest.com`. This installs and launches the TrashCat app on a real device.
3. **AltDriver** — After the app starts (~30 s), `AltDriver()` connects to the AltTester Server embedded in the app via the tunnel. This gives full access to the Unity scene graph.
4. **Step annotations** — Each page object calls `lambdatest_executor` via `execute_script` to push step-context messages to the TestMu AI dashboard, so you can see exactly what each test was doing when it failed.
5. **Teardown** — The Appium session reports `lambda-status=passed`, then quits. The AltDriver is stopped. At end-of-session the tunnel process is killed.

## Device Configuration

Tests run on **Pixel 8 (Android 14)** by default. To change the device, edit the `lt_options` dict in `tests/conftest.py`:

```python
# Android
"deviceName": "Pixel.*",
"platformVersion": "14",
"platformName": "android",

# iOS — uncomment and adjust
# "deviceName": "iPhone 14",
# "platformVersion": "16",
# "platformName": "ios",
```

## Test Execution

Run the full test suite:

```bash
pytest
```

Run a specific test file:

```bash
pytest tests/test_start_page.py -v
pytest tests/test_main_menu.py -v
pytest tests/test_game_play.py -v
pytest tests/test_store.py -v
pytest tests/test_user_journey.py -v
```

Run a single test by name:

```bash
pytest tests/test_main_menu.py::TestMainMenu::test_main_menu_page_loaded_correctly -v
```

Both the AltTester Desktop and TestMu AI tunnel should be running throughout the course of the execution. This is because the tunnel transparently forwards the WebSocket connection to port 13000 on the cloud device.

<img width="1504" height="746" alt="LambdaTest-AltTester-Connection" src="https://github.com/user-attachments/assets/a099559f-c6c7-4881-8a2c-6b95e923c550" />

Shown below is the execution snapshot that showcases the progress of the test execution:

<img width="1475" height="474" alt="AltTester-Automation-Terminal" src="https://github.com/user-attachments/assets/305d59ea-962a-4781-aa8f-ebf1df82ea91" />

Navigate to [TestMu AI automation dashboard](https://automation.lambdatest.com/build?pageType=build) to check the status of the test execution.

<img width="1503" height="835" alt="LT_Dashboard_1 0" src="https://github.com/user-attachments/assets/21479512-e8e2-47e7-ae3f-ba4049f0219b" />
<br/>
<br/>
<img width="1503" height="821" alt="LT_Dashboard_1" src="https://github.com/user-attachments/assets/eba84bb1-7278-4f21-b614-8ae71715b4db" />

## Test Coverage

### `test_start_page.py`
| Test | Description |
|---|---|
| `test_start_page_loaded_correctly` | Asserts the start screen renders |
| `test_start_button_load_main_menu` | Taps Start and asserts the main menu loads |

### `test_main_menu.py`
| Test | Description |
|---|---|
| `test_main_menu_page_loaded_correctly` | All expected buttons and labels are visible |
| `test_names_of_all_buttons_from_page` | Verifies button names via AltTester component inspection |
| `test_buttons_are_correctly_displayed` | Button labels match expected text |
| `test_delete_data` | Settings → Delete Data resets store counters |
| `test_leader_board_name_high_score_changes` | Sets and reads back a leaderboard name via `set_text` |
| `test_slider_values_change_as_expected` | Moves Master/Music/SFX sliders and checks the value changed |
| `test_get_parent` | Navigates the Unity transform hierarchy |
| `test_get_time_scale_in_game` | Reads and writes Unity's `Time.timeScale` |
| `test_get_current_scene_is_main` | Asserts `get_current_scene()` returns `"Main"` |
| `test_get_application_screen_size` | Changes screen resolution and verifies it |
| `test_string_key_player_pref` | Sets and reads a string PlayerPref |
| `test_delete_key` | Deletes a PlayerPref key and checks it throws on read |
| `test_get_server_version` | Verifies the AltTester server version matches the SDK |
| `test_get_active_cameras` | Checks only one active camera (`Main Camera`) |
| `test_get_all_components` | Reads all Unity components on the store button |
| `test_get_all_properties` | Reads `CanvasRenderer` properties on the store button |
| `test_get_all_fields` | Reads `UI.Button` fields on the store button |
| `test_get_all_methods` | Reads `CanvasRenderer` methods on the store button |
| `test_get_screenshot` | Captures a PNG screenshot via AltDriver |

### `test_user_journey.py`
| Test | Description |
|---|---|
| `test_user_journey_play_and_pause` | Plays the game, avoids obstacles, pauses, resumes, dies, reaches game-over screen |
| `test_user_journey_buy_items` | Buys magnet and night theme from store, verifies they appear in-game |
| `test_user_journey_revive_and_get_a_second_chance` | Buys life item, loses a life, activates revive, reaches game-over |
| `test_the_number_of_all_enabled_elements_from_different_pages_is_different` | Counts enabled Unity objects across scenes |
| `test_the_number_of_all_disabled_elements_from_different_pages_is_different` | Counts disabled Unity objects across scenes |
| `test_methods_that_handle_scenes` | Exercises `load_scene`, `unload_scene`, `get_current_scene` |

## Troubleshooting

**Tunnel fails to start** — Check that `tunnel/LT` is executable (`chmod +x tunnel/LT`) and that your `LT_USERNAME`/`LT_ACCESS_KEY` are correct.

**`AltDriver` connection timeout** — The 30-second sleep in `conftest.py` may not be enough on slower devices. Increase `time.sleep(30)` in the `setup` fixture.

**App not found** — Confirm the `LT_APP_URL` value starts with `lt://` and matches the app you uploaded for the target platform (Android vs iOS).

**Wrong AltTester server version** — `test_get_server_version` asserts version `2.2.5`. If your APK was instrumented with a different version, update that assertion or re-instrument the app.

## Have feedback or need assistance?
Feel free to fork the repo and contribute to make it better! Thanks to the awesome AltTester team for the great support that they provided throughout the course of the integration!

Email to [himanshu[dot]sheth[at]gmail[dot]com](mailto:himanshu.sheth@gmail.com) for any queries or ping me on the following social media sites:

<b>LinkedIn</b>: [@hjsblogger](https://linkedin.com/in/hjsblogger)<br/>
<b>Twitter</b>: [@hjsblogger](https://www.twitter.com/hjsblogger)

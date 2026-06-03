---
name: alttester-unity-skill
description: >
  Generates production-grade AltTester automation tests for Unity mobile games
  running on Android and iOS real devices via TestMu AI cloud. Use when the user
  asks to automate a Unity game, test game objects, drive in-game UI, or mentions
  "AltTester", "AltDriver", "Unity game testing", "TrashCat", "game automation",
  "real device Unity", "TestMu AI game". Triggers on: "AltTester", "AltDriver",
  "Unity", "game automation", "game testing", "TrashCat", "TestMu", "LambdaTest",
  "real device", "mobile game", "instrumented app".
languages:
  - Python
category: game-testing
license: MIT
metadata:
  author: TestMu AI
  version: "1.0"
---

# AltTester Unity Game Automation Skill

You are a senior mobile game QA architect. You write production-grade AltTester
tests for Unity games running on Android/iOS real devices via TestMu AI cloud.

## How AltTester Works

```
Unity Game (APK/IPA instrumented with AltTester SDK)
    └─ starts AltTester Server (WebSocket) on device port 13000
           │
           │ ← LT Tunnel bridges localhost:13000 ↔ cloud device port
           │
AltDriver (Python) ──── connects to localhost:13000
           │
           └─ inspects / manipulates Unity game objects at runtime

Appium (Python) ─────── manages the remote device session on TestMu AI
                         (launches app, keeps session alive, reports status)
```

## Step 1 — Execution Target

```
User asks to automate a Unity game
│
├─ Mentions "TestMu AI", "cloud", "real device", "LambdaTest"?
│  └─ TestMu AI cloud via Appium + LT Tunnel
│
├─ Mentions "local", "emulator", "simulator"?
│  └─ Local AltTester Server (no Appium/tunnel needed)
│
└─ Ambiguous? → Default to TestMu AI cloud (matches this repo's setup)
```

## Step 2 — Platform Detection

```
├─ Mentions "Android", "APK", "Pixel", "Samsung"?
│  └─ platformName: android, deviceName: "Pixel.*", automationName: UiAutomator2
│
├─ Mentions "iOS", "IPA", "iPhone", "iPad"?
│  └─ platformName: ios, deviceName: "iPhone 14", automationName: XCUITest
│
└─ Both? → Create separate lt_options dicts; AltDriver code is identical for both
```

## Step 3 — Language

This skill generates **Python + pytest** (the language used in this repo).
All patterns use `alttester-driver`, `Appium-Python-Client`, and `python-dotenv`.

---

## Core Patterns — Python + pytest

### Environment Variables

Store credentials in a `.env` file (gitignored). Load with `python-dotenv`:

```python
from dotenv import load_dotenv
load_dotenv()

username   = os.environ["LT_USERNAME"]
access_key = os.environ["LT_ACCESS_KEY"]
app_url    = os.environ["LT_APP_URL"]   # e.g. lt://APP1234567890
```

### LT Tunnel — Start / Stop

```python
import subprocess, time
from urllib.request import urlopen
from urllib.error import URLError

TUNNEL_NAME      = "alttester-tunnel"
TUNNEL_INFO_PORT = 8000

def start_tunnel(username, access_key):
    process = subprocess.Popen(
        ["./tunnel/LT", "--user", username, "--key", access_key,
         "--tunnelName", TUNNEL_NAME, "--verbose",
         "--infoAPIPort", str(TUNNEL_INFO_PORT)],
        stdout=subprocess.PIPE, stderr=subprocess.PIPE,
    )
    _wait_for_tunnel(timeout_seconds=60)
    return process

def _wait_for_tunnel(timeout_seconds=60):
    deadline = time.time() + timeout_seconds
    while time.time() < deadline:
        try:
            with urlopen(f"http://127.0.0.1:{TUNNEL_INFO_PORT}/api/v1.0/info", timeout=5) as r:
                if r.status == 200:
                    return
        except (URLError, OSError):
            pass
        time.sleep(2)
    raise RuntimeError("LT Tunnel did not start within timeout")

def stop_tunnel(process):
    if process and process.poll() is None:
        process.kill()
        process.wait(timeout=10)
```

### conftest.py — Full Fixture Chain

```python
import os, time, warnings
from datetime import datetime

import pytest
from appium import webdriver as appium_webdriver
from appium.options.common import AppiumOptions
from alttester import AltDriver
from dotenv import load_dotenv

load_dotenv()
TUNNEL_NAME = "alttester-tunnel"

@pytest.fixture(scope="session")
def lt_tunnel():
    """Start tunnel once for the entire test session."""
    process = start_tunnel(os.environ["LT_USERNAME"], os.environ["LT_ACCESS_KEY"])
    yield process
    stop_tunnel(process)

@pytest.fixture(scope="class")
def setup(request, lt_tunnel):
    """One Appium + AltDriver session per test class."""
    username   = os.environ["LT_USERNAME"]
    access_key = os.environ["LT_ACCESS_KEY"]
    app_url    = os.environ["LT_APP_URL"]

    lt_options = {
        "user": username, "accessKey": access_key,
        "app": app_url,
        "deviceName": "Pixel.*", "platformVersion": "14", "platformName": "android",
        "build": "AltTester Demo", "name": f"tests - {datetime.now().strftime('%B %d - %H:%M')}",
        "isRealMobile": True, "idleTimeout": 300,
        "tunnel": True, "tunnelName": TUNNEL_NAME,
    }
    options = AppiumOptions()
    options.set_capability("lt:options", lt_options)
    options.set_capability("platformName", "android")

    with warnings.catch_warnings():
        warnings.simplefilter("ignore", UserWarning)
        appium_driver = appium_webdriver.Remote(
            command_executor=f"https://{username}:{access_key}@mobile-hub.lambdatest.com/wd/hub",
            options=options,
        )

    time.sleep(30)                 # wait for Unity app to fully launch
    alt_driver = AltDriver()       # connects to AltTester Server via tunnel

    request.cls.alt_driver    = alt_driver
    request.cls.appium_driver = appium_driver
    yield alt_driver

    try:
        appium_driver.execute_script("lambda-status=passed")
    except Exception:
        pass
    appium_driver.quit()
    alt_driver.stop()

@pytest.fixture(autouse=True)
def per_test_annotation(request, setup):
    """Annotate each test start/end; ping Appium to prevent idle timeout."""
    appium_driver = getattr(request.cls, "appium_driver", None)
    if appium_driver:
        _annotate(appium_driver, f"Starting: {request.node.name}")
    yield
    if appium_driver:
        passed = getattr(getattr(request.node, "rep_call", None), "passed", True)
        _annotate(appium_driver, f"{'PASS' if passed else 'FAIL'}: {request.node.name}",
                  "info" if passed else "error")
        try:
            appium_driver.get_display_density()   # keeps session alive (Android)
        except Exception:
            pass

def _annotate(driver, message, level="info"):
    escaped = message.replace("\\", "\\\\").replace('"', '\\"')
    driver.execute_script(
        f'lambdatest_executor: {{"action":"stepcontext","arguments":{{"data":"{escaped}","level":"{level}"}}}}'
    )

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    setattr(item, f"rep_{outcome.get_result().when}", outcome.get_result())
```

### Test Class Structure

```python
import pytest
from pages import MainMenuPage, GamePlay   # import your page objects

@pytest.mark.usefixtures("setup")
class TestMyFeature:
    def setup_method(self):
        self.main_menu = MainMenuPage(self.alt_driver)
        self.game_play = GamePlay(self.alt_driver)
        self.main_menu.load_scene()           # always reset to a known scene

    def test_something(self):
        self.main_menu.press_run()
        assert self.game_play.is_displayed()

    def teardown_method(self):
        self.main_menu.load_scene()           # restore scene
        time.sleep(1)
```

### Finding Unity Objects (AltDriver)

```python
from alttester import By

# By object name (fastest)
obj = driver.find_object(By.NAME, "StartButton")

# By full hierarchy path
obj = driver.find_object(By.PATH, "/UICamera/Loadout/StoreButton")

# By object ID (use after find to re-fetch a stale reference)
obj = driver.find_object(By.ID, str(obj.id))

# Wait until object appears (raises on timeout)
obj = driver.wait_for_object(By.NAME, "PauseButton", timeout=10)

# Find all matching objects
objs = driver.find_objects_which_contain(By.NAME, "Button")

# Find using a camera filter
obj = driver.find_object(By.NAME, "ThemeZone", By.NAME, "Main Camera")
```

### Interacting with Objects

```python
# Tap
obj.tap()

# Set text
obj.set_text("HighScore")

# Read text
text = obj.get_text()

# Call a C# component method
character.call_component_method(
    "CharacterInputController", "Jump", "Assembly-CSharp", []
)

# Read a C# component property
life = character.get_component_property(
    "CharacterInputController", "currentLife", "Assembly-CSharp"
)

# Read a C# component field
fields = obj.get_all_fields(
    component_name="UnityEngine.UI.Button", assembly_name="UnityEngine.UI"
)

# Get parent object
parent = obj.get_parent()

# Get world / screen position
world_pos  = obj.get_world_position()
screen_pos = obj.get_screen_position()

# Update stale reference
obj = obj.update_object()
```

### Scene Management

```python
driver.load_scene("Main")               # load scene by name
driver.get_current_scene()              # returns scene name string
driver.get_all_loaded_scenes()          # returns list of scene names
driver.unload_scene("Main")
```

### Static Methods & Player Prefs

```python
# Call a Unity static method
driver.call_static_method(
    "UnityEngine.Screen", "SetResolution", "UnityEngine.CoreModule",
    ["375", "667", "false"],
    ["System.Int32", "System.Int32", "System.Boolean"],
)

# Player prefs
driver.set_key_player_pref("coins", 1000000)
value = driver.get_int_key_player_pref("coins")
driver.delete_key_player_pref("coins")
driver.delete_player_pref()             # wipe all prefs
```

### Time Scale

```python
driver.set_time_scale(0.1)             # slow motion for assertion windows
assert driver.get_time_scale() == 0.1
driver.set_time_scale(1)               # restore normal speed
```

### Page Object Model

Every page inherits `BasePage`. Expose Unity objects as `@property` (lazy lookup on access):

```python
from alttester import By
from .base_page import BasePage

class BasePage:
    annotate_callback = None

    def __init__(self, driver):
        self.driver = driver

    def log(self, message, level="info"):
        if BasePage.annotate_callback:
            BasePage.annotate_callback(message, level)

class MyGameScreen(BasePage):
    def __init__(self, driver):
        super().__init__(driver)

    def load_scene(self):
        self.log("MyScreen: loading scene")
        self.driver.load_scene("MyScene")

    @property
    def play_button(self):
        return self.driver.wait_for_object(By.NAME, "PlayButton", timeout=10)

    def is_displayed(self):
        return self.play_button is not None

    def press_play(self):
        self.log("MyScreen: pressing Play")
        self.play_button.tap()
```

---

## Anti-Patterns

| Bad | Good | Why |
|-----|------|-----|
| `time.sleep(5)` between every action | `driver.wait_for_object(By.NAME, "X", timeout=10)` | Waits are flaky and slow |
| Hardcoded object IDs | `By.NAME` or `By.PATH` | IDs regenerate across builds |
| Accessing Unity objects directly in tests | Page Object properties | Keeps test logic readable |
| One giant test class for all screens | One class per screen | Matches `setup` fixture scope |
| Forgetting `load_scene()` in `setup_method` | Always reset to known scene first | Prevents test pollution |
| Polling with `while timeout > 0` without try/except | Wrap in try/except; decrement only on failure | Avoids infinite loops on hard crashes |
| Re-using the same `AltObject` reference after scene change | Call `obj.update_object()` or re-find | References go stale on scene reload |

---

## TestMu AI Cloud — Quick Setup

```bash
# 1. Upload the AltTester-instrumented APK/IPA
curl -u "$LT_USERNAME:$LT_ACCESS_KEY" \
  --location --request POST \
  'https://manual-api.lambdatest.com/app/upload/realDevice' \
  --form 'name="TrashCat"' \
  --form 'appFile=@"/path/to/TrashCat.apk"'
# Response: { "app_url": "lt://APP1234567890" }

# 2. Set your .env
echo "LT_USERNAME=<username>"   >> .env
echo "LT_ACCESS_KEY=<key>"      >> .env
echo "LT_APP_URL=lt://APP1234567890" >> .env

# 3. Make the tunnel binary executable (macOS)
chmod +x tunnel/LT

# 4. Run tests (tunnel starts automatically via lt_tunnel fixture)
pytest
```

---

## Test Status Reporting

```python
# In teardown / fixture cleanup — mark session passed or failed
appium_driver.execute_script("lambda-status=passed")
appium_driver.execute_script("lambda-status=failed")

# Mid-test step annotation
driver.execute_script(
    'lambdatest_executor: {"action":"stepcontext","arguments":{"data":"Step description","level":"info"}}'
)
```

---

## Quick Reference

| Task | Code |
|------|------|
| Connect AltDriver | `AltDriver()` — default host `localhost:13000` |
| Load a scene | `driver.load_scene("SceneName")` |
| Find object | `driver.find_object(By.NAME, "ObjectName")` |
| Wait for object | `driver.wait_for_object(By.NAME, "X", timeout=10)` |
| Tap object | `obj.tap()` |
| Call C# method | `obj.call_component_method("Component", "Method", "Assembly", [args])` |
| Read C# property | `obj.get_component_property("Component", "prop", "Assembly")` |
| Screenshot | `driver.get_png_screenshot("screenshot.png")` |
| Get all elements | `driver.get_all_elements(enabled=True)` |
| Active cameras | `driver.get_all_active_cameras()` |
| Server version | `driver.get_server_version()` |
| App screen size | `driver.get_application_screen_size()` |

---

## Reference Files

| File | When to Read |
|------|-------------|
| `reference/alttester-api.md` | Full AltDriver API surface — finders, waits, gestures, inspection |
| `reference/page-object-patterns.md` | POM templates, scene-reset patterns, multi-screen flows |
| `reference/testmuai-cloud.md` | App upload, capability options, tunnel flags, iOS setup |
| `reference/playbook.md` | Deep patterns: parallel runs, time-scale tricks, invincibility testing, obstacle avoidance logic |

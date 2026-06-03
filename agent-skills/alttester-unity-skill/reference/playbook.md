# AltTester Playbook — Deep Patterns

## § 1 — Project Setup

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
chmod +x tunnel/LT
cp .env.example .env    # fill in LT_USERNAME, LT_ACCESS_KEY, LT_APP_URL
```

`requirements.txt` must include:
```
alttester-driver>=2.2.5
pytest>=8.0.0
Appium-Python-Client>=4.0.0
python-dotenv>=1.0.0
```

---

## § 2 — pytest.ini Defaults

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
```

---

## § 3 — Thread-Safe AltDriver (Parallel Runs)

By default, `AltDriver()` and `appium_webdriver.Remote()` are not thread-safe.
For parallel test execution, use `pytest-xdist` with `--dist=no` (class-level
isolation) or a thread-local driver pattern:

```python
import threading

_local = threading.local()

def get_alt_driver():
    if not hasattr(_local, "driver"):
        _local.driver = AltDriver()
    return _local.driver
```

For most use cases, run tests serially (one device session per run).

---

## § 4 — Time-Scale Tricks

Slow time down to open assertion windows during fast-moving gameplay:

```python
def test_character_moves(self):
    self.alt_driver.set_time_scale(0.1)   # 10× slower
    time.sleep(1)                          # real 1 s ≈ 0.1 s game time
    initial_z = self.game_play.character.get_world_position().z
    time.sleep(1)
    final_z   = self.game_play.character.update_object().get_world_position().z
    self.alt_driver.set_time_scale(1)
    assert initial_z != final_z
```

Always restore in teardown — a leaked `set_time_scale(0.1)` makes subsequent
tests run in slow motion.

---

## § 5 — Invincibility Testing Pattern

Force the character into an invincible state to test post-death flows without
depending on random obstacle timing:

```python
def test_revive_flow(self):
    # Make character die predictably
    self.game_play.set_character_invincible("False")
    # Wait for GetAnotherChance screen
    timeout = 20
    while timeout > 0:
        try:
            self.get_another_chance_page.is_displayed()
            break
        except Exception:
            timeout -= 1
    assert self.get_another_chance_page.is_displayed()
    self.get_another_chance_page.press_game_over()
    assert self.game_over_screen.is_displayed()

def test_survive_with_invincibility(self):
    self.game_play.set_character_invincible("True")
    time.sleep(20)                                          # survive 20 s
    self.alt_driver.wait_for_object_not_be_present(By.NAME, "GameOver")
    self.game_play.set_character_invincible("False")
```

`set_character_invincible` calls a C# method on `CharacterCollider`:

```python
def set_character_invincible(self, state: str):
    self.character_slot.call_component_method(
        "CharacterCollider", "SetInvincibleExplicit", "Assembly-CSharp", [state]
    )
```

---

## § 6 — Obstacle Avoidance Algorithm

`GamePlay.avoid_obstacles(n)` drives the character through `n` obstacles:

1. Wait for any obstacle to appear via `wait_for_object_which_contains(By.NAME, "Obstacle")`.
2. Find all obstacles, sort by `worldZ`, filter to those ahead of the character.
3. Determine type from name: `ObstacleHighBarrier` → slide; `ObstacleLowBarrier` / `Rat` → jump; otherwise → lane change.
4. For lane changes, look at the next obstacle's position to choose left vs right.
5. After passing, undo the lane change to re-centre.

This is a game-specific heuristic — extend it as new obstacle types are added to the Unity project.

---

## § 7 — Reading and Setting UI Properties at Runtime

You can read or patch any serialised field on any component without rebuilding
the game:

```python
# Read the current fishbone count
fishbones = store_page.alt_driver.get_static_property(
    "UnityEngine.PlayerPrefs", "fishbones", "UnityEngine.CoreModule"
)

# Force a coin amount via static method
driver.call_static_method(
    "UnityEngine.PlayerPrefs", "SetInt", "UnityEngine.CoreModule",
    ["Fishbones", "1000000"]
)

# Change an item's display name at runtime
item_obj.call_component_method("ShopItemList", "set_Name", "Assembly-CSharp", ["magneeeeeeet"])
```

---

## § 8 — Color-State Assertions

AltTester can read Unity `Color` structs to assert that UI elements change
appearance on press / release:

```python
def get_button_color(self, obj):
    return obj.get_component_property(
        "UnityEngine.UI.Image", "color", "UnityEngine.UI"
    )

def compare_object_color_by_state(self, obj):
    color_before = self.get_button_color(obj)
    obj.pointer_down_from_object()
    time.sleep(0.5)
    color_pressed = self.get_button_color(obj)
    obj.pointer_up_from_object()
    assert color_before != color_pressed
```

---

## § 9 — Screenshot on Failure

Add a pytest hook to capture a screenshot when a test fails:

```python
@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield
    rep = outcome.get_result()
    setattr(item, f"rep_{rep.when}", rep)
    if rep.when == "call" and rep.failed:
        alt_driver = getattr(item.cls, "alt_driver", None)
        if alt_driver:
            alt_driver.get_png_screenshot(f"screenshot_{item.name}.png")
```

---

## § 10 — Scene-Level Test Isolation

When a test modifies persistent state (player prefs, purchased items), always
call `settings_page.delete_data()` in `setup_method` or `teardown_method` to
prevent cross-test contamination:

```python
def setup_method(self):
    self.main_menu.load_scene()
    self.settings.delete_data()    # wipe save data before each test
```

`delete_data` typically calls:

```python
def delete_data(self):
    self.driver.delete_player_pref()
    self.driver.load_scene("Main")
```

---

## § 11 — Debugging Checklist

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `AltDriver` connection refused | Tunnel not running or app not launched | Check tunnel binary, add `time.sleep(30)` after Appium connect |
| Object not found by `By.NAME` | Wrong name / object not active | Inspect with AltTester Inspector in Unity Editor |
| `wait_for_object` times out | Scene not loaded yet | Call `load_scene()` first; increase timeout |
| Session idle disconnect | No Appium command between tests | Add `appium_driver.get_display_density()` in `per_test_annotation` |
| Stale object reference error | Object re-created after scene load | Use `obj.update_object()` or re-find with `wait_for_object` |
| `lambda-status` not updating | Appium driver already quit | Call status before `appium_driver.quit()` |
| Tests pass locally, fail on cloud | Timing differences on real device | Increase `wait_for_object` timeout; add `time.sleep(30)` startup buffer |
| Wrong scene loaded | Previous test crashed mid-teardown | Call `load_scene()` at start of `setup_method`, not just teardown |

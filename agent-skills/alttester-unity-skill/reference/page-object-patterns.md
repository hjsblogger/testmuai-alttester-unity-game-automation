# Page Object Model — Patterns for AltTester

## BasePage

Every page inherits `BasePage`. Keep it minimal — it only needs the driver and
the optional annotation hook.

```python
class BasePage:
    annotate_callback = None          # set by conftest.py after Appium driver is ready

    def __init__(self, driver):
        self.driver = driver          # AltDriver instance

    def log(self, message, level="info"):
        if BasePage.annotate_callback:
            BasePage.annotate_callback(message, level)
```

## Page Class Template

```python
from alttester import By
from .base_page import BasePage

class MyScreenPage(BasePage):
    def __init__(self, driver):
        super().__init__(driver)

    # --- Scene entry point ---
    def load_scene(self):
        self.log("MyScreen: loading")
        self.driver.load_scene("MySceneName")

    # --- Elements as @property (lazy; re-fetched on every access) ---
    @property
    def action_button(self):
        return self.driver.wait_for_object(By.NAME, "ActionButton", timeout=10)

    @property
    def score_text(self):
        return self.driver.wait_for_object(By.NAME, "ScoreText", timeout=5)

    # --- Compound check ---
    def is_displayed(self):
        self.log("MyScreen: checking visibility")
        return all([self.action_button, self.score_text])

    # --- Actions ---
    def press_action(self):
        self.log("MyScreen: pressing Action")
        self.action_button.tap()

    def get_score(self):
        return int(self.score_text.get_text())
```

## Why @property for Elements

`@property` means the object is looked up fresh every time the property is
accessed. This avoids stale-reference errors after scene reloads or object
pool recycling. If you cache an `AltObject` in `__init__`, it will be stale
the next time the scene loads.

```python
# BAD — cached in __init__, stale after load_scene()
self.button = self.driver.find_object(By.NAME, "StartButton")

# GOOD — re-fetched each access
@property
def button(self):
    return self.driver.wait_for_object(By.NAME, "StartButton", timeout=10)
```

## Scene Reset Pattern

Every test class's `setup_method` should navigate to a known scene before
creating page objects:

```python
def setup_method(self):
    self.main_menu = MainMenuPage(self.alt_driver)
    self.game_play = GamePlay(self.alt_driver)
    self.main_menu.load_scene()     # ← always first; ensures clean state
```

And `teardown_method` should reset data and return to a stable state:

```python
def teardown_method(self):
    self.main_menu.load_scene()
    self.settings_page.delete_data()
    time.sleep(1)
```

## Multi-Screen Flow Example

```python
@pytest.mark.usefixtures("setup")
class TestPurchaseFlow:
    def setup_method(self):
        self.main_menu = MainMenuPage(self.alt_driver)
        self.store      = StorePage(self.alt_driver)
        self.game_play  = GamePlay(self.alt_driver)
        self.main_menu.load_scene()

    def test_buy_item_and_use_in_game(self):
        self.main_menu.press_store()
        self.store.get_more_money()
        self.store.buy("Items", 0)          # buy magnet
        self.store.close_store()

        self.main_menu.tap_arrow_button("power", "Left")
        self.main_menu.press_run()

        self.game_play.activate_in_game_power_up()
        assert self.game_play.power_up_icon is not None
```

## Handling Objects That Appear After a Delay

Use `wait_for_object` rather than `find_object` for elements that appear after
a game event (death screen, transition, etc.):

```python
# BAD — may race against the animation
screen = self.driver.find_object(By.NAME, "GameOverScreen")

# GOOD — waits up to 15 s
screen = self.driver.wait_for_object(By.NAME, "GameOverScreen", timeout=15)
```

For objects that appear based on game events with unknown timing, poll in a
tight try/except loop with a decreasing counter:

```python
timeout = 20
while timeout > 0:
    try:
        self.get_another_chance_page.is_displayed()
        break
    except Exception:
        timeout -= 1
assert self.get_another_chance_page.is_displayed()
```

## Checking Object State Without Raising

Some helper methods need to return `True`/`False` without raising an exception
when the object is absent. Wrap the lookup in try/except:

```python
def button_is_interactable(self, obj_name):
    try:
        obj = self.driver.find_object(By.NAME, obj_name)
        return obj.get_component_property(
            "UnityEngine.UI.Button", "interactable", "UnityEngine.UI"
        )
    except Exception:
        return False
```

## Parameterized Tests

Use `@pytest.mark.parametrize` for tests that repeat the same flow across
different data (slider names, item indices, tab names):

```python
@pytest.mark.parametrize("slider_name", ["MasterSlider", "MusicSlider", "MasterSFXSlider"])
def test_slider_changes(self, slider_name):
    self.settings.move_slider(slider_name, -1000)
    before = self.settings.get_slider_value(slider_name)
    self.settings.move_slider(slider_name, 20)
    assert before != self.settings.get_slider_value(slider_name)
```

## pages/__init__.py Convention

Re-export all page classes so tests import from a single location:

```python
# pages/__init__.py
from .base_page import BasePage
from .start_page import StartPage
from .main_menu_page import MainMenuPage
from .game_play_page import GamePlay
from .pause_overlay_page import PauseOverlayPage
from .get_another_chance_page import GetAnotherChancePage
from .settings_page import SettingsPage
from .store_page import StorePage
from .game_over_screen_page import GameOverScreen
```

Tests then do:

```python
from pages import MainMenuPage, GamePlay, StorePage
```

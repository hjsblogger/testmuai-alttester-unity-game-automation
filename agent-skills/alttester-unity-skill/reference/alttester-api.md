# AltTester — AltDriver API Reference

## Connection

```python
from alttester import AltDriver

# Default: connects to localhost:13000 (bridged by LT Tunnel)
driver = AltDriver()

# Custom host/port
driver = AltDriver(host="localhost", port=13000, enable_logging=True)

# Disconnect
driver.stop()
```

## Finding Objects

| Method | Description |
|--------|-------------|
| `find_object(by, value)` | Single object; raises if not found |
| `find_object(by, value, by_camera, camera_value)` | With camera filter |
| `find_objects(by, value)` | All matching objects |
| `find_object_which_contains(by, value)` | Partial name match, first result |
| `find_objects_which_contain(by, value)` | Partial name match, all results |
| `wait_for_object(by, value, timeout)` | Polls until found or timeout |
| `wait_for_object_which_contains(by, value, timeout)` | Partial match with wait |
| `wait_for_object_not_be_present(by, value, timeout)` | Wait until gone |

### By Selectors

```python
from alttester import By

By.NAME      # exact Unity object name
By.PATH      # full hierarchy path: "/Canvas/Panel/Button"
By.ID        # AltTester internal ID (string)
By.LAYER     # Unity layer name
By.TAG       # Unity tag
By.COMPONENT # component type name
By.TEXT      # UI Text value
```

## Interacting with Objects (AltObject methods)

```python
obj.tap()                          # single tap at object centre
obj.click()                        # alias for tap
obj.double_tap()                   # double tap
obj.pointer_down_from_object()     # begin hold
obj.pointer_up_from_object()       # release hold

obj.set_text("value")              # set UI text field
obj.get_text()                     # read UI text

obj.get_parent()                   # returns parent AltObject
obj.get_screen_position()          # (x, y) in screen pixels
obj.get_world_position()           # (x, y, z) in world space
obj.update_object()                # re-fetch stale reference
```

## Inspecting Components / Properties / Fields / Methods

```python
obj.get_all_components()
# returns list of ComponentDto: .component_name, .assembly_name

obj.get_all_properties(component_name="...", assembly_name="...")
# returns list of AltProperty: .name, .value

obj.get_all_fields(component_name="...", assembly_name="...")
# returns list of AltField: .name, .value

obj.get_all_methods(component_name="...", assembly_name="...")
# returns list of method signature strings

obj.get_component_property("ComponentName", "propertyName", "AssemblyName")
# returns the value as a Python native type

obj.set_component_property("ComponentName", "propertyName", "AssemblyName", value)
# sets the property at runtime
```

## Calling C# Methods

```python
# Instance method
obj.call_component_method(
    "ComponentName",         # C# class name
    "MethodName",            # method name
    "AssemblyName",          # assembly (e.g. "Assembly-CSharp", "UnityEngine.UI")
    ["arg1", "arg2"],        # arguments as strings
    ["System.Int32", ...],   # optional: parameter type hints
)

# Static method
driver.call_static_method(
    "UnityEngine.Screen",
    "SetResolution",
    "UnityEngine.CoreModule",
    ["375", "667", "false"],
    ["System.Int32", "System.Int32", "System.Boolean"],
)
```

## Scene Management

```python
driver.load_scene("SceneName")             # additive=False by default
driver.load_scene("SceneName", load_single=True)
driver.unload_scene("SceneName")
driver.get_current_scene()                 # returns scene name string
driver.get_all_loaded_scenes()             # returns list of scene names
driver.get_all_elements(enabled=True)      # all enabled game objects
driver.get_all_elements(enabled=False)     # all disabled game objects
```

## Time Scale

```python
driver.set_time_scale(0.1)   # slow motion
driver.get_time_scale()      # returns float
driver.set_time_scale(1)     # restore
```

## Player Preferences

```python
driver.set_key_player_pref("key", "string_value")
driver.set_key_player_pref("key", 42)         # int
driver.set_key_player_pref("key", 3.14)       # float

driver.get_string_key_player_pref("key")
driver.get_int_key_player_pref("key")
driver.get_float_key_player_pref("key")

driver.delete_key_player_pref("key")
driver.delete_player_pref()                   # wipe all
```

## Touch & Gesture Input

```python
# Multi-touch: begin/move/end
finger_id = driver.begin_touch((x, y))
driver.move_touch(finger_id, (new_x, new_y))
driver.end_touch(finger_id)

# Swipe
driver.swipe((start_x, start_y), (end_x, end_y), duration_seconds=0.5)

# Tilt
driver.tilt((0, 0, 2.5), duration_seconds=0.1)
```

## Camera & Screen

```python
driver.get_all_active_cameras()        # list of active camera AltObjects
driver.get_application_screen_size()   # AltVector2: .x, .y
driver.get_png_screenshot("path.png")  # save screenshot to file
```

## Server Info

```python
driver.get_server_version()   # returns version string e.g. "2.2.5"
```

## AltObject Attributes

After `find_object`, the returned `AltObject` has these attributes:

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | str | Unity object name |
| `id` | int | Internal AltTester ID |
| `worldX/Y/Z` | float | World-space position |
| `x / y` | float | Screen-space position |
| `enabled` | bool | Whether the object is active |
| `transformParentId` | int | Parent object ID |
| `transformId` | int | Transform ID |

# TestMu AI Cloud — Integration Reference

## App Upload

Upload your AltTester-instrumented APK or IPA before running tests.
The response gives you the `lt://` URL to put in `LT_APP_URL`.

```bash
# Android APK
curl -u "$LT_USERNAME:$LT_ACCESS_KEY" \
  --location --request POST \
  'https://manual-api.lambdatest.com/app/upload/realDevice' \
  --form 'name="YourGameName"' \
  --form 'appFile=@"/path/to/YourGame.apk"'

# iOS IPA
curl -u "$LT_USERNAME:$LT_ACCESS_KEY" \
  --location --request POST \
  'https://manual-api.lambdatest.com/app/upload/realDevice' \
  --form 'name="YourGameName"' \
  --form 'appFile=@"/path/to/YourGame.ipa"'

# Response
{ "app_url": "lt://APP1234567890" }
```

## Appium Hub Endpoint

```
https://<LT_USERNAME>:<LT_ACCESS_KEY>@mobile-hub.lambdatest.com/wd/hub
```

## Android Capabilities

```python
lt_options = {
    "user":          os.environ["LT_USERNAME"],
    "accessKey":     os.environ["LT_ACCESS_KEY"],
    "app":           os.environ["LT_APP_URL"],       # lt://APP...
    "deviceName":    "Pixel.*",                       # regex supported
    "platformVersion":"14",
    "platformName":  "android",
    "build":         "AltTester Android Build",
    "name":          "Test run name",
    "isRealMobile":  True,
    "idleTimeout":   300,                             # seconds; increase for long tests
    "tunnel":        True,
    "tunnelName":    "alttester-tunnel",              # must match tunnel binary --tunnelName
    # Optional extras:
    "video":         True,
    "network":       True,
    "console":       True,
}

options = AppiumOptions()
options.set_capability("lt:options", lt_options)
options.set_capability("platformName", "android")
```

## iOS Capabilities

```python
lt_options = {
    "user":          os.environ["LT_USERNAME"],
    "accessKey":     os.environ["LT_ACCESS_KEY"],
    "app":           os.environ["LT_APP_URL"],
    "deviceName":    "iPhone 14",
    "platformVersion":"16",
    "platformName":  "ios",
    "build":         "AltTester iOS Build",
    "name":          "Test run name",
    "isRealMobile":  True,
    "idleTimeout":   300,
    "tunnel":        True,
    "tunnelName":    "alttester-tunnel",
}

options = AppiumOptions()
options.set_capability("lt:options", lt_options)
options.set_capability("platformName", "ios")
# For iOS, Appium keep-alive ping: use appium_driver.get_clipboard_text() instead of get_display_density()
```

## LT Tunnel Binary

| Flag | Purpose |
|------|---------|
| `--user` | TestMu AI username |
| `--key` | TestMu AI access key |
| `--tunnelName` | Name that matches `tunnelName` capability |
| `--verbose` | Debug logging |
| `--infoAPIPort` | Port for `/api/v1.0/info` health check (default 8000) |

```bash
# Start manually (conftest.py does this automatically)
./tunnel/LT \
  --user "$LT_USERNAME" \
  --key "$LT_ACCESS_KEY" \
  --tunnelName "alttester-tunnel" \
  --verbose \
  --infoAPIPort 8000
```

The tunnel binary for macOS is committed at `tunnel/LT`. Make it executable:

```bash
chmod +x tunnel/LT
```

## Test Status Reporting

```python
# Mark session passed
appium_driver.execute_script("lambda-status=passed")

# Mark session failed
appium_driver.execute_script("lambda-status=failed")
```

## Step Annotation

Annotations appear as timeline steps in the TestMu AI dashboard:

```python
def annotate(appium_driver, message, level="info"):
    escaped = message.replace("\\", "\\\\").replace('"', '\\"')
    appium_driver.execute_script(
        f'lambdatest_executor: {{"action":"stepcontext",'
        f'"arguments":{{"data":"{escaped}","level":"{level}"}}}}'
    )

# Usage
annotate(appium_driver, "Waiting for app to start...")
annotate(appium_driver, "Test failed at store", level="error")
```

## Idle Timeout Prevention

TestMu AI sessions disconnect after `idleTimeout` seconds of no Appium command.
Between tests, ping the driver to keep it alive:

```python
# Android — non-raising ping
try:
    appium_driver.get_display_density()
except Exception:
    pass

# iOS — non-raising ping
try:
    appium_driver.get_clipboard_text()
except Exception:
    pass
```

## Session Time Considerations

- Real device sessions are slower than emulators.
- Add `time.sleep(30)` after Appium `Remote()` connect to allow the Unity app
  to fully launch before `AltDriver()` tries to connect.
- Use `idleTimeout: 300` and the Appium keep-alive ping between tests.
- For tests that use `set_time_scale(0.1)`, remember to restore to `1` in
  teardown; otherwise the next test runs in slow motion.

## Environment File

```ini
# .env (gitignored)
LT_USERNAME=your_testmuai_username
LT_ACCESS_KEY=your_testmuai_access_key
LT_APP_URL=lt://APP1234567890
```

Load with:

```python
from dotenv import load_dotenv
load_dotenv()
```

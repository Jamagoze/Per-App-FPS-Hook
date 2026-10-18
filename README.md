# Per-App FPS Cap

A Magisk module that caps the frame rate of chosen apps on HyperOS, for example Terraria at 60 FPS on a 90Hz screen, and puts the display back to 90Hz everywhere else.

Built and tested on a Redmi Pad SE (Android 14, HyperOS, Magisk). It relies on HyperOS-specific settings (see Limitations), so other devices may need changes.

**Current version: v3.2.** Nothing polls. The module runs once at boot and a small accessibility app reports app switches, so it uses almost no CPU between them.

---

## How it works

Two things have to happen for a cap to hold on HyperOS, so the module does both.

### 1. Game-mode FPS override (the cap itself)

For every app in the config, the module registers a frame-rate override with Android's game manager:

```
device_config put game_overlay <package> "mode=2,fps=N:mode=3,fps=N"
cmd game set --mode 2 --fps N <package>
cmd game mode 2 <package>
```

The `device_config` line is what makes Android accept performance mode for an app that doesn't declare game-mode support. Without it, `cmd game mode 2` reports "not supported".

### 2. Display pin (stops the touch boost)

HyperOS raises the display to 90Hz on every touch, and the FPS override alone didn't hold against that. The standard `peak_refresh_rate` setting is ignored on HyperOS, so the module pins the display through the two keys HyperOS actually follows:

```
settings put secure miui_refresh_rate <Hz>
settings put secure user_refresh_rate <Hz>
```

With the display pinned to 60Hz, the render range becomes `(0.0 60.0)` and the 60 FPS cap holds. When you leave the app, both keys go back to 90.

### Who switches the refresh rate: the hook app

The module can't see which app is in front without checking constantly. Instead it ships a tiny accessibility app, **Per-App FPS Hook** (`com.perappfps.hook`), as a system priv-app.

- It listens for window-change events only, and does not read screen content.
- It ignores system UI, the keyboard and itself.
- On each app change it runs `focus.sh <package>` through one long-lived root shell.
- `focus.sh` looks the app up in the config and sets 60 or 90 (or does nothing if the rate is already right).

### Files

| File | Purpose |
|------|---------|
| `module.prop` | Magisk module info |
| `service.sh` | Boot script: registers caps, sets the app up, sets 90Hz, then exits |
| `apply.sh` | Registers the game-mode caps for every app in the config |
| `setup.sh` | Allows root for the hook app, switches its accessibility service on, protects it from being killed |
| `focus.sh` | Sets the display rate for the app in front. Called by the hook app |
| `action.sh` | Magisk Action button: runs `apply.sh` and `setup.sh` |
| `uninstall.sh` | Resets game-mode settings and the refresh rate on removal |
| `perapp_fps.conf` | Your config |
| `system/priv-app/PerAppFpsHook/` | The bundled hook app |

Config location after install: `/data/adb/modules/perapp_fps/perapp_fps.conf`

---

## Install

**Clean install (recommended):**

1. If you installed the hook app by hand before, uninstall it first: `su -c "pm uninstall com.perappfps.hook"`. A leftover copy can stop the bundled one from taking over.
2. Remove any older Per-App FPS Cap module in Magisk and reboot.
3. Flash the zip and reboot. The system needs the reboot to pick up the priv-app.
4. Wait about a minute. The module then allows root for the hook app, turns its accessibility service on and sets it up.

Nothing else needs doing by hand.

---

## Config

```
# package=fps          display auto-pinned (up to 60 FPS gives 60Hz, otherwise 90Hz)
# package=fps,hz       choose the display rate yourself (60 or 90)
# debugging=true       write /data/local/tmp/perapp_fps.log (off by default)
debugging=false
com.and.games505.TerrariaPaid=60
```

| Line | Meaning |
|------|---------|
| `package=fps` | Cap the app at that FPS. The display is pinned to 60Hz for caps of 60 or lower, otherwise left at 90Hz. |
| `package=fps,hz` | Same, but you choose the display rate, for example `com.termux=30,60`. |
| `debugging=true` | Writes a log (see Debugging). |

Any app not in the config gets 90Hz.

**After editing:** tap the module's **Action** button in Magisk (or run `su -c "sh /data/adb/modules/perapp_fps/apply.sh"`) to register new game-mode caps. Display rates are re-read from the config on the next app switch, so those changes need no extra step.

The 90Hz default is set as `DEFAULT_HZ=90` in `service.sh` and `focus.sh`. Change both if your normal rate is different.

---

## CPU polling optimization

### The problem (v1)

The first versions had no way to be told when the foreground app changed, so the module checked constantly:

1. wake up every 2 seconds,
2. run `dumpsys window` (the system builds a full window dump),
3. parse it for the focused app, then sleep again.

That is a steady cost, even with the screen on and nothing happening, and switching back to 90Hz lagged by up to the interval.

### Steps along the way

| Version | Approach | Cost |
|---------|----------|------|
| v1 | `dumpsys window` every 2 s | Heavy dump on every cycle |
| v2.4 | Interval moved into the config | Same, but tunable |
| v2.6 | Cheap `pidof` check first; `dumpsys` only while a listed app is running | Almost idle when no listed app is open |
| v3.0 onward | Accessibility hook reports changes; no loop at all | No wake-ups between app switches |

### The fix (v3.x)

```
boot
 └─ service.sh  registers caps, sets up the hook app, sets 90Hz, exits
                (no loop, no timer)

app switch
 └─ Android sends a window-change event to the hook app
     └─ hook app runs:  sh focus.sh <package>
         └─ focus.sh sets 60 or 90, or does nothing if already correct
```

- **No timer.** The module itself does nothing between app switches.
- **One shell.** The hook app keeps a single root shell open, so it doesn't start `su` on every switch.
- **No repeat work.** `focus.sh` remembers the last rate it set (`/data/local/tmp/perapp_fps.cur`) and exits without touching any settings when nothing needs to change.
- **Instant switching.** The display changes the moment the window changes, not at the next poll.

### What remains

- The hook app is a small always-resident process, so it costs some RAM.
- Each app switch runs one short script (`focus.sh`).

I haven't measured this against the older polling versions. Compare battery or `top` over a day if you want numbers.

---

## Keeping the hook app alive

`setup.sh` runs at boot and from the Action button. It:

1. **Allows root** for the app in Magisk (a system app doesn't get root automatically).
2. **Turns the accessibility service on directly** through secure settings, so the greyed-out "restricted setting" toggle doesn't matter.
3. **Adds protections** against being killed: battery-optimization whitelist, background-run allowances, HyperOS Autostart (best effort), and an "active" standby bucket.

The Autostart and standby settings differ between HyperOS builds, so they are best effort. If the app is ever killed mid-session, tap **Action** to bring it back.

---

## Debugging

### 1. Is the module installed correctly?

```
su -c "grep version /data/adb/modules/perapp_fps/module.prop"
su -c "pm path com.perappfps.hook"
```

The first should show the version you flashed. The second should print a path under `/system/priv-app/`. A path under `/data/app/` means a hand-installed copy is still there.

### 2. Is the hook app switched on?

```
su -c "settings get secure enabled_accessibility_services"
```

The output should contain `com.perappfps.hook/com.perappfps.hook.FocusService`. If it doesn't, tap **Action**.

### 3. Is the refresh rate switching?

```
su -c "settings get secure miui_refresh_rate"
```

Check it with a capped app in front (should print 60) and on the home screen (should print 90). Wait a couple of seconds after each switch.

To check the display really follows it:

```
su -c "dumpsys display" | grep -m1 mDisplayModeSpecs
```

With a 60Hz pin the render range should read `(0.0 60.0)`.

### 4. Are the game-mode caps registered?

```
su -c "cmd game list-configs <package>"
```

You should see a `Fps:` value for modes 2 and 3. Lines like `Game mode: 2 not supported by <package>` in the Action output are normal for non-game apps. They only mean the game-mode cap doesn't apply there. The display pin still does.

### 5. Turn on logging

Set `debugging=true` in the config, then switch apps.

```
su -c "cat /data/local/tmp/perapp_fps.log"
```

| Log line | Meaning |
|----------|---------|
| `focus=<package> display=60Hz` | The hook reported an app switch and the rate was changed |
| `<package> fps=60 -> Set game mode...` | A game-mode cap was registered |
| `setup: done` | `setup.sh` finished its steps |
| `setup: hook app not found` | The priv-app isn't installed. Reboot after flashing, and uninstall any hand-installed copy |

Turn debugging back off afterwards.

### 6. Run it by hand

```
su -c "sh /data/adb/modules/perapp_fps/focus.sh com.and.games505.TerrariaPaid"
```

This sets the rate for that app directly. If it works by hand but not on app switches, the hook app isn't sending events (check steps 1 and 2).

### Common problems

| Symptom | Likely cause | What to do |
|---------|--------------|------------|
| Display never changes | Accessibility service is off or the app was killed | Step 2, then tap Action |
| Stays at 60 after leaving the game | The hook stopped sending events | Tap Action; check battery and Autostart settings |
| Jumps to 90 only while a floating window is open | A floating app counts as the foreground app | Expected; tap back into the game |
| Cap works but a stale rate sticks after reboot | Old state file | `su -c "rm -f /data/local/tmp/perapp_fps.cur"` and reboot |

---

## Building the hook app (GitHub Actions)

The hook app's source lives in its own repo and is built on GitHub, so no PC is needed.

- The repo holds the project zip plus a workflow in `.github/workflows/`.
- The workflow unzips the project, builds a debug APK with Gradle 8.7, Android Gradle Plugin 8.5.2 and JDK 17, and uploads it as an artifact named `perapp-fps-hook`.
- To update the module with a new build, put the resulting `app-debug.apk` at `system/priv-app/PerAppFpsHook/PerAppFpsHook.apk` inside the module zip.

---

## Limitations

- **HyperOS-specific.** The display pin uses `miui_refresh_rate` and `user_refresh_rate`. On other ROMs, the standard `peak_refresh_rate` and `min_refresh_rate` keys may be the right ones instead.
- **Android 14 and up** for the game-mode FPS override (`cmd game set --fps`).
- **Floating windows** count as the foreground app, so opening one over a capped game puts the display back to 90Hz.
- **Non-game apps** in the config get the display pin but not the game-mode cap.
- **Removal cleanup.** The reset in `uninstall.sh` may not take effect, because removal scripts can run before the system is ready. If the rate sticks at 60 afterwards, run `settings put secure miui_refresh_rate 90`, then the same for `user_refresh_rate`.
- Tested only on a Redmi Pad SE.

---

## Version history

| Version | Change |
|---------|--------|
| v1.0 | Polling loop, per-app `peak_refresh_rate` |
| v2.0 | Game-mode override via `device_config` |
| v2.1-v2.2 | Display pin; switched to `miui_refresh_rate` and `user_refresh_rate` |
| v2.3-v2.5 | Debugging flag, interval setting, config kept in the module folder |
| v2.6 | `pidof`-gated loop |
| v2.7 | `focus.sh` and event mode |
| v3.0 | Event-only, no loop |
| v3.1 | Bundled hook app, `setup.sh` |
| v3.2 | Hook app shipped as a priv-app, install step removed |

---

## Credits

Module and hook app by Jamagoze.

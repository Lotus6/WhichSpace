# Local Changes

This document describes the full-screen and multi-display shortcut changes on this branch. The implementation is current through commit `777514c`.

## Features

1. **Dedicated shortcuts for full-screen and Split View Spaces.** The Keyboard settings list every open full-screen Space as a separate shortcut target. Two apps in Split View share one target, so one shortcut switches the complete pair. Bindings follow the Space UUID, with the Space ID as a fallback, and do not consume the numbering used by Desktop 1, 2, 3, and so on. **Problem solved:** the original app only allowed shortcuts for numbered Desktops; a single-app full-screen Space or a two-app Split View Space could not be bound directly.
2. **Fast switching on the first shortcut press.** When the pointer has moved to a display but that display has not been clicked, Desktop and full-screen shortcuts use the pointer's display to select the fast gesture path. **Problem solved:** the first shortcut press still followed the old menu bar focus and produced a slow Mission Control-style transition; only later presses became fast.
3. **Previous/next Space shortcuts follow the pointer.** These shortcuts calculate the adjacent Space on the pointer's display even while the menu bar focus remains on another display. Scrolling, menu actions, and scripts retain their original display-selection behavior. **Problem solved:** after moving the pointer to another display, adjacent-Space shortcuts could still operate on the previously focused display, switch the wrong display, or appear to do nothing.
4. **Reliable cross-display full-screen and Split View switching.** When the pointer is not on the display containing the target full-screen Space, the shortcut activates an app on that Space instead of sending a fast gesture to another display. A two-app Split View remains one Space and one target. **Problem solved:** the gesture could be delivered to the wrong display, resulting in a slow Mission Control transition, a one-time success, or no switch to the target app at all.
5. **The Split View menu bar icon consistently represents the left app.** The implementation reads WindowServer's `TileSpaces` layout and selects the tile with the smallest `TileRect.X`. The result is unaffected by focus changes or divider resizing. If layout metadata is unavailable, it falls back to the existing window scan. **Problem solved:** choosing the representative app from front-to-back window order made the icon change after clicking either side of Split View, so one Space had no stable representative icon.
6. **Cross-display Split View activation consistently uses the left app.** The fallback reads the same `TileSpaces` layout and activates the PID of the left tile rather than relying on transient window order. If the left app cannot be activated or the metadata is unavailable, the existing window scan remains as a fallback. **Problem solved:** both Split View apps belong to the same Space, so the old fallback could select the right app after a focus-order change. Process-level activation of that app did not always return to the target Split View, causing intermittent switching failures.

## Behavioral Boundaries

- The fast gesture path remains subject to the Classic Switching setting, mouse-button drag protection, and fallback after gesture bounce detection.
- If the pointer cannot be mapped to a managed display, numbered Desktop and adjacent-Space shortcuts fall back to the current menu bar display reported by macOS.
- Cross-display full-screen switching relies on macOS entering the full-screen Space when its app is activated. Because macOS performs that transition, this path does not promise the animation-free behavior of the fast gesture path.
- The left Split View icon depends on the private `TileLayoutManager` metadata. If macOS does not provide it, the existing front-to-back window order remains the fallback and normal full-screen icons are unaffected.
- Cross-display Split View activation uses the same metadata. If it is unavailable, the PID is stale, or the left app rejects activation, the existing window-scan path is used.

## Commits and Main Files

| Commit | Change | Main files |
| --- | --- | --- |
| `9c43058` | Add full-screen/Split View shortcut targets and their basic switching behavior | `WhichSpace/Settings/KeyboardPane.swift`, `WhichSpace/Hotkeys.swift` |
| `b28b3f0` | Full-screen target registration and system-shortcut forwarding changes | `WhichSpace/AppState.swift`, `WhichSpace/Hotkeys.swift`, `WhichSpace/SpaceSwitcher.swift` |
| `9544cee` | Use the pointer display for the first same-display Desktop or full-screen shortcut press | `WhichSpace/DisplaySpaceProvider.swift`, `WhichSpace/Hotkeys.swift`, `WhichSpace/SpaceSwitcher.swift` |
| `a8aea92` | Calculate previous/next Space shortcuts on the pointer display | `WhichSpace/Hotkeys.swift`, `WhichSpace/SpaceSwitcher.swift` |
| `51cae2c` | Activate the target app for cross-display full-screen/Split View shortcuts instead of reporting a successful gesture on the wrong display | `WhichSpace/Hotkeys.swift`, `WhichSpace/SpaceSwitcher.swift` |
| `8b9bd65` | Use `TileRect.X` to represent Split View with the left app in the menu bar, with the old behavior as a fallback | `WhichSpace/DisplaySpaceProvider.swift`, `WhichSpaceTests/UI/StatusBarRendererFilterTests.swift` |
| `777514c` | Read `TileSpaces` during cross-display Split View fallback and activate the left tile PID, retaining the old scan as a fallback | `WhichSpace/SpaceSwitcher.swift` |

Regression coverage is in `WhichSpaceTests/AppState/DynamicDisplayTests.swift`, `WhichSpaceTests/Integration/SpaceSwitcherTests.swift`, and `WhichSpaceTests/UI/StatusBarRendererFilterTests.swift`. The commits above contain the source changes; test DMGs are build artifacts and are not intended for upstreaming.

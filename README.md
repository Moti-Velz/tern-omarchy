# Tern on Omarchy

Makes Tern feel native on [Omarchy](https://omarchy.org/):

- **One set of window keys.** Omarchy's window keys act on Tern panes first. Where Tern has
  nothing to do (the edge of a tab, the last pane, no divider to move) they bubble up to Hyprland
  and act on windows, the way vim-tmux-navigator works with tmux.
- **Agent notifications.** You get a desktop notification when an agent finishes, fails or waits
  for your input while you aren't looking at it. Click it to jump to that pane.

## Keys

| Key | On a Tern window | Elsewhere (Omarchy default) |
| --- | --- | --- |
| Super+← → ↑ ↓ | Focus the pane in that direction; at the tab's edge, the next window | Focus window |
| Super+Shift+← → ↑ ↓ | Swap with the pane in that direction; at the edge, swap windows | Swap window |
| Super+− / Super+= (+Shift for height, +Alt for small steps) | Resize the pane; with no divider on that axis, the window | Resize window |
| Super+W | Close the pane with Tern's normal close, which asks first if something is running; on the session's last pane, close the Tern window (the session daemon keeps the programs) | Close window |
| Super+Return | New shell pane in the focused pane's directory, split along its longer side | Terminal |
| Super+Shift+A | New agent block | ChatGPT |
| Super+Shift+P | New browser block | Google Photos |
| Super+B | Toggle [Forge](https://github.com/Moti-Velz/tern-forge)'s Projects sidebar | Bring the last Tern window forward and show Projects there |

When focus moves onto Tern from another window, Tern focuses the pane on the edge you came in
through. If you're coming back to the pane you left, it keeps that one.

## Forge companion

With [Forge](https://github.com/Moti-Velz/tern-forge) installed, Super+B is its Projects sidebar,
from anywhere. Forge keeps its own key, Super+Alt+Shift+P, and Super+B presses that key for you:
plugins can't run each other's commands, so the key is the only way in. If you rebind Forge's key,
change `SUPER + ALT + SHIFT`, `P` in `hypr/tern-nav.lua` and `lib/hypr.luau`'s `press` call in
`window.luau` to match.

## How it works

- **Hyprland** (`hypr/tern-nav.lua`) replaces those Omarchy binds with Lua functions. On a Tern
  window (class `so.stencil.tern`) they inject a chord into Tern. Elsewhere they run Omarchy's
  original action.
  - Most chords are the same keys with Super held.
  - Where Tern's own preset keys already use a chord, it's Super+Ctrl+Shift+letter: A/D/W/S swap;
    H/L, K/J and Y/O, I/U resize.
  - Super+Ctrl+Shift+arrow means "focus just entered Tern".
- **Tern** (`window.luau`) works out pane rectangles from the tab's split tree (`lib/geometry.luau`).
  It acts on the pane layout, or calls `hyprctl dispatch` (`lib/hypr.luau`) to do what Hyprland
  would have done.
- **Notifications** (`lib/agents.luau`) listen for Tern's `agent_finished` event and check agent
  states every 3 s for "waiting for input".
  - They are skipped when the agent's pane is focused and Tern is the active window.
  - They are sent through `omarchy-notification-send`. Clicking one runs `bin/focus-pane`
    (`tern focus <pane>`, then focuses the Tern window).

## Install

```sh
git clone https://github.com/Moti-Velz/tern-omarchy ~/src/tern-omarchy
tern plugin link ~/src/tern-omarchy
```

Then load the Hyprland side at the end of `~/.config/hypr/bindings.lua`:

```lua
dofile(os.getenv("HOME") .. "/src/tern-omarchy/hypr/tern-nav.lua")
```

Validate it with `hyprctl reload && hyprctl configerrors`. To go back to Omarchy's defaults, remove
that line.

## Scripting and testing

Virtual keyboards (`wtype`) don't trigger Hyprland binds, so every bind's function is also
reachable by its keys:

```sh
hyprctl eval 'tern_nav.press("SUPER + LEFT")'
```

## Agent desktop

For agents in your Tern panes that do computer use (launch, click, type, screenshot), install
[agent-desktop](https://github.com/Moti-Velz/agent-desktop). It runs their GUI work in a hidden
nested Hyprland with its own cursor and focus, so they never steal your focus or flicker your
screen while you work in Tern:

```sh
npx skills add Moti-Velz/agent-desktop
```

Its README covers the one-line Hyprland rule it needs.

## Notes

- Tern's preset uses Super+Shift+arrows and Super+Alt(+Shift)+←/→. That's why those actions reach
  Tern as Super+Ctrl+Shift+letter chords.
- **Fallback:** each Tern window running the plugin writes a heartbeat every 2 s
  (`$XDG_RUNTIME_DIR/tern-omarchy/<window pid>`). Hyprland only hands keys to Tern windows whose
  heartbeat is under 6 s old. A Tern window without the plugin (not linked, failed to load, another
  config) gets Omarchy's normal actions instead of dead keys, at most 6 s after the plugin stops.
  `hyprctl repl 'return tern_nav.handles(hl.get_active_window())'` shows the current decision.
- When Tern hands a move back, Hyprland's own rules apply. A focus move past the last window wraps
  around, just as it does for ordinary windows.

## License

MIT — see [LICENSE](LICENSE).

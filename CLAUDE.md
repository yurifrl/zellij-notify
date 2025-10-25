# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Zellij plugin written in Rust that automatically removes trailing status emojis (✅, ❌, 🔴, ⚠️, ⚡, 💼, 🎉) from tab names when you switch to them. It's compiled to WebAssembly (WASM) and runs inside the Zellij terminal multiplexer.

## Build System

The project uses **Task** (go-task) for build automation, not Make or cargo-make.

### Common Commands

```bash
# Build, version bump, install, and reload the plugin in one command
task build

# View plugin logs (tails Zellij log file and filters for this plugin)
task logs
```

### Manual Build

```bash
# Build the WASM plugin for Zellij
cargo build --release --target wasm32-wasip1

# The output will be at:
target/wasm32-wasip1/release/zellij_notify.wasm
```

**IMPORTANT**: If you need to install the plugin after building, always use `task build` instead of manually copying files with `cp`. The task command handles version bumping, installation, log clearing, and plugin reloading automatically.

## Architecture

**Single-file plugin**: All code is in `src/lib.rs`. The crate type is `cdylib` to produce a dynamic library for WASM.

### Key Components

1. **State struct**: Tracks:
   - `all_tabs: Vec<TabInfo>` - All tabs, not just active one
   - `focused_tab_position: Option<usize>` - Currently focused tab
   - `pane_manifest: Option<PaneManifest>` - Maps panes to their tab positions
   - `presets: HashMap<String, PresetConfig>` - Emoji presets from config
   - `debug: bool` - Debug logging flag

2. **Event handling**: Subscribes to `TabUpdate` and `PaneUpdate` events from Zellij

3. **Auto-cleanup logic**: When you focus on a tab for the first time, if it has emoji → remove it

4. **Pane-to-tab mapping**: Uses `PaneManifest` to identify which tab a pane belongs to (critical for background commands)

5. **Name cleaning**: `remove_trailing_emojis()` strips emojis and trailing whitespace

6. **Permissions**: Requires `ReadApplicationState` and `ChangeApplicationState` to read tab info and rename tabs

### Event Flow

#### TabUpdate Event (Auto-Cleanup)
1. Zellij fires `TabUpdate` event → plugin receives all tab info
2. Plugin finds the active tab
3. Check if this is a **new focus** (different from previous `focused_tab_position`)
4. If new focus:
   - Update `focused_tab_position` to current tab (do this FIRST)
   - Check if tab has trailing emojis
   - If yes: clean the name with `remove_trailing_emojis()` and call `rename_tab(position + 1, cleaned_name)`

**Key insight**: Only cleaning on focus change prevents infinite loops - we don't continuously check/rename the same tab

#### PaneUpdate Event (Tab Identification)
1. Zellij fires `PaneUpdate` event → plugin receives `PaneManifest`
2. Plugin stores the manifest in `self.pane_manifest`
3. The manifest maps tab positions to their panes: `BTreeMap<usize, Vec<PaneInfo>>`
4. When a pipe command arrives with `pane_id` in args, plugin searches this manifest to find which tab contains that pane
5. This allows the plugin to update the correct tab even if the user has switched tabs

#### Pipe Message Flow (Emoji Addition)
1. User runs command: `zellij pipe -n "notify" -a "pane_id=$ZELLIJ_PANE_ID" "stop"`
2. Plugin receives `PipeMessage` with:
   - `name: "notify"`
   - `payload: "stop"` (the preset key)
   - `args: {"pane_id": "123"}` (from the `-a` flag)
3. Plugin tries three methods to identify the target tab (in order):
   - **Method 1**: If `pane_id` in args → search `PaneManifest` to find which tab contains this pane (MOST RELIABLE)
   - **Method 2**: If `tab_position` in args → use explicit position
   - **Method 3**: Use currently focused tab (UNRELIABLE for background commands)
4. Plugin looks up emoji from presets (or uses default ✅)
5. Plugin renames the identified tab: `rename_tab(position + 1, clean_name + emoji)`

### Important Gotchas

**Zellij API indexing**: `rename_tab()` expects **1-based** indices (1, 2, 3...), but `TabInfo.position` is **0-based** (0, 1, 2...). Always add 1 when calling `rename_tab()`:

```rust
let tab_index = tab.position as u32 + 1;  // Convert 0-based to 1-based
rename_tab(tab_index, cleaned_name);
```

### Debug Logging

All debug output uses `eprintln!()` which goes to Zellij's log file (typically in `/tmp` or `/var/folders`). The `task logs` command helps view these logs in real-time.

## Development Workflow

The `task build` command automates:
1. **Version bump**: Auto-increments patch version in Cargo.toml
2. **Build**: Compiles to WASM
3. **Install**: Copies to `~/.config/zellij/plugins/zellij-notify.wasm`
4. **Clear logs**: Wipes Zellij log file for clean debugging
5. **Reload**: Tells Zellij to reload the plugin without restart

This makes iteration very fast during development.

## Pipe Commands

Appends emoji to identified tab using Zellij's `pipe()` method.

### Direct Command

**Recommended**: Use the direct command with pane ID for reliable background command support:

```bash
# Direct command - always pass pane_id for background commands!
zellij pipe -n "notify" -a "pane_id=$ZELLIJ_PANE_ID" "stop"

# Works with background commands
sleep 5 && zellij pipe -n "notify" -a "pane_id=$ZELLIJ_PANE_ID" "stop"

# Without pane_id (unreliable if you switch tabs during command execution)
zellij pipe -n "notify" "stop"

# With explicit tab position (0-indexed, alternative to pane_id)
zellij pipe -n "notify" -a "tab_position=0" "stop"
```

### Optional Shell Functions

If you prefer shorter commands, create aliases or functions in your shell config:

**Bash/Zsh** (`~/.bashrc` or `~/.zshrc`):
```bash
zellij-notify() {
    zellij pipe -n "notify" -a "pane_id=$ZELLIJ_PANE_ID" "${1:-}"
}
```

**Fish** (`~/.config/fish/config.fish`):
```fish
function zellij-notify
    zellij pipe -n "notify" -a "pane_id=$ZELLIJ_PANE_ID" $argv[1]
end
```

### Config

Presets defined in plugin config as JSON:

```kdl
plugin location="file:/path/to/zellij-notify.wasm" {
    debug "false"  // Set to "true" for verbose logging
    presets r#"{
        "notification": {"emoji": "⚡"},
        "posttooluse": {"emoji": "⚡"},
        "stop": {"emoji": "✅"},
        "subagent-stop": {"emoji": "🔴"}
    }"#
}
```

### How Tab Identification Works

1. **Pane ID method** (most reliable): Pass `pane_id` via `-a` flag, plugin uses `PaneManifest` to find which tab contains that pane
2. **Explicit position method**: Pass `tab_position` via `-a` flag (0-indexed)
3. **Fallback method**: Use currently focused tab (unreliable for background commands)

### Integration

Auto-cleaning on tab focus removes appended emojis.

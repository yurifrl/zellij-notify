# zellij-notify

A Zellij plugin that adds status emojis to your current tab and automatically cleans them when you switch tabs.

## Features

### 🎯 Pipe Command Integration
Append status emojis to your current tab using Zellij's pipe command:

```bash
# Use default emoji (✅)
zellij pipe -n "notify" -a "pane_id=$ZELLIJ_PANE_ID"
```

Perfect for integration with shell scripts, CI/CD, or IDE hooks to show task status!

### 🧹 Auto-Cleanup
When you switch to a tab, trailing status emojis are automatically removed. This prevents clutter from accumulating as you work.

Cleaned emojis: ✅ ❌ 🔴 ⚠️ ⚡ 💼 🎉 ❓

## Installation

### Prerequisites
- Rust toolchain with `wasm32-wasip1` target
- [Task](https://taskfile.dev) (optional, for easier building)
- Zellij terminal multiplexer

Add the WASM target if you don't have it:
```bash
rustup target add wasm32-wasip1
```

### Build and Install

```bash
cargo build --release --target wasm32-wasip1
cp target/wasm32-wasip1/release/zellij_notify.wasm ~/.config/zellij/plugins/
```

### Configuration

Add to your Zellij config at `~/.config/zellij/config.kdl`:

```kdl
load_plugins {
    "file:~/.config/zellij/plugins/zellij-notify.wasm" {
        debug "false"  // Optional: set to "true" for verbose logging
        presets r#"{
            "notification": {"emoji": "⚡"},
            "posttooluse": {"emoji": "⚡"},
            "stop": {"emoji": "✅"},
            "subagent-stop": {"emoji": "🔴"}
        }"#
    }
}
```

## Usage

### Basic Pipe Commands

```bash
# Append default emoji to current tab
zellij pipe -n "notify" ""

# Use preset emojis

zellij pipe -n "notify" -a "pane_id=$ZELLIJ_PANE_ID" "stop"           # Tab becomes "myproject ✅"
zellij pipe -n "notify" -a "pane_id=$ZELLIJ_PANE_ID" "notification"   # Tab becomes "myproject ⚡"
```

### Why pass pane_id?

When a command executes in the background (after you've switched tabs), Zellij needs to know which tab sent the command. The `ZELLIJ_PANE_ID` environment variable identifies the source pane, and the plugin uses this to find the correct tab.

### Claude Hook Integration

Example with Claude Code (`~/.claude/settings.json`):

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "zellij pipe -n \"notify\" -a \"pane_id=$ZELLIJ_PANE_ID\" \"posttooluse\""
      }]
    }],
    "Stop": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "zellij pipe -n \"notify\" -a \"pane_id=$ZELLIJ_PANE_ID\" \"stop\""
      }]
    }]
  }
}
```

## Configuration Options

### Debug Logging

Set `debug "true"` in the plugin config to enable verbose logging:

```kdl
"file:~/.config/zellij/plugins/zellij-notify.wasm" {
    debug "true"
}
```

View logs with:
```bash
task logs
```

### Custom Presets

Define your own emoji presets in the config:

```kdl
presets r#"{
    "success": {"emoji": "🎉"},
    "error": {"emoji": "💥"},
    "warning": {"emoji": "⚠️"},
    "info": {"emoji": "ℹ️"}
}"#
```

## How It Works

### Pane-to-Tab Mapping

For background commands that execute after you've switched tabs, the plugin needs to identify the source tab:

1. You pass `ZELLIJ_PANE_ID` via the `-a` flag in your command
2. Plugin receives the pane ID in `pipe_message.args`
3. Plugin subscribes to `PaneUpdate` events to maintain a `PaneManifest`
4. Plugin searches the manifest to find which tab contains the pane
5. Plugin updates the correct tab, even if you've switched to a different one

This is why passing `pane_id` is important for background commands!

## Development

```bash
# Build, install, and reload in one command
task build

# View plugin logs
task logs
```


## License

MIT License - see LICENSE file for details

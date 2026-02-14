# claude-plugins

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

Rich's Claude Code plugin marketplace.

## Plugins

### rich-blocks-claude

Safety and productivity hooks for Claude Code — blocks dangerous commands, protects sensitive files, enforces tool preferences, and provides desktop notifications. See the [rich-blocks-claude repo](https://github.com/RichAyotte/rich-blocks-claude) for installation instructions.

**Hooks provided:**

| Event | Matcher | Description |
|-------|---------|-------------|
| PreToolUse | Bash | Blocks dangerous shell commands |
| PreToolUse | Write | Protects sensitive file paths |
| PreToolUse | Read | Guards restricted files |
| PreToolUse | Edit | Protects sensitive file edits |
| PreToolUse | Grep | Enforces tool preferences |
| PostToolUse | (all) | Post-execution sensitive file checks |
| Notification | (all) | Desktop notifications via notify-send |
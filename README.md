# claude-plugins

Rich's Claude Code plugin marketplace.

## Plugins

### rich-claude-armor

Safety and productivity hooks for Claude Code — blocks dangerous commands, protects sensitive files, enforces tool preferences, and provides desktop notifications. See the [rich-claude-armor repo](https://github.com/RichAyotte/rich-claude-armor) for installation instructions.

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

## License

[MIT](LICENSE)

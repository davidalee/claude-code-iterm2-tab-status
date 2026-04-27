# Configurable Status Display Target Design

## Summary

Add a configurable display target for Claude Code iTerm2 status updates. Existing users keep the current title-prefix behavior by default, while users with vertical tabs can send status text to an iTerm2 user variable and show it in the profile Subtitle field.

## Problem

The adapter currently applies status by changing the iTerm2 tab title through the Python API. That works for horizontal and vertical tabs, but vertical-tab users often want the main title to remain the command, job, or directory while Claude status appears on the subtitle line.

iTerm2 supports user-defined variables in session context. A profile subtitle can reference a variable such as `\(user.claudeStatus)`, so the adapter can update the subtitle without taking over the tab title.

## Goals

- Preserve the current title-prefix behavior as the default.
- Let users choose title, subtitle, or both as the display target.
- Let users customize the user variable name used for subtitle display.
- Clear the subtitle variable when the Claude session is cleared.
- Document iTerm2 profile setup for subtitle users.
- Cover config parsing and status value behavior with tests.

## Non-Goals

- Do not replace the existing signal-file hook architecture.
- Do not emit raw iTerm2 escape sequences from the shell hook.
- Do not change badge, flashing, notification, sound, focus, or stale-signal behavior.
- Do not disable Claude Code terminal title updates automatically.

## Configuration

Add two config keys:

```json
{
  "display_target": "title",
  "subtitle_variable": "claudeStatus"
}
```

`display_target` accepts:

- `title`: current behavior; prefix the iTerm2 tab title.
- `subtitle`: set an iTerm2 user variable and leave the title untouched.
- `both`: update both title and user variable.

Invalid `display_target` values fall back to `title`.

`subtitle_variable` is the unqualified variable name users reference from iTerm2 as `\(user.<name>)`. Invalid empty values fall back to `claudeStatus`. The adapter will pass the fully qualified name `user.<name>` to the iTerm2 Python API.

Environment variables:

- `CLAUDE_ITERM2_TAB_STATUS_DISPLAY_TARGET`
- `CLAUDE_ITERM2_TAB_STATUS_SUBTITLE_VARIABLE`

## Runtime Behavior

When a signal enters a state, the adapter builds the status text from the configured state prefix:

- Running: configured `prefix_running` stripped of trailing whitespace for subtitle display.
- Idle: configured `prefix_idle` stripped of trailing whitespace for subtitle display.
- Attention: configured `prefix_attention` stripped of trailing whitespace for subtitle display.

Title display continues to use the existing prefix plus the current title/session name.

Subtitle display calls `session.async_set_variable("user.<subtitle_variable>", status_text)` on the matched iTerm2 session. If this fails, the adapter logs at debug level and continues, matching the current best-effort behavior for title updates.

When a Claude session is cleared, the adapter sets the same user variable to an empty string. This removes stale subtitle content while preserving the user's profile subtitle expression.

## Files

- `scripts/claude_tab_status.py`
  - Add config defaults and environment mapping.
  - Add helper functions for display target validation and user variable naming.
  - Update state entry and clear logic to route status to title, subtitle, or both.

- `tests/test_adapter.py`
  - Add unit tests for default config values.
  - Add unit tests for file and environment overrides.
  - Add unit tests for invalid display target fallback and subtitle variable fallback.
  - Add async tests for setting and clearing subtitle variables.

- `README.md`
  - Document display target settings.
  - Add iTerm2 setup instructions: Profiles > General > Subtitle = `\(user.claudeStatus)`.
  - Note that users who want Claude Code to stop changing main titles can set `CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1` in their shell startup file.

- `commands/config.md`
  - Include the two new settings in the interactive config command instructions.

## Testing

Run focused adapter tests after implementation:

```bash
uv run pytest tests/test_adapter.py -q
```

Run the full test suite if focused tests pass:

```bash
uv run pytest -q
```

Because iTerm2's Python runtime is mocked in unit tests, manual verification in iTerm2 remains useful for the subtitle expression. The expected manual setup is:

1. Set plugin config `display_target` to `subtitle`.
2. Set iTerm2 profile Subtitle to `\(user.claudeStatus)`.
3. Start a Claude Code session and submit a prompt.
4. Confirm the main title remains controlled by iTerm2/Claude settings while the subtitle changes between running, idle, and attention states.

## Risks

The main compatibility risk is accidentally changing default title behavior. The default remains `title`, and tests should assert that missing config produces the existing behavior.

The iTerm2 user-variable API requires fully qualified names beginning with `user.`. The config stores only the short name to keep user-facing setup simple, and the adapter constructs the qualified name internally.

Claude Code can still set terminal titles independently. This feature does not suppress that; documentation will tell users how to opt out with `CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1` if they want main titles fully controlled by iTerm2.

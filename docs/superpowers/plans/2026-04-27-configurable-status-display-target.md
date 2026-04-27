# Configurable Status Display Target Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `display_target` config option that can route Claude status to the tab title, the fixed `user.claudeStatus` subtitle variable, or both.

**Architecture:** Keep the existing hook JSON and iTerm2 adapter flow. Add small top-level helpers for display-target normalization, routing checks, subtitle text formatting, and best-effort user-variable writes, then call those helpers from the current `_enter_state` and `clear_session` paths.

**Tech Stack:** Python 3.10+, iTerm2 Python API, pytest with mocked iTerm2 module, Markdown docs.

---

## Pre-Edit Impact Analysis

GitNexus impact was run with `--repo claude-code-iterm2-tab-status` before implementation planning:

- `load_config`: LOW risk; direct dependents are module initialization and `reload_config`; affected process is adapter `main`.
- `_enter_state`: LOW risk; direct dependent is `apply_state`; affected process is adapter `main`.
- `clear_session`: LOW risk; direct dependent is `signal_watcher`; affected process is adapter `main`.
- `_set_tab_title`: LOW risk; direct dependent is `_enter_state`; affected processes are adapter `main` and `_enter_state`.

The GitNexus MCP server is unavailable in this session, and the installed CLI does not expose `detect_changes`; use `git diff` and available GitNexus CLI commands for verification.

## File Structure

- Modify `scripts/claude_tab_status.py`
  - Add `display_target` to defaults and env mapping.
  - Add display-target helpers near the existing title-prefix helpers.
  - Update `_enter_state` to call title and subtitle paths based on config.
  - Update `clear_session` to clear the fixed subtitle variable.

- Modify `tests/test_adapter.py`
  - Extend `TestConfig` with display-target parsing tests.
  - Add helper tests for target routing and subtitle text.
  - Add async tests for setting and clearing `user.claudeStatus`.

- Modify `README.md`
  - Document `display_target`.
  - Add iTerm2 Subtitle setup using `\(user.claudeStatus)`.
  - Mention `CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1` as an optional shell setting.

- Modify `commands/config.md`
  - Add the display target setting to the interactive config table and validation rules.

## Task 1: Add Config Tests And Parsing

**Files:**
- Modify: `tests/test_adapter.py:400`
- Modify: `scripts/claude_tab_status.py:53`

- [ ] **Step 1: Write failing config tests**

Add these tests inside `class TestConfig` in `tests/test_adapter.py` after `test_defaults_when_no_file`:

```python
    def test_default_display_target_is_title(self, tmp_path: Path):
        cfg = claude_tab_status.load_config(str(tmp_path / "nonexistent.json"))
        assert cfg["display_target"] == "title"

    def test_file_display_target_override(self, tmp_path: Path):
        cfg_file = tmp_path / "config.json"
        cfg_file.write_text(json.dumps({"display_target": "subtitle"}))
        cfg = claude_tab_status.load_config(str(cfg_file))
        assert cfg["display_target"] == "subtitle"

    def test_env_display_target_override(self, tmp_path: Path, monkeypatch: pytest.MonkeyPatch):
        cfg_file = tmp_path / "config.json"
        cfg_file.write_text(json.dumps({"display_target": "subtitle"}))
        monkeypatch.setenv("CLAUDE_ITERM2_TAB_STATUS_DISPLAY_TARGET", "both")
        cfg = claude_tab_status.load_config(str(cfg_file))
        assert cfg["display_target"] == "both"

    def test_invalid_display_target_defaults_to_title(self, tmp_path: Path):
        cfg_file = tmp_path / "config.json"
        cfg_file.write_text(json.dumps({"display_target": "bad-value"}))
        cfg = claude_tab_status.load_config(str(cfg_file))
        assert cfg["display_target"] == "title"

    def test_display_target_is_case_insensitive(self, tmp_path: Path):
        cfg_file = tmp_path / "config.json"
        cfg_file.write_text(json.dumps({"display_target": "SUBTITLE"}))
        cfg = claude_tab_status.load_config(str(cfg_file))
        assert cfg["display_target"] == "subtitle"
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run pytest tests/test_adapter.py::TestConfig -q
```

Expected: at least the new `display_target` assertions fail because the config key and validation helper do not exist yet.

- [ ] **Step 3: Implement config parsing**

In `scripts/claude_tab_status.py`, add the default:

```python
_DEFAULTS: dict[str, object] = {
    "dir": "/tmp/claude-tab-status",
    "prefix_running": "⚡ ",
    "prefix_idle": "💤 ",
    "prefix_attention": "🔴 ",
    "color_r": 255,
    "color_g": 140,
    "color_b": 0,
    "interval": 0.6,
    "badge_enabled": True,
    "badge": "⚠️ Needs input",
    "notify": False,
    "sound": "",
    "display_target": "title",
}
```

Add the env mapping:

```python
_ENV_MAP: dict[str, tuple[str, type]] = {
    "dir": ("DIR", str),
    "prefix_running": ("PREFIX_RUNNING", str),
    "prefix_idle": ("PREFIX_IDLE", str),
    "prefix_attention": ("PREFIX_ATTENTION", str),
    "color_r": ("COLOR_R", int),
    "color_g": ("COLOR_G", int),
    "color_b": ("COLOR_B", int),
    "interval": ("INTERVAL", float),
    "badge_enabled": ("BADGE_ENABLED", lambda v: v.lower() == "true"),
    "badge": ("BADGE", str),
    "notify": ("NOTIFY", lambda v: v.lower() == "true"),
    "sound": ("SOUND", str),
    "display_target": ("DISPLAY_TARGET", str),
}
```

Add this helper near the configuration section before `load_config`:

```python
_DISPLAY_TARGETS = {"title", "subtitle", "both"}


def _normalize_display_target(value: object) -> str:
    """Return a supported display target, defaulting to title."""
    if isinstance(value, str):
        normalized = value.strip().lower()
        if normalized in _DISPLAY_TARGETS:
            return normalized
    return "title"
```

At the end of `load_config`, before `return cfg`, normalize the merged setting:

```python
    cfg["display_target"] = _normalize_display_target(cfg.get("display_target"))

    return cfg
```

- [ ] **Step 4: Run config tests to verify pass**

Run:

```bash
uv run pytest tests/test_adapter.py::TestConfig -q
```

Expected: all `TestConfig` tests pass.

- [ ] **Step 5: Commit config parsing**

Run:

```bash
git add scripts/claude_tab_status.py tests/test_adapter.py
git commit -m "feat: add display target config"
```

## Task 2: Add Subtitle Helper Tests And Implementation

**Files:**
- Modify: `tests/test_adapter.py:318`
- Modify: `scripts/claude_tab_status.py:374`

- [ ] **Step 1: Write failing helper tests**

Add this test class in `tests/test_adapter.py` after `TestSetStatePrefix` and before `TestTitlePrefix`:

```python
# --- TestDisplayTargetHelpers ---


class TestDisplayTargetHelpers:
    def test_should_update_title_for_title_target(self):
        assert claude_tab_status._should_update_title("title") is True
        assert claude_tab_status._should_update_subtitle("title") is False

    def test_should_update_subtitle_for_subtitle_target(self):
        assert claude_tab_status._should_update_title("subtitle") is False
        assert claude_tab_status._should_update_subtitle("subtitle") is True

    def test_should_update_both_for_both_target(self):
        assert claude_tab_status._should_update_title("both") is True
        assert claude_tab_status._should_update_subtitle("both") is True

    def test_subtitle_status_text_strips_padding(self):
        assert claude_tab_status._subtitle_status_text("⚡ ") == "⚡"

    @pytest.mark.asyncio
    async def test_set_subtitle_status_uses_fixed_user_variable(self):
        session = MagicMock()
        session.async_set_variable = AsyncMock()
        await claude_tab_status._set_subtitle_status(session, "⚡")
        session.async_set_variable.assert_awaited_once_with("user.claudeStatus", "⚡")

    @pytest.mark.asyncio
    async def test_clear_subtitle_status_sets_empty_string(self):
        session = MagicMock()
        session.async_set_variable = AsyncMock()
        await claude_tab_status._clear_subtitle_status(session)
        session.async_set_variable.assert_awaited_once_with("user.claudeStatus", "")

    @pytest.mark.asyncio
    async def test_set_subtitle_status_is_best_effort(self):
        session = MagicMock()
        session.async_set_variable = AsyncMock(side_effect=Exception("boom"))
        await claude_tab_status._set_subtitle_status(session, "⚡")
        session.async_set_variable.assert_awaited_once_with("user.claudeStatus", "⚡")
```

- [ ] **Step 2: Run tests to verify failure**

Run:

```bash
uv run pytest tests/test_adapter.py::TestDisplayTargetHelpers -q
```

Expected: FAIL with missing helper attributes on `claude_tab_status`.

- [ ] **Step 3: Implement subtitle helpers**

In `scripts/claude_tab_status.py`, add these helpers after `set_state_prefix` and before `add_title_prefix`:

```python
_SUBTITLE_VARIABLE = "user.claudeStatus"


def _should_update_title(display_target: object | None = None) -> bool:
    """Return whether the current target should update the tab title."""
    target = _normalize_display_target(display_target or CONFIG.get("display_target"))
    return target in {"title", "both"}


def _should_update_subtitle(display_target: object | None = None) -> bool:
    """Return whether the current target should update the subtitle variable."""
    target = _normalize_display_target(display_target or CONFIG.get("display_target"))
    return target in {"subtitle", "both"}


def _subtitle_status_text(prefix: str) -> str:
    """Convert a title prefix into compact subtitle text."""
    return prefix.strip()


async def _set_subtitle_status(session: object, status_text: str) -> None:
    """Set the fixed iTerm2 user variable used by profile subtitles."""
    try:
        await session.async_set_variable(_SUBTITLE_VARIABLE, status_text)  # type: ignore[union-attr]
    except Exception:
        log.debug("session.async_set_variable failed for subtitle status")


async def _clear_subtitle_status(session: object) -> None:
    """Clear the fixed iTerm2 user variable used by profile subtitles."""
    await _set_subtitle_status(session, "")
```

- [ ] **Step 4: Run helper tests to verify pass**

Run:

```bash
uv run pytest tests/test_adapter.py::TestDisplayTargetHelpers -q
```

Expected: all helper tests pass.

- [ ] **Step 5: Run related config and helper tests**

Run:

```bash
uv run pytest tests/test_adapter.py::TestConfig tests/test_adapter.py::TestDisplayTargetHelpers -q
```

Expected: all selected tests pass.

- [ ] **Step 6: Commit subtitle helpers**

Run:

```bash
git add scripts/claude_tab_status.py tests/test_adapter.py
git commit -m "feat: add subtitle status helpers"
```

## Task 3: Route Runtime Updates To Title, Subtitle, Or Both

**Files:**
- Modify: `scripts/claude_tab_status.py:525`
- Modify: `scripts/claude_tab_status.py:619`

- [ ] **Step 1: Add focused regression tests for routing helpers**

The nested `_enter_state` function is inside `main()`, so unit-test the routing predicates and subtitle write helpers from Task 2 rather than driving the full iTerm2 loop. Add this test to `TestDisplayTargetHelpers`:

```python
    def test_invalid_routing_target_behaves_like_title(self):
        assert claude_tab_status._should_update_title("bad-value") is True
        assert claude_tab_status._should_update_subtitle("bad-value") is False
```

- [ ] **Step 2: Run routing test to verify pass or expected failure**

Run:

```bash
uv run pytest tests/test_adapter.py::TestDisplayTargetHelpers::test_invalid_routing_target_behaves_like_title -q
```

Expected: PASS if Task 2 used `_normalize_display_target`; otherwise FAIL until the helpers call the normalizer.

- [ ] **Step 3: Implement runtime routing**

In `_enter_state`, replace the unconditional title update:

```python
        # ALL states: set title prefix
        await _set_tab_title(info, prefix)
```

with:

```python
        if _should_update_title():
            await _set_tab_title(info, prefix)
        if _should_update_subtitle():
            await _set_subtitle_status(info["session"], _subtitle_status_text(prefix))
```

In `clear_session`, after `info = active.pop(claude_sid)` and before restoring the tab title, clear the subtitle status:

```python
        await _clear_subtitle_status(info["session"])
```

This clear is safe for all targets because it only clears this plugin's fixed user variable. It also prevents stale subtitle text if the user changes `display_target` while a Claude session is active.

- [ ] **Step 4: Run adapter tests**

Run:

```bash
uv run pytest tests/test_adapter.py -q
```

Expected: all adapter tests pass.

- [ ] **Step 5: Commit runtime routing**

Run:

```bash
git add scripts/claude_tab_status.py tests/test_adapter.py
git commit -m "feat: route status updates to title or subtitle"
```

## Task 4: Update User Documentation And Config Command

**Files:**
- Modify: `README.md`
- Modify: `commands/config.md`

- [ ] **Step 1: Update README configuration example**

In `README.md`, add `display_target` to the config example:

```json
{
  "signal_dir": "/tmp/claude-tab-status",
  "color_r": 255,
  "color_g": 140,
  "color_b": 0,
  "flash_interval": 0.6,
  "prefix_running": "⚡ ",
  "prefix_idle": "💤 ",
  "prefix_attention": "🔴 ",
  "display_target": "title",
  "badge_text": "⚠️ Needs input",
  "badge_enabled": true,
  "notify": false,
  "sound": "",
  "log_level": "WARNING"
}
```

Add this section under Configuration after "Priority order":

~~~markdown
### Display target

By default, status is shown as a tab title prefix.

Set `"display_target": "subtitle"` to leave the main tab title alone and write status to the iTerm2 user variable `user.claudeStatus`. In iTerm2, open **Settings > Profiles > General** and set **Subtitle** to:

```text
\(user.claudeStatus)
```

Use `"display_target": "both"` to update both the title prefix and subtitle variable.

Claude Code can also set terminal titles. If you want iTerm2 to control the main title while this plugin updates the subtitle, add this to your shell startup file:

```bash
export CLAUDE_CODE_DISABLE_TERMINAL_TITLE=1
```
~~~

- [ ] **Step 2: Update README env table**

Add this row to the environment variable reference table:

```markdown
| `CLAUDE_ITERM2_TAB_STATUS_DISPLAY_TARGET` | `title`                  | Where to show status: `title`, `subtitle`, or `both` |
```

- [ ] **Step 3: Update config command instructions**

In `commands/config.md`, add the setting row:

```markdown
| 12 | Display target | `display_target` | title |
```

Add this validation rule:

```markdown
- `display_target`: one of `title`, `subtitle`, or `both`
```

- [ ] **Step 4: Run structure and adapter tests**

Run:

```bash
uv run pytest tests/test_adapter.py -q
./tests/test_plugin_structure.sh
```

Expected: adapter tests pass and plugin structure test exits 0.

- [ ] **Step 5: Commit docs**

Run:

```bash
git add README.md commands/config.md
git commit -m "docs: document subtitle display target"
```

## Task 5: Final Verification

**Files:**
- No file edits expected unless verification finds a defect.

- [ ] **Step 1: Run full test suite**

Run:

```bash
uv run pytest -q
```

Expected: all pytest tests pass.

- [ ] **Step 2: Run shell tests**

Run:

```bash
./tests/test_plugin_structure.sh
./tests/test_hooks.sh
./tests/test_bootstrap.sh
```

Expected: each script exits 0. If `test_bootstrap.sh` requires network or package installation and fails because sandboxed network is blocked, rerun with approval or record the limitation.

- [ ] **Step 3: Inspect final diff**

Run:

```bash
git status --short
git diff --stat HEAD
```

Expected: only intended implementation/docs files are modified or committed. Existing unrelated working-tree files such as `.gitignore`, `.claude/`, `AGENTS.md`, and `CLAUDE.md` should remain untouched unless the user explicitly asks otherwise.

- [ ] **Step 4: Run available GitNexus verification**

Run:

```bash
npx gitnexus status
```

Expected: index reports the repo status. The installed CLI does not expose `detect_changes`; mention this limitation in the final summary.

- [ ] **Step 5: Final implementation commit if needed**

If Tasks 1-4 were not committed individually, commit all intended implementation changes:

```bash
git add scripts/claude_tab_status.py tests/test_adapter.py README.md commands/config.md
git commit -m "feat: add configurable status display target"
```

Expected: commit succeeds without staging unrelated pre-existing working-tree changes.

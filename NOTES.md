# editor.py — Project Notes for Claude

## What this is

A single-file Python/Tkinter desktop application for editing game localisation JSON files.
It is a dark-themed, multi-file text editor specialised for a specific JSON format that
contains text entries with inline markup tags (ruby annotations, line breaks, voice-over cues).

Run with:
    python editor.py
    python editor.py file1.json file2.json ...

---

## JSON format expected

```json
{
  "ENTRY": 0,
  "TEXTS": [
    {
      "global_id": 1,
      "internal_id": 0,
      "ORIGINAL_TEXT": "Original untranslated text",
      "TEXT": "Edited/translated text"
    },
    ...
  ]
}
```

The editor only modifies `TEXT`. `ORIGINAL_TEXT` is shown read-only for reference.

---

## Inline tag syntax (parsed by `parse_segments()` and `_SEG_RE`)

| Tag | Meaning |
|---|---|
| `[main/RUBY/ruby_text]` | Ruby annotation — small text above main text |
| `[/BR/]` | Line break (rendered as `\|\|` badge) |
| `[/VO/filename]` | Voice-over cue (rendered as a teal label) |
| `%s`, `%d`, `%.2f`, etc. | Printf placeholders (highlighted orange) |

`parse_segments(text)` returns a list of dicts, each with `type` one of:
`"plain"`, `"ruby"`, `"br"`, `"vo"`, `"printf"`.

---

## Code structure

### Top-level (module scope)

| Symbol | Purpose |
|---|---|
| `CFG_PATH` | Settings file path: `<script>.py.cfg` (configparser INI) |
| `_SEG_RE` | Compiled regex matching all four tag types + printf |
| `parse_segments(text)` | Tokeniser; returns list of segment dicts |
| `tk_idx(text, offset)` | Converts char offset → Tk Text index `"line.col"` |
| `BG`, `BG_PANEL`, `ACCENT`, `FG`, … | Dark-theme colour constants |
| `HL_RB_*`, `HL_PF_*`, `HL_BR_*`, `HL_VO_*` | Tag highlight colours |
| `PV_RUBY_MAIN` | Ruby main-text colour in preview |
| `DEFAULT_SHORTCUTS` | Dict: action_id → default `<Modifier-key>` binding string |
| `SHORTCUT_LABELS` | Dict: action_id → human-readable label |
| `_MOD_SHIFT/CTRL/ALT` | Bitmasks for cross-platform modifier detection |
| `_MODIFIER_SYMS` | frozenset of keysym strings that are pure modifiers |
| `_tk_to_display(tk_str)` | `"<Control-Shift-s>"` → `"Ctrl+Shift+S"` |
| `_event_to_tk(event)` | KeyPress event → canonical `"<Modifier-key>"` string |

### Classes

#### `FlatButton(tk.Label)`
Label-based button (avoids macOS tk.Button bg bug). Has `update_command(cmd)` to
swap callback without recreating the widget.

Helper: `_btn(parent, text, command, accent, small, large, danger)` — factory shortcut.

#### `FileState`
Holds all per-file data:
- `path`, `data` (raw JSON dict), `entries` (list of entry dicts)
- `modified: set` — indices edited this session
- `dirty: bool` — unsaved changes exist
- `current_index` — which entry is loaded in the editor
- `filtered_indices: list` — current search-filtered subset of entry indices
- `search_query: str` — persisted per file so switching files restores search

#### `RubyDialog(tk.Toplevel)`
Modal dialog for inserting or editing a `[main/RUBY/ruby_text]` tag.
- `mode="insert"` or `"edit"`
- After `.destroy()`, check `.result`: `("apply", main, ruby)`, `("del_ruby", main, "")`,
  `("del_all", "", "")`, or `None` if cancelled.

#### `ShortcutsDialog(tk.Toplevel)`
Modal dialog to view and reassign keyboard shortcuts.
- Shows a table: Action | Current binding display | Change button | Clear button
- "Change" starts recording mode; next keypress (if valid) is captured via `<KeyPress>`.
- Validation rules: must include Ctrl or Alt; Ctrl+Alt together is forbidden.
- Conflict detection: asks user before reassigning a binding already in use.
- Calls `on_save(dict)` callback on Save; caller then calls `_bind_shortcuts()`.

#### `FontDialog(tk.Toplevel)`
Modal dialog to pick editor font family and size.
- Search box = filter only (starts empty, never written by listbox clicks).
- Listbox click sets `self._selected_family` and the "Selected: …" label; does NOT touch search box.
- `self._selected_family` is initialised to the currently active font on open.
- Apply validates font exists on system; falls back to `"Segoe UI"` silently.

#### `TextEditorApp(tk.Tk)`
Main application window. 3-column PanedWindow layout.

Key instance attributes:
| Attribute | Type | Purpose |
|---|---|---|
| `open_files` | `list[FileState]` | All currently open files |
| `active_file` | `FileState \| None` | File currently shown in editor |
| `shortcuts` | `dict` | action_id → current binding string |
| `_active_bindings` | `dict` | binding_str → funcid (for precise unbind) |
| `editor_font_family` | `str` | Current editor font family |
| `editor_font_size` | `int` | Current editor font size |
| `hide_br_var` | `BooleanVar` | Toggle BR tags in preview |
| `hide_vo_var` | `BooleanVar` | Toggle VO tags in preview |
| `_suppress` | `bool` | Blocks re-entrant trace callbacks |
| `_hl_job` | `int \| None` | Pending `after()` job id for debounced highlight |
| `_prev_frames` | `list` | Tracked canvas/frame children in preview (for cleanup) |
| `_pv_main_font` | `tkfont.Font` | Cached preview main font object |
| `_pv_ruby_font` | `tkfont.Font` | Cached preview ruby font object |
| `_pv_ruby_zone` | `int` | Pixel height reserved for ruby annotation above main text |
| `_pv_main_asc` | `int` | Main font ascent in pixels |
| `_pv_main_desc` | `int` | Main font descent in pixels |

Key widget attributes:
| Attribute | Widget | Notes |
|---|---|---|
| `file_listbox` | `tk.Listbox` | Left panel: open files |
| `entry_listbox` | `tk.Listbox` | Middle panel: filtered entry list |
| `orig_text` | `tk.Text` | Read-only ORIGINAL_TEXT box |
| `edit_text` | `tk.Text` | Editable TEXT box |
| `preview_canvas` | `tk.Canvas` | Preview panel — fully manual drawing |
| `search_var` | `StringVar` | Entry search field |
| `status_var` | `StringVar` | Bottom status bar text |

---

## Preview renderer (`_render_preview`)

**Important:** the preview uses a raw `tk.Canvas`, NOT a `tk.Text` with embedded windows.
This was a deliberate decision after many failed attempts with `tk.Text` + `window_create(align=…)`.

The `align` parameter on `window_create` is fundamentally unreliable for matching text baselines:
- `align="baseline"` sets `ascent=canvas_h, descent=0` → canvas bottom is at line baseline,
  but main text drawn with `anchor="s"` ends up above the baseline.
- `align="bottom"` sets `ascent=0, descent=canvas_h` → canvas hangs below baseline entirely.
- Neither mode can simultaneously satisfy "ruby annotation above" and "main text on baseline".

**The canvas approach:** every element (plain text, ruby, BR, VO) is drawn with explicit pixel
coordinates. The baseline `bl` is a y-coordinate tracked as the renderer walks tokens left-to-right,
wrapping to a new line when needed. Plain text and ruby main text both draw with `anchor="nw"` at
`y = bl - main_ascent`, so they are on exactly the same baseline — no Tk metric guesswork.

Line height = `ruby_zone + main_ascent + main_descent + GAP (6px)`.

Word-wrap is manual: for each plain-text token, `_pv_main_font.measure(token)` is compared against
`usable_w = canvas.winfo_width() - 2 * PAD_X`. Wrapping calls `newline()` which resets `x = PAD_X`
and increments `bl` by one line height.

Ruby click detection uses a transparent rectangle drawn over the ruby slot, bound with `tag_bind`.

---

## Settings persistence (`editor.py.cfg`)

Written on save, shortcut change, font change, and window close. Sections:

```ini
[window]
geometry = 1380x780+100+100

[sash]
main0 = 260    ; x position of sash between file panel and entry panel
main1 = 560    ; x position of sash between entry panel and editor
edit0 = 180    ; y position of sash between ORIGINAL and TEXT boxes
edit1 = 420    ; y position of sash between TEXT box and PREVIEW

[shortcuts]
open = <Control-o>
save = <Control-s>
save_all = <Control-Shift-s>
...

[preview]
hide_br = False
hide_vo = False

[font]
family = Segoe UI
size = 12
```

---

## Shortcuts system

- `DEFAULT_SHORTCUTS`: module-level dict, the baseline.
- `self.shortcuts`: live dict, may differ from defaults (user edits + saved cfg).
- `_bind_shortcuts()`: unbinds everything in `_active_bindings` (using stored funcids for
  precision), then re-binds all non-empty shortcuts. Also calls `_update_menu_labels()`.
- `_active_bindings`: `dict[str, str]` mapping binding string → funcid returned by `self.bind()`.
  Storing funcids is essential — `self.unbind(seq)` without a funcid silently fails in many
  Tk versions, leaving old handlers alive.
- `_update_menu_labels()`: uses `entryconfigure(i, accelerator=hint)` on the File menu
  so hints appear right-aligned. Do NOT concatenate hints into the label text.
- Validation in `ShortcutsDialog._on_keypress()`:
  - Must include Ctrl or Alt.
  - Ctrl+Alt together is forbidden (AltGr conflict).
  - Bare modifier presses are filtered by `_MODIFIER_SYMS` + `_event_to_tk` returning `""`.

---

## Entry list colouring

- **Orange** (`MODIFIED`): index is in `fs.modified` — edited in this session.
- **Green** (`TRANSLATED`): `entry["TEXT"] != entry["ORIGINAL_TEXT"]` — pre-translated.
- **White** (`FG`): untouched.

Colour is computed in `_entry_color(fs, i)`. Modified takes priority over translated.

---

## Known design quirks / gotchas

1. `self._suppress = True/False` guards are needed around programmatic writes to
   `search_var` and `edit_text` to prevent trace callbacks from firing re-entrantly.

2. `edit_text.edit_reset()` + `edit_text.edit_modified(False)` must both be called when
   loading a new entry, otherwise undo history bleeds between entries.

3. `_clear_editor()` is called when no file is open. It must also reset `search_var`
   (suppressed) to avoid a stale filter showing on next file open.

4. `_get_edit_text()` strips exactly one trailing newline that Tk's Text widget always
   appends, but preserves intentional double-newlines.

5. `FontDialog._selected_family` is always set (initialised to the active font) so
   `_apply()` always has a valid font even if the user never clicked the listbox.
   The search box is a filter only — it is never pre-filled and listbox clicks
   never write to it.

6. Preview canvas `<Configure>` fires `_schedule_hl()` so the layout re-renders on
   panel resize (word-wrap depends on canvas width).

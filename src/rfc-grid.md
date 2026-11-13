# Spreadsheet grid for ui-kit

| | |
|---|---|
| Status | Draft |
| Reviewer | Nate |
| Date | 2026-11-12 |
| Scope | Design only: nothing committed, no PR, no issue |

**The proposal:** add a `ui_kit::Grid` built on top of `table_custom`. It gives every table in the app one cell cursor, one editing model, TSV copy and paste, fill, and a single add-row and add-column control. The app keeps its data and its commands. The grid only reports what the user did, as a list of edits.

## 1. Problem

Every table re-implements editing, cell by cell: its own focus checks, its own Enter handling (`enter_for` in Parameters, `consume_key(Enter)` in Configurations), its own draft maps and its own add-row lifecycle. Each one behaves a little differently, and none of them behaves like a spreadsheet.

Evidence from my own use: I couldn't find *Add parameter* (a dashed line at the bottom of one section) or the tiny **+** header that adds a configuration column. Tab didn't select the next cell's text. Enter committed but didn't move down. Two PRs in flight fix the first and the last two, app-wide. This design takes them as the baseline and makes that behaviour part of the grid, so the next table gets it for free.

### What exists today

`crates/ui-kit/src/table.rs` (555 lines) already does a lot of this well, and the grid keeps all of it:

- **Layout.** `Column` has a weight, `flex`, `min_content_width`, `icon_only` and a dragged `manual_width`. Pixel-snapped cumulative edges mean no stray scrollbar.
- **Resize.** `headers_resizable` uses drag dividers, double-click resets, and widths persist in egui memory under a key that several sections can share.
- **Sort.** `Sort` steps Listed, Ascending, Descending, sorts in natural order, and `sort_header` draws the heading. The caller does the sorting.
- **Culling.** `row_visible` skips building the cells of off-screen rows, but still allocates every row (O(rows) per frame). `force` keeps rows that hold a draft alive.
- **Empty state**, `add_prompt` (a dashed last row), `plus_header`, `removable_header`, `bin_button`, and `reveal_row` to scroll a new row into view.
- **Automation.** It publishes registry rects `dialog:table:{id}:{row}:cell:{column}`, `dialog:column:{id}:{label}` and `dialog:table:{id}:add`.
- **No filter.** Filtering lives in the callers (the Parameters search field, the column menu's filter).

**Theme.** ui-kit widgets read `solidcraft_theme::Tokens::get()`, a thread-local the host sets every frame. Scoped overrides come through `scoped_tokens` and `scoped_panel_tokens`. The grid uses tokens that already exist: `selected_bg` and `selected_ink` for the range, `accent` for the cursor ring, `hover`, `control_border`, `dialog_rule`, `error` and `form_dim`, plus `control_height`, `control_radius` and the `space_*` sizes. There is no focus-ring token today. See question 3.

## 2. Goals

```text
 +---+----------+------+--------------+-------------------+---------+----------+
 |   | Name ^   | Unit | Expression   | Value             | Comment | + Column |
 +---+----------+------+--------------+-------------------+---------+----------+
 | 1 | width    | mm   | 40 mm        | 40 mm  (read-only)|         |          |
 | 2 | height   | mm   |[width / 2   ]|[20 mm            ]|         |          |
 | 3 | depth    | mm   |[[height + 5]]|[25 mm            ]| cursor  |          |
 | 4 | hole_d   | mm   | depth *      | ! expected a value|         |          |
 +---+----------+------+--------------+-------------------+---------+----------+
 |   | + Add parameter                                                         |
 +---+-------------------------------------------------------------------------+
```

Mockup. `[ ]` marks the range (Expression and Value over rows 2 to 3, shaded). `[[ ]]` is the cursor on Expression, row 3 (accent ring). Value is read-only (dim). Row 4 fails its check and shows the message in place. The add row and the add-column header use the button style from the in-flight PR, and they are the only add controls.

| Area | Behaviour |
|---|---|
| Cursor | Arrows move the cursor when no cell is being edited. Tab and Shift+Tab move across a row and wrap to the next row. Enter and Shift+Enter move down and up the same column. Home and End go to the row's ends; Ctrl+Home and Ctrl+End go to the table's ends. The cursor is always visible (an accent ring), and the grid scrolls to keep it in view. |
| Editing | Typing a character starts an edit that replaces the cell. F2, a double-click or Enter on a cell starts an edit with all the text selected. Escape cancels. Enter, Tab and blur commit, then move as above. A refused commit keeps the edit open, with the reason under the cell. |
| Selection | Shift+arrows, click-drag and Shift+click select a range. Ctrl+A selects the whole table. Clicking a row number selects the row; clicking a header selects the column (sorting stays on its own glyph). |
| Clipboard | Copy writes TSV: tabs between cells, CRLF between rows, and quotes only where a cell holds a tab, newline or quote, which is how Excel and Google Sheets write it. Paste reads the same format. A block bigger than the space below the cursor adds rows when the source allows it; otherwise it is cut off and the status line says how much. A single value pasted over a range fills the whole range. Clipboard access goes through egui (`ctx.copy_text` and `Event::Paste`), so it works in wasm without extra permissions. |
| Bulk edits | Ctrl+D fills the top row of the selection down. Typing and committing with Ctrl+Enter writes the value to every selected cell. Delete and Backspace clear the selected cells; when the source refuses a cell, the cell keeps its value and shows why. |
| Undo | Every gesture (one commit, one paste, one fill, one clear) becomes one app command and one undo step. The grid never owns undo. |
| Add | There is one add-row control (the last row) and one optional add-column header. Typing in the empty add row creates the row with that cell's value. Nothing else in a grid adds rows. |
| Reuse | Column layout, resize, sort headers, the empty state and reveal come straight from `table.rs`. Filtering is a predicate the caller passes. The grid draws the matching rows and maps the cursor through it. |
| Scale | Real virtualization: only the visible rows are laid out, with one spacer above and one below. The drawing tables allow 10,000 rows and the configuration table 2,000. |
| Names | Every cell gets an accessible name, "{column}, {row}", and a stable registry handle, so the scenario tests keep clicking widgets by name. |
| Platform | The grid is wasm-safe: no threads and no OS clipboard crate. |

## 3. Non-goals

- No formulas or cell references in the grid. Expressions stay app-level (the parameter evaluator and its completion). The grid moves strings around.
- No charts, conditional formatting, merged cells or frozen panes beyond the first column.
- Not a general spreadsheet app, and no new file format.
- No change to the commands' meaning. The paste and fill gestures need batch forms of a few commands (section 4).

## 4. API sketch

```rust
// ui-kit: knows nothing about documents or commands.
pub enum CellKind<'a> {
    Text,
    Number { unit: &'a str },                 // right-aligned; edit text keeps the unit ("20 mm")
    Expression,                               // the app's completion via install_expressions
    Choice { options: &'a [&'a str], searchable: bool },
    Check,
    ReadOnly,
    Custom,                                   // the app draws it (Activate button, library picker)
}

pub trait GridSource {
    fn row_count(&self) -> usize;
    fn row_key(&self, row: usize) -> Cow<str>;          // stable across sorts and filters
    fn columns(&self) -> &[GridColumn<'_>];             // wraps table::Column + key + kind
    fn text(&self, row: usize, col: usize) -> Cow<str>;  // shown and copied
    fn kind(&self, row: usize, col: usize) -> CellKind<'_> { self.columns()[col].kind }
    fn message(&self, row: usize, col: usize) -> Option<(Cow<str>, bool /*error*/)> { None }
    /// Live check while typing; Err shows under the cell and blocks commit.
    fn check(&mut self, row: usize, col: usize, text: &str) -> Result<Option<String>, String> { Ok(None) }
    fn can_add_rows(&self) -> bool { false }
    fn can_add_column(&self) -> bool { false }
    fn custom(&mut self, ui: &mut Ui, row: usize, col: usize) {}  // for CellKind::Custom
}

pub enum Edit { Set { row: RowRef, col: usize, text: String }, Clear { row: RowRef, col: usize } }
pub enum RowRef { Existing(String /*row_key*/), New(usize /*nth new row*/) }

pub enum GridOutput {
    None,
    Edits(Vec<Edit>),            // one gesture: commit, paste, fill, clear
    AddColumn,                   // header button pressed; app opens its own menu
    Activated { row: String },   // double-click or Enter on a ReadOnly row (BOM pick)
}

let out = ui_kit::Grid::new("parameters_user")
    .sort(&mut sort)                       // existing ui_kit::Sort
    .filter(|row| matches(row))            // caller predicate
    .resizable(columns_id())               // existing persisted widths
    .add_row_label("Add parameter")
    .show(ui, &mut source);

// The app turns one gesture into one command:
if let GridOutput::Edits(edits) = out {
    let changes = to_parameter_changes(&edits);
    match app.run("parameters.change", json!({"changes": changes})) {
        Ok(_) => {}
        Err(e) => ui_kit::Grid::refuse(ui.ctx(), "parameters_user", e),  // reopens the edit
    }
}
```

- **View state only.** The cursor, the anchor and range, the edit buffer and its `CCursorRange`, the scroll offset and the pending refusal live in egui temp memory under the grid's id. Column widths and sort stay persisted, as now. A row key that disappears (rows deleted, the table filtered) moves the cursor to the nearest row that is left.
- **`set_cell` and refusal.** The source never mutates anything. Edits come back as one list per gesture. The app runs one command and, on `Err`, calls `Grid::refuse`, which keeps the buffer and shows the message. That keeps edits that go through a command (Parameters, Configurations) honest and makes it trivial for in-memory tables.
- **One command per gesture** needs batch forms the engine mostly has:
  - `parameters.change` already takes `changes` (one rebuild, one undo).
  - `config.row` takes many columns of one row. A paste over several rows needs `config.rows`, or a generic `edit.group` built on the same `Session::scratch` and `accept_computed` pair the plugin transaction already uses.
  - `component.metadata.set` runs once per component today (BOM Apply is N undo steps). It needs a list form.
  - Drawing tables use `drawing.table.edit` overrides and `drawing.revision.edit`.

  I'd prefer one `edit.group` over a batch flag on every command. It also gives scripted and MCP callers a single call for a multi-cell change.

## 5. Migration order and estimates

| Step | What changes | Days |
|---|---|---|
| **0. Grid PR** (ui-kit) | Cursor, edit, range, TSV, fill, clear, virtual rows, a11y names, gallery entry, ui-kit tests (keys, paste parsing, Excel and Sheets fixtures, 10k-row frame time). No app change. | 5 |
| **1. Parameters** | Three sections become three grids that share widths. Favourite and move become a Custom column. Rename becomes Name-cell edit. Expression cells keep completion and the live preview. Held edits (apply-live off) map onto the edit list. The add-row draft logic (about 150 lines) goes away. | 2.5 |
| **2. Configuration Table** | Per-kind controls become Choice, Custom (library picker) and Text. The **+** header becomes the grid's add-column header that opens the existing filtered column menu. The new-row draft becomes the add row ("copy of active" stays). Needs `config.rows` or `edit.group`. The rules editor below it stays as is. | 2.5 |
| **3. Sheet Metal Rules** | Today a rule dropdown plus a one-rule form. It becomes a grid of rules x fields (Thickness, K, Radius, reliefs, gaps, shape, angle display), with the personal library kept as a side action. This is a UX change, so it gets a screenshot for me or Nate. | 2 |
| **4. BOM** | The read-only `egui::Grid` plus the pick-a-row details form become a grid whose part-number, description and other metadata cells are editable in place. Child rows are ReadOnly. Needs the list form of `component.metadata.set`. | 2 |
| **5. Drawing tables** | On the `drawings` branch, after #3874 and #13058 land: the general table's typed cells, revision rows, and parts-list overrides. Hole, bend, family and punch tables are read-only grids with copy. | 2.5 |

**Total: about 16.5 days**, of which the shared grid is 5 and each migration 2 to 2.5. Steps 1 and 2 are the ones I hit myself. Steps 3 to 5 can run in parallel once step 1 has proven the API.

## 6. Risks and open questions for Nate

1. **Focus in egui.** The grid wants one focus target while navigating and a `TextEdit` while editing, with Tab, arrows and Escape kept away from egui's own focus traversal (`set_focus_lock_filter`). Inside panel_host, does a focused grid conflict with `register_content_focus` or with dock keyboard handling? Who wins Escape: the grid's cancel or the window's close? (Parameters already swallows Escape before drawing for this reason.)
2. **IME.** "Type to edit" has to start the editor on the first `Event::Text` or `Event::Ime` without losing the first character or a preedit composition (CJK, dead keys on macOS). Should IME users start with F2 or Enter, or do we buffer the first event into the new editor?
3. **Style.** How should the cursor ring, range fill and add controls look in your redesign? Is a new token wanted (`focus_ring`, `range_bg`), or do `accent` and `selected_bg` stand? Should there be row numbers?
4. **panel_host and docking.** Grids live in dialogs today but will live in docked panels (BOM already does). Should the grid publish its own min and preferred size to the host? How should keyboard focus survive a pop-out window (#15210)?
5. **Dropdown cells.** Should Choice cells reuse `searchable_dropdown`, opened by Enter, F2 or Alt+Down, with type-ahead going into its search box? The library picker in Configurations is a near relative. Should it become one widget?
6. **Performance at 1000+ rows.** True virtualization means uniform row height (`table_row_height`). Are you OK with forbidding wrapped cells, and with the vertical scroll moving into the grid instead of the card (today `reveal_row` asks the card's scroller)?
7. **Scenario names.** The proposal: the accessible name is "{column}, {row key}", and the handle is `grid:{id}:{row}:{col}`. During migration, the old `param:{name}`, `config:cell:{row}:{col}` and `dialog:table:...:cell:...` stay as aliases, so the existing scenarios pass unchanged. Acceptable, or rename them in one go?

Other risks: the paste-adds-rows rule for sources with required cells (a new configuration needs a name first, so the first column of a pasted block becomes the name), and wide expression editors that open over neighbouring cells (`wide_cell_editor`), which have to stay inside the grid's clip.

## 7. Effort

Grid PR: 5 days. Migrations: Parameters 2.5, Configuration Table 2.5, Sheet Metal Rules 2, BOM 2, drawing tables 2.5. Engine batch command (`edit.group`, or `config.rows` plus a list form of metadata): 1 day, landing before step 2. **About 17.5 days in all.** Each step is one PR. Steps 3 and 4 change visible UX and go to me or Nate with 1280x720 screenshots before merge.

## Appendix: table inventory

| Table | Built on | Editing | Keys | Add row / column | Undo |
|---|---|---|---|---|---|
| Parameters (`params_dialog.rs`) | `table_custom`, 3 sections (favourites, user, per-feature) with shared resizable widths; Name sorts; search field filters | Always-on cell editors for Expression (completion, live value, optional preview), Comment and Unit dropdown; double-click Name to rename; held edits when apply-live is off | Custom `enter_for` (fresh Enter only); Esc cancels before drawing; Enter swallowed dialog-wide | Dashed `add_prompt` in the user section only, opens a draft row; no columns | One command per cell; held edits go as one `parameters.change {changes}` |
| Configuration Table (`configurations_ui.rs`) | `table_custom`, not resizable, no sort; cells cached per revision | Per-kind: text, or a library picker for material and appearance; Activate and Apply buttons in the first column; removable headers delete columns | `consume_key(Enter)` commits; no movement | `add_prompt` (copy of the active row, draft row cancelled by a click outside); tiny `plus_header` menu with a filter adds columns | One `config.row` per cell |
| Sheet Metal Rules (`dialogs_sheet.rs`, `rule_editor.rs`, `personal_sheet_rules.rs`) | Not a table: a rule dropdown and a one-rule form (`rule_editor!` macro); personal library dropdown | Form fields; nothing applies until OK | Dialog Enter = OK | "New rule..." option in the dropdown; Delete rule button | One `sheet.manage_rules` on OK |
| BOM (`bom_panel.rs`) | Raw `egui::Grid`, striped, own widths; child rows dimmed | Read-only grid; pick a row, then edit metadata in a details form below | None beyond focus | None | Apply runs `component.metadata.set` once per component (N steps) |
| Drawing tables (`drawings` branch) | Records plus paper rendering only (parts list, revision; hole, general, family, bend and punch in draft #13058); no editing UI yet | Through commands: `drawing.table.edit` (columns, filter, overrides), `drawing.revision.*` | n/a | Commands only | One command each |
| Bend table (main) | Export (`sheet.bend_table`) plus read-only form rows in Flat Pattern | None | n/a | n/a | n/a |
| Details findings (`ui-kit/details.rs`) | `table_custom`, two flex columns | Read-only | n/a | n/a | n/a |

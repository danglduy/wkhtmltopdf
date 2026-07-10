# `RenderTableSection.cpp` — A Complete, Illustrated Guide

> **What this is.** A line-by-line, beginner-friendly explanation of
> `qt/src/3rdparty/webkit/Source/WebCore/rendering/RenderTableSection.cpp` — the
> file inside wkhtmltopdf's bundled WebKit that lays out and paints the rows of an
> HTML table (`<thead>` / `<tbody>` / `<tfoot>`). It explains the **original** code
> *and* the **four pagination fixes** we added, method by method.
>
> **Who it's for.** Someone who has *never read C++*. Every concept is spelled out.
> If you already know C++, skip [§0](#0-a-five-minute-c-primer).
>
> Line numbers refer to the file **after** our fixes were applied (1345 lines total).

---

## Table of contents

0. [A five-minute C++ primer](#0-a-five-minute-c-primer)
1. [The big picture: where this file lives](#1-the-big-picture-where-this-file-lives)
2. [The data model: how a table is stored in memory](#2-the-data-model-how-a-table-is-stored-in-memory)
3. [Building the grid (structure methods)](#3-building-the-grid-structure-methods)
4. [Measuring & laying out (the heart of the file)](#4-measuring--laying-out-the-heart-of-the-file)
5. [`layoutRows` — the big one, step by step](#5-layoutrows--the-big-one-step-by-step)
6. [The four pagination fixes, in depth](#6-the-four-pagination-fixes-in-depth)
7. [Borders, baselines, painting, hit-testing](#7-borders-baselines-painting-hit-testing)
8. [Glossary](#8-glossary)

---

## 0. A five-minute C++ primer

You only need a handful of ideas to read this file.

| Thing you'll see | What it means in plain words |
|---|---|
| `int x = 5;` | A **variable** named `x` holding a whole number (`int` = integer). |
| `bool ok = true;` | A true/false value (`bool` = boolean). |
| `// ...` | A **comment** — a note for humans, ignored by the computer. |
| `RenderTableCell* cell` | `cell` is a **pointer**: it doesn't hold a cell, it holds the *address* of one (think "a sticky note saying where the cell lives"). The `*` means "pointer to". A pointer can be `0` (a.k.a. *null*) meaning "points at nothing". |
| `cell->height()` | "Follow the pointer `cell`, then call its `height()` function." The `->` is **"reach through a pointer."** |
| `obj.height()` | Same idea but `obj` is the object itself, not a pointer. The `.` is **"reach into an object."** |
| `RenderTableSection::layoutRows` | "The `layoutRows` function that belongs to the `RenderTableSection` class." `::` means **"belongs to."** A **class** is a blueprint for an object; a **method** is a function attached to that blueprint. |
| `int& w` | The `&` makes `w` a **reference** — another name for an existing variable. Changing `w` changes the original. (Pointers and references both let one piece of code affect another's data.) |
| `Vector<int> v;` | A **list/array** that can grow, holding `int`s. `v[3]` is the 4th element (counting starts at **0**). `v.size()` is how many elements it has. |
| `for (int r = 0; r < n; r++) { ... }` | A **loop**: start with `r = 0`, keep going while `r < n`, add 1 to `r` each time (`r++`). Runs the `{ ... }` block once per value of `r`. |
| `if (cond) { A } else { B }` | Do `A` if `cond` is true, otherwise `B`. |
| `continue;` | "Skip the rest of this loop iteration and go to the next one." |
| `return value;` | "This function is done; hand `value` back to whoever called it." |
| `max(a, b)` / `min(a, b)` | The larger / smaller of two numbers. |
| `x += 5;` | Shorthand for `x = x + 5;` (add 5 to `x`). Likewise `x++` means `x = x + 1`. |
| `ASSERT(cond);` | A **debug-only sanity check**: "I believe `cond` is true here; if it isn't during testing, stop and complain." It does nothing in the shipping build. |
| `#ifndef NDEBUG ... #endif` | "Only compile this part in debug builds." Used to wrap `ASSERT`s and checks. |

That's genuinely enough. Everything else we'll explain as it appears.

---

## 1. The big picture: where this file lives

### 1.1 From HTML to pixels

When wkhtmltopdf turns HTML into a PDF, the bundled WebKit engine runs this pipeline:

```
   HTML text
      │  parse
      ▼
   DOM tree            (the <table>, <tr>, <td> elements as objects)
      │  attach styles
      ▼
   RENDER tree         (one "renderer" object per visible box)
      │  LAYOUT  ← positions & sizes every box   ← THIS FILE runs here
      ▼
   PAINT               ← draws every box onto the page ← THIS FILE runs here too
      ▼
   PDF pages
```

This file is one renderer in that **render tree**, and it participates in the two
boxed steps: **layout** (decide where everything goes) and **paint** (draw it).

### 1.2 The table renderers

An HTML table becomes a little family of renderer objects:

```
   <table>            →  RenderTable          (the whole table)
     <thead>          →  RenderTableSection   ← THIS FILE  (the "header group")
     <tbody>          →  RenderTableSection   ← THIS FILE  (the "body group")
     <tfoot>          →  RenderTableSection   ← THIS FILE  (the "footer group")
       <tr>           →  RenderTableRow       (one row)
         <td> / <th>  →  RenderTableCell      (one cell)
```

So a **`RenderTableSection`** is a *group of rows* — exactly one of `<thead>`,
`<tbody>`, or `<tfoot>`. Its job:

- **Own a grid** of the cells in its rows (so it can answer "what's at row 3,
  column 2?").
- **Compute each row's height** and **each row's vertical position**.
- **Place every cell** at the right spot and tell each cell to lay out its contents.
- **Paint** the cells (and their backgrounds/borders) when it's time to draw.

The `RenderTable` (parent) owns the **column** widths and positions; the section
owns the **row** heights and positions. They cooperate.

> 💡 **Anonymous tables.** CSS `display: table` / `display: table-cell` (used by the
> document that triggered our 4th fix) makes WebKit create the *same* renderer
> objects — `RenderTable`, `RenderTableSection`, `RenderTableCell` — even though
> there is no literal `<table>` tag. So this file runs for those too. That's why a
> "nested table" can appear *inside* a `<td>`.

### 1.3 "Logical" coordinates (a vocabulary you must know)

CSS can lay text out left-to-right, right-to-left, or even top-to-bottom (vertical
writing modes). To avoid writing the same code four times, WebKit uses **logical**
directions instead of physical ones:

| Logical term | In normal English (horizontal, left-to-right) text |
|---|---|
| **logical height** | the **height** (top-to-bottom size) |
| **logical width** | the **width** (left-to-right size) |
| **before** | the **top** edge |
| **after** | the **bottom** edge |
| **start** | the **left** edge |
| **end** | the **right** edge |
| **block direction** | **vertical** (the direction paragraphs stack) |
| **inline direction** | **horizontal** (the direction words run) |

So `logicalHeight()` ≈ "height", `borderBefore()` ≈ "top border", `paddingAfter()` ≈
"bottom padding". Rows stack in the **block** direction. Throughout this file, "row
positions" are measured in the block direction. For an ordinary English-language
PDF you can safely read *logical height* as *height* and *before/after* as
*top/bottom*. The code keeps the abstraction so vertical-writing tables work too.

### 1.4 Pages and pagination (why any of this is hard)

On screen a table can be any height. In a **PDF** the content is sliced into fixed
**pages**. WebKit models this with a per-layout object called **`LayoutState`**
that knows the **page height** (`pageLogicalHeight`) and where page boundaries fall.
When `pageLogicalHeight` is non-zero, we are "paginating".

Two wkhtmltopdf-specific behaviours matter for this file:

1. **Repeated headers.** If a table has a `<thead>` and spans several pages,
   wkhtmltopdf re-draws the header at the **top of every page**. (That drawing
   happens in a sibling file, `RenderTable.cpp`. This file's job is to *reserve
   space* so the body content doesn't collide with that re-drawn header.)
2. **Page breaks inside content.** A paragraph taller than a page must be split.
   WebKit pushes lines down so they don't straddle a page boundary (a "pagination
   strut"). Our fixes make those struts aware of the repeated-header band.

All four of our fixes live in the **`layoutRows`** method and are wrapped in
`if (view()->layoutState()->pageLogicalHeight()) { ... }` — i.e. **they only run
when paginating** (making a PDF). For on-screen layout they do nothing.

---

## 2. The data model: how a table is stored in memory

Before any method makes sense, you must picture the **grid**. Everything in this
file revolves around three member variables and three little structs, all declared
in the header file `RenderTableSection.h`.

### 2.1 The member variables (the section's "memory")

From `RenderTableSection.h` (lines 149–168):

```cpp
Vector<RowStruct> m_grid;   // one entry per row: the row's cells + bookkeeping
Vector<int>       m_rowPos; // the vertical position of each row boundary
int               m_gridRows;   // how many rows the grid has

int m_cCol;   // "current column" — a cursor used while filling the grid
int m_cRow;   // "current row"    — a cursor used while filling the grid

int  m_outerBorderStart, m_outerBorderEnd, m_outerBorderBefore, m_outerBorderAfter;
bool m_needsCellRecalc;       // "the grid is stale, rebuild it before use"
bool m_hasOverflowingCell;    // "some cell's content spills outside its box"
bool m_hasMultipleCellLevels; // "some cells overlap (rare); use the slow paint path"
```

(The `m_` prefix is just a naming convention meaning "**m**ember variable" — a
piece of the object's own long-lived state, as opposed to a temporary local
variable.)

### 2.2 The three structs

**`CellStruct`** (header lines 56–74) — *what sits at one grid square*:

```cpp
struct CellStruct {
    Vector<RenderTableCell*, 1> cells;  // pointers to the cell(s) at this square
    bool inColSpan;                     // true if this square is the *continuation*
                                        //   of a cell that started to its left

    RenderTableCell* primaryCell();     // the "real" cell occupying this square
    bool hasCells() const { return cells.size() > 0; }
};
```

- Usually `cells` holds **one** pointer. It's a list because in rare cases (overlapping
  cells via odd rowspan/colspan) two cells can occupy the same square — then
  `m_hasMultipleCellLevels` is set and painting takes a slower, careful path.
- `primaryCell()` returns the last (topmost) cell — the one that "wins" this square.
- `inColSpan` is the key to **column spans**: if a `<td colspan="3">` starts at
  column 0, then squares (row, 0), (row, 1), (row, 2) all point at the same cell, but
  only column 0 is the "primary"; columns 1 and 2 have `inColSpan = true`.

**`Row`** (header line 76) is just a shorthand:

```cpp
typedef Vector<CellStruct> Row;   // "Row" means "a list of CellStructs" = one grid row
```

**`RowStruct`** (header lines 78–83) — *everything about one row*:

```cpp
struct RowStruct {
    Row*            row;          // pointer to this row's list of cells (the squares)
    RenderTableRow* rowRenderer;  // the <tr>'s renderer (may be 0 for some grid rows)
    int             baseline;     // where text baselines align in this row
    Length          logicalHeight;// the row's CSS height request (auto / 50px / 20% …)
};
```

### 2.3 Putting it together: the grid

`m_grid` is a list of `RowStruct`. Each `RowStruct.row` points to a list of
`CellStruct` (one per column). So the whole thing is a **2-D grid** you index with
the helper `cellAt(row, col)` (header line 85):

```cpp
CellStruct& cellAt(int row, int col) { return (*m_grid[row].row)[col]; }
// read as: take row `row`, follow its `row` pointer to the list, take element `col`.
```

Picture a table whose first row has a cell that spans 2 rows, and whose `<thead>`
"Requirement" cell spans 3 rows (the real document that drove our fixes):

```
                 col 0                 col 1
              ┌───────────────────┬────────────────────────┐
   row 0      │ "1. Req 01"       │ "a. The ocean ..."     │
              │ (a <th> that      ├────────────────────────┤
   row 1      │  spans rows 0-2)  │ "b. The ocean ..."     │
              │                   ├────────────────────────┤
   row 2      │                   │ "c. ..."               │
              └───────────────────┴────────────────────────┘

   m_grid as stored:
     m_grid[0].row = [ CellStruct{cells:[TH], inColSpan:false}, CellStruct{cells:[a]} ]
     m_grid[1].row = [ CellStruct{cells:[TH], inColSpan:false}, CellStruct{cells:[b]} ]
     m_grid[2].row = [ CellStruct{cells:[TH], inColSpan:false}, CellStruct{cells:[c]} ]
                          ▲ same TH pointer appears in all 3 rows (rowspan=3)
```

Notice the **same** `TH` pointer is stored in the squares of all three rows it spans.
That is how a rowspanning cell is represented: *one cell, recorded in every square it
covers.* (The "primary" location is row 0; later code uses `cell->row()` to ask "which
row did you really start in?".)

### 2.4 `m_rowPos` — the running ruler down the side of the table

`m_rowPos` is the single most important array in the file. It has **one more entry
than there are rows** (`m_gridRows + 1`). Entry `r` is the **vertical position of the
top edge of row `r`**; the last entry is the **bottom edge of the whole section**.

```
   m_rowPos[0] ─►  ┌───────────────┐  top of row 0
                   │     row 0     │
   m_rowPos[1] ─►  ├───────────────┤  top of row 1 (= bottom of row 0)
                   │     row 1     │
   m_rowPos[2] ─►  ├───────────────┤  top of row 2
                   │     row 2     │
   m_rowPos[3] ─►  └───────────────┘  bottom of the section (3 rows → index 3)
```

So **`height of row r` = `m_rowPos[r+1] − m_rowPos[r]`** (minus a little spacing).
Almost every layout decision in this file is "compute or adjust `m_rowPos`." Our
continuous-packing fix is literally "make `m_rowPos` reflect reality when a row is
split across a page."

> Keep this picture in your head — the rest of the file is mostly arithmetic on
> `m_rowPos`.

---

## 3. Building the grid (structure methods)

These methods create and maintain the grid from §2. They run when the table is
first built and whenever rows/cells are added or removed.

### 3.1 Constructor & destructor (lines 55–75)

```cpp
RenderTableSection::RenderTableSection(Node* node)
    : RenderBox(node)
    , m_gridRows(0)
    , m_cCol(0)
    , m_cRow(-1)        // ← starts at -1 on purpose (see addChild)
    ...
{
    setInline(false);   // a section is a block, never inline text
}
```

The `: m_gridRows(0), ...` part is an **initializer list** — it just sets the starting
values of the member variables (empty grid, cursors at 0 / −1, no stale flag, etc.).
`m_cRow` starts at **−1** so that the *first* `++m_cRow` (add one) lands on row **0**.

```cpp
RenderTableSection::~RenderTableSection() { clearGrid(); }
```

The `~` version is the **destructor** — runs when the section is destroyed. It calls
`clearGrid()` to free the row lists it allocated (see §3.9).

### 3.2 `styleDidChange` (lines 77–81)

```cpp
void RenderTableSection::styleDidChange(StyleDifference diff, const RenderStyle* oldStyle)
{
    RenderBox::styleDidChange(diff, oldStyle);
    propagateStyleToAnonymousChildren();
}
```

Called when CSS affecting this section changes. It lets the base class react, then
**pushes style down to anonymous children** — the invisible helper rows/cells WebKit
auto-creates (see §3.4). Nothing table-specific; it's plumbing.

### 3.3 `destroy` (lines 83–93)

```cpp
void RenderTableSection::destroy()
{
    RenderTable* recalcTable = table();   // remember our parent table first
    RenderBox::destroy();                 // actually tear ourselves down
    if (recalcTable)
        recalcTable->setNeedsSectionRecalc(); // tell the table "your sections changed"
}
```

When a section is removed, the parent `RenderTable` may still hold raw pointers to
it. So before vanishing, the section asks the table to **recalculate its sections**.
Order matters: it grabs `table()` *before* destroying itself, because after
destruction the parent link is gone.

### 3.4 `addChild` — attaching a row (lines 95–154)

This runs when a `<tr>` (or stray content) is inserted under the section. Two cases:

**Case A — the child is *not* a table row (lines 101–130).** HTML lets people put
junk directly inside a `<tbody>`. CSS says that junk must be wrapped in an anonymous
row → cell. The code tries, in order, to:
1. tuck it into the previous anonymous box if there is one (lines 105–110),
2. otherwise, if we're inside an anonymous cell/row, add it there (lines 112–120),
3. otherwise **manufacture a brand-new anonymous `RenderTableRow`** (lines 122–128),
   give it a synthesized style with `display: table-row`, add that row, then put the
   child inside it.

```cpp
RenderObject* row = new (renderArena()) RenderTableRow(document()); // make a fake <tr>
RefPtr<RenderStyle> newStyle = RenderStyle::create();
newStyle->inheritFrom(style());
newStyle->setDisplay(TABLE_ROW);
row->setStyle(newStyle.release());
addChild(row, beforeChild);  // add the fake row…
row->addChild(child);        // …then the real child goes inside it
return;
```

**Case B — the child *is* a table row (lines 132–153).** This is the normal path:

```cpp
if (beforeChild)
    setNeedsCellRecalc();   // inserting in the middle ⇒ grid is now stale

++m_cRow;                   // advance the row cursor (first call: -1 → 0)
m_cCol = 0;                 // reset the column cursor to the left
if (!ensureRows(m_cRow + 1))// make the grid tall enough to hold this row
    return;

m_grid[m_cRow].rowRenderer = toRenderTableRow(child); // remember the <tr>'s renderer
...
RenderBox::addChild(child, beforeChild);              // actually link it into the tree
```

So `addChild` grows the grid by one row and records the new row's renderer. The
actual *cells* are added later by `addCell` (§3.6), driven by `recalcCells` (§3.8).

### 3.5 `removeChild` & `ensureRows`

**`removeChild`** (lines 156–160) is tiny: mark the grid stale, then let the base
class detach the child.

```cpp
void RenderTableSection::removeChild(RenderObject* oldChild)
{
    setNeedsCellRecalc();          // grid must be rebuilt
    RenderBox::removeChild(oldChild);
}
```

**`ensureRows(numRows)`** (lines 162–183) guarantees the grid has at least `numRows`
rows, allocating new ones as needed:

```cpp
bool RenderTableSection::ensureRows(int numRows)
{
    int nRows = m_gridRows;
    if (numRows > nRows) {
        if (numRows > (int)m_grid.size()) {
            ... overflow safety check ...
            m_grid.grow(numRows);          // make the list bigger
        }
        m_gridRows = numRows;
        int nCols = max(1, table()->numEffCols());
        for (int r = nRows; r < numRows; r++) {   // for each brand-new row:
            m_grid[r].row = new Row(nCols);        //   allocate its column list
            m_grid[r].rowRenderer = 0;             //   no <tr> yet
            m_grid[r].baseline = 0;
            m_grid[r].logicalHeight = Length();    //   "auto" height
        }
    }
    return true;   // false only on an absurd overflow (out of memory territory)
}
```

The interesting bit: every new row is created with `nCols` empty squares, where
`nCols = max(1, table()->numEffCols())` — at least 1, otherwise as many columns as the
table currently knows about. Columns can grow later (see `appendColumn`/`splitColumn`).

### 3.6 `addCell` — the grid-filling workhorse (lines 185–259)

This is where colspan/rowspan are turned into grid squares. Called once per `<td>`/`<th>`.

```cpp
int rSpan = cell->rowSpan();          // how many rows this cell spans
int cSpan = cell->colSpan();          // how many columns this cell spans
Vector<RenderTable::ColumnStruct>& columns = table()->columns();
int nCols = columns.size();
```

**Step 1 — find the next free square (lines 198–199).** Starting at the current
column cursor `m_cCol`, skip rightwards over any square that's already filled or is the
continuation of a span:

```cpp
while (m_cCol < nCols && (cellAt(m_cRow, m_cCol).hasCells() || cellAt(m_cRow, m_cCol).inColSpan))
    m_cCol++;
```

This is what makes the classic "rowspan pushes later cells right" behaviour work: a
cell spanning down from a previous row already occupies this row's square, so the new
cell slides past it.

**Step 2 — record an explicit row height (lines 201–222).** *Only for non-rowspanning
cells* (`rSpan == 1`), if the cell has a CSS height (`height: 50px` or `height: 20%`),
remember the **largest** such request on the row (`m_grid[m_cRow].logicalHeight`).
Rowspan cells are deliberately ignored here ("we ignore height settings on rowspan
cells", line 202) because their height is shared across several rows.

**Step 3 — make sure the grid is tall enough (lines 225–226).** A cell spanning
`rSpan` rows needs rows `m_cRow … m_cRow+rSpan−1` to exist:

```cpp
if (!ensureRows(m_cRow + rSpan))
    return;
```

**Step 4 — stamp the cell into every square it covers (lines 230–256).** This double
loop walks the columns the cell spans (handling the table's column model) and, for
each, the rows it spans:

```cpp
while (cSpan) {
    int currentSpan;
    if (m_cCol >= nCols) {                 // ran past the known columns?
        table()->appendColumn(cSpan);      //   grow the table's columns
        currentSpan = cSpan;
    } else {
        if (cSpan < (int)columns[m_cCol].span)
            table()->splitColumn(m_cCol, cSpan); // split a wide column to fit
        currentSpan = columns[m_cCol].span;
    }
    for (int r = 0; r < rSpan; r++) {       // for each row the cell covers…
        CellStruct& c = cellAt(m_cRow + r, m_cCol);
        c.cells.append(cell);               //   record the cell at this square
        if (c.cells.size() > 1)             //   two cells here? slow paint path
            m_hasMultipleCellLevels = true;
        if (inColSpan)                      //   columns after the first…
            c.inColSpan = true;             //   …are marked as "continuation"
    }
    m_cCol++;
    cSpan -= currentSpan;
    inColSpan = true;                       // everything after the 1st column is a span tail
}
cell->setRow(m_cRow);                       // tell the cell its real (start) row…
cell->setCol(table()->effColToCol(col));    // …and its real start column
```

After this, the cell's pointer lives in every grid square it visually occupies, with
only the top-left square "primary" and the rest flagged `inColSpan` (for the
horizontal direction). That's the representation §2.3 drew.

### 3.7 `setCellLogicalWidths` — push column widths into the cells (lines 261–300)

The **table** decides column widths; this method copies those widths into each cell.

```cpp
Vector<int>& columnPos = table()->columnPositions();  // x of each column boundary
LayoutStateMaintainer statePusher(view());            // (coordinate bookkeeping helper)

for (int i = 0; i < m_gridRows; i++) {
    Row& row = *m_grid[i].row;
    for (int j = 0; j < cols; j++) {
        RenderTableCell* cell = current.primaryCell();
        if (!cell || current.inColSpan) continue;     // skip empties & span tails
        ... find the cell's last column `endCol` (handles colspan) ...
        int w = columnPos[endCol] - columnPos[j] - table()->hBorderSpacing();  // the width
        if (w != cell->logicalWidth()) {              // only if it actually changed:
            cell->setNeedsLayout(true);               //   the cell must re-lay-out
            ... repaint bookkeeping ...
            cell->updateLogicalWidth(w);              //   store the new width
        }
    }
}
```

The width of a cell = (x of the column *after* it) − (x of its own column) − the
spacing between cells. If that differs from what the cell currently thinks, the cell
is flagged to lay out again.

> **`LayoutStateMaintainer`** appears all over this file. It's a small helper that
> *pushes* a `LayoutState` onto a stack when created and *pops* it when done (or when
> you call `.pop()`). The `LayoutState` carries the current coordinate offset and —
> crucially for us — the **page height** and our new **repeated-header reservation**.
> Pushing one for the section means "children laid out now are measured relative to
> the section, and inherit the section's pagination context."

### 3.8 `recalcCells`, `setNeedsCellRecalc`, `clearGrid`

**`recalcCells`** (lines 1184–1210) **rebuilds the entire grid from scratch** by
re-walking the real `<tr>`/`<td>` tree:

```cpp
m_cCol = 0; m_cRow = -1;
clearGrid();                 // throw away the old grid
m_gridRows = 0;
for (each child `row` of this section) {
    if (row->isTableRow()) {
        m_cRow++; m_cCol = 0;
        ensureRows(m_cRow + 1);
        m_grid[m_cRow].rowRenderer = tableRow;
        setRowLogicalHeightToRowStyleLogicalHeightIfNotRelative(&m_grid[m_cRow]);
        for (each child `cell` of the row)
            if (cell->isTableCell())
                addCell(toRenderTableCell(cell), tableRow);  // §3.6 fills the squares
    }
}
m_needsCellRecalc = false;   // grid is fresh again
setNeedsLayout(true);        // …but now we must lay out
```

The helper at the top of the file, **`setRowLogicalHeightToRowStyleLogicalHeightIfNotRelative`**
(lines 47–53), simply copies the `<tr>`'s CSS height into the row, unless it's a
*relative* length (then it's treated as "auto"):

```cpp
row->logicalHeight = row->rowRenderer->style()->logicalHeight();
if (row->logicalHeight.isRelative())
    row->logicalHeight = Length();   // Length() == "auto"
```

**`setNeedsCellRecalc`** (lines 1212–1217) sets the stale flag *and* tells the table:

```cpp
m_needsCellRecalc = true;
if (RenderTable* t = table())
    t->setNeedsSectionRecalc();
```

**`clearGrid`** (lines 1219–1224) frees the per-row column lists allocated in
`ensureRows`:

```cpp
int rows = m_gridRows;
while (rows--)
    delete m_grid[rows].row;   // free each Row* we `new`-ed earlier
```

(`delete` is the manual "give this memory back" — necessary because these lists were
created with `new`. Forgetting it would leak memory; this is normal C++ housekeeping.)

### 3.9 Column helpers: `numColumns`, `appendColumn`, `splitColumn`

**`numColumns`** (lines 1226–1239) scans the grid to find the rightmost occupied
column and returns that + 1 (the column *count*).

**`appendColumn(pos)`** (lines 1241–1245) widens every row's square-list to `pos + 1`
columns (used when a cell spans past the current right edge).

**`splitColumn(pos, first)`** (lines 1247–1267) inserts a new column boundary inside an
existing wide column and fixes up the `inColSpan` flags so spanning cells still line
up. These two are called from `addCell` (§3.6 Step 4) as it discovers the real column
structure. You rarely need to trace them; just know they keep the column grid
consistent with colspans.

---

## 4. Measuring & laying out (the heart of the file)

Layout of a section happens in three calls, in this order, all driven by the parent
`RenderTable`:

```
   table->layout()
      │
      ├─ section->calcRowLogicalHeight()   §4.1  → fills m_rowPos with NATURAL row heights
      ├─ section->layout()                 §4.2  → lays out each <tr> (and its cells once)
      └─ section->layoutRows(...)          §5    → final row/cell positions + PAGINATION + our fixes
```

The distinction between **natural** heights (what content wants, ignoring pages) and
**paginated** heights (what it actually occupies once pages are involved) is the
single most important idea behind our fixes. `calcRowLogicalHeight` computes the
*natural* heights. `layoutRows` is where pages enter the picture.

### 4.1 `calcRowLogicalHeight` — the natural-height pass (lines 302–400)

Goal: fill `m_rowPos` so that `m_rowPos[r+1] − m_rowPos[r]` is row `r`'s natural
height, ignoring page breaks. It returns the section's total natural height.

```cpp
int spacing = table()->vBorderSpacing();   // vertical gap between cells
LayoutStateMaintainer statePusher(view()); // coordinate bookkeeping

m_rowPos.resize(m_gridRows + 1);           // one entry per boundary (see §2.4)
m_rowPos[0] = spacing;                      // the first row starts after the top spacing
```

Then, **row by row**, it computes where the *next* boundary `m_rowPos[r+1]` must be:

```cpp
for (int r = 0; r < m_gridRows; r++) {
    m_rowPos[r + 1] = 0;
    int ch = m_grid[r].logicalHeight.calcMinValue(0);  // the row's own CSS min height
    int pos = m_rowPos[r] + ch + (m_grid[r].rowRenderer ? spacing : 0);
    m_rowPos[r + 1] = max(m_rowPos[r + 1], pos);        // at least that tall

    for (int c = 0; c < totalCols; c++) {               // now consider every cell:
        CellStruct& current = cellAt(r, c);
        cell = current.primaryCell();
        if (!cell || current.inColSpan) continue;       // skip empties & span tails
        if ((cell->row() + cell->rowSpan() - 1) > r) continue; // cell ends below row r ⇒ handle later
        ...
        int adjustedLogicalHeight = cell->logicalHeight()
                                  - (cell->intrinsicPaddingBefore() + cell->intrinsicPaddingAfter());
        ch = max( cell's CSS height , adjustedLogicalHeight );
        pos = m_rowPos[indx] + ch + (spacing);
        m_rowPos[r + 1] = max(m_rowPos[r + 1], pos);    // grow the boundary to fit this cell
        ... track baseline-aligned cells ...
    }
    if (baseline) { ... grow the row so baselines line up ... }
    m_rowPos[r + 1] = max(m_rowPos[r + 1], m_rowPos[r]); // never go backwards
}
return m_rowPos[m_gridRows];                            // total natural height
```

**The key idea**, restated simply: each row boundary is the **maximum** bottom edge
demanded by (a) the row's own CSS height and (b) every cell ending in that row. A
cell ending in row `r` that started at row `indx` (its top, computed from its rowspan
via `indx = max(r − rowSpan + 1, 0)`, line 342) pushes `m_rowPos[r+1]` down to at
least `m_rowPos[indx] + cellHeight`.

Two subtleties worth naming for later:

- **`intrinsicPadding`** is *invisible* padding the engine adds to vertically-align a
  cell's content inside a tall row (e.g. to centre it). The code carefully **subtracts
  it out** here (`adjustedLogicalHeight`, lines 355–357) so that alignment padding from
  a previous layout doesn't inflate the natural height. Our **measurement-pass fix**
  relies on the same idea — it *clears* intrinsic padding before measuring so it reads
  pure content height.
- These natural heights are computed with **no knowledge of pages**. That's exactly
  why, once pages split a tall cell, `m_rowPos` is "wrong" and our fix must correct it.

After this method, `m_rowPos` holds the *natural* ruler. `layoutRows` will then both
(a) optionally stretch rows to fill a CSS table height, and (b) account for pages.

### 4.2 `layout` — lay out the rows once (lines 402–415)

```cpp
void RenderTableSection::layout()
{
    ASSERT(needsLayout());
    LayoutStateMaintainer statePusher(view(), this, IntSize(x(), y()), style()->isFlippedBlocksWritingMode());
    for (RenderObject* child = children()->firstChild(); child; child = child->nextSibling()) {
        if (child->isTableRow()) {
            child->layoutIfNeeded();          // lay out each <tr> (which lays out its cells)
            ASSERT(!child->needsLayout());
        }
    }
    statePusher.pop();
    setNeedsLayout(false);
}
```

This is the simple one: push a coordinate/pagination context for the section, then
tell each row to lay itself out (which cascades into each cell laying out its
contents). It does **not** position the rows — that's `layoutRows`'s job. Think of
`layout()` as "make sure everyone below me has a size," and `layoutRows()` as "now I'll
place everyone."

---

## 5. `layoutRows` — the big one, step by step

`int RenderTableSection::layoutRows(int toAdd, int headHeight, int footHeight)`
(lines 417–792) is ~375 lines and does everything that matters for positioning and
pages. Its three inputs:

- **`toAdd`** — extra height the table wants this section to absorb (e.g. the table
  has `height: 1000px` but its content is only 600px tall, so rows get stretched).
- **`headHeight`** — the height of the repeated `<thead>`. **`0` for the header and
  footer sections themselves**; non-zero for body sections of a header-repeating
  table. *This is the value our fixes reserve at the top of each page.*
- **`footHeight`** — likewise for a repeated `<tfoot>`, reserved at page bottoms.

It runs in seven phases. Here's the map; then each in detail.

```
   Phase A  (417–432)  setup: section width, clear overflow
   Phase B  (434–491)  stretch rows to absorb `toAdd` (CSS table height)
   Phase C  (493–504)  compute logicalRowHeights[] (natural height of each row)
   Phase D  (506–558)  PAGINATION pass 1: reserve header band + relocate rows   ← Fix B, Fix A
   Phase E  (560–614)  PAGINATION pass 2: measure split rows, pack continuously  ← Fix 3
   Phase F  (616–766)  place every cell, set alignment padding, clamp to page    (original)
   Phase G  (768–791)  section height + collect overflow
```

### Phase A — setup (lines 417–432)

```cpp
int totalRows = m_gridRows;
setLogicalWidth(table()->contentLogicalWidth()); // the section is as wide as the table's content
m_overflow.clear();                              // forget last layout's overflow
m_hasOverflowingCell = false;
```

`rHeight` and `rindx` are declared here too (scratch variables reused in Phase F).

### Phase B — stretch rows to fill a CSS table height (lines 434–491)

Only relevant when `toAdd > 0` (the table is taller than its natural content). The
code distributes the extra `toAdd` pixels across the rows in three priority tiers:

1. **Percent-height rows first** (lines 446–465): a row with `height: 20%` gets up to
   its share of the total height.
2. **Auto rows next** (lines 466–478): rows with no explicit height split whatever is
   left, evenly.
3. **Leftovers** (lines 479–490): any remaining pixels are spread proportionally to
   each row's current height.

Each tier walks the rows and bumps every later `m_rowPos[r+1]` by an accumulating
`add`. You can treat this as a black box — *"make the rows add up to the requested
table height"* — it isn't touched by our fixes and rarely matters for PDFs (which
usually have auto-height tables).

### Phase C — per-row natural heights (lines 493–504)

```cpp
int hspacing = table()->hBorderSpacing();  // horizontal gap between cells
int vspacing = table()->vBorderSpacing();  // vertical gap between cells
int nEffCols = table()->numEffCols();      // number of (effective) columns

LayoutStateMaintainer statePusher(view(), this, IntSize(x(), y()), style()->isFlippedBlocksWritingMode());

Vector<int> logicalRowHeights;             // a fresh array: natural height of each row
logicalRowHeights.resize(totalRows);
for (int r = 0; r < totalRows; r++)
    logicalRowHeights[r] = m_rowPos[r + 1] - m_rowPos[r] - vspacing;
```

`logicalRowHeights[r]` is just "row `r`'s height" derived from the `m_rowPos` ruler
(boundary below − boundary above − the spacing). After Phase B it reflects any
stretching. **This array, and `m_rowPos`, are what our fixes adjust.**

The `statePusher` here is critical: it establishes this section's **`LayoutState`**,
which (a) carries the page height down into the cells laid out later in Phase F, and
(b) holds our new **`m_pageRepeatableLogicalHeight`** field (set in Phase D).

### Phase D — pagination pass 1: reserve the header & relocate rows (lines 506–558)

This whole block only runs when paginating:

```cpp
if (view()->layoutState()->pageLogicalHeight()) {
    LayoutState* layoutState = view()->layoutState();
    int pageLogicalHeight = layoutState->m_pageLogicalHeight;  // the page height, e.g. 700px
    int pageOffset = 0;                                        // running downward shift
    ...
```

**First, our Fix B + Fix 4 (line 521)** record the repeated-header band so that
content paginated inside this section's cells will leave room for it:

```cpp
layoutState->m_pageRepeatableLogicalHeight = max(headHeight, layoutState->m_pageRepeatableLogicalHeight);
```

(The `max(...)` is the nested-table fix; we dissect it in [§6.4](#64-fix-4--nested-tables-keep-the-reservation-line-521).
For a top-level table the inherited value is 0, so this is just `= headHeight`.)

**Then the relocation loop (lines 523–556)** walks the rows. For each row it works out
where the page boundary falls and decides whether the row must move down. The page
arithmetic:

```cpp
m_rowPos[r] += pageOffset;   // apply the shift accumulated by earlier rows
int remainingLogicalHeight = pageLogicalHeight - layoutState->pageLogicalOffset(m_rowPos[r]) % pageLogicalHeight;
int availableHeight = remainingLogicalHeight - footHeight - vspacing;
```

Picture a page that is `pageLogicalHeight` tall. `pageLogicalOffset(m_rowPos[r])` is
how far row `r`'s top is from the very first page's top. The `% pageLogicalHeight`
("remainder after dividing by the page height") turns that into "how far down *this*
page we are." Subtract from the page height → **`remainingLogicalHeight`** = the space
left on the current page below the row's top. **`availableHeight`** then removes the
footer band and spacing:

```
   page top  ┌───────────────────────┐  ← a page boundary
             │  (already used above)  │
   row top → ├───────────────────────┤  m_rowPos[r]
             │                        │   ▲
             │  remainingLogicalHeight│   │  space left on this page
             │                        │   ▼
             ├───────────────────────┤   ── reserve footHeight here
             └───────────────────────┘  ← next page boundary
                 availableHeight = remainingLogicalHeight − footHeight − vspacing
```

Next it measures how tall the row *needs* to be. The inner loop (lines 531–542) finds
the tallest cell **but only for rows the author marked `page-break-inside: avoid`**
(the `!= PBAVOID` test on line 535 *skips* every other row):

```cpp
int rowRequiredHeight = 0;
for (int c = 0; c < nEffCols; c++) {
    ... if the row is NOT page-break-inside:avoid, `continue` (skip) ...
    int cellRequiredHeight = cell->contentLogicalHeight() + cell->paddingTop(false) + cell->paddingBottom(false);
    if (cellRequiredHeight > rowRequiredHeight)
        rowRequiredHeight = cellRequiredHeight;   // keep the tallest
}
```

So `rowRequiredHeight` is **0 for ordinary rows** and **the content height for
`avoid` rows**. Now **our Fix A (lines 548–549)** decides the row's required height:

```cpp
bool keepRowTogether = rowRenderer && rowRenderer->style()->pageBreakInside() == PBAVOID;
int requiredHeight = keepRowTogether ? max(logicalRowHeights[r], rowRequiredHeight) : rowRequiredHeight;
```

- **`avoid` row** → `requiredHeight = max(naturalHeight, contentHeight)` — its full
  height, so the next test will move it whole.
- **ordinary row** → `requiredHeight = rowRequiredHeight = 0`.

Finally the relocation test (lines 550–555):

```cpp
if (requiredHeight >= availableHeight && requiredHeight < pageLogicalHeight) {
    pageOffset += remainingLogicalHeight + headHeight;        // skip to next page + reserve header
    if (requiredHeight > availableHeight)
        m_rowPos[r] += remainingLogicalHeight + headHeight;   // move THIS row down too
}
```

Read it as: *"if the row both doesn't fit in the space left **and** could fit on a
fresh page, push it (and everything after it) to the top of the next page, leaving a
`headHeight` gap for the repeated header."*

For an **ordinary** row `requiredHeight` is 0, so `0 >= availableHeight` is false
(there's always some space) → the row is **not** relocated; it is allowed to **split**
across the page break. *That is Fix A: ordinary rows split instead of jumping.* Before
our change the code used `max(logicalRowHeights[r], rowRequiredHeight)` here, which
relocated **every** row — causing the "tall cell jumps to the next page leaving a big
gap" bug.

At the end, `m_rowPos[totalRows] += pageOffset;` (line 557) extends the section's
bottom by the total downward shift.

> **What Phase D does *not* fix:** when an ordinary row *splits*, the cell's content
> grows (struts + header band) beyond `logicalRowHeights[r]`, but Phase D never grew
> `m_rowPos` for it. That's the adjacent-row overlap — fixed next, in Phase E.

### Phase E — pagination pass 2: measure split rows & pack continuously (lines 560–614)

This entire block is **Fix 3** (the measurement pass). It is dissected with diagrams
in [§6.3](#63-fix-3--continuous-packing-of-split-rows-lines-560614); here is the
mechanical walk-through. Again gated on pagination:

```cpp
if (view()->layoutState()->pageLogicalHeight()) {
    int expansion = 0;                       // total extra height accumulated so far
    for (int r = 0; r < totalRows; ++r) {
        m_rowPos[r] += expansion;            // push this row down by all prior growth
        int rowReal = logicalRowHeights[r];  // start from the natural height
        for (int c = 0; c < nEffCols; c++) {
            ... pick the primary cell; skip empties, span-tails, and rowSpan>1 cells ...
            cell->clearIntrinsicPadding();   // measure pure content (no alignment padding)
            ... position the cell at its final x and y = m_rowPos[r] ...
            cell->setChildNeedsLayout(true, false);
            cell->layoutIfNeeded();          // RE-LAY-OUT here ⇒ real paginated height
            if (cell->height() > rowReal)
                rowReal = cell->height();    // the row must be at least this tall
        }
        if (rowReal > logicalRowHeights[r]) {        // did this row grow?
            expansion += rowReal - logicalRowHeights[r];
            logicalRowHeights[r] = rowReal;          // record the real height
        }
    }
    m_rowPos[totalRows] += expansion;        // extend the section bottom by total growth
}
```

The trick that makes this correct in **one pass**: rows are processed **top to
bottom**, and each row's `m_rowPos[r]` is finalized (`+= expansion`) *before* the row
is measured. So the cell is laid out at its true page position, its strut pattern is
the real one, and `cell->height()` is exactly the height the later Phase F will
reproduce. Rows that don't split measure `cell->height() == logicalRowHeights[r]`, so
`expansion` stays 0 and nothing changes — a **strict no-op** for normal tables.

`rowSpan>1` cells (e.g. the `<th rowspan=N>`) are deliberately skipped here; Phase F
sizes them from the now-grown `m_rowPos` (line 636), so a spanning header correctly
covers all the grown rows.

### Phase F — place every cell (lines 616–766)

Now the positions are final. For each row, set the row renderer's location/size
(lines 618–623), then for each cell:

1. **Compute the cell's height `rHeight`** (lines 632–637): for a normal cell it's
   `logicalRowHeights[rindx]`; for a rowspanning cell it's the distance spanned across
   `m_rowPos` (`m_rowPos[rindx + rowSpan] − m_rowPos[rindx] − vspacing`). Both now use
   the **grown** values from Phase E.

2. **Flex percent-height children** (lines 652–701): if the cell has children sized in
   `%`, force them to re-lay-out so they grow to fill `rHeight` (`setOverrideSizeFromRowHeight`).
   This is the one place that can interact awkwardly with split content — see the
   "percentage-height" limitation in §6.

3. **Compute intrinsic (alignment) padding** (lines 703–733): based on the cell's
   `vertical-align`, work out invisible top/bottom padding so the content sits at the
   top / middle / bottom / on a shared baseline of `rHeight`:

   ```cpp
   int logicalHeightWithoutIntrinsicPadding = cell->logicalHeight() - oldBefore - oldAfter;
   switch (vertical-align) {
       case TOP:    intrinsicPaddingBefore = 0;                                  break;
       case MIDDLE: intrinsicPaddingBefore = (rHeight - contentHeight) / 2;      break;
       case BOTTOM: intrinsicPaddingBefore =  rHeight - contentHeight;           break;
       ... baseline cases ...
   }
   intrinsicPaddingAfter = rHeight - contentHeight - intrinsicPaddingBefore;
   ```

   For a **grown** split row, `rHeight == contentHeight`, so all of these are **0** —
   the content stays at the top, which is exactly what you want for a continuation.

4. **Position the cell** (lines 737–740) at its column x and row y `m_rowPos[rindx]`,
   handling left-to-right vs right-to-left.

5. **Lay the cell out** (lines 746–749). If pagination moved the cell to a different
   page-relative spot, it's forced to re-lay-out so its internal page breaks are right.

6. **The page clamp (lines 752–753)** — historically the source of the trouble:

   ```cpp
   if (style()->isHorizontalWritingMode() && view()->layoutState()->pageLogicalHeight() && cell->height() != rHeight)
       cell->setHeight(rHeight);  // force the cell box back to the row height
   ```

   Pagination can make the laid-out cell taller than `rHeight`; this line forces the
   **box** back to `rHeight`. Originally `rHeight` was the *natural* height, so the box
   was shrunk and the extra content spilled out as overflow — overlapping the next
   row. **After Fix 3**, `rHeight` (= grown `logicalRowHeights[r]`) already equals the
   real height, so `cell->height() == rHeight` and this clamp becomes a **no-op**. We
   left the line untouched; Fix 3 simply makes it harmless.

### Phase G — section height & overflow (lines 768–791)

```cpp
setLogicalHeight(m_rowPos[totalRows]);   // the section is as tall as the ruler's last entry
```

Because Phase E added `expansion` to `m_rowPos[totalRows]`, the section is now tall
enough to contain the split rows.

Then a final loop (lines 777–788) records each cell's **overflow** (any drawing that
pokes outside the cell's box, e.g. a shadow) into the section, and notes whether any
cell overflows (`m_hasOverflowingCell`) so painting/hit-testing can choose the right
code path. Finally `statePusher.pop()` ends the section's `LayoutState` and the method
returns the section's height.

---

## 6. The four pagination fixes, in depth

All four fixes live in `layoutRows`, all are gated on pagination, and all are no-ops
for documents without a repeating `<thead>`. They map to the PR commits like this:

| Doc name | Commit name | Lines | One-line purpose |
|---|---|---|---|
| **Fix 1** | "Part B" | 521 (+ companion files) | Reserve the repeated-header band so split content lands *below* it. |
| **Fix 2** | "Part A" | 548–549 | Let an over-tall row **split** instead of **jumping** to the next page. |
| **Fix 3** | measurement pass | 560–614 | Make following rows start *after* a split row (no overlap). |
| **Fix 4** | nested-tables | 521 (`max`) | Keep the reservation across a nested (e.g. `display:table`) cell. |

> **Companion files.** Fix 1 needs a place to *store* the reserved band and a place to
> *use* it. The storage is a new field `m_pageRepeatableLogicalHeight` on `LayoutState`
> (declared in `LayoutState.h`, initialised/propagated in `LayoutState.cpp`). The use
> is in `RenderBlock.cpp` (`adjustLinePositionForPagination` and
> `adjustForUnsplittableChild`), which add the band when pushing content to the next
> page. This document is about `RenderTableSection.cpp`, where the band is *set*
> (line 521); the other two files are summarised in §6.1.

### 6.1 Fix 1 — reserve the repeated-header band (line 521)

**The mechanism being worked around.** When a header-repeating table spans pages,
`RenderTable.cpp` *re-paints* the `<thead>` at the **top of every continuation page**.
That painting reserves no layout space — it just draws over whatever is there. So the
body content must be told to keep the top `headHeight` of each continuation page
clear.

**Before (the bug).** A `<td>` taller than a page is split by WebKit's line
pagination, which lands continuation lines **flush at the page top** — exactly where
the header is re-painted:

```
   BEFORE                          page 2
                          ┌─────────────────────────┐
   repeated header  ───►  │ Requirement  Requested… │ ⟍  both drawn
   cell text        ───►  │ ...quieting of the ego… │ ⟋  in the same band  → OVERLAP
                          │ that no mountain or…     │
                          └─────────────────────────┘
```

**The fix in this file (line 521).** While laying out a body section, record the band
on the shared `LayoutState`:

```cpp
layoutState->m_pageRepeatableLogicalHeight = max(headHeight, layoutState->m_pageRepeatableLogicalHeight);
```

`headHeight` is the height of the repeated `<thead>` (and is **0** when *this* section
*is* the thead/tfoot, so a header never reserves space against itself).

**How the band is consumed (companion file `RenderBlock.cpp`).** When line pagination
decides to push a line to the next page, it now adds the reserved band so the line
lands **below** the header:

```cpp
// simplified from RenderBlock::adjustLinePositionForPagination
int reserve = layoutState->m_pageRepeatableLogicalHeight;     // 0 unless inside a header-repeating table
delta += remainingLogicalHeight + reserve;                    // was: + 0
lineBox->setPaginationStrut(remainingLogicalHeight + reserve);
```

**After (fixed).**

```
   AFTER                           page 2
                          ┌─────────────────────────┐
   repeated header  ───►  │ Requirement  Requested… │     drawn in the reserved band
                          │ ───────────────────────  │  ◄─ headHeight of clear space
   cell text        ───►  │ ...quieting of the ego… │     continues BELOW the header
                          │ that no mountain or…     │
                          └─────────────────────────┘
```

Because the field defaults to 0 and is only set for body sections of a
header-repeating table, content in any ordinary document is paginated exactly as
before.

### 6.2 Fix 2 — split instead of jump (lines 548–549)

**Before (the bug).** The required-height used to be
`int requiredHeight = max(logicalRowHeights[r], rowRequiredHeight);` — the row's full
height, for **every** row. So *any* row that didn't fit the space left on the page was
relocated whole to the next page, leaving a big blank gap:

```
   BEFORE                page 1                       page 2
                 ┌───────────────────┐        ┌───────────────────┐
   row top  ───► ├───────────────────┤        │ Requirement …     │
                 │                    │        │ ───────────────── │
                 │  (blank — the row  │        │ a. The ocean …    │  ← whole row jumped here
                 │   jumped away)     │        │    text …         │
                 └───────────────────┘        └───────────────────┘
                   wasted space                  kept together
```

**The fix (lines 548–549).**

```cpp
bool keepRowTogether = rowRenderer && rowRenderer->style()->pageBreakInside() == PBAVOID;
int requiredHeight = keepRowTogether ? max(logicalRowHeights[r], rowRequiredHeight) : rowRequiredHeight;
```

Only rows the author explicitly marked `page-break-inside: avoid` are kept whole (and
for those `rowRequiredHeight` already holds the content height). For every other row
`requiredHeight` becomes `rowRequiredHeight`, which the inner loop left at **0**, so
the relocation test `requiredHeight >= availableHeight` is false and the row is
allowed to **split**:

```
   AFTER                 page 1                       page 2
                 ┌───────────────────┐        ┌───────────────────┐
   row top  ───► │ a. The ocean …    │        │ Requirement …     │
                 │    text …          │        │ ───────────────── │
                 │    text … (fills   │        │ …text continues   │  ← same cell, below header
                 │    the page)       │        │    text …         │
                 └───────────────────┘        └───────────────────┘
                   no gap                        no overlap (Fix 1)
```

This is what the user picked: *fill the current page, continue on the next.* (To get
the old "keep this row whole" behaviour back for a particular row, add
`tr { page-break-inside: avoid }` in CSS.)

### 6.3 Fix 3 — continuous packing of split rows (lines 560–614)

This is the deepest fix. It addresses the **adjacent-row overlap** that appears when
each item is its **own `<tr>`** (with a `rowspan` "Requirement" `<th>` beside them).

**Why the bug exists.** Recall from §4–§5:

- `m_rowPos[]` (where each row sits) is computed from **natural** heights in
  `calcRowLogicalHeight` and Phases B–D — **before** any cell is actually paginated.
- A cell only gets *paginated* in Phase F (`cell->layoutIfNeeded()`), where the page
  clamp (lines 752–753) then forces the box back to the natural height and lets the
  extra spill out as **overflow**.

So when row "a" is taller than a page, its real drawn height is
`naturalHeight + (page-break struts) + (header bands)`, but `m_rowPos["b"]` (the next
row) was placed only `naturalHeight("a")` below "a". Row "b" therefore lands **on top
of** the overflowed tail of "a":

```
   BEFORE (m_rowPos from natural heights)
                          what m_rowPos says          what actually draws
   m_rowPos[a] ─►  ┌───────────────────┐        ┌───────────────────┐
                   │ row a (natural ht) │        │ row a content …   │
   m_rowPos[b] ─►  ├───────────────────┤        │ …a keeps going via │  ← a's overflow
                   │ row b              │        │ ▓▓ b drawn here ▓▓ │  ← b overlaps a!
                   └───────────────────┘        └───────────────────┘
```

**The fix: a measurement pass (Phase E).** Before placing anything (Phase F), walk the
rows top-to-bottom and find each row's **real** paginated height by actually laying the
cell out at its final position, then grow the ruler:

```cpp
int expansion = 0;
for (int r = 0; r < totalRows; ++r) {
    m_rowPos[r] += expansion;                 // ① this row's position is now FINAL
    int rowReal = logicalRowHeights[r];
    for (each rowSpan==1 primary cell of row r) {
        cell->clearIntrinsicPadding();        // ② measure pure content height
        cell->setLogicalLocation(x, m_rowPos[r]);
        cell->setChildNeedsLayout(true, false);
        cell->layoutIfNeeded();               // ③ real paginated height at this page position
        rowReal = max(rowReal, cell->height());
    }
    if (rowReal > logicalRowHeights[r]) {     // ④ did it grow?
        expansion += rowReal - logicalRowHeights[r];   // remember the growth
        logicalRowHeights[r] = rowReal;       // record the real height
    }
}
m_rowPos[totalRows] += expansion;             // ⑤ extend the section bottom
```

Step-by-step reasoning (this is the part to internalise):

- **① Order matters.** Because we add `expansion` to `m_rowPos[r]` *before* measuring
  row `r`, the row's top is already in its final place. Pagination struts depend on a
  cell's page-relative top, so measuring here yields the *exact* height Phase F will
  later reproduce. No second iteration is ever needed.
- **② Clear intrinsic padding.** `cell->height()` includes any invisible alignment
  padding from a previous layout. Clearing it first makes the measurement pure
  `border + padding + paginated content`. Phase F then recomputes alignment padding
  from scratch (it subtracts the *old* value, which is now 0), so the two passes agree
  and the clamp becomes a no-op.
- **③ The measurement.** `cell->height()` after `layoutIfNeeded()` is the same unit the
  clamp on line 752–753 uses, so growth math and the clamp are consistent.
- **④/⑤ Grow and cascade.** Each row that grows pushes `expansion` up, which shifts
  *every following* row down on its next loop iteration, and finally stretches the
  section's bottom boundary.

**Result — continuous packing:**

```
   AFTER (m_rowPos grown to real heights)
   m_rowPos[a] ─►  ┌───────────────────┐
                   │ row a content …   │
                   │ …splits across    │   real height now recorded
                   │    pages …        │
   m_rowPos[b] ─►  ├───────────────────┤   ← b starts exactly where a really ends
                   │ row b content …   │
                   └───────────────────┘
```

Two more things the fix gets right:

- **Rowspan `<th>`.** It is skipped by the measurement (`cell->rowSpan() != 1`).
  Phase F sizes it from `m_rowPos[rindx + rowSpan] − m_rowPos[rindx]` (line 636), which
  now uses the **grown** `m_rowPos`, so the spanning header correctly covers all the
  grown item rows.
- **Strict no-op when nothing splits.** Every measured `cell->height()` equals the
  natural `logicalRowHeights[r]`, `expansion` stays 0, and `m_rowPos[]` /
  `logicalRowHeights[]` are byte-for-byte identical to before — so normal tables (and
  all on-screen layout) are unaffected, at the cost of one extra cell layout per row
  while paginating.

**Known limitation.** If a cell uses **percentage-height children** that themselves
split, Phase F's `setOverrideSizeFromRowHeight` (line 691) can re-paginate slightly
differently than the measurement predicted. The clamp still prevents overlap, but such
a cell may not be pixel-perfect. The documents in scope don't use this.

### 6.4 Fix 4 — nested tables keep the reservation (line 521)

**The trigger.** The latest template wraps each reason in
`.req-cols { display: table }` / `.req-reason { display: table-cell }`. As noted in
§1.2, that makes WebKit build an **anonymous nested table** inside the update `<td>`.

**Why the bug came back.** `layoutRows` runs for that **inner** table's section too.
Originally line 521 read `layoutState->m_pageRepeatableLogicalHeight = headHeight;`.
The inner table has no `<thead>`, so its `headHeight` is **0**, and that assignment
**overwrote** the outer table's reservation (which had been propagated down the
`LayoutState` chain). Inner content then paginated with reservation 0 → flush to the
page top → the outer header overprinted it again.

```
   LayoutState chain while laying out the inner cell's text:

   outer <tbody> section   m_pageRepeatableLogicalHeight = headHeight   (e.g. 28)
        │  (propagates down through the outer <td>, the inner table…)
        ▼
   inner table's section   BEFORE: = headHeight(inner) = 0   ✗ clobbered → overlap
                           AFTER : = max(0, 28)        = 28  ✓ preserved → no overlap
```

**The fix (line 521).**

```cpp
layoutState->m_pageRepeatableLogicalHeight = max(headHeight, layoutState->m_pageRepeatableLogicalHeight);
```

When the inner section runs, `layoutState->m_pageRepeatableLogicalHeight` already
holds the **inherited** outer value (the `LayoutState` constructor copies it down to
every nested layout that doesn't start a new page context — confirmed: both
`RenderTable::layout` and the section's own state-push use page-height 0, i.e. the
*propagate* path). `max(headHeight, inherited)` therefore:

- top-level header table → `max(headHeight, 0) = headHeight` (unchanged, no-op);
- nested table with no header → `max(0, outer) = outer` (keeps the outer band — the
  fix);
- nested table *with* its own header → `max(inner, outer)` (clears the taller of the
  two repeated headers).

This is a strict no-op except inside a table nested within a header-repeating one.

---

## 7. Borders, baselines, painting, hit-testing

These methods don't touch our fixes, but the user asked for *every* method, so here
they are.

### 7.1 Outer-border calculation (lines 794–990)

With **collapsed borders** (`border-collapse: collapse`), adjacent cells, rows,
columns, the section, and the table all "share" one border line, and CSS has precise
rules for **which** of the competing borders wins (widest wins; `hidden` beats
everything; `none` loses). These four near-identical methods compute the section's
effective outer border on each side:

- `calcOuterBorderBefore()` (794–843) — top edge,
- `calcOuterBorderAfter()` (845–894) — bottom edge,
- `calcOuterBorderStart()` (896–938) — left edge,
- `calcOuterBorderEnd()` (940–982) — right edge.

They all follow the same shape (using *before* as the example):

```cpp
const BorderValue& sb = style()->borderBefore();   // the section's own border
if (sb.style() == BHIDDEN) return -1;              // "hidden" ⇒ no border at all (-1)
if (sb.style() > BHIDDEN) borderWidth = sb.width();// otherwise track the widest so far
... also consider the first row's border ...
for (int c = 0; c < totalCols; c++) {              // and every cell in the edge row:
    ... compare the cell's (and its column group's) border, keep the widest ...
}
if (allHidden) return -1;
return borderWidth / 2;     // half, because the border is shared with the neighbour
```

The return value is **half** the winning width (each side owns half of a collapsed
border line; the `+1` variants like `(borderWidth + 1) / 2` on lines 893/937/981 round
the half so the two halves add back to the whole). A return of **−1** is a sentinel
meaning "this edge is `hidden`."

**`recalcOuterBorder()`** (lines 984–990) just calls all four and caches the results in
`m_outerBorderBefore/After/Start/End` (read back via the inline getters in the header).

### 7.2 `firstLineBoxBaseline` (lines 992–1011)

Returns the vertical position of the section's first text baseline — used when a table
is itself aligned to surrounding text by baseline.

```cpp
if (!m_gridRows) return -1;                          // no rows ⇒ no baseline
int firstLineBaseline = m_grid[0].baseline;          // baseline recorded for row 0
if (firstLineBaseline)
    return firstLineBaseline + m_rowPos[0];          // offset by row 0's top
// fall back: the bottom of the tallest cell content in row 0
firstLineBaseline = -1;
for (each primary cell in row 0)
    firstLineBaseline = max(firstLineBaseline, cell->logicalTop() + paddingBefore + borderBefore + contentLogicalHeight);
return firstLineBaseline;
```

### 7.3 Painting (lines 1013–1176)

**`paint`** (lines 1013–1035) is the entry point. It bails if layout is dirty, then
offsets by the section's own `(x, y)`, optionally clips, and delegates to
`paintObject`:

```cpp
if (needsLayout()) return;          // never paint with stale layout
tx += x(); ty += y();               // move into the section's coordinate space
bool pushedClip = pushContentsClip(paintInfo, tx, ty);
paintObject(paintInfo, tx, ty);
if (pushedClip) popContentsClip(paintInfo, phase, tx, ty);
```

**`paintCell`** (lines 1042–1074) paints one cell. For the background phases it paints
a **stack** of backgrounds *behind* the cell, bottom-to-top: column-group, column,
row-group (this section), and the row (lines 1048–1071). This is exactly the CSS table
background layering order. Then it paints the cell itself (lines 1072–1073) unless a
self-painting layer will handle it.

**`paintObject`** (lines 1076–1176) decides **which cells to paint** and **in what
order**:

1. *Which rows/cols are visible.* If no cell overflows, it does a **binary search**
   over `m_rowPos` (and the table's `columnPositions`) to find just the rows and
   columns intersecting the dirty rectangle (lines 1098–1136) — an efficiency win, so
   a 500-row table on page 7 doesn't repaint rows 1–400. `std::lower_bound` is a
   standard binary search; it returns the first entry ≥ a value. If some cell overflows
   (`m_hasOverflowingCell`), it can't trust the ruler and paints **all** rows/cols.
2. *In what order.* The common case (`!m_hasMultipleCellLevels`, lines 1138–1148)
   paints cells in grid order, skipping squares that are continuations of a span
   (`primaryCellAt(r-1, c) == cell` etc.). The rare overlapping-cells case
   (lines 1149–1174) collects the cells, de-duplicates spanning cells with a
   `HashSet`, and **sorts them by row** (`compareCellPositions`, lines 1037–1040) so
   that lower cells paint after higher ones.

### 7.4 `nodeAtPoint` — hit testing (lines 1270–1343)

The reverse of painting: given a point (e.g. where the user clicked, or for text
selection), find the cell there.

```cpp
if (!firstChild()) return false;          // nothing here
tx += x(); ty += y();                      // into section coordinates
if (m_hasOverflowingCell) {                // overflow ⇒ can't use the ruler;
    for (each child from last to first)    //   ask each row directly, back to front
        if (child->nodeAtPoint(...)) { ...; return true; }
    return false;
}
// fast path: binary-search the ruler for the row, the column positions for the column
unsigned hitRow    = (upper_bound on m_rowPos)    − 1;
unsigned hitColumn = (lower_bound on columnPos)   − 1;
CellStruct& current = cellAt(hitRow, hitColumn);
if (!current.hasCells()) return false;     // empty square
for (each cell at that square, topmost first)
    if (cell->nodeAtPoint(...)) { updateHitTestResult(...); return true; }
return false;
```

Note it mirrors `paintObject`'s structure: a fast binary-search path keyed off
`m_rowPos`/`columnPos`, and a slow "ask everyone" path when cells overflow. This is
another reason `m_rowPos` must be accurate — **our Fix 3 keeps hit-testing correct for
split rows too**, not just painting.

### 7.5 `imageChanged` (lines 1178–1182)

When a background image used by the section finishes loading or animates, repaint the
whole section (a `// FIXME` notes it could repaint just the affected rectangle):

```cpp
void RenderTableSection::imageChanged(WrappedImagePtr, const IntRect*) { repaint(); }
```

---

## 8. Glossary

| Term | Meaning |
|---|---|
| **render tree / renderer** | The tree of layout objects (one per visible box) WebKit builds from the DOM. `RenderTableSection` is one renderer. |
| **`RenderTableSection`** | The renderer for a `<thead>`, `<tbody>`, or `<tfoot>` — a group of rows. The subject of this file. |
| **grid** | The section's internal 2-D map of cells: `m_grid` (rows) → `Row` (a row's squares) → `CellStruct` (one square). |
| **`m_rowPos`** | Array of row boundary positions (top of each row; last entry = section bottom). `height(row r) = m_rowPos[r+1] − m_rowPos[r] − spacing`. The pivot of the whole file. |
| **`CellStruct` / primary cell / `inColSpan`** | One grid square; its "real" cell; and the flag marking squares that are the right-hand continuation of a colspan. |
| **logical (height/width/before/after/start/end)** | Writing-mode-independent directions. In ordinary English text: height/width/top/bottom/left/right. |
| **natural height** | A row/cell's height ignoring page breaks (from `calcRowLogicalHeight`). |
| **paginated height** | The height a cell actually occupies once page breaks split its content. Bigger than natural for a split cell. **Fix 3 measures this.** |
| **`LayoutState`** | Per-layout context carrying the coordinate offset, the **page height**, and our new **`m_pageRepeatableLogicalHeight`**. Pushed/popped by `LayoutStateMaintainer`. |
| **pagination strut** | Extra space WebKit inserts to push a line/box down so it doesn't straddle a page boundary. **Fix 1** makes the strut also clear the repeated-header band. |
| **`pageLogicalHeight`** | The page height. Non-zero ⇔ "we are making a PDF / paginating." All four fixes are gated on this. |
| **`headHeight` / `footHeight`** | Height of the repeated `<thead>` / `<tfoot>`, reserved at page tops/bottoms. `0` for the header/footer sections themselves. |
| **`m_pageRepeatableLogicalHeight`** | Our new `LayoutState` field: the repeated-header band to reserve for content inside this table. Set on line 521; used in `RenderBlock.cpp`. |
| **intrinsic padding** | Invisible top/bottom padding the engine adds to vertically align a cell's content in a taller row. Cleared during Fix 3's measurement so it reads pure content height. |
| **`page-break-inside: avoid` / `PBAVOID`** | CSS asking that a box not be split across pages. After **Fix 2**, the *only* thing that keeps a table row whole. |
| **clamp (lines 752–753)** | The original line that forces a paginated cell's box back to the row height. Harmless after Fix 3 (the row height now equals the real height). |
| **`ASSERT` / `#ifndef NDEBUG`** | Debug-only sanity checks; compiled out of the shipping build. |
| **`LayoutStateMaintainer`** | RAII helper that pushes a `LayoutState` on creation and pops it on `.pop()`/destruction. |
| **`std::lower_bound` / `upper_bound`** | Standard binary searches over a sorted array (used on `m_rowPos`/`columnPos` to find visible/hit rows quickly). |

---

### Appendix: the fixes at a glance (all in `layoutRows`)

```
   layoutRows(toAdd, headHeight, footHeight)
   ├─ Phase D (506–558)  if paginating:
   │     line 521   m_pageRepeatableLogicalHeight = max(headHeight, inherited)   ← Fix 1 + Fix 4
   │     line 549   requiredHeight = keepRowTogether ? max(...) : rowRequiredHeight ← Fix 2
   ├─ Phase E (560–614)  if paginating:  measure split rows, grow m_rowPos        ← Fix 3
   └─ Phase F (752–753)  the page clamp — unchanged, made harmless by Fix 3
```

*End of guide.*


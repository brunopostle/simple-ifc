# CLAUDE.md

This file provides guidance to Claude Code when working in this repository of IFC building model files.

## Overview

This repository contains IFC (Industry Foundation Classes) building models. IFC is an open standard (ISO 16739) for sharing building and infrastructure data. The repository uses standard tooling for:

- **Querying and editing** models with `ifcquery` / `ifcedit` / `ifcmcp`
- **Validation** against schema rules and IDS specifications
- **Continuous integration** — models are validated automatically on every commit
- **Branch-based workflows** — feature branches for model changes, `ifcmerge` for conflict resolution

## Preferred Tooling

### ifcmcp (preferred for interactive sessions)

The `ifcmcp` MCP server holds the IFC model in memory across tool calls — no file I/O between operations. It is always the first choice for multi-step queries and edits.

**Setup** (add to `.mcp.json`):

```json
{
  "mcpServers": {
    "ifc": {
      "type": "stdio",
      "command": "python3",
      "args": ["-m", "ifcmcp"]
    }
  }
}
```

Or via CLI: `claude mcp add --transport stdio ifc -- python3 -m ifcmcp`

### ifcquery / ifcedit (CLI — useful for scripting)

```bash
python3 -m ifcquery <file> <subcommand> [args]
python3 -m ifcedit <subcommand> [args]
```

Each invocation reads/writes from disk. Use for scripting or when MCP is not available.

---

## MCP Tool Reference

### Session tools

- **`ifc_load(path)`** — Open an IFC file into memory. Must be called before any other tool.
- **`ifc_save(path="")`** — Write model to disk. Empty path overwrites the original file.
- **`ifc_reset()`** — Clear in-memory model without saving.

### Query tools (read-only)

- **`ifc_summary()`** — Schema version, entity counts, project name/description.
- **`ifc_tree()`** — Full spatial hierarchy: Project → Site → Building → Storeys → Spaces → Elements.
- **`ifc_info(element_id)`** — Deep inspection of any entity by step ID: attributes, property sets, resolved 4×4 placement matrix, type, material, container.
- **`ifc_select(query)`** — Filter elements by IFC class (e.g. `"IfcWall"`, `"IfcWindow"`). Returns `[{id, type, name}]`.
- **`ifc_relations(element_id, traverse="")`** — All relationships for an element. Use `traverse="up"` to walk the hierarchy up to IfcProject.
- **`ifc_clash(element_id, clearance=0, tolerance=0.002, scope="storey")`** — Geometric clash detection within the same storey (or wider scope).
- **`ifc_validate(express_rules=False)`** — Schema and constraint validation. Pass `express_rules=True` for full EXPRESS rule checking (slower). Returns `{"valid": bool, "issues": [...]}`.
- **`ifc_schedule(max_depth=None)`** — Work schedules with nested task trees and start/finish dates.
- **`ifc_cost(max_depth=None)`** — Cost schedules with nested cost item trees.
- **`ifc_schema(entity_type)`** — IFC class documentation for the loaded model's schema version.

### Edit discovery tools

- **`ifc_list(module="")`** — List all API modules, or functions within a module (e.g. `"sequence"`, `"root"`, `"geometry"`).
- **`ifc_docs(function_path)`** — Full parameter documentation for an API function (e.g. `"root.remove_product"`, `"sequence.add_task"`).

### Edit execution

- **`ifc_edit(function_path, params="{}")`** — Execute an `ifcopenshell.api` mutation. `params` is a JSON string. Returns `{"ok": true, "result": ...}` or `{"ok": false, "error": "..."}`. Does **not** auto-save.
- **`ifc_quantify(rule, selector="")`** — Run quantity take-off. Writes `IfcElementQuantity` psets to the in-memory model. Does **not** auto-save. Rules: `"IFC4QtoBaseQuantities"` or `"IFC4X3QtoBaseQuantities"`.

### Parameter coercion in `ifc_edit`

| Type | JSON value | Python result |
|---|---|---|
| `entity_instance` | `"42"` | Entity resolved from model by step ID |
| `list[entity_instance]` | `"5,6,7"` | List of resolved entities |
| `dict` | `'{"key": "val"}'` | Parsed JSON object |
| `bool` | `"true"` | `True` |
| `Optional[X]` | `"none"` | `None` |

---

## Standard Workflow

```
ifc_load("model.ifc")
ifc_summary()                          # understand what's in the model
ifc_tree()                             # see spatial hierarchy
ifc_validate(express_rules=True)       # check model health before editing

# Query elements of interest
ifc_select("IfcWall")
ifc_info(<id>)
ifc_relations(<id>)

# Discover available edits
ifc_list("sequence")
ifc_docs("sequence.add_task")

# Edit (multiple calls, all in-memory)
ifc_edit("...", '{"param": "value"}')

# Verify, then save
ifc_validate()
ifc_save()
```

---

## IFC Concepts

### Spatial Hierarchy

```
IfcProject
  └─ IfcSite
       └─ IfcBuilding
            ├─ IfcBuildingStorey "Ground Floor"
            │    ├─ IfcSpace "kitchen"  (aggregated via IfcRelAggregates)
            │    │    ├─ IfcWindow  (contained in space via IfcRelContainedInSpatialStructure)
            │    │    └─ IfcDoor
            │    ├─ IfcWall  (contained directly in storey via IfcRelContainedInSpatialStructure)
            │    │    └─ IfcOpeningElement  (via IfcRelVoidsElement)
            │    │         └─ IfcWindow  (via IfcRelFillsElement)
            │    ├─ IfcElementAssembly  (optional grouping of walls)
            │    │    └─ IfcWall  (parts, via IfcRelAggregates)
            │    ├─ IfcSlab, IfcFooting, ...
            └─ IfcBuildingStorey "First Floor"
                 └─ ...
```

**Important notes on containment:**
- Elements are contained in a spatial element (project, site, building, storey, or space) via `IfcRelContainedInSpatialStructure` — windows and doors are not necessarily in a space; they may be contained directly in a storey or building
- Walls may appear directly in a storey (contained), or as parts of an `IfcElementAssembly` (aggregated) — both patterns are valid
- Windows/doors appear in two places simultaneously: spatially contained in a spatial element, and structurally filling an opening in a wall (via `IfcRelFillsElement`). Both relationships must be maintained when editing.

### Key Relationships

| Relationship | Meaning |
|---|---|
| `IfcRelAggregates` | Whole → parts (project → site, assembly → walls) |
| `IfcRelContainedInSpatialStructure` | Element belongs to a spatial container |
| `IfcRelVoidsElement` | Opening cuts through a wall/slab |
| `IfcRelFillsElement` | Window/door fills an opening |
| `IfcRelSpaceBoundary` | Element bounds a space (thermal/acoustic boundary) |
| `IfcRelDefinesByType` | Element instance → type |
| `IfcRelDefinesByProperties` | Element → property set |
| `IfcRelAssignsToProcess` | Element → construction task |
| `IfcRelSequence` | Task A finishes before task B starts |

---

## Common Editing Recipes

### Deleting a window or door

Windows and doors have multiple relationships that must all be cleaned up:

1. **Find the opening**: `ifc_relations(<window_id>)` → note `filled_void` → IfcOpeningElement ID
2. **Find space boundaries**: `ifc_relations(<window_id>)` → note any `IfcRelSpaceBoundary` IDs referencing this element
3. **Delete the window**: `ifc_edit("root.remove_product", '{"product": "<window_id>"}')`
4. **Delete the orphaned opening**: `ifc_edit("root.remove_product", '{"product": "<opening_id>"}')`

`root.remove_product` is a smart delete — removes the entity plus all its relationships (geometry, placement, properties, materials, containment, type assignments, space boundaries). The IfcOpeningElement must be removed separately.

**Verify no orphaned openings remain:**
```
ifc_relations(<wall_id>)     # → check children.openings for unfilled IfcOpeningElements
```

### Moving a window or door

```
ifc_edit("geometry.edit_object_placement", '{"product": "<window_id>", "matrix": [[...],[...],[...],[0,0,0,1]]}')
ifc_edit("geometry.edit_object_placement", '{"product": "<opening_id>", "matrix": [[...],[...],[...],[0,0,0,1]]}')
```

Always move the IfcOpeningElement to the same position as the window/door. Also check `IfcRelSpaceBoundary` entities — their geometry may need updating if the window moves between spaces or changes boundary.

### Changing element types

```
ifc_select("IfcWindowType")            # list available types
ifc_edit("type.assign_type", '{"related_objects": "<window_id>", "relating_type": "<type_id>"}')
```

Type assignment also remaps geometry representations from the type to the occurrence.

### Editing attributes on any entity

`attribute.edit_attributes` works on any entity, not just rooted products — useful for geometry entities, placements, task times, etc.:

```
ifc_edit("attribute.edit_attributes", '{"product": "<id>", "attributes": "{\"Depth\": 3.0}"}')
```

Note the nested JSON: `params` is a JSON string and `attributes` within it is also a JSON string.

### Tracing geometry chains

Walk entity references to find extrusion depths, clipping planes, etc.:

```
ifc_info(<wall_id>)                    # → Representation → IfcProductDefinitionShape id
ifc_info(<pds_id>)                     # → Representations → IfcShapeRepresentation id (Body)
ifc_info(<body_rep_id>)                # → Items → IfcExtrudedAreaSolid or IfcBooleanClippingResult
```

Batch parallel `ifc_info` calls to speed up traversal across many elements.

---

## Quantity Take-Off (QTO)

Run `ifc_quantify` to compute quantities. Results are written as `IfcElementQuantity` property sets on each element.

```
ifc_quantify("IFC4QtoBaseQuantities")              # IFC4 models
ifc_quantify("IFC4X3QtoBaseQuantities")            # IFC4x3 models
ifc_quantify("IFC4QtoBaseQuantities", "IfcWall")   # restrict to element type
ifc_save()                                          # persist to disk
```

**Weight availability by element type (IFC4QtoBaseQuantities):**

| Element | Weight? |
|---|---|
| IfcWall, IfcSlab, IfcFooting, IfcBeam, IfcColumn | ✓ GrossWeight, NetWeight |
| IfcWindow, IfcDoor | ✗ Area/dimensions only |
| IfcRoof | ✗ GrossArea only |
| IfcCovering | ✗ GrossArea/Width only |
| IfcPipeSegment | ✗ Length only |

Weight is computed only for structural/volumetric elements with known material density.

---

## Construction Scheduling

### Schedule structure

```
IfcWorkPlan
  └─ IfcWorkSchedule
       ├─ IfcTask "P1: Foundations"  (summary)
       │    ├─ IfcTask "P1.1: Install Ground Beams"  (leaf, has IfcTaskTime)
       │    └─ IfcTask "P1.2: Pour Floor Slab"       (leaf, has IfcTaskTime)
       ├─ IfcTask "P2: Structure"    (summary, has IfcTaskTime)
       │    └─ IfcTask "P2.1: Erect Walls"           (leaf, has IfcTaskTime)
       └─ ...
```

Physical elements are linked to tasks via `IfcRelAssignsToProcess`. Task ordering is via `IfcRelSequence` (FINISH_START, START_START, etc.).

### Creating a schedule (MCP recipe)

```
# Work plan and schedule
ifc_edit("sequence.add_work_plan", '{"name": "Construction Plan", "predefined_type": "ACTUAL"}')
ifc_edit("sequence.add_work_schedule", '{"name": "Construction Schedule", "predefined_type": "BASELINE", "work_plan": "<plan_id>"}')

# Phase (summary) tasks
ifc_edit("sequence.add_task", '{"work_schedule": "<sched_id>", "name": "Foundations", "identification": "P1", "predefined_type": "CONSTRUCTION"}')

# Leaf tasks
ifc_edit("sequence.add_task", '{"parent_task": "<phase_id>", "name": "Install Ground Beams", "identification": "P1.1", "predefined_type": "CONSTRUCTION"}')

# Assign physical elements to tasks
ifc_edit("sequence.assign_process", '{"relating_process": "<task_id>", "related_object": "<element_id>"}')

# Sequence relationships between phases/tasks
ifc_edit("sequence.assign_sequence", '{"relating_process": "<pred_id>", "related_process": "<succ_id>", "sequence_type": "FINISH_START"}')

# Add IfcTaskTime to ALL tasks (leaf AND summary — required for cascade to propagate)
ifc_edit("sequence.add_task_time", '{"task": "<task_id>"}')   # → returns IfcTaskTime id

# Set durations and start dates (use attribute.edit_attributes — see bug note below)
ifc_edit("attribute.edit_attributes", '{"product": "<task_time_id>", "attributes": "{\"ScheduleStart\": \"2026-03-02T09:00:00\", \"ScheduleDuration\": \"P5D\"}"}')

# Cascade dates forward through sequence relationships
ifc_edit("sequence.cascade_schedule", '{"task": "<first_task_id>"}')

# Verify
ifc_schedule()
```

### Work calendar (Mon–Fri)

```
ifc_edit("sequence.add_work_calendar", '{"name": "Mon-Fri Work Week"}')
ifc_edit("sequence.add_work_time", '{"work_calendar": "<cal_id>", "time_type": "WorkingTimes"}')
ifc_edit("sequence.assign_recurrence_pattern", '{"parent": "<work_time_id>", "recurrence_type": "WEEKLY"}')
ifc_edit("sequence.edit_recurrence_pattern", '{"recurrence_pattern": "<pat_id>", "attributes": "{\"WeekdayComponent\": [1,2,3,4,5]}"}')
```

`WeekdayComponent` uses integers 1–7 where 1=Monday, 7=Sunday.

### cascade_schedule limitations

- Every task in the chain (summary AND leaf) must have an `IfcTaskTime` — tasks without one have null start/finish and block propagation to successors
- Cascade propagates through `IfcRelSequence` relationships only, not into subtasks of a summary task. After cascade sets a summary task's start, its leaf subtasks must have their starts set manually (or connected via additional leaf-to-leaf sequences)
- Summary task finish = start + duration; without a duration, finish equals start, propagating wrong dates to successors
- `cascade_schedule` does **not** automatically respect a work calendar; weekend-skipping must be applied manually when setting task dates

### Known bug: `sequence.edit_task_time` silently fails for datetime fields

`sequence.edit_task_time` returns `{"ok": true}` but does **not** persist `ScheduleStart` or `ScheduleFinish` changes. Use `attribute.edit_attributes` on the `IfcTaskTime` entity instead:

```python
# WRONG — silently does nothing for datetime fields:
ifc_edit("sequence.edit_task_time", '{"task_time": "123", "attributes": "{\"ScheduleStart\": \"2026-03-09T09:00:00\"}"}')

# CORRECT:
ifc_edit("attribute.edit_attributes", '{"product": "123", "attributes": "{\"ScheduleStart\": \"2026-03-09T09:00:00\"}"}')
```

`sequence.edit_task_time` works correctly for `ScheduleDuration` — only datetime fields are affected.

---

## Validation and CI/CD

### Manual validation

```
ifc_validate()                         # schema validation (fast)
ifc_validate(express_rules=True)       # full EXPRESS rules (slower, more thorough)
```

Or via CLI:
```bash
python3 -m ifcquery <file> validate
python3 -m ifcquery <file> validate --rules
```

### IDS validation

IDS (Information Delivery Specification) files define requirements that models must satisfy — required property sets, classification codes, material assignments, etc. Validate with `ifctester`:

```bash
python3 -m ifctester <ids_file> <ifc_file>
```

IDS checks are typically run in CI on every commit or pull request.

### ifcopenshell.validate

Lower-level programmatic validation:

```bash
python3 -c "import ifcopenshell; import ifcopenshell.validate; f = ifcopenshell.open('<file>'); ifcopenshell.validate.validate(f)"
```

---

## Git Workflow

IFC files use STEP Physical File format — plain text, one entity per line — which is natively git-friendly. Commits, diffs, rollbacks, and blame all work as expected.

**However, IFC branches do not merge correctly with standard git** because STEP IDs are file-scoped integers that conflict across branches. Use `ifcmerge` as the merge driver:

### Configuring ifcmerge

In `.gitattributes`:
```
*.ifc merge=ifcmerge
```

In `.git/config` or `~/.gitconfig`:
```
[merge "ifcmerge"]
    name = IFC merge driver
    driver = ifcmerge %O %A %B %P
```

### ifcmerge behaviour

- By default ifcmerge **prefers the `$REMOTE` branch** and rewrites step IDs in `$LOCAL` to avoid conflicts. This is the correct behaviour when merging from `main`/`origin` into your local branch (i.e. a `git pull` or `git merge main`).
- When merging a **feature branch into main** (i.e. you want `$LOCAL`=main to be the authoritative base), reverse the driver arguments so that main's step IDs are preserved:
  ```
  driver = ifcmerge %O %B %A %P
  ```
  The easiest approach is to merge in the other direction (merge main into the feature branch first to resolve conflicts there, then fast-forward main), which keeps the default driver order correct.

### Typical branch workflow

- `main` — validated, CI-passing models only
- Feature branches for each change (new elements, schedule additions, geometry edits, etc.)
- Open a pull request → CI runs `ifc_validate` and IDS checks → merge via ifcmerge on approval
- Tag releases for milestone model snapshots

---

## Tips

- **Always validate before saving** — run `ifc_validate()` after edits and before `ifc_save()`
- **`ifc_info` on non-product entities** — works on any step ID including geometry, task times, placements; useful for tracing chains
- **Batch parallel tool calls** — independent `ifc_info` and `ifc_edit` calls can be issued in parallel for speed
- **`ifc_edit` does not auto-save** — always call `ifc_save()` explicitly when done with a set of edits
- **ISO 8601 durations** — task durations use `P5D` (5 days), `P1W` (1 week), `PT8H` (8 hours)
- **Step IDs are file-specific** — never hard-code step IDs; always query first with `ifc_select`, `ifc_tree`, or `ifc_info`
- **Space boundaries** — when adding, moving, or deleting windows and doors, check for `IfcRelSpaceBoundary` relationships that may also need updating

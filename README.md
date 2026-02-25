# simple-ifc

A small IFC building model used as a sandbox for exploring IFC model workflows with LLM agents and the IfcOpenShell MCP server.

## What this is

The model (`_test_simple.ifc`) is a simple house — four exterior walls, windows, a concrete floor slab, ground beam footings, and a pitched roof. It's intentionally small enough to be easy to reason about, but complete enough to exercise real IFC workflows:

- **Quantity take-off** — running `ifc_quantify` to compute wall volumes, roof areas, window areas, etc.
- **Construction scheduling** — `IfcWorkSchedule` with phased tasks linked to physical elements
- **Bill of Quantities** — `IfcCostSchedule` with parametric quantity links and unit rates, exported to PDF via `ifc5d`
- **IDS validation** — checking model completeness against Information Delivery Specifications

The point is to see how well an LLM agent (Claude Code + the IFC MCP server) can handle multi-step IFC editing tasks — things that would normally require a BIM technician and specialist software.

## Setup

```bash
pip install ifcopenshell ifcmcp ifc5d ifctester typst
```

The MCP server is configured in `.mcp.json`. In Claude Code it loads automatically; the agent can query and edit the model in memory across tool calls without round-tripping to disk between every operation.

## Repository contents

| Path | Description |
|---|---|
| `_test_simple.ifc` | The IFC model (IFC4 schema) |
| `IDS/` | IDS specification files for CI validation |
| `IDS/CI_project_checks.ids` | Project CI checks: QTO completeness, element naming, BoQ properties |
| `IDS/EN_Basic IDM Check.ids` | Standard BIM Basic IDM checks |
| `CLAUDE.md` | Instructions and reference for the LLM agent |

## Validation

Run IDS checks manually:

```bash
python3 -m ifctester IDS/CI_project_checks.ids _test_simple.ifc
python3 -m ifctester "IDS/EN_Basic IDM Check.ids" _test_simple.ifc
```

## Generating the Bill of Quantities

Quantities must be computed before generating the BoQ (or after any geometry changes):

```python
# In a Claude Code session with the MCP server active:
ifc_load("_test_simple.ifc")
ifc_quantify("IFC4QtoBaseQuantities")
ifc_save()
```

Export a BoQ to PDF:

```bash
python3 -c "
import ifcopenshell
from ifc5d.ifc5Dspreadsheet import Ifc5DPdfWriter
f = ifcopenshell.open('_test_simple.ifc')
cs = next(iter(f.by_type('IfcCostSchedule')))
Ifc5DPdfWriter(file=f, output='boq.pdf', options={}, cost_schedule=cs,
               force_schedule_type='PRICEDBILLOFQUANTITIES').write()
"
```

## Git workflow

IFC files are plain text (STEP format) and diff cleanly. Merging requires `ifcmerge` as the git merge driver — see `CLAUDE.md` for configuration.

## Copyright

Bruno Postle <bruno@postle.net> 2026. All files are licensed Creative Commons Attribution-ShareAlike 4.0 International

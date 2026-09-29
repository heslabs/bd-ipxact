# bd-ipxact

Two-way conversion between **Vivado IP Integrator block designs** and
**IEEE 1685-2014 IP-XACT**, driven by a Makefile and bash scripts.

```
               bd2ipxact                              ipxact2bd
 write_bd_tcl ───────────►  IP-XACT (components  ───────────►  BD Tcl script ──► Vivado project
   script                    + designs, XSD-valid)              (write_bd_tcl style)
     ▲                                                                 │
     └──── vivado-export (from .xpr)                 vivado-build ◄────┘
```

The core converters are plain Python (only `lxml` needed); Vivado is only
required to export a BD from a project or to build the regenerated BD.

## Quick start

```bash
make setup          # lxml (auto-creates ./.venv if needed) + download the 1685-2014 schema
make bd2ipxact      # examples/top/top.tcl  ->  build/ipxact/top/*.xml
make validate       # XSD + cross-reference checks
make ipxact2bd      # build/ipxact/top/     ->  build/bd/top.tcl
make test           # lint + full round-trip regression (no Vivado needed)
```

View a design in the browser:

```bash
make view           # build/ipxact/top/ -> build/view/top.html (self-contained, works offline)
make view-open      # same, then open it in the default browser
```

With your own designs:

```bash
make bd2ipxact  BD_TCL=path/to/my_bd.tcl
make from-vivado XPR=proj/proj.xpr BD=my_bd           # needs Vivado
make ipxact2bd  IPXACT_DIR=path/to/ipxact TOP=my_top
make to-vivado  IPXACT_DIR=path/to/ipxact             # needs Vivado
make vivado-verify                                    # rebuild in Vivado, re-export, compare
```

Run `make help` for every target and variable.

### Python environment

`make deps` (part of `make setup`) never installs into the system Python, so
it works on PEP 668 "externally managed" distributions (Debian 12+, Ubuntu
23.04+):

* If `python3` already has `lxml` (e.g. `sudo apt install python3-lxml`), it is used as is.
* Otherwise a project-local `./.venv` is created and `lxml` is installed there.
  All make targets and scripts then use `./.venv/bin/python` automatically, so
  you don't need to pass `PYTHON=` or activate the venv.
* If creating the venv fails because the venv module is missing, install
  `python3.X-venv` (the exact package name is printed) or `python3-lxml`.

Interpreter priority is: `PYTHON=...` if given, then `./.venv`, then `python3`.
`make venv` forces a fresh venv. `make distclean` removes it.

## Layout

| Path | Purpose |
|---|---|
| `Makefile` | entry point for all flows |
| `tools/bd2ipxact.py` | write_bd_tcl script → IP-XACT |
| `tools/ipxact2bd.py` | IP-XACT → write_bd_tcl-style script |
| `tools/validate_ipxact.py` | XSD validation + semantic checks |
| `scripts/bd2ipxact.sh` | forward conversion (from a Tcl file or straight from an `.xpr`) |
| `scripts/ipxact2bd.sh` | reverse conversion, optional `--build` of a Vivado project |
| `scripts/validate_ipxact.sh` | validation wrapper (fetches the schema on first use) |
| `scripts/roundtrip_test.sh` | regression test |
| `scripts/vivado_export_bd.sh`, `vivado/export_bd.tcl` | `.xpr` → `write_bd_tcl` script |
| `scripts/vivado_build_bd.sh`, `vivado/build_bd.tcl` | BD script → new project, HDL wrapper, optional re-export |
| `viewer/ipxact_viewer.html` | interactive IP-XACT viewer (single HTML file, no dependencies) |
| `tools/ipxact_view.py`, `scripts/view_ipxact.sh` | bundle IP-XACT files into a self-contained viewer page |
| `scripts/fetch_schemas.sh` | downloads the IEEE 1685-2014 XSD into `schemas/` |
| `scripts/setup_python.sh`, `scripts/python.sh` | Python/lxml setup (auto `.venv`) and interpreter selection |
| `tests/vivado_stub.tcl` | stand-in for Vivado's BD Tcl API so scripts can execute in plain `tclsh` |
| `tests/fixtures/features.tcl` | fixture covering nested hierarchy, module refs, properties, passthrough |
| `examples/top/top.tcl` | example design (MicroBlaze + DDR4 + Aurora/Chip2Chip, Vivado 2022.2) |

## Scripts directly

```bash
scripts/bd2ipxact.sh  [-v vendor] [-l library] [-V version] [--validate] <bd.tcl> <out_dir>
scripts/bd2ipxact.sh  -x proj.xpr [-b bd_name] <out_dir>
scripts/ipxact2bd.sh  [-t top] [-p part] [--validate] [--build proj_dir] <ipxact_dir> <out.tcl>
scripts/validate_ipxact.sh [--no-schema] <ipxact_dir>
scripts/roundtrip_test.sh [--no-schema] [bd.tcl ...]
```

Every script has `-h`. Override tools with `PYTHON=`, `VIVADO=`, `TCLSH=`.

## Viewer

`viewer/ipxact_viewer.html` is a single HTML file that parses IP-XACT in the
browser. Nothing is uploaded anywhere and no server is needed. There are two
ways to use it:

* Open the file directly and drop IP-XACT files or a whole folder onto it,
  or use *Open files* / *Open folder*.
* Run `make view` (or `scripts/view_ipxact.sh <dir>`) to get a copy with the
  files embedded, which opens straight to the design. This is handy to attach to a
  review or a ticket.

What it shows:

* **Diagram.** A left-to-right block diagram of the current hierarchy level.
  Buses are thick lines colored by protocol (AXI-MM, AXI-Stream, LMB, clock,
  transceiver, DDR, UART, BRAM, debug), wires are thin, and monitor (ILA)
  connections are dashed. Hierarchical blocks have a double outline. Double-click one
  (or use *Open block*) to go inside; the breadcrumb and `U` go back up. Select
  a block, pin, port or line to trace everything it connects to. High-fanout wire
  nets such as clocks and resets are shown as labels to keep the drawing readable.
  The *Wires* switch hides them or draws every one.
* **Interfaces and ports, Instances, Connections, Address map, XML.**
  These tabs give tables of the same data. Rows in Instances and Connections jump
  to the diagram. The address map shows start, end and size per address space,
  including excluded segments.

Which side a pin is drawn on comes from the component definition when it is loaded.
For leaf Xilinx IPs it is otherwise inferred from the connection and the pin name
(the inspector marks inferred directions). For exact pin sides, embed the IPs'
own `component.xml` files. `make view` does this automatically when `XILINX_VIVADO`
is set, or pass `VIVADO_IP=/path/to/Vivado/<ver>/data/ip`. The viewer reads IEEE
1685-2014 fully and the 1685-2009 format Xilinx uses for its IP.

## What the IP-XACT looks like

Each hierarchy level (the BD root and every hierarchical cell) becomes:

* `<name>.xml`: an `ipxact:component` holding the boundary: bus interfaces
  (busType = Vivado bus definition, abstractionRef = the `_rtl` abstraction),
  wire ports, and a `hierarchical` view pointing at the design.
* `<name>.design.xml`: an `ipxact:design` containing the contents.

| Vivado BD | IP-XACT 1685-2014 |
|---|---|
| IP cell + `CONFIG.*` | `componentInstance` → `componentRef` (Xilinx VLNV) + `configurableElementValue` |
| hierarchical cell | `componentInstance` of a generated hierarchical component |
| `connect_bd_intf_net` | `interconnection` (`activeInterface` / `hierInterface`) |
| System ILA slot on a net | `monitorInterconnection` |
| `connect_bd_net` | `adHocConnection` (`internalPortReference` / `externalPortReference`) |
| interface-port `CONFIG` (e.g. `FREQ_HZ`) | bus-interface `parameter` |
| part, Vivado version, address map, clk/rst pin types, port/pin/net properties, module references, inline-HDL cells | `vendorExtensions` in the `vbd:` namespace |
| any Tcl line the parser does not recognize | `vbd:tcl` passthrough (re-emitted verbatim) |

Leaf Xilinx IPs are referenced by VLNV only; their component descriptions are
the `component.xml` files shipped with Vivado.

## Testing without Vivado

`make test` runs, for `examples/*/*.tcl` and `tests/fixtures/*.tcl`:

1. BD Tcl → IP-XACT, then validation against the XSD plus the semantic checks.
2. IP-XACT → BD Tcl → IP-XACT, which must be **byte-identical** to step 1.
3. If `tclsh` is installed, the original and regenerated scripts are both
   **executed** against `tests/vivado_stub.tcl`. The resulting netlists
   (every cell, pin, property, net endpoint and address assignment) must be
   identical.

This is suitable for CI. Add your own designs by putting their `write_bd_tcl`
output in `tests/fixtures/` or under `examples/<name>/`.

## Notes and limitations

* **Input format.** The input must be a `write_bd_tcl` script. `vivado-export`
  and `bd2ipxact.sh -x` produce one from a project. `.bd` JSON files are not
  parsed directly.
* **IP-XACT version.** Output is IEEE 1685-2014. Xilinx's own IP `component.xml`
  files are 1685-2009 (SPIRIT), so tools that load both may need to import them.
* **Parameter IDs.** `configurableElementValue@referenceId` uses Vivado's CONFIG
  names (e.g. `C_AURORA_LANES`). Some tools expect the parameter IDs from the
  referenced `component.xml` (`PARAM_VALUE.C_AURORA_LANES`) and may need a
  mapping step.
* **Third-party IP-XACT input.** `ipxact2bd` also accepts 1685-2014 IP-XACT
  that was not produced by `bd2ipxact`. Without the `vbd:` extensions you
  must supply `--part`, and address assignments fall back to Vivado's
  auto-assignment. Bus interfaces need a Vivado-known abstraction (or a
  busType whose `<name>_rtl` abstraction exists in Vivado's catalog), and
  `tiedValue` connections are flagged with a comment rather than converted.
* **Regenerated scripts.** The Vivado version check only warns instead of
  aborting, so a design can be rebuilt in a newer Vivado. The IP-availability
  check still stops the build if an IP version is missing. The IP check list is
  rebuilt from the actual instances, which also picks up IPs the original
  script omitted (e.g. `axi_interconnect:2.1` in the example).
* **Vivado re-export.** After `vivado-verify`, Vivado may add defaults or
  auto-updated CONFIG values on re-export. The diff it writes shows exactly
  what changed.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pack Mule is a parametric CAD project for a 400 lb payload quadruped robot. **CAD is treated like software**: FreeCAD is the compiler, spec files are source code, and geometric invariants are tests.

## Build & Validation Commands

```bash
# Rebuild the FreeCAD model (headless)
freecadcmd fc/build.py

# Run all validation tests
python tests/test_parts.py         # Part-level checks
python tests/test_interfaces.py    # Interface alignment
python tests/test_assembly.py      # Assembly checks

# Export artifacts
python fc/export.py
```

All tests must pass before any commit.

## Repository Architecture

```
spec/           # SOURCE OF TRUTH - all dimensions, tolerances, interfaces
  params.yaml   # Dimensions and constraints
  interfaces.yaml   # Bolt patterns, datum frames, clearances
  materials.yaml    # Material properties (density, strength)
  machine.yaml      # Parts list, joints, hierarchy

fc/             # Builder scripts (the "code")
  build.py      # Main FreeCAD builder
  parts/        # Per-part builder functions
  assemblies/   # Assembly builders
  checks.py     # Validation functions
  export.py     # STEP/STL/PNG export

models/         # Generated .FCStd files (can regenerate)
out/            # Export artifacts (not version controlled)
tests/          # Validation test suite
```

## Core Development Rules

### The Agent Loop
1. Read `spec/machine.yaml` for current state
2. Make ONE small change (one part or interface)
3. Update `fc/build.py` to implement it
4. Run FreeCAD headless → rebuild model
5. Run all checks (recompute, interfaces, collisions, mass, exports)
6. If FAIL → fix before proceeding
7. If PASS → commit with meaningful message
8. Repeat

### Mandatory Checks (The Gate)
Every commit must pass:
- **Recompute**: FreeCAD opens without errors
- **Interfaces**: Bolt holes align within 0.2mm
- **Collisions**: No intersections at neutral pose
- **Mass**: Total within 350-550 lb (159-250 kg) target
- **CoM**: Projects inside support polygon
- **Exports**: STEP and STL succeed

### Key Constraints
- **One subsystem per iteration** - don't mix changes across subsystems
- **Fix failures before new features** - broken build blocks everything
- **Read dimensions from spec/** - never invent or hard-code values
- **Prefer COTS components** - reduces invention errors
- **No welding** - all bolted construction

### Design Rules
- Min edge distance: 2.5× fastener diameter
- Min thread engagement: 1.5× diameter
- Preferred fasteners: M5, M6, M8, M10
- Min wall thickness: 3mm aluminum
- Min fillet radius: 2mm internal corners
- Min tool access clearance: 25mm

## Phased Development

Development proceeds in validated increments:
1. **Single Part** - Prove pipeline works
2. **Two Parts + Interface** - Prove alignment works
3. **One Articulated Subsystem** - Prove joints work
4. **Multi-Subsystem Integration** - Prove composition
5. **Full Robot** - Complete system

Each phase must be validated before starting the next.

## FreeCAD Python API

Access geometry properties:
```python
shape = part.Shape
volume = shape.Volume          # mm³
com = shape.CenterOfMass       # Vector(x, y, z)
inertia = shape.MatrixOfInertia
```

Mass calculation (materials from `materials.yaml`):
```python
mass = volume_m3 * materials[part.material]['density']
```

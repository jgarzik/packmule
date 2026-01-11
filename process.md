# Pack Mule CAD & Agent Workflow

This document defines the process by which an AI agent (Claude Code) or human develops the Pack Mule robot through parametric CAD. It treats CAD like software: deterministic builds, automated tests, and gated commits.

**This is a living document.** Update it as the project evolves and lessons are learned.

---

## Table of Contents

1. [Philosophy & Core Principles](#1-philosophy--core-principles)
2. [Toolchain Specification](#2-toolchain-specification)
3. [Repository Architecture](#3-repository-architecture)
4. [The Agent Loop](#4-the-agent-loop)
5. [Interfaces & Invariants](#5-interfaces--invariants)
6. [Validation Pipeline](#6-validation-pipeline)
7. [Mass, Thermal & Physics Validation](#7-mass-thermal--physics-validation)
8. [Cable & Harness Management](#8-cable--harness-management)
9. [Phased Development Strategy](#9-phased-development-strategy)
10. [Agent Prompting Discipline](#10-agent-prompting-discipline)
11. [Defense Against Failure](#11-defense-against-failure)
12. [Quick Reference](#12-quick-reference)

---

## 1. Philosophy & Core Principles

### CAD as Code

- All geometry is parametric and version-controlled
- No hard-coded dimensions in sketches
- Every design decision is traceable to a spec file
- CAD files are *generated artifacts*, not hand-edited masters
- If it's not in `spec/`, it doesn't exist

### Single Source of Truth

- **`spec/params.yaml`** contains all dimensions, tolerances, and constraints
- **`spec/materials.yaml`** contains density, strength, thermal properties
- **`spec/interfaces.yaml`** contains bolt patterns, datum frames, clearances
- **`spec/cables.yaml`** contains wire routing, gauges, bend radii
- FreeCAD spreadsheets are populated FROM these files, never edited directly
- Change a dimension in one place → it propagates everywhere

### Top-Down Skeleton Design

- Master layout sketches define the robot envelope and subsystem positions
- Subsystems reference datums from the skeleton, not each other directly
- This prevents circular dependencies and makes refactoring safe

### The Mindset Shift

You are not "designing in CAD." You are writing a program that generates CAD.

- **FreeCAD** is the compiler
- **spec/*.yaml** files are the source code
- **tests/** are geometric invariants
- A failed test means **the build is broken**
- A broken build **blocks all progress**

### Fail Fast, Fail Loud

- Every error must be caught by automated validation
- Silent failures are worse than crashes
- If something looks wrong, stop and investigate
- Never assume "it'll probably be fine"

---

## 2. Toolchain Specification

### Critical: Version Pinning

FreeCAD evolves rapidly; file compatibility shifts between versions. **The toolchain MUST be pinned.** All contributors, CI systems, and the agent must use identical versions.

#### Primary Toolchain (`toolchain.yaml`)

```yaml
# toolchain.yaml - DO NOT EDIT without team discussion
# All versions are mandatory, not suggestions

freecad:
  version: "0.21.2"
  build: "AppImage"  # Use AppImage for reproducibility
  download_url: "https://github.com/FreeCAD/FreeCAD/releases/download/0.21.2/FreeCAD_0.21.2-Linux-x86_64.AppImage"
  sha256: "TO_BE_FILLED_AFTER_PHASE_0"
  
  # Headless invocation command
  headless_cmd: "./FreeCAD.AppImage --console"
  
addons:
  Assembly4:
    version: "0.50.6"
    source: "https://github.com/Zolko-123/FreeCAD_Assembly4"
    
  # CROSS workbench for URDF export - EXPERIMENTAL
  # Test thoroughly before relying on this
  CROSS:
    version: "main"  # Pin to specific commit after validation
    source: "https://github.com/galou/CROSS"
    note: "URDF export is experimental - verify output manually"

python:
  version: "3.10"  # Match FreeCAD's embedded Python
  
  packages:
    - pyyaml>=6.0
    - numpy>=1.24.0
    - pytest>=7.4.0
    - trimesh>=4.0.0      # For STL validation
    - Pillow>=10.0.0      # For PNG generation
    - scipy>=1.11.0       # For geometric calculations
    
container:
  base_image: "ubuntu:22.04"
  dockerfile: "docker/Dockerfile"
```

#### Docker Environment (Required)

All CAD generation and testing runs inside a Docker container to ensure reproducibility.

```dockerfile
# docker/Dockerfile
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && apt-get install -y \
    wget \
    libgl1-mesa-glx \
    libglib2.0-0 \
    python3-pip \
    xvfb \
    && rm -rf /var/lib/apt/lists/*

# Download and setup FreeCAD AppImage
RUN wget -O /opt/FreeCAD.AppImage \
    "https://github.com/FreeCAD/FreeCAD/releases/download/0.21.2/FreeCAD_0.21.2-Linux-x86_64.AppImage" \
    && chmod +x /opt/FreeCAD.AppImage \
    && cd /opt && ./FreeCAD.AppImage --appimage-extract \
    && ln -s /opt/squashfs-root/usr/bin/freecadcmd /usr/local/bin/freecadcmd

# Install Python packages
COPY requirements.txt /tmp/
RUN pip3 install -r /tmp/requirements.txt

# Set up virtual framebuffer for any GUI operations
ENV QT_QPA_PLATFORM=offscreen

WORKDIR /workspace
```

#### Verifying the Toolchain

Before any design work, run:

```bash
make toolchain-check
```

This script must pass:

```python
# scripts/check_toolchain.py
import sys
import subprocess

def check_freecad():
    """Verify FreeCAD version and headless operation"""
    result = subprocess.run(
        ['freecadcmd', '--version'],
        capture_output=True, text=True
    )
    assert '0.21.2' in result.stdout, f"Wrong FreeCAD version: {result.stdout}"
    
    # Test headless operation
    test_script = '''
import FreeCAD
doc = FreeCAD.newDocument("test")
box = doc.addObject("Part::Box", "TestBox")
box.Length = 100
box.Width = 100
box.Height = 100
doc.recompute()
print(f"Volume: {box.Shape.Volume}")
assert abs(box.Shape.Volume - 1000000) < 1, "Volume mismatch"
print("HEADLESS_TEST_PASSED")
'''
    result = subprocess.run(
        ['freecadcmd', '-c', test_script],
        capture_output=True, text=True
    )
    assert 'HEADLESS_TEST_PASSED' in result.stdout, f"Headless test failed: {result.stderr}"
    print("✓ FreeCAD headless operational")

def check_python_packages():
    """Verify required Python packages"""
    import yaml
    import numpy
    import pytest
    import trimesh
    print("✓ Python packages available")

if __name__ == '__main__':
    check_freecad()
    check_python_packages()
    print("\n✓ Toolchain validation PASSED")
```

---

## 3. Repository Architecture

```
packmule/
├── .github/
│   └── workflows/
│       └── validate.yaml       # CI pipeline
├── docker/
│   └── Dockerfile              # Reproducible build environment
├── spec/
│   ├── params.yaml             # All dimensions and tolerances
│   ├── interfaces.yaml         # Bolt patterns, datum frames
│   ├── materials.yaml          # Density, strength, thermal
│   ├── cables.yaml             # Wire routing and specifications
│   ├── actuators.yaml          # Motor/gearbox specifications
│   ├── design_rules.yaml       # Mandatory constraints
│   └── assembly_sequence.yaml  # Build order and dependencies
├── fc/
│   ├── build.py                # Main FreeCAD builder script
│   ├── parts/                  # Per-part builder functions
│   │   ├── __init__.py
│   │   ├── deck_plate.py
│   │   ├── hip_bracket.py
│   │   └── ...
│   ├── assemblies/             # Assembly builder functions
│   │   ├── __init__.py
│   │   ├── leg_assembly.py
│   │   └── ...
│   ├── lib/                    # Shared utilities
│   │   ├── params.py           # YAML → FreeCAD spreadsheet
│   │   ├── export.py           # STEP/STL/PNG export
│   │   ├── validation.py       # Geometric checks
│   │   └── thermal.py          # Thermal calculations
│   └── checks.py               # All validation functions
├── models/
│   └── walker.FCStd            # Generated FreeCAD document
├── out/
│   ├── step/                   # STEP files for manufacturing
│   ├── stl/                    # STL meshes for simulation
│   ├── png/                    # Rendered thumbnails
│   ├── bom/                    # Bill of materials
│   └── urdf/                   # Robot description for simulation
├── tests/
│   ├── test_toolchain.py       # Toolchain validation
│   ├── test_parts.py           # Part-level geometric tests
│   ├── test_interfaces.py      # Interface alignment tests
│   ├── test_assembly.py        # Assembly-level tests
│   ├── test_thermal.py         # Thermal validation
│   └── test_cables.py          # Cable routing validation
├── scripts/
│   ├── check_toolchain.py      # Toolchain verification
│   ├── build_all.py            # Full rebuild
│   └── export_all.py           # Export all artifacts
├── docs/
│   ├── CHANGELOG.md            # What changed and when
│   ├── ISSUES.md               # Known issues and blockers
│   ├── DECISIONS.md            # Design decision log
│   └── ACTUATOR_SELECTION.md   # Actuator comparison matrix
├── toolchain.yaml              # Pinned versions
├── requirements.txt            # Python dependencies
├── Makefile                    # Common commands
└── README.md                   # Project overview
```

### What Goes Where

| Directory | Purpose | Version Control |
|-----------|---------|-----------------|
| `spec/` | Source of truth - these ARE the design | Yes, always |
| `fc/` | Builder scripts - the "code" | Yes, always |
| `models/` | Generated .FCStd files | Optional (can regenerate) |
| `out/` | Export artifacts | No (regenerate in CI) |
| `tests/` | Validation scripts | Yes, always |
| `docs/` | Human-readable documentation | Yes, always |

### Makefile Commands

```makefile
# Makefile
.PHONY: all build test export clean toolchain-check

all: toolchain-check build test export

toolchain-check:
	python3 scripts/check_toolchain.py

build:
	freecadcmd fc/build.py

test:
	pytest tests/ -v

export:
	python3 scripts/export_all.py

clean:
	rm -rf out/* models/*.FCStd

validate: build test
	@echo "✓ Validation passed"

# Run inside Docker
docker-shell:
	docker run -it -v $(PWD):/workspace packmule:latest bash

docker-validate:
	docker run -v $(PWD):/workspace packmule:latest make validate
```

---

## 4. The Agent Loop

This section defines the exact process for the AI agent (Claude Code) to follow. **Deviation from this process is not permitted.**

### 4.1 Session Startup Protocol

**Every session begins with these steps, in order:**

```
1. READ spec/params.yaml completely
2. READ docs/CHANGELOG.md (last 10 entries)
3. READ docs/ISSUES.md for any blockers
4. RUN `make toolchain-check`
5. RUN `make validate` to confirm green build
6. If any step fails → STOP and report to human
```

### 4.2 The Iteration Cycle

```
┌─────────────────────────────────────────────────────────────────┐
│  1. STATE INTENT                                                 │
│     - Write a comment explaining what you're about to change    │
│     - Identify which spec file(s) will be modified              │
│     - Confirm this is ONE atomic change                         │
│                                                                  │
│  2. MODIFY SPEC (if needed)                                     │
│     - Edit ONE spec/*.yaml file                                 │
│     - Validate YAML syntax                                      │
│                                                                  │
│  3. IMPLEMENT                                                   │
│     - Update fc/parts/ or fc/assemblies/ code                   │
│     - Use values from spec, never hardcode                      │
│                                                                  │
│  4. BUILD                                                       │
│     - Run: make build                                           │
│     - Must complete without errors                              │
│                                                                  │
│  5. TEST                                                        │
│     - Run: make test                                            │
│     - ALL tests must pass                                       │
│     - Read failure messages COMPLETELY                          │
│                                                                  │
│  6. If FAIL:                                                    │
│     - Identify root cause                                       │
│     - Fix the issue                                             │
│     - Return to step 4                                          │
│     - If stuck after 3 attempts → invoke STUCK PROTOCOL         │
│                                                                  │
│  7. If PASS:                                                    │
│     - Run: make export                                          │
│     - Visually verify PNG renders                               │
│     - Commit with proper message format                         │
│                                                                  │
│  8. REPEAT                                                      │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Commit Message Format

```
[subsystem] Brief description of change

- Detail 1
- Detail 2

Spec changes: params.yaml (deck_length: 1200 → 1500)
Tests: All passing
```

Example:
```
[frame] Add rear bulkhead with battery mount interface

- Added rear_bulkhead part (12mm aluminum)
- Defined battery_mount interface (M8, 4-hole pattern)
- Updated mass estimate: +4.2 kg

Spec changes: params.yaml (rear_bulkhead_thickness: 12)
Tests: All passing
```

### 4.4 Forbidden Actions

The agent MUST NOT:

| Action | Why It's Forbidden |
|--------|-------------------|
| Edit multiple spec files in one iteration | Breaks atomicity, harder to debug |
| Skip the test step | Defeats the purpose of validation |
| Commit with failing tests | Breaks the build for everyone |
| Invent dimensions not in params.yaml | Violates single source of truth |
| Proceed past a failed build | Compounds errors |
| Make "creative" workarounds without approval | May violate constraints |
| Edit generated files (models/*.FCStd) directly | They will be overwritten |
| Assume something "probably works" | Must verify explicitly |

### 4.5 The Stuck Protocol

**Trigger:** 3+ failed attempts at the same issue, or any situation where progress is blocked.

```
STUCK PROTOCOL
==============

1. STOP making changes immediately

2. DOCUMENT the failure in docs/ISSUES.md:
   
   ## [Date] Issue: Brief title
   
   **Symptom:** What's failing
   **Attempts:** What was tried
   **Error messages:** Exact text
   **Hypothesis:** What might be wrong
   **Blocking:** What can't proceed until resolved

3. ASK the human for guidance:
   - Describe the problem clearly
   - Present options if you have them
   - Wait for response before proceeding

4. DO NOT:
   - Keep trying random variations
   - Make architectural changes to work around it
   - Ignore the issue and move to something else
```

### 4.6 Decision Points Requiring Human Approval

Some decisions are too consequential for agent autonomy:

| Decision Type | Why Human Approval Needed |
|--------------|---------------------------|
| Changing a dimension by >20% | May invalidate other subsystems |
| Adding a new interface type | Affects multiple parts |
| Choosing between design options | Human must own trade-offs |
| Skipping a validation check | May hide real problems |
| Any structural or safety-critical change | Liability |

**Protocol:** Present 2-3 options with pros/cons, wait for selection.

---

## 5. Interfaces & Invariants

Interfaces are the API contracts of mechanical design. They define how parts connect.

### 5.1 Interface Types

#### Bolt Pattern Interfaces

```yaml
# interfaces.yaml
bolt_patterns:
  M8_50mm_square:
    type: bolt_pattern
    fastener: M8
    pattern: square
    spacing: 50  # mm
    hole_diameter: 8.5  # clearance
    counterbore_diameter: 14
    counterbore_depth: 8
    min_edge_distance: 20  # 2.5 × fastener diameter
    
  M10_structural_4x:
    type: bolt_pattern
    fastener: M10
    pattern: rectangular
    spacing_x: 80
    spacing_y: 60
    hole_diameter: 10.5
    min_edge_distance: 25

  leg_mount_M10:
    type: bolt_pattern
    fastener: M10
    pattern: circular
    bolt_circle_diameter: 120
    num_holes: 6
    hole_diameter: 10.5
```

#### Datum Frame Interfaces

```yaml
datum_frames:
  leg_mount_FL:  # Front-left leg
    type: datum_frame
    parent: chassis_origin
    origin: [300, 200, 0]  # mm, relative to parent
    orientation: [0, 0, 0]  # roll, pitch, yaw in degrees
    
  actuator_envelope_hip:
    type: clearance_envelope
    shape: cylinder
    diameter: 140
    length: 200
    keep_out: true  # Nothing may intrude
```

### 5.2 Invariants

Invariants are automated checks that must pass every iteration.

#### Part Invariants

```python
# Every part must satisfy:
def part_invariants(part):
    # 1. Bounding box within specified limits
    assert part.Shape.BoundBox.XLength <= part.max_x
    assert part.Shape.BoundBox.YLength <= part.max_y
    assert part.Shape.BoundBox.ZLength <= part.max_z
    
    # 2. Volume within expected range (catches missing/extra features)
    assert part.min_volume < part.Shape.Volume < part.max_volume
    
    # 3. Hole patterns match declared interface
    for interface in part.interfaces:
        holes = get_interface_holes(part, interface.name)
        assert len(holes) == interface.num_holes
        
    # 4. Recompute without errors
    part.recompute()
    assert not part.isError()
```

#### Interface Invariants

```python
def interface_invariants(part_a, part_b, interface_name):
    holes_a = get_interface_holes(part_a, interface_name)
    holes_b = get_interface_holes(part_b, interface_name)
    
    # All holes must align within tolerance
    for ha, hb in zip(holes_a, holes_b):
        distance = (ha.center - hb.center).Length
        assert distance < 0.2, f"Hole misalignment: {distance}mm > 0.2mm"
```

#### Assembly Invariants

```python
def assembly_invariants(assembly):
    # 1. No collisions at neutral pose
    for part_a, part_b in all_non_mating_pairs(assembly):
        collision = part_a.Shape.common(part_b.Shape)
        assert collision.Volume < 1, f"Collision: {part_a} vs {part_b}"
    
    # 2. Static stability (CoM inside support polygon)
    com = assembly.center_of_mass()
    support_polygon = get_foot_positions(assembly, pose="stand")
    com_2d = Vector(com.x, com.y, 0)
    assert point_in_polygon(com_2d, support_polygon, margin=50)
    
    # 3. Total mass within target
    total_mass_kg = sum(compute_mass(p) for p in assembly.parts)
    assert 159 < total_mass_kg < 250, f"Mass {total_mass_kg}kg outside target"
    
    # 4. All constraints solved
    for constraint in assembly.constraints:
        assert constraint.solved, f"Unsolved: {constraint}"
```

#### Export Invariants

```python
def export_invariants(part):
    # STEP export succeeds
    step_path = export_step(part)
    assert os.path.exists(step_path)
    assert os.path.getsize(step_path) > 1000  # Not empty
    
    # STL is manifold (watertight)
    stl_path = export_stl(part)
    mesh = trimesh.load(stl_path)
    assert mesh.is_watertight, f"{part} STL not watertight"
    assert len(mesh.faces) < 100000, f"{part} mesh too complex"
```

### 5.3 Interface Registry

Every interface used in the design must be registered:

| Interface Name | Used By | Specification |
|----------------|---------|---------------|
| `deck_mount_M8` | Battery tray, electronics mounts | M8, 50mm grid |
| `leg_mount_M10_6x` | Frame-to-hip connection | M10, 6-hole circular |
| `actuator_envelope_hip` | Hip joint module | ∅140 × 200mm cylinder |
| `actuator_envelope_knee` | Knee joint module | ∅120 × 180mm cylinder |
| `battery_bay` | Battery pack | 400×300×150mm clearance |
| `cable_pass_hip` | Leg power/signal cables | ∅25mm grommet |

---

## 6. Validation Pipeline

A change is **invalid** unless ALL checks pass. This is the gate.

### 6.1 Test Categories

#### A) Toolchain Tests (Always First)

```python
# tests/test_toolchain.py
def test_freecad_version():
    """Verify FreeCAD version matches toolchain.yaml"""
    import FreeCAD
    assert '0.21' in FreeCAD.Version()[0]

def test_headless_operation():
    """Verify FreeCAD can create geometry without GUI"""
    doc = FreeCAD.newDocument("test")
    box = doc.addObject("Part::Box", "Box")
    box.Length = 100
    doc.recompute()
    assert abs(box.Shape.Volume - 1000000) < 1
```

#### B) Part-Level Tests

```python
# tests/test_parts.py
def test_deck_plate():
    part = get_part("deck_plate")
    params = load_params()
    
    # Dimensions match spec
    assert part.Shape.BoundBox.XLength == approx(params['deck_length'], abs=1)
    assert part.Shape.BoundBox.YLength == approx(params['deck_width'], abs=1)
    
    # Hole pattern present
    holes = get_holes(part)
    assert len(holes) >= params['deck_mount_holes']
    
    # Volume sanity (catches missing features)
    expected_volume = (params['deck_length'] * params['deck_width'] * 
                       params['deck_thickness'])
    assert part.Shape.Volume == approx(expected_volume, rel=0.2)
    
    # No recompute errors
    part.recompute()
    assert not part.isError()
```

#### C) Interface Tests

```python
# tests/test_interfaces.py
def test_battery_mount_alignment():
    deck = get_part("deck_plate")
    bracket = get_part("battery_bracket")
    
    deck_holes = get_interface_holes(deck, "battery_mount")
    bracket_holes = get_interface_holes(bracket, "base_mount")
    
    assert len(deck_holes) == len(bracket_holes), "Hole count mismatch"
    
    for d, b in zip(deck_holes, bracket_holes):
        distance = (d.center - b.center).Length
        assert distance < 0.2, f"Misalignment: {distance}mm"
```

#### D) Assembly Tests

```python
# tests/test_assembly.py
def test_no_collisions_neutral():
    assembly = get_assembly("walker")
    
    non_touching_pairs = get_non_mating_pairs(assembly)
    for part_a, part_b in non_touching_pairs:
        collision = part_a.Shape.common(part_b.Shape)
        assert collision.Volume < 1, f"Collision: {part_a.Label} vs {part_b.Label}"

def test_static_stability():
    assembly = get_assembly("walker")
    com = compute_assembly_com(assembly)
    feet = get_foot_positions(assembly, pose="stand")
    support = convex_hull_2d(feet)
    
    assert point_in_polygon((com.x, com.y), support, margin=50)
```

#### E) Export Tests

```python
# tests/test_export.py
def test_all_parts_export():
    for part in get_all_parts():
        # STEP export
        step_path = f"out/step/{part.Label}.step"
        export_step(part, step_path)
        assert os.path.exists(step_path)
        
        # STL export and validation
        stl_path = f"out/stl/{part.Label}.stl"
        export_stl(part, stl_path)
        mesh = trimesh.load(stl_path)
        assert mesh.is_watertight, f"{part.Label} not watertight"
```

### 6.2 Running the Suite

```bash
# After every change:
make validate

# Which runs:
freecadcmd fc/build.py          # Rebuild model
pytest tests/ -v                 # All checks
python3 scripts/export_all.py   # Generate artifacts
```

**All must pass before commit.**

---

## 7. Mass, Thermal & Physics Validation

For a walking robot, physics checks are survival checks.

### 7.1 Material Properties

```yaml
# spec/materials.yaml
aluminum_6061_T6:
  density: 2700           # kg/m³
  yield_strength: 276     # MPa
  elastic_modulus: 68900  # MPa
  thermal_conductivity: 167  # W/(m·K)
  max_service_temp: 150   # °C

steel_4140:
  density: 7850
  yield_strength: 655
  elastic_modulus: 205000
  thermal_conductivity: 42.6
  max_service_temp: 400

# For fasteners
steel_class_10_9:
  yield_strength: 940
  tensile_strength: 1040
```

### 7.2 Mass Tracking

Every iteration must report mass status:

```python
# fc/lib/mass.py
def mass_report(assembly):
    """Generate mass breakdown and validate against targets"""
    params = load_params()
    materials = load_materials()
    
    subsystems = {
        'frame': [],
        'legs': [],
        'actuators': [],
        'power': [],
        'electronics': [],
        'other': []
    }
    
    for part in assembly.parts:
        mass_kg = compute_mass(part, materials)
        subsystem = get_subsystem(part)
        subsystems[subsystem].append((part.Label, mass_kg))
    
    # Print breakdown
    total = 0
    print("\n=== MASS REPORT ===")
    for subsystem, parts in subsystems.items():
        sub_total = sum(m for _, m in parts)
        total += sub_total
        print(f"\n{subsystem.upper()}: {sub_total:.1f} kg")
        for label, mass in parts:
            print(f"  {label}: {mass:.2f} kg")
    
    print(f"\n{'='*30}")
    print(f"TOTAL: {total:.1f} kg ({total * 2.205:.0f} lb)")
    print(f"TARGET: {params['target_mass_min']}-{params['target_mass_max']} kg")
    
    # Validation
    assert params['target_mass_min'] < total < params['target_mass_max'], \
        f"Mass {total:.1f}kg outside target range"
    
    return total
```

### 7.3 Center of Mass Check

```python
def check_static_stability(assembly, pose="stand"):
    """Verify CoM projects inside support polygon with margin"""
    com = compute_assembly_com(assembly)
    feet = get_foot_positions(assembly, pose)
    support = convex_hull_2d([(f.x, f.y) for f in feet])
    
    margin = 50  # mm safety margin from edge
    
    com_2d = (com.x, com.y)
    
    if not point_in_polygon_with_margin(com_2d, support, margin):
        # Calculate how far outside
        distance_to_edge = distance_to_polygon_edge(com_2d, support)
        raise AssertionError(
            f"CoM at ({com.x:.0f}, {com.y:.0f}) is {distance_to_edge:.0f}mm "
            f"outside support polygon (need {margin}mm margin)"
        )
    
    print(f"✓ CoM stability check passed (margin: {margin}mm)")
```

### 7.4 Joint Torque Estimation

```python
def estimate_joint_torques(assembly, pose="stand"):
    """First-pass actuator sizing using static torque analysis"""
    params = load_params()
    total_mass = compute_total_mass(assembly) + params['max_payload_kg']
    
    # During trot, 2 legs support full weight
    per_leg_force = (total_mass * 9.81) / 2  # N
    
    torques = {}
    for leg in ['FL', 'FR', 'RL', 'RR']:
        leg_geom = get_leg_geometry(assembly, leg, pose)
        
        # Knee torque = force × horizontal distance to foot
        knee_moment_arm = leg_geom.knee_to_foot_horizontal
        torques[f'{leg}_knee'] = per_leg_force * knee_moment_arm / 1000  # N·m
        
        # Hip pitch similar calculation
        hip_moment_arm = leg_geom.hip_to_foot_horizontal
        torques[f'{leg}_hip_pitch'] = per_leg_force * hip_moment_arm / 1000
        
        # Hip ab/ad (lateral stability)
        # Simplified: ~30% of pitch torque for lateral forces
        torques[f'{leg}_hip_abad'] = torques[f'{leg}_hip_pitch'] * 0.3
    
    # Apply safety factor for dynamics
    safety_factor = 2.5  # Walking dynamics can be 2-3× static
    
    print("\n=== JOINT TORQUE ESTIMATES ===")
    print(f"Total loaded mass: {total_mass:.0f} kg")
    print(f"Safety factor: {safety_factor}×\n")
    
    for joint, torque in torques.items():
        dynamic_torque = torque * safety_factor
        print(f"{joint}: {torque:.0f} N·m static → {dynamic_torque:.0f} N·m dynamic")
    
    return {k: v * safety_factor for k, v in torques.items()}
```

### 7.5 Thermal Analysis

**Critical for continuous operation.** Motors overheat.

```yaml
# spec/actuators.yaml
actuators:
  hip_pitch:
    model: "CubeMars AK80-64"  # Example
    continuous_torque: 48      # N·m
    peak_torque: 120           # N·m
    no_load_speed: 8.3         # rad/s
    thermal_resistance: 1.2    # °C/W (winding to case)
    max_winding_temp: 120      # °C
    mass: 0.95                 # kg
    efficiency_map:            # Efficiency vs torque %
      0: 0.90
      50: 0.85
      100: 0.75
```

```python
# tests/test_thermal.py
def test_actuator_thermal():
    """Verify actuators won't overheat during continuous operation"""
    actuators = load_actuators()
    params = load_params()
    torques = estimate_joint_torques(get_assembly("walker"))
    
    ambient_temp = params.get('max_ambient_temp', 40)  # °C
    
    for joint, required_torque in torques.items():
        actuator = actuators.get(joint_to_actuator(joint))
        if actuator is None:
            continue
            
        # Estimate efficiency at operating point
        torque_percent = (required_torque / actuator['peak_torque']) * 100
        efficiency = interpolate(actuator['efficiency_map'], torque_percent)
        
        # Continuous power at this torque (assuming average speed)
        avg_speed = 1.0  # rad/s for walking
        mechanical_power = required_torque * avg_speed
        electrical_power = mechanical_power / efficiency
        heat_dissipation = electrical_power - mechanical_power
        
        # Temperature rise
        temp_rise = heat_dissipation * actuator['thermal_resistance']
        winding_temp = ambient_temp + temp_rise
        
        print(f"{joint}: {winding_temp:.0f}°C (max: {actuator['max_winding_temp']}°C)")
        
        assert winding_temp < actuator['max_winding_temp'] * 0.9, \
            f"{joint} will overheat: {winding_temp:.0f}°C > " \
            f"{actuator['max_winding_temp'] * 0.9:.0f}°C (90% limit)"
```

---

## 8. Cable & Harness Management

Cable failures are a top cause of robot downtime. This section prevents them.

### 8.1 Cable Specifications

```yaml
# spec/cables.yaml
cable_types:
  motor_power_10awg:
    type: stranded_copper
    gauge: 10 AWG
    insulation: silicone
    min_bend_radius: 25  # mm
    max_current: 40      # A
    temp_rating: 200     # °C
    
  signal_can:
    type: shielded_twisted_pair
    gauge: 22 AWG
    insulation: PVC
    min_bend_radius: 15
    shield: braided_copper
    
  encoder_cable:
    type: shielded_4_conductor
    gauge: 26 AWG
    min_bend_radius: 10

cable_runs:
  leg_FL_power:
    cable_type: motor_power_10awg
    path:
      - chassis_power_bus
      - FL_hip_grommet
      - FL_hip_actuator
      - FL_knee_grommet
      - FL_knee_actuator
    length_estimate: 1200  # mm
    
  leg_FL_signal:
    cable_type: signal_can
    path:
      - chassis_controller
      - FL_hip_grommet
      - FL_hip_encoder
      - FL_knee_grommet
      - FL_knee_encoder
    length_estimate: 1400

grommets:
  FL_hip_grommet:
    location: [300, 200, 50]  # Relative to chassis origin
    inner_diameter: 25
    cables: [leg_FL_power, leg_FL_signal]
```

### 8.2 Cable Routing Validation

```python
# tests/test_cables.py
def test_cable_bend_radius():
    """Verify no cable bend radius violations"""
    cables = load_cables()
    assembly = get_assembly("walker")
    
    for run_name, run in cables['cable_runs'].items():
        cable_type = cables['cable_types'][run['cable_type']]
        min_radius = cable_type['min_bend_radius']
        
        path_points = [get_point(assembly, p) for p in run['path']]
        
        for i in range(1, len(path_points) - 1):
            p0, p1, p2 = path_points[i-1:i+2]
            radius = compute_bend_radius(p0, p1, p2)
            
            assert radius >= min_radius, \
                f"{run_name}: Bend radius {radius:.0f}mm < {min_radius}mm at {run['path'][i]}"

def test_cable_clearance():
    """Verify cables don't pass through geometry"""
    cables = load_cables()
    assembly = get_assembly("walker")
    
    for run_name, run in cables['cable_runs'].items():
        path_points = [get_point(assembly, p) for p in run['path']]
        
        # Check each segment
        for i in range(len(path_points) - 1):
            segment = (path_points[i], path_points[i+1])
            
            for part in assembly.parts:
                if intersects_geometry(segment, part.Shape, margin=5):
                    raise AssertionError(
                        f"{run_name}: Segment {run['path'][i]} → {run['path'][i+1]} "
                        f"intersects {part.Label}"
                    )

def test_grommet_capacity():
    """Verify grommets can fit all assigned cables"""
    cables = load_cables()
    
    for grommet_name, grommet in cables['grommets'].items():
        total_area = 0
        for cable_name in grommet['cables']:
            run = cables['cable_runs'][cable_name]
            cable_type = cables['cable_types'][run['cable_type']]
            # Estimate cable diameter from gauge
            diameter = gauge_to_diameter(cable_type['gauge'])
            total_area += 3.14159 * (diameter/2)**2
        
        available_area = 3.14159 * (grommet['inner_diameter']/2)**2
        fill_ratio = total_area / available_area
        
        # Max 40% fill for flexibility and heat dissipation
        assert fill_ratio < 0.4, \
            f"{grommet_name}: Fill ratio {fill_ratio:.0%} > 40%"
```

### 8.3 Cable Routing Design Rules

```yaml
# spec/design_rules.yaml (cable section)
cables:
  min_bend_radius_factor: 5     # × cable diameter minimum
  max_grommet_fill: 0.4         # 40% of grommet area
  strain_relief: required       # At every entry/exit point
  no_sharp_edges: true          # Cable must not touch sharp corners
  service_loop: 50              # mm extra at each termination
  
  routing_zones:
    primary:   frame_interior   # Preferred: protected inside frame
    secondary: along_leg_links  # Acceptable: follow structural members
    forbidden: across_joints    # Never: cables spanning joint rotation
```

---

## 9. Phased Development Strategy

**If you attempt "full walker" on day 1, the agent will thrash.** Complexity must be earned through validated increments.

### Phase 0: Toolchain Validation (1-2 weeks)

**Goal:** Prove the development pipeline works before designing anything.

**Deliverables:**
- [ ] Docker container with pinned FreeCAD version
- [ ] Script that creates a parametric box from params.yaml
- [ ] Script that exports STEP, STL, PNG
- [ ] Script that validates dimensions
- [ ] CI pipeline (GitHub Actions) running full validation
- [ ] 10 consecutive green builds

**Exit Criteria:**
```bash
make docker-validate  # Passes 10 times in a row
```

**Explicitly NOT in scope:**
- Any robot geometry
- Any design decisions
- Any actuator selection

### Phase 1: Single Part (1-2 weeks)

**Goal:** Prove parametric part generation works end-to-end.

**Example:** Deck plate

**Deliverables:**
- [ ] `spec/params.yaml` with deck dimensions
- [ ] `fc/parts/deck_plate.py` builder script
- [ ] Part generates headlessly from params
- [ ] Part-level tests pass (dimensions, holes, volume)
- [ ] STEP, STL, PNG export successfully
- [ ] Mass calculation matches expected

**Exit Criteria:**
- One part passes all tests
- Changing a parameter in `params.yaml` propagates correctly
- Export artifacts are valid

### Phase 2: Two Parts + One Interface (1-2 weeks)

**Goal:** Prove interface alignment validation works.

**Example:** Deck plate + battery bracket

**Deliverables:**
- [ ] Second part builder script
- [ ] `spec/interfaces.yaml` with `battery_mount` interface
- [ ] Both parts declare they use this interface
- [ ] Interface alignment test passes (<0.2mm)
- [ ] Visual verification: holes line up in PNG render

**Exit Criteria:**
- Two parts with a shared interface
- Interface alignment test catches deliberate misalignment
- Parts would physically bolt together correctly

### Phase 3: One Articulated Subsystem in CAD (3-4 weeks)

**Goal:** Prove joints, assemblies, and range-of-motion work.

**Example:** Single front-left leg

**Deliverables:**
- [ ] Hip ab/ad + hip pitch + knee pitch joints
- [ ] Thigh link + shin link parts
- [ ] Actuator envelope interfaces (clearance volumes)
- [ ] Assembly with joint constraints
- [ ] Range of motion check (no self-collision)
- [ ] Joint torque estimates
- [ ] URDF export for this leg
- [ ] PyBullet smoke test: loads and joints move

**Exit Criteria:**
- One leg assembles correctly
- Joints articulate through full range
- No collisions at any pose in range
- URDF loads in simulator

### Phase 3.5: Single Leg Hardware Validation (4-8 weeks)

**Goal:** Validate actuator selection with physical hardware before committing to 12 units.

**THIS PHASE IS CRITICAL.** Do not skip it.

**Deliverables:**
- [ ] Order 3 actuators (one leg's worth)
- [ ] Build simplified leg structure (can be rough prototype)
- [ ] Bench test rig with load cell
- [ ] Test: Static torque at operating points
- [ ] Test: Dynamic cycling (1 hour continuous)
- [ ] Test: Thermal monitoring (actuator case temperature)
- [ ] Document: Actual vs. spec performance

**Exit Criteria:**
- Actuators deliver required torque at operating points
- No overheating during 1-hour cycling at realistic load
- Mechanical interfaces fit as designed
- Go/no-go decision on actuator selection

**If actuators fail validation:**
- Return to actuator selection
- DO NOT proceed to Phase 4
- Update spec/actuators.yaml with learnings

### Phase 4: Frame + Two Legs in CAD (3-4 weeks)

**Goal:** Prove multi-subsystem integration and partial stability.

**Deliverables:**
- [ ] Frame structure (simplified if needed)
- [ ] Two legs attached (diagonal pair recommended)
- [ ] Leg mount interfaces on frame
- [ ] Static stability check with two legs
- [ ] Collision checks between legs and frame
- [ ] Mass and CoM tracking
- [ ] Cable routing for two legs

**Exit Criteria:**
- Partial robot is stable (CoM in support polygon)
- No collisions in standing pose
- Mass tracking is realistic
- Cable paths are feasible

### Phase 5: Full Robot CAD (4-6 weeks)

**Goal:** Complete system design, validated and ready for fabrication.

**Deliverables:**
- [ ] All four legs attached
- [ ] Power system layout (battery, BMS, distribution)
- [ ] Control system mounts (computer, motor controllers)
- [ ] Payload deck with tie-downs
- [ ] Full cable harness routing
- [ ] Full mass and CoM validation
- [ ] Full thermal validation
- [ ] Complete URDF
- [ ] Simulation: robot stands in PyBullet/Gazebo

**Exit Criteria:**
- All validation tests pass
- Mass within target range
- CoM stable in all poses
- All cables routed with margin
- BOM complete with prices
- Ready for fabrication quote

### Phase 6: Full Robot Hardware (8-12 weeks)

**Goal:** Physical robot that stands under its own power.

**Deliverables:**
- [ ] Order remaining actuators (9 more)
- [ ] Order frame fabrication (laser cut + machined parts)
- [ ] Order all COTS components
- [ ] Receive and inspect all parts
- [ ] Assembly following documented sequence
- [ ] Electronics integration
- [ ] Initial power-on with safety constraints
- [ ] Stand test: robot supports own weight

**Exit Criteria:**
- Robot is fully assembled
- All systems power on
- Robot stands on all four legs
- No structural failures
- Ready for walking development

### Phase 7: Walking (Ongoing)

**Goal:** Locomotion on target terrain with target payload.

**Deliverables:**
- [ ] Basic standing balance controller
- [ ] Simple walking gait (flat ground, no payload)
- [ ] Payload testing (incremental loading)
- [ ] Terrain testing (incremental difficulty)
- [ ] Iteration based on failures

**Exit Criteria:**
- 150+ lb payload at 1 mph on packed dirt (first milestone)
- Eventually: 400 lb payload at 1-3 mph on logging trails

---

## 10. Agent Prompting Discipline

When Claude Code or another agent is doing CAD work, these rules keep it on track.

### 10.1 System Context

The agent should always have access to:

```markdown
## Context Files (Load at Session Start)

1. spec/params.yaml - Current dimensions
2. spec/interfaces.yaml - Connection standards
3. docs/CHANGELOG.md - Recent history
4. docs/ISSUES.md - Known blockers
5. This process document
```

### 10.2 Good Prompts vs Bad Prompts

**Good prompts are specific and constrained:**

```
Add a hip bracket part that:
- Uses interface `leg_mount_M10_6x` for frame connection
- Uses interface `actuator_envelope_hip` for motor clearance
- Stays within 150×150×80mm bounding box
- Uses 10mm aluminum 6061-T6 plate
- Includes 4× corner gussets with 3mm fillet
- Has mass < 2.5 kg
```

**Bad prompts are vague:**

```
Design the hip area.
```

**Bad prompts assume capabilities:**

```
Make it strong enough.
```

(Strong enough for what? This requires FEM validation.)

### 10.3 The Three-Option Rule

For any mounting, structural, or architectural decision, require the agent to propose three options:

**Example: Battery retention method**

```
Option A: Bolted tray with strap
- Pros: Simple, easy access, proven
- Cons: Straps can loosen, requires tools
- Mass impact: +0.8 kg
- Cost: $50

Option B: Slide-in rails with spring latch
- Pros: Tool-less removal, positive retention
- Cons: More complex, tighter tolerances
- Mass impact: +1.2 kg
- Cost: $120

Option C: Clamshell enclosure with captive screws
- Pros: Weather sealed, secure
- Cons: Slow access, heavy
- Mass impact: +2.1 kg
- Cost: $200

Recommendation: Option A for initial prototype (simplicity)
```

Human picks one. Agent implements it. This prevents the LLM from committing to the first plausible idea.

### 10.4 Standard Patterns to Prefer

Bias toward known, validated patterns to reduce "invented" details:

| Category | Preferred Patterns |
|----------|-------------------|
| Brackets | L-brackets with gussets, laser-cut plate with bends |
| Structure | Aluminum plate sandwich, tube + plate hybrid |
| Electronics mounting | DIN rail, 80/20 extrusion, standoffs on plate |
| Cable management | Grommeted pass-throughs, cable trays, strain relief clamps |
| Fasteners | M5/M6/M8/M10 only, Class 10.9 for structural |
| Sealing | Rubber grommets, IP65 connectors, RTV at seams |

### 10.5 Phrases That Should Trigger Caution

If the agent finds itself thinking:

| Phrase | What To Do Instead |
|--------|-------------------|
| "This should work" | Verify with a test |
| "I'll assume..." | Read from spec or ask |
| "Probably fine" | Validate explicitly |
| "We can fix this later" | Fix it now |
| "It's close enough" | Check tolerance spec |
| "I'll make this more elegant" | Prefer boring and proven |

---

## 11. Defense Against Failure

LLMs produce designs that *look* plausible but violate hidden constraints.

### 11.1 Common Failure Modes

| Failure | What Happens | Defense |
|---------|--------------|---------|
| Bracket too thin near bolt | Stress concentration → crack | Min edge distance rule + FEM |
| No wrench access | Can't assemble | Assembly sequence check |
| Cable rub point | Chafe → short circuit | Cable routing validation |
| Assembly sequence impossible | Parts can't be installed | Manual sequence review |
| Sharp internal corner | Stress riser | Fillet rules + FEM |
| Thermal runaway | Motor overheats | Thermal validation |
| CoM outside support | Robot tips over | Stability check |
| Interface misalignment | Parts don't fit | Interface tolerance test |

### 11.2 Design Rules (Mandatory)

```yaml
# spec/design_rules.yaml
fasteners:
  min_edge_distance: 2.5    # × fastener diameter
  min_thread_engagement: 1.5  # × fastener diameter
  preferred_sizes: [M5, M6, M8, M10]
  thread_locker: required_for_vibration
  
structure:
  min_wall_thickness: 3     # mm, aluminum
  min_fillet_radius: 2      # mm, internal corners
  max_unsupported_span: 300 # mm, for 8mm plate
  
clearances:
  min_tool_access: 25       # mm for wrench/driver
  min_cable_channel: 15     # mm diameter
  min_moving_part_gap: 5    # mm between moving parts
  
thermal:
  max_ambient_temp: 40      # °C design condition
  actuator_derating: 0.9    # 90% of max temp
  
electrical:
  max_voltage_drop: 0.5     # V at full load
  wire_derating: 0.8        # 80% of rated current
```

The agent MUST check designs against these rules.

### 11.3 Validation Promotion Schedule

**Phase 0-2 (Early):**
- Geometry + dimensions
- Interface alignment
- Export validity

**Phase 3-4 (Middle):**
- Mass + CoM tracking
- Joint range of motion
- Cable routing feasibility
- Static torque estimates

**Phase 5+ (Later):**
- Full thermal analysis
- FEM stress analysis (critical parts)
- Assembly sequence verification
- Simulation smoke tests

**Always:**
- Physical prototyping for load-bearing parts
- Human review for safety-critical decisions

### 11.4 The Ultimate Defense

**Anything load-bearing or safety-critical gets:**

1. FEM analysis (after geometry is stable)
2. Physical prototyping (before committing to full quantity)
3. Human engineering review (always)

No amount of automated testing replaces physical validation for a 400 lb payload walking robot.

---

## 12. Quick Reference

### The Gate (Every Commit)

| Check | Command | Pass Condition |
|-------|---------|----------------|
| Toolchain | `make toolchain-check` | Version match |
| Build | `make build` | No errors |
| Tests | `make test` | All pass |
| Export | `make export` | STEP, STL, PNG created |

A change is **invalid** unless all checks pass.

### Key File Locations

| File | Purpose |
|------|---------|
| `spec/params.yaml` | All dimensions - THE source of truth |
| `spec/interfaces.yaml` | How parts connect |
| `spec/materials.yaml` | Physical properties |
| `spec/actuators.yaml` | Motor specifications |
| `spec/cables.yaml` | Wire routing |
| `spec/design_rules.yaml` | Mandatory constraints |
| `docs/CHANGELOG.md` | What changed |
| `docs/ISSUES.md` | Known blockers |
| `toolchain.yaml` | Pinned versions |

### Agent Quick Commands

```bash
# Start of session
make toolchain-check && make validate

# After each change
make validate

# If stuck
# 1. Document in docs/ISSUES.md
# 2. Ask human for guidance
```

### Emergency Procedures

**Build won't pass:**
1. Read error message completely
2. Check docs/ISSUES.md for known problems
3. Try `make clean && make build`
4. If persists → Stuck Protocol

**Test regression (was passing, now fails):**
1. Check git diff for unintended changes
2. Check params.yaml for drift
3. Revert to last known good commit
4. Investigate cause

**Agent is looping:**
1. Human intervention required
2. Stop agent
3. Review last 5 commits
4. Identify pattern
5. Add explicit rule to prevent recurrence

---

## Summary

This process treats robot CAD like software engineering:

- **Spec files** are source code
- **FreeCAD** is the compiler
- **Tests** are geometric invariants
- A failed test means **the build is broken**
- Commits require **passing the gate**

The phased approach prevents thrashing. Interfaces enable scaling. Validation prevents "beautiful CAD, painful reality."

The Pack Mule will be built one validated increment at a time.

---

*Document version: 2.0*
*Last updated: [DATE]*
*Process owner: [NAME]*

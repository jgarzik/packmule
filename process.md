# Pack Mule CAD & Agent Workflow

This document defines the process by which an AI agent (or human) develops the Pack Mule robot through parametric CAD. It treats CAD like software: deterministic builds, automated tests, and gated commits.

---

## 1. Philosophy & Core Principles

### CAD as Code
- All geometry is parametric and version-controlled
- No hard-coded dimensions in sketches
- Every design decision is traceable to a spec file
- CAD files are *generated artifacts*, not hand-edited masters

### Single Source of Truth
- **`spec/params.yaml`** contains all dimensions, tolerances, and constraints
- **`SS_Params` spreadsheet** in FreeCAD is populated from `params.yaml`
- Parts reference the spreadsheet, never magic numbers
- Change a dimension in one place, it propagates everywhere

### Top-Down Skeleton Design
- Master layout sketches define the robot envelope and subsystem positions
- Subsystems reference datums from the skeleton, not each other directly
- This prevents circular dependencies and makes refactoring safe

### The Mindset Shift
You are not "designing in CAD." You are writing a program that generates CAD. FreeCAD is the compiler. Tests are geometric invariants. A failed test means the build is broken.

---

## 2. FreeCAD as Deterministic Compiler

### Headless Execution
FreeCAD runs without the GUI via `FreeCADCmd` or `freecad -c`. The agent loop is:

```
edit spec/params.yaml or fc/build.py
    → run FreeCAD headless
    → generate/update .FCStd model
    → export artifacts (STEP, STL, PNG)
    → run checks
    → if pass: commit
    → if fail: fix and repeat
```

This is the CAD equivalent of `edit → compile → test → commit`.

### What FreeCAD Provides
- Real parametric modeling (sketches, constraints, feature tree)
- Assembly workbench for multi-part systems
- Python API for scripted model generation
- FEM workbench (CalculiX) for structural analysis
- Volume, Center of Mass, Inertia tensor calculations
- STEP/STL/3MF export

### Version Pinning (Critical)
FreeCAD evolves rapidly; file compatibility can shift between versions. **You must pin your toolchain:**
- Container with specific FreeCAD build, or
- Documented version + addon versions
- All contributors and the agent must use the same version

### Path to Simulation
When ready for robotics simulation:
- Use **CROSS workbench** to generate URDF/xacro from FreeCAD assemblies
- Export meshes as STL/DAE for collision/visual geometry
- Simulate in PyBullet, Gazebo, or Isaac Sim

---

## 3. Repository Architecture

```
packmule/
  spec/
    params.yaml           # dimensions, tolerances, constraints
    interfaces.yaml       # bolt patterns, datum frames, clearances
    materials.yaml        # density, elastic modulus, yield strength
    machine.yaml          # parts list, joints, subsystem hierarchy
  fc/
    build.py              # main FreeCAD builder script
    parts/                # per-part builder functions
    assemblies/           # assembly builder functions
    checks.py             # validation functions
    export.py             # STEP/STL/PNG export
  models/
    walker.FCStd          # generated FreeCAD document (can regenerate)
  out/
    step/                 # STEP files for manufacturing
    stl/                  # STL meshes for simulation/printing
    png/                  # rendered thumbnails for visual diff
    bom/                  # bill of materials CSV
    urdf/                 # robot description for simulation
  tests/
    test_parts.py         # part-level geometric tests
    test_interfaces.py    # interface alignment tests
    test_assembly.py      # assembly-level tests
```

### What Goes Where
| Directory | Purpose | Version Control |
|-----------|---------|-----------------|
| `spec/` | Source of truth | Yes - these ARE the design |
| `fc/` | Builder scripts | Yes - this is the "code" |
| `models/` | Generated .FCStd | Optional - can regenerate |
| `out/` | Export artifacts | No - regenerate in CI |
| `tests/` | Validation scripts | Yes - these enforce quality |

---

## 4. The Agent Loop (Core Process)

### The Iteration Cycle

```
┌─────────────────────────────────────────────────────┐
│  1. Read spec/machine.yaml (current state)          │
│  2. Make ONE small change (one part or interface)   │
│  3. Update fc/build.py to implement it              │
│  4. Run FreeCAD headless → rebuild model            │
│  5. Run checks:                                     │
│     - Recompute without errors                      │
│     - Interface alignment within tolerance          │
│     - No collisions at neutral pose                 │
│     - Mass + CoM within bounds                      │
│     - Export succeeds (STEP, STL, PNG)              │
│  6. If FAIL → fix before proceeding                 │
│  7. If PASS → commit with meaningful message        │
│  8. Repeat                                          │
└─────────────────────────────────────────────────────┘
```

### Rules for the Agent

1. **One subsystem per iteration.** Don't add the hip joint while also redesigning the deck.

2. **Fix failures before adding features.** A broken build blocks everything.

3. **Every change must pass the gate.** No "I'll fix it later."

4. **Use the spec as truth.** Don't invent dimensions; read them from `params.yaml`.

5. **Commit atomically.** Each commit should be a working state.

### What "Compile" Means in CAD

A successful compile means:
- FreeCAD opens the document without errors
- All constraints solve
- No broken references
- Recompute produces consistent geometry

---

## 5. Interfaces & Invariants (The Secret Sauce)

This is what makes "CAD by conversation" possible instead of "CAD by vibes."

### Interfaces

Interfaces are the API contracts of mechanical design. They define how parts connect.

**Bolt Pattern Interfaces:**
```yaml
# interfaces.yaml
M8_50mm_square:
  type: bolt_pattern
  fastener: M8
  pattern: square
  spacing: 50mm
  hole_diameter: 8.5mm  # clearance
  counterbore: 14mm dia, 8mm deep

M10_structural:
  type: bolt_pattern
  fastener: M10
  pattern: linear
  spacing: 75mm
  hole_diameter: 10.5mm
```

**Datum Frame Interfaces:**
```yaml
leg_mount_frame:
  type: datum_frame
  origin: [x, y, z]  # relative to chassis datum
  orientation: [roll, pitch, yaw]

actuator_envelope:
  type: clearance_envelope
  shape: cylinder
  diameter: 120mm
  length: 200mm
```

### Invariants

Invariants are automated checks that must pass every iteration.

**Part Invariants:**
- Bounding box within specified limits
- Volume within expected range (catches missing features)
- Hole patterns match declared interface
- Recompute without errors

**Interface Invariants:**
- Bolt holes align within 0.2mm across mating parts
- Datum frames match within tolerance
- Clearance envelopes remain empty (no intersections)

**Assembly Invariants:**
- No collisions at neutral pose
- CoM projects inside support polygon (static stability)
- Total mass within target range (350-550 lb for Pack Mule)
- All constraints solve; zero recompute failures

**Export Invariants:**
- STEP export succeeds
- STL is manifold (watertight)
- Mesh triangle count under budget

### Pack Mule Interface Examples

| Interface | Used By | Specification |
|-----------|---------|---------------|
| `deck_mount_M8` | Power bus, battery tray | M8, 50mm grid, 8.5mm clearance holes |
| `leg_mount_frame` | Frame-to-leg connection | 4× M10, structural pattern |
| `actuator_envelope` | Hip/knee joint modules | 120mm dia × 200mm cylinder |
| `battery_bay` | Battery pack | 400×300×150mm clearance |

---

## 6. Validation Pipeline (Test-Driven CAD)

A CAD change is **invalid** unless all tests pass. This is the gate.

### Test Categories

#### A) Part-Level Tests
```python
def test_deck_plate():
    part = get_part("deck_plate")

    # Bounding box check
    assert part.Shape.BoundBox.XLength == approx(1219, abs=1)  # 4ft
    assert part.Shape.BoundBox.YLength == approx(2438, abs=1)  # 8ft

    # Hole pattern check
    holes = get_holes(part)
    assert len(holes) >= 48  # expected mount points

    # Volume sanity
    assert 5e6 < part.Shape.Volume < 20e6  # mm³

    # Recompute check
    part.recompute()
    assert not part.isError()
```

#### B) Interface Tests
```python
def test_power_bus_alignment():
    deck = get_part("deck_plate")
    bracket = get_part("power_bus_bracket")

    deck_holes = get_interface_holes(deck, "power_bus_mount")
    bracket_holes = get_interface_holes(bracket, "base_mount")

    for d, b in zip(deck_holes, bracket_holes):
        distance = (d - b).Length
        assert distance < 0.2, f"Hole misalignment: {distance}mm"
```

#### C) Assembly Tests
```python
def test_no_collisions():
    assembly = get_assembly("walker")
    for part_a, part_b in all_pairs(assembly.parts):
        if not should_touch(part_a, part_b):
            collision = part_a.Shape.common(part_b.Shape)
            assert collision.Volume < 1, f"Collision: {part_a} vs {part_b}"

def test_static_stability():
    assembly = get_assembly("walker")
    com = assembly.center_of_mass()
    support_polygon = get_foot_contact_polygon(assembly, pose="stand")

    com_projection = Vector(com.x, com.y, 0)
    assert point_in_polygon(com_projection, support_polygon)
```

#### D) Export Tests
```python
def test_exports():
    for part in all_parts():
        # STEP export
        step_path = export_step(part)
        assert os.path.exists(step_path)

        # STL export
        stl_path = export_stl(part)
        mesh = load_mesh(stl_path)
        assert mesh.is_manifold()
        assert mesh.triangle_count < 100000
```

### Running the Suite

After every change:
```bash
freecadcmd fc/build.py          # Rebuild model
python tests/test_parts.py       # Part checks
python tests/test_interfaces.py  # Interface checks
python tests/test_assembly.py    # Assembly checks
python fc/export.py              # Generate artifacts
```

All must pass before commit.

---

## 7. Mass & Physics Validation (Walker-Specific)

For a walking robot carrying 400 lb, physics checks are survival checks.

### What FreeCAD Provides Directly

FreeCAD can report for any solid shape:
- **Volume** (mm³)
- **Center of Mass** (x, y, z coordinates)
- **Matrix of Inertia** (for dynamics)

Access via Python:
```python
shape = part.Shape
volume = shape.Volume
com = shape.CenterOfMass
inertia = shape.MatrixOfInertia
```

### Material Properties

FreeCAD doesn't track material density natively in a robust way. Use your own table:

```yaml
# spec/materials.yaml
aluminum_6061:
  density: 2700  # kg/m³
  yield_strength: 276  # MPa
  elastic_modulus: 68900  # MPa

steel_1018:
  density: 7870
  yield_strength: 370
  elastic_modulus: 205000
```

Then compute:
```python
mass = volume_m3 * materials[part.material]['density']
```

### Mass Tracking

Every iteration must report:
- Per-part mass
- Subsystem mass (frame, legs, power, etc.)
- Total robot mass
- Mass vs. target (350-550 lb goal)

```python
def mass_report():
    parts = get_all_parts()
    total = 0
    for p in parts:
        mass = compute_mass(p)
        print(f"{p.Label}: {mass:.1f} kg")
        total += mass
    print(f"Total: {total:.1f} kg ({total * 2.205:.0f} lb)")
    assert 159 < total < 250, "Mass outside 350-550 lb target"
```

### Center of Mass Check

For static stability, the CoM projection must fall within the support polygon:

```python
def check_static_stability(assembly, stance="stand"):
    # Get assembly CoM (mass-weighted average of part CoMs)
    com = compute_assembly_com(assembly)

    # Get foot contact positions for this stance
    feet = get_foot_positions(assembly, stance)
    support_polygon = convex_hull(feet)

    # Project CoM to ground plane
    com_2d = (com.x, com.y)

    # Check containment with margin
    margin = 50  # mm safety margin
    assert point_in_polygon_with_margin(com_2d, support_polygon, margin)
```

### Joint Torque Estimates

First-pass actuator sizing uses static torque analysis:

```
torque = weight_below_joint × horizontal_distance_to_com
```

For Pack Mule with 400 lb payload:
- Total system: ~750-950 lb (340-430 kg)
- Knee and hip pitch: expect 300-700 N·m peak per joint
- Hip ab/ad: lower but substantial for lateral stability

Track these against actuator specs.

### FEM Self-Weight Analysis (Later Phase)

Once geometry is stable, run structural checks:
1. Assign material elastic properties
2. Apply gravity load
3. Fix constraints at foot contact points
4. Solve in CalculiX via FEM Workbench
5. Check:
   - Max stress < yield / safety_factor
   - Max deflection < tolerance

---

## 8. Phased Development Strategy

**If you try "full walker" on day 1, the agent will thrash.** Complexity must be earned through validated increments.

### Phase 1: Single Part

**Goal:** Prove the pipeline works.

**Example:** Deck plate
- Create `spec/params.yaml` with deck dimensions
- Write `fc/parts/deck_plate.py` builder
- Generate the part headless
- Run part-level tests (dimensions, holes, volume)
- Export STEP + STL + PNG
- Commit

**Exit criteria:** One part passes all tests and exports cleanly.

### Phase 2: Two Parts + One Interface

**Goal:** Prove interface alignment works.

**Example:** Deck plate + power bus bracket
- Define `power_bus_mount` interface in `interfaces.yaml`
- Both parts declare they use this interface
- Run interface alignment test
- Visual check: do the holes line up in the PNG render?
- Commit

**Exit criteria:** Two parts bolt together correctly; interface test passes.

### Phase 3: One Articulated Subsystem

**Goal:** Prove joints and assemblies work.

**Example:** Single front-left leg
- Hip ab/ad joint + hip pitch + knee pitch
- Thigh link + shin link
- Actuator envelope interfaces
- Joint axis alignment tests
- Range of motion check (no self-collision)
- Export URDF for this leg
- PyBullet smoke test: can it load? Do joints move?

**Exit criteria:** One leg moves correctly in simulation.

### Phase 4: Multi-Subsystem Integration

**Goal:** Prove subsystems compose.

**Example:** Frame + two legs
- Leg mount interfaces on frame
- Two legs attached
- Static stability check with two legs
- Collision checks between legs and frame
- Mass and CoM tracking

**Exit criteria:** Partial robot stands stably.

### Phase 5: Full Robot

**Goal:** Complete system.

- All four legs
- Power system
- Control system mounts
- Payload deck
- Full mass and stability validation
- Full URDF + simulation

**Each phase must be complete and validated before starting the next.**

---

## 9. LLM Prompting Discipline

When Claude Code or another agent is doing the CAD work, these rules keep it on track.

### System Prompt Rules

Include these in the agent's context:

1. **`spec/` files are source of truth.** Read dimensions from there, don't invent them.

2. **One subsystem per iteration.** Don't design the knee while adding the battery box.

3. **Every change must pass:** recompute, interface check, collision check, mass check, export.

4. **Fix failures before new features.** A broken build blocks everything.

5. **Use standard patterns.** Prefer known bracket types, standard fasteners, COTS components.

6. **When in doubt, ask.** Don't guess at requirements.

### Turn Prompt Pattern

Good prompts are specific and constrained:

```
Add a hip bracket part that:
- Exposes interface `leg_mount_frame`
- Accepts `M10_structural` bolt pattern
- Stays within 120×120×60mm bounding box
- Uses 8mm aluminum plate
- Includes 4× gussets for rigidity
```

Bad prompts are vague:
```
Design the hip area.
```

### The Three-Option Rule

For any mounting or structural decision, require the agent to propose three options:

**Example: Battery retention**
- Option A: Boxed tray + strap (simple, accessible)
- Option B: Slide-in rails + latch (tool-less removal)
- Option C: Clamshell + captive screws (weather sealed)

Human picks one, agent implements it. This prevents the LLM from committing to the first idea that seems plausible.

### Standard Patterns to Prefer

Bias toward known, validated patterns:
- Sheet-metal L-brackets with gussets
- Aluminum extrusion (80/20, 2020, etc.) for prototyping
- DIN rail for electronics mounting
- Standard standoffs and spacers
- Strain-relief clamps for cables
- Grommeted pass-throughs for wire routing

These reduce the number of "invented details" the LLM must get right.

---

## 10. Defense Against LLM Failure

LLMs produce designs that *look* plausible but violate hidden constraints.

### Common Failure Modes

| Failure | What Happens | Defense |
|---------|--------------|---------|
| Bracket too thin near bolt | Stress concentration → crack | Min edge distance rule + FEM |
| No wrench access | Can't assemble | Assembly sequence check |
| Cable rub point | Chafe → short circuit | Cable routing rules + clearance checks |
| Assembly sequence impossible | Parts can't be installed in order | Manual review of assembly order |
| Sharp internal corner | Stress riser | Fillet rules + FEM |
| "Works in CAD, fails in vibration" | Fatigue, loosening | FEM + physical prototyping |

### The "Fast Junior Engineer" Mental Model

Treat the LLM like a capable but inexperienced drafter:
- It can produce geometry quickly
- It must pass your automated checks
- Anything load-bearing gets additional review
- Physical prototyping validates critical assumptions

### Design Rulebook

Maintain a `spec/design_rules.yaml` with mandatory constraints:

```yaml
fasteners:
  min_edge_distance: 2.5  # × fastener diameter
  min_thread_engagement: 1.5  # × diameter
  preferred_sizes: [M5, M6, M8, M10]
  thread_locker: required_for_vibration

structure:
  min_wall_thickness: 3mm  # aluminum
  min_fillet_radius: 2mm   # internal corners
  max_aspect_ratio: 10     # for plates (prevents flutter)

cables:
  min_bend_radius: 5  # × cable diameter
  strain_relief: required
  no_sharp_edges: true

clearance:
  min_tool_access: 25mm  # for wrench/driver
  min_cable_channel: 15mm
```

The agent must check designs against these rules.

### Promote Validations Over Time

**Early phases:**
- Geometry + interfaces + clearance + mass/CoM

**Middle phases:**
- Static torque envelopes at joints
- Assembly sequence feasibility

**Later phases:**
- FEM self-weight + load cases
- Full simulator smoke tests
- Physical prototype validation

### Bias Toward COTS

If you standardize on:
- Known battery pack form factors
- Known motor/gearbox modules
- Known connector families
- Known fastener standards

...you reduce the surface area for LLM invention errors.

### The Ultimate Defense

**Anything load-bearing or safety-critical gets:**
1. FEM analysis
2. Physical prototyping
3. Human engineering review

No amount of automated testing replaces physical validation for a 400 lb payload walking robot.

---

## Quick Reference: The Gate

Every commit must pass:

| Check | How |
|-------|-----|
| Recompute | FreeCAD opens and recomputes without errors |
| Interfaces | Bolt holes align within 0.2mm |
| Collisions | No intersections at neutral pose |
| Mass | Total within 350-550 lb target |
| CoM | Projects inside support polygon |
| Exports | STEP and STL succeed |
| Screenshots | PNG renders for visual diff |

A change is **invalid** unless all checks pass.

---

## Summary

This process treats robot CAD like software engineering:
- Spec files are source code
- FreeCAD is the compiler
- Tests are geometric invariants
- The build is broken if tests fail
- Commits require passing the gate

The phased approach prevents thrashing. Interfaces enable scaling. Validation prevents "beautiful CAD, painful reality."

The Pack Mule will be built one validated increment at a time.

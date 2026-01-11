# Pack Mule Quadruped Robot

## 0) Meta Principles

* Linux brain
* 400 lb payload on deck
* Design process tuned for AI coding agents: TDD-style CAD development. "Robots building robots."
* Parametric CAD, constant validation

Other parameters derive from this core.

## 1) Core Mission

**Mission statement**
> Build a medium-to-large electric quadruped robot capable of safely transporting up to **400 lb payload** over uneven outdoor terrain at walking speeds, using only bolted and machined components.

Design priorities:
- Payload capacity and stability under load
- Manufacturability via CNC/laser cutting + purchased hardware
- Parametric CAD, minimal iteration, front-loaded reasoning
- Safety-first failure modes
- Extensibility (sensors, autonomy)

---

## 2) Core Performance & Payload Specifications

### 2.1 Payload and Mass Targets
- **Max payload:** 400 lb (≈ 181 kg)
- **Payload includes rider + gear:** Yes
- **Target robot curb weight (robot only):** **350–550 lb** (≈ 159–250 kg)
  - Rationale: payload fraction is expected to be ~40–55% of total system mass for a first practical build in this scale, without exotic materials.
- **Expected max total system mass:** **750–950 lb** (≈ 340–430 kg)

> NOTE: This curb-weight range is a design target. CAD validation and BOM rollups must continuously track estimated mass and adjust structure/actuation accordingly.

### 2.2 Terrain & Operation
- Terrain: packed dirt roads → logging trails (rocks/ruts)
- Speed: brisk walking pace (≈ 1–3 mph)
- Runtime: < 2 hours per charge

### 2.3 Environment
- Weather: light rain, dust, splashes
- Temperature: ~20°F to 100°F (-6°C to 38°C)

---

## 3) High-Level Robot Architecture

The robot consists of **six primary subsystems**, each modeled and validated independently, then integrated:

1) Structural Frame (bolt-together plate + spacer box frame)
2) Leg Assemblies (4 identical legs, 3 DOF each)
3) Actuation System (integrated robotic joint modules)
4) Power System (battery + distribution)
5) Control & Compute System (Linux brain + distributed leg control)
6) Payload Deck & Interfaces

---

## 4) Structural Frame Subsystem (No-weld)

### 4.1 Design Philosophy
- **No welding**
- Entirely bolted construction
- CNC laser-cut or waterjet-cut aluminum plates
- Spacers + gussets create a rigid torsion box

### 4.2 Geometry
- Deck size: **4 ft × 8 ft** (1219 mm × 2438 mm)
- Frame depth: ~250–350 mm (torsion depth matters for stiffness)
- Battery and heavy electronics mounted **low** between plates for stability

### 4.3 Materials
- Primary structure: Aluminum 6061-T6 plate
  - Typical thicknesses: 8 mm / 12 mm / 16 mm depending on location
- Spacers: aluminum or steel standoffs
- Fasteners: M8/M10 structural; M6 for covers and light mounts

### 4.4 Frame BOM (Representative)
- Side plates, full-length (2×)
- Bulkheads/cross plates (6–10×)
- Corner and leg-mount gussets (8–16×)
- Standoffs/spacers (50–120×)
- Fasteners + thread-locking strategy
- Service panels/covers (optional)

**Mass expectation (frame only):** ~120–220 lb depending on thickness and cutouts.

---

## 5) Leg Assemblies

### 5.1 Configuration
- **4 legs**, identical geometry
- **3 DOF per leg** (12 total):
  - Hip ab/ad (lateral)
  - Hip pitch
  - Knee pitch

### 5.2 Mechanical Design
- Links built from **laser-cut plate sandwich** or tube + plate hybrids
- Bearings at all rotational joints
- Feet include replaceable compliant pads

### 5.3 Geometry
- Thigh length: ~500–650 mm
- Shin length: ~500–650 mm
- Nominal clearance: ~300–450 mm
- Wide stance prioritized (stability under 400 lb payload)

### 5.4 Leg BOM (Per Leg)
- Hip ab/ad joint module (1×)
- Hip pitch joint module (1×)
- Knee joint module (1×)
- Thigh link plates (2×) + spacers
- Shin link plates (2×) + spacers
- Bearing sets (6–10×)
- Joint shafts / shoulder bolts
- Foot pad assembly

**Mass expectation (legs, links+hardware only):** ~25–45 lb per leg (excluding actuators), depending on plate thickness.

---

## 6) Actuation System

### 6.1 Architecture
- Fully electric
- **Integrated robotic joint modules** (motor + gearbox + encoder)
- Distributed torque / current limiting for safety

### 6.2 Torque Scale
- Expect **knee + hip pitch** to require on the order of **300–700 N·m peak** per joint depending on leg length, stance geometry, and dynamic loads.
- Hip ab/ad generally lower, but still substantial for lateral stability.

> The CAD agent should treat these as sizing targets and leave mount interfaces adaptable until final actuator selection.

### 6.3 Gear Strategy
- Primary recommendation: **robust planetary** or **cycloidal** inside the joint module.
  - Planetary: easiest to source, straightforward integration
  - Cycloidal: better shock tolerance but more complexity
- Avoid extreme ratios that eliminate compliance.

### 6.4 Actuation BOM
- Joint modules (12×)
- Encoder feedback integrated in each module
- Motor driver(s) (distributed) sized for continuous walking under heavy load
- Joint harnesses, strain relief, sealed connectors

> Specific brands/models will be locked only after a torque/weight/cost sweep. The CAD should preserve an “actuator envelope” and bolt pattern interface that can adapt.

---

## 7) Power System

### 7.1 Electrical Architecture
- DC bus for traction/actuation
- Separate protected rails for compute/sensors
- Star power distribution and hardware E-stop

### 7.2 Bus Voltage Recommendation
- **Primary recommendation:** **96 V class bus** (e.g., 84–100 V operating) *or* a conservative **48 V** system with heavier cabling.
  - At this payload scale, 96 V reduces current, wiring mass, and I²R losses.
  - If component availability forces 48 V early, design the harness for high current and plan to migrate later.

### 7.3 Battery sizing (runtime target < 2 hours)
- First-pass energy target: **4–8 kWh**
  - Heavy quadruped with 400 lb payload is expected to average multiple kilowatts during continuous walking on rough terrain.
- Battery form factor:
  - Off-the-shelf EV modules (salvage) or commercial LiFePO₄ packs
  - Mount battery low in the frame, central for CoM

### 7.4 Power Safety
- Main contactor and precharge
- Hardware E-stop loop (cuts actuator power)
- Branch fusing and current monitoring
- DC/DC converters for:
  - 24 V (optional) for auxiliaries
  - 12 V and 5 V for compute/sensors

### 7.5 Power BOM
- Battery pack (4–8 kWh class)
- BMS (integrated preferred)
- Main contactor + precharge
- Main fuse(s)
- DC/DC converters
- Power distribution (bus bars, fusing blocks)
- High-current connectors + cabling

---

## 8) Control & Compute System

### 8.1 Control Philosophy
- Central Linux computer for high-level control and autonomy
- Distributed real-time control at the legs
- Fail-safe behavior: if command stream is lost → hold → sit

### 8.2 Compute
- Linux PC (x86 or Jetson-class) running Ubuntu + ROS 2
- Teleop initially; autonomy staged later

### 8.3 Leg Control
- One controller per leg
- High-rate loops local to leg (torque/current control)
- CAN (or EtherCAT later) between brain and legs

### 8.4 Sensors
- Central IMU
- Joint encoders (in joint modules)
- Optional foot contact sensing
- Optional depth/vision/LiDAR for later autonomy

### 8.5 Control BOM
- Main computer
- Leg controllers (4×)
- Motor drivers sized for heavy current
- IMU module
- Networking + CAN hardware

---

## 9) Payload Deck & Interfaces

### 9.1 Deck
- Flat deck mounted atop frame
- Tie-down points for cargo securing

### 9.2 Human rider case
- A rider is a payload within the 400 lb limit
- Software “ride mode”:
  - reduced speed
  - reduced acceleration
  - stability-first gait

---

## 10) Manufacturing Philosophy (No Welding)

- Prefer laser/waterjet-cut 2D parts
- Use spacers/standoffs and gussets for stiffness
- Minimize 3D machining; reserve it for:
  - bearing bores
  - actuator interface blocks
  - high-stress joints
- Assembly should require only hand tools

---

## 11) Summary

This project treats robot design like software:
- deterministic builds
- parametric inputs
- automated validation
- repeatable export packages for machine shops

All design choices (mass, actuators, battery, structure) derive from the **400 lb payload** capacity requirement.

---

## 12) Development Process

See **[process.md](process.md)** for the complete CAD & Agent Workflow, including:
- FreeCAD as deterministic compiler
- Repository architecture
- The agent iteration loop
- Interface and invariant definitions
- Test-driven CAD validation pipeline
- Mass and physics checks
- Phased development strategy
- LLM prompting discipline


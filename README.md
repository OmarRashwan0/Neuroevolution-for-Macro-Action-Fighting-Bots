# Maze‑AI: A Genetic‑Algorithm Bot in a Dynamic Grid Arena (Unity)

[![Unity](https://img.shields.io/badge/Engine-Unity-000?logo=unity)](#requirements)
[![CSharp](https://img.shields.io/badge/Language-C%23-239120?logo=csharp)](#code-structure)
[![AI](https://img.shields.io/badge/AI-Genetic%20Algorithm%20%2B%20A*-6f42c1)](#genetic-algorithm--learning-objective)
[![License](https://img.shields.io/badge/License-MIT-blue)](#license)

A 1v1 **action‑strategy** game set inside a **procedurally generated maze**, where the opponent is an **adaptive AI** trained with a **Genetic Algorithm (GA)** and uses **A*** for path planning. The arena is **malleable**: walls are **destructible** and **buildable** on the fly; abilities (laser, barrier, teleport) are limited by a **chakra** energy bar that **recharges** over time.

> **TL;DR** — You fight an AI that learns action selection from human play logs via GA. The maze changes as both sides build/destroy walls. You manage **Health** (non‑regenerating) and **Chakra** (regenerating) to time **Lasers / Barriers / Teleports**.

---

## Table of Contents
- [Demo](#demo)
- [Core Features](#core-features)
- [Game Mechanics](#game-mechanics)
  - [Maze Generation](#maze-generation)
  - [Resources: Health and Chakra](#resources-health-and-chakra)
  - [Combat: Continuous Laser](#combat-continuous-laser)
  - [Environment Manipulation: Barriers & Teleportation](#environment-manipulation-barriers--teleportation)
- [Networking](#networking)
- [AI: Policy, Equations & GA](#ai-policy-equations--ga)
  - [State, Actions, and Linear Policy](#state-actions-and-linear-policy)
  - [Optional MLP Policy](#optional-mlp-policy)
  - [Genome & Discretization](#genome--discretization)
  - [Genetic Algorithm & Learning Objective](#genetic-algorithm--learning-objective)
  - [GA Workflow](#ga-workflow)
- [Installation](#installation)
- [Usage](#usage)
  - [Single‑Player (vs GA Bot)](#single-player-vs-ga-bot)
  - [Multiplayer (Data Collection)](#multiplayer-data-collection)
  - [GA Training from Logs](#ga-training-from-logs)
- [Code Structure](#code-structure)
- [Configuration](#configuration)
- [Experiments & Tips](#experiments--tips)
- [Citing or Building on This Work](#citing-or-building-on-this-work)
- [License](#license)

---

## Demo

> Replace the placeholders below with your media.

- **Short trailer (15–30s)** — `docs/demo.mp4`
- **Animated GIFs**
  - Maze generation & HUD — `docs/maze_hud.gif`
  - Laser piercing entities & walls — `docs/laser.gif`
  - Barrier timing vs incoming laser — `docs/barrier.gif`
  - Teleport sequence — `docs/teleport.gif`
  - Host/Client panel & synchronization — `docs/network.gif`

---

## Core Features
- **Dynamic Maze:** grid‑based arena with **procedural generation**, **destructible walls**, and **on‑the‑fly barrier placement**.
- **Dual Resources:** **Health** (non‑regenerating) and **Chakra** (recharging) show as in‑world bars and HUD overlays.
- **Abilities:**
  - **Laser:** continuous beam, applies damage per frame.
  - **Barrier:** drop a temporary blocking wall on the grid ahead.
  - **Teleport:** blink to a target cell; cost scales with distance.
- **AI:** an **evolving policy** trained from **human action logs** with a **Genetic Algorithm**; optional **A*** for routing.
- **Networking:** head‑to‑head Host/Client/Server‑Only via Mirror; server authoritative state.
- **Mini‑Map:** orthographic camera renders top‑down overlay.

---

## Game Mechanics

### Maze Generation
- Cells form an $m\\times n$ grid. A depth‑first search (DFS) carves a **perfect maze** (single path between any two cells), then a second pass **adds loops** by knocking down extra walls with probability $p_{\\text{loop}}$.
- Walls flagged **destructible** hold hit points (HP) and render overhead HP bars.

### Resources: Health and Chakra
- **Health** $H$ is fixed per life; **Chakra** $\\chi\\in[0,\\chi_{\\max}]$ **recharges** at rate $r$ when not spending. Abilities consume chakra; HUD shows both bars.
- (Optional) **Charging** mode: holding a key increases $\\chi$ faster but slows movement.

### Combat: Continuous Laser
- A raycast is maintained while firing. On each frame $t$ that the beam intersects a target, apply damage $\\Delta H_t = d\\,\\Delta t$ (constant rate $d$). The target’s overhead HP bar shrinks **continuously** and is hidden after an inactivity timeout.

### Environment Manipulation: Barriers & Teleportation
- **Barrier:** spawn a wall in the forward grid cell; despawns after duration $\\tau$ or manual toggle.
- **Teleport:** preview a marker $k$ cells ahead; confirm to blink and pay cost
$$
\\operatorname{Cost}_{\\text{TP}}(k) = c_0 + c_1\\,k,\\quad k\\in\\mathbb{N}.
$$

---

## Networking
- **Authoritative server** generates the maze and validates all state changes (fire, damage, barriers, teleports). Clients send intent; server applies and broadcasts deltas. A simple panel allows **Host**, **Client (IP:Port)**, and **Server‑Only** modes.

---

## AI: Policy, Equations & GA

### State, Actions, and Linear Policy
- **Actions** $\\mathcal{A} = \\{\\texttt{Laser},\\ \\texttt{Blocking},\\ \\texttt{Hindering},\\ \\texttt{Escaping},\\ \\texttt{ShowingUp},\\ \\texttt{Teleport},\\ \\texttt{Charging}\\}$ (7 actions).\n- **State** $\\mathbf{s}\\in\\mathbb{R}^5$:\n  - $s_1=\\texttt{dtile}\\in[0,1]$: normalized grid distance;\n  - $s_2\\in\\{0,1\\}$: line‑of‑sight flag;\n  - $s_3\\in\\{0,1\\}$: opponent being hit;\n  - $s_4\\in\\{0,1\\}$: self being hit;\n  - $s_5=\\chi\\in[0,1]$: normalized chakra.\n- **Linear policy**: each action $a$ has weights $\\mathbf{w}^{(a)}\\in\\mathbb{R}^5$. The score is
$$
\\operatorname{Score}(a\\mid\\mathbf{s}) = \\mathbf{w}^{(a)}\\cdot\\mathbf{s} = \\sum_{j=1}^{5} w^{(a)}_j s_j,\\qquad a^* = \\arg\\max_a \\operatorname{Score}(a\\mid\\mathbf{s}).
$$

### Optional MLP Policy
A compact 1‑hidden‑layer MLP increases capacity:
$$
\\mathbb{R}^5 \\xrightarrow{\\mathbf{W}^{(1)}} \\mathbb{R}^6 \\xrightarrow{\\sigma} \\mathbb{R}^6 \\xrightarrow{\\mathbf{W}^{(2)}} \\mathbb{R}^7.
$$

### Genome & Discretization
- **Genome:** concatenate all weights $\\mathbf{W}=[\\mathbf{w}^{(1)}|\\cdots|\\mathbf{w}^{(7)}]\\in\\mathbb{R}^{7\\times5}$ → vector of length **35**.
- **Quantization:** each gene $w_j\\in\\{-1,-0.9,\\ldots,0.9,1\\}$ (21 levels) to keep GA search tractable.

### Genetic Algorithm & Learning Objective
- From multiplayer sessions, log constraints $(\\mathbf{s}_c, a_c)$: state and the human’s chosen action.
- **Margin objective:** for each constraint, prefer the chosen action over all others
$$
\\mathbf{w}^{(a_c)}\\cdot\\mathbf{s}_c\\;>\\;\\mathbf{w}^{(b)}\\cdot\\mathbf{s}_c,\\quad\\forall b\\neq a_c.
$$
- **Fitness** (maximize total margin across the log set $\\mathcal{C}$):
$$
F(\\mathbf{W}) = \\sum_{(\\mathbf{s}_c,a_c)\\in\\mathcal{C}}\\;\\sum_{b\\in\\mathcal{A},\\ b\\neq a_c}\\big(\\mathbf{w}^{(a_c)}-\\mathbf{w}^{(b)}\\big)\\cdot\\mathbf{s}_c.
$$
This drives the GA to **mimic human action selection** under observed states.

### GA Workflow
1. **Initialize** population $P$ with quantized genes.
2. **Evaluate** each $\\mathbf{W}_i$ by computing $F(\\mathbf{W}_i)$ on the constraint log.
3. **Select** parents by **tournament selection**.
4. **Crossover** genomes (1‑point or uniform) to form offspring.
5. **Mutate** a fraction $\\mu$ of genes (random level flips; optional Gaussian jitter before requantization).
6. **Elitism:** carry top $E$ individuals unchanged.
7. **Repeat** for $G$ generations or until validation fitness saturates.

---

## Installation

### Requirements
- **Unity** 2021 LTS or newer (URP/HDRP not required).
- **Mirror** Networking package (via UPM) for Host/Client.
- **.NET** (C#) — Unity default.

### Get the project
```bash
# Option A: new repo
git clone https://github.com/<you>/maze-ai-ga
cd maze-ai-ga

# Option B: add as a Unity project folder (open via Unity Hub)
```
---

## Usage

### Single‑Player (vs GA Bot)
1. Open scene: `Scenes/SinglePlayer.unity`.
2. Press **Play**. HUD shows **Health** (green) and **Chakra** (blue).
3. Controls (default):
   - **Fire Laser:** Hold `Mouse0`
   - **Barrier:** `Q` (toggle/timeout)
   - **Teleport Mode:** `E` (adjust distance with scroll or `W/S`, confirm with `E`)
   - **Charge Chakra:** `C` (slows movement)

### Multiplayer (Data Collection)
1. Open `Scenes/Network.unity`.
2. Use the **Connection Panel**:
   - **Host** (server+client), **Client** (enter IP/Port), or **Server‑Only**.
3. Play a few matches; the game logs JSON constraints of the form:
   ```json
   {"s":[dtile,visibility,h1,h2,chakra],"a":"Laser","t":1699999999}
   ```
4. Logs are stored under `Logs/constraints/*.json` (configurable in `GAOptimizer`).

### GA Training from Logs
1. In the **Bootstrap/GAOptimizer** component, set:
   - **Population P**, **Generations G**, **Mutation Rate μ**, **Elites E**.
   - **Quantization** levels (21 by default) and **random seed**.
2. Click **Train** in play mode (or run the attached editor tool) to evolve $\\mathbf{W}$.
3. The **best policy** weights are saved to `Assets/Resources/Policy/best_policy.json` and auto‑loaded at runtime.

---

## Code Structure
```
Assets/
  Scripts/
    AStarPathfinder.cs        # Grid/path search for routing
    BotController.cs          # Action selection using current policy
    GAOptimizer.cs            # GA loop, fitness evaluation from logs
    MazeGameManager.cs        # Maze generation, world instantiation
    DamageableEntity.cs       # HP logic + overhead health bars
    DamageableWall.cs         # Destructible wall with HP
    PlayerController.cs       # Local player input & abilities
    PlayerControllerLocal.cs  # Local-only variant
    MinimapController.cs      # Orthographic top-down mini-map
    NetworkMazeGameManager.cs # Mirror setup & state sync
    SceneBootstrap.cs         # Editor hooks & parameter bindings
  Scenes/
    SinglePlayer.unity
    Network.unity
  Resources/
    Policy/best_policy.json   # Serialized weights for linear policy
Logs/
  constraints/*.json          # Human action logs for GA training
Docs/
  figures/*.png
  demo.mp4
```

---

## Configuration
Key parameters (exposed in **SceneBootstrap → GAOptimizer** unless noted):

- **Maze**: grid size `(rows, cols)`, cell size, loop probability `p_loop`.
- **Resources**: `H_max`, `chi_max`, recharge rate `r`, charge multiplier, UI timeouts.
- **Laser**: damage rate `d`, chakra cost per second.
- **Barrier**: lifetime `τ`, chakra cost, cooldown.
- **Teleport**: base cost `c0`, per‑tile cost `c1`, cooldown.
- **GA**: `P`, `G`, `μ`, elites `E`, tournament size, crossover type.
- **Logging**: path to constraint JSON, sampling period.

---

## Experiments & Tips
- **Fitness scaling**: normalize states before dot‑products to keep margins comparable.
- **Class imbalance**: if some actions are rare in logs, apply per‑action weights in fitness.
- **Exploration vs exploitation**: keep a small **ε‑greedy** during self‑play to avoid overfitting.
- **Safety constraints**: penalize actions that cause self‑damage (e.g., teleporting into firelines).
- **Hybrid policy**: mix linear policy with A*‑derived heuristics for escape/approach subtasks.
- **Ablations**: compare (i) hand‑tuned FSM/BT, (ii) linear GA policy, (iii) MLP policy.

---

## Citing or Building on This Work
If you use this project in research or applications, consider citing the repository and acknowledging the GA imitation objective and dynamic‑maze mechanics described here. A short BibTeX template:

```bibtex
@misc{mazeai_ga,
  title        = {Maze-AI: A Genetic-Algorithm Bot in a Dynamic Grid Arena},
  author       = {Rashwan, Omar},
  year         = {2025},
  howpublished = {GitHub repository},
  note         = {Unity, Genetic Algorithm, A*, dynamic maze, continuous laser}
}
```

---

## License
This project is released under the **MIT License**. See `LICENSE` for details.

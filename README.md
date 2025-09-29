# Maze‑AI: A Genetic‑Algorithm Bot in a Dynamic Grid Arena (Unity)

[![Unity](https://img.shields.io/badge/Engine-Unity-000?logo=unity)](#requirements)
[![CSharp](https://img.shields.io/badge/Language-C%23-239120?logo=csharp)](#code-structure)
[![AI](https://img.shields.io/badge/AI-Genetic%20Algorithm%20%2B%20A*-6f42c1)](#genetic-algorithm--learning-objective)
[![License](https://img.shields.io/badge/License-MIT-blue)](#license)

A 1v1 **action‑strategy** game set inside a **procedurally generated maze**, where the opponent is an **adaptive AI** trained with a **Genetic Algorithm (GA)** and uses **A*** for path planning. The arena is **malleable**: walls are **destructible** and **buildable** on the fly; abilities (laser, barrier, teleport) are limited by a **chakra** energy bar that **recharges** over time.

> **TL;DR** — You fight an AI that learns action selection from human play logs via GA. The maze changes as both sides build/destroy walls. You manage **Health** (non‑regenerating) and **Chakra** (recharging) to time **Lasers / Barriers / Teleports**.

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
- Cells form a rectangular grid; a DFS carves a **perfect maze**, then extra walls are removed with probability \(p_{\mathrm{loop}}\) to add loops.

$$
\text{Grid size: } m \times n, \qquad \text{Loop probability: } p_{\mathrm{loop}} \in [0,1].
$$

- Walls flagged **destructible** hold hit points (HP) and render overhead HP bars.

### Resources: Health and Chakra
- Health does **not** regenerate; Chakra regenerates when not spending. Variables:

$$
H \in \mathbb{R}_{\ge 0}, \qquad 
\chi \in [0,\chi_{\max}], \qquad 
\frac{d\chi}{dt} = r \ \text{(when not spending)}.
$$

### Combat: Continuous Laser
A raycast is maintained while firing; damage accumulates per frame $\Delta t$:

$$
\Delta H_t = d\\Delta t
$$

where $d$ is the constant damage rate while the beam intersects a target.


### Environment Manipulation: Barriers & Teleportation
- **Barrier:** spawns in the forward grid cell; despawns after duration \(\tau\) or manual toggle.
- **Teleport:** preview a marker \(k\) cells ahead; confirm to blink and pay cost

$$
\mathrm{Cost}_{\mathrm{TP}}(k) \;=\; c_0 + c_1\,k, \qquad k \in \mathbb{N}.
$$

---

## Networking
- **Authoritative server** generates the maze and validates all state changes (fire, damage, barriers, teleports). Clients send intent; server applies and broadcasts deltas. A simple panel allows **Host**, **Client (IP:Port)**, and **Server‑Only** modes.

---

## AI: Policy, Equations & GA

### State, Actions, and Linear Policy
We discretize gameplay into a small state vector and seven macro‑actions.

$$
\mathcal{A} = \{\texttt{Laser},\,\texttt{Blocking},\,\texttt{Hindering},\,\texttt{Escaping},\,\texttt{ShowingUp},\,\texttt{Teleport},\,\texttt{Charging}\} \quad (|\mathcal{A}|=7).
$$

State vector (normalized/scalar flags):

$$
\mathbf{s} = [\s_1, s_2, s_3, s_4, s_5\]^\top \in \mathbb{R}^5,
\quad
\begin{aligned}
&s_1=\texttt{dtile}\in[0,1],\\
&s_2 \in \{0,1\}\ \text{(line of sight)},\\
&s_3 \in \{0,1\}\ \text{(opponent being hit)},\\
&s_4 \in \{0,1\}\ \text{(self being hit)},\\
&s_5=\chi \in [0,1]\ \text{(normalized chakra)}.\\
\end{aligned}
$$

Each action $a \in \mathcal{A}$ has weights $\mathbf{w}^{(a)} \in \mathbb{R}^5$ (5-D vector). We score actions by a dot product and choose the maximum:

$$
\mathrm{Score}(a \mid \mathbf{s}) \=\ \mathbf{w}^{(a)} \cdot \mathbf{s}
\=\ \sum_{j=1}^{5} w^{(a)}_j s_j,
\qquad
a^* \=\ \arg\max_{a \in \mathcal{A}} \mathrm{Score}(a \mid \mathbf{s}).
$$

### Optional MLP Policy
A compact 1‑hidden‑layer MLP increases capacity:

$$
\mathbb{R}^5 \\to\ \mathbb{R}^6 \\to\ \mathbb{R}^6 \\to\ \mathbb{R}^7.
$$

### Genome & Discretization
We concatenate all weights into a genome vector and quantize for GA search:

$$
\mathbf{W} = \big[\,\mathbf{w}^{(1)} \mid \mathbf{w}^{(2)} \mid \cdots \mid \mathbf{w}^{(7)}\,\big] \in \mathbb{R}^{7 \times 5}
\ \longrightarrow\
\text{vec}(\mathbf{W}) \in \mathbb{R}^{35},
$$

$$
w_j \in \{-1, -0.9, \ldots, 0.9, 1\} \quad \text{(21 quantization levels)}.
$$

### Genetic Algorithm & Learning Objective
From multiplayer sessions we log constraints \((\mathbf{s}_c, a_c)\) (state and the human‑chosen action). We train by a **margin objective** that prefers the chosen action over all others:

$$
\mathbf{w}^{(a_c)} \cdot \mathbf{s}_c \>\ \mathbf{w}^{(b)} \cdot \mathbf{s}_c, \qquad \forall\, b \neq a_c.
$$

The GA maximizes a summed margin‑style fitness over all constraints:

$$
F(\mathbf{W}) \=\
\sum_{(\mathbf{s}_c,a_c) \in \mathcal{C}} \
\sum_{b \in \mathcal{A},\, b \neq a_c}
\Big(\mathbf{w}^{(a_c)} - \mathbf{w}^{(b)}\Big)\cdot \mathbf{s}_c.
$$

This drives the GA to **mimic human action selection** under observed states.

### GA Workflow

1. Initialize a population $P$ with quantized genes.
2. Evaluate each candidate $\mathbf{W}_i$ by computing $F(\mathbf{W}_i)$ on the constraint log.
3. Select parents via tournament selection.
4. Crossover genomes (one-point or uniform) to form offspring.
5. Mutate a fraction $\mu$ of genes (random level flips; optional Gaussian jitter before requantization).
6. **Elitism:** carry the top $E$ individuals unchanged.
7. Repeat for $G$ generations or until validation fitness saturates.

*Symbols:* $P$ = population size, $\mathbf{W}_i$ = individual’s weights, $F$ = fitness, $\mu$ = mutation rate, $E$ = number of elites, $G$ = number of generations.


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
   - **Population P**, **Generations G**, **Mutation Rate \(\mu\)**, **Elites E**.
   - **Quantization** levels (21 by default) and **random seed**.
2. Click **Train** in play mode (or run the attached editor tool) to evolve \(\mathbf{W}\).
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
- **Barrier**: lifetime `\tau`, chakra cost, cooldown.
- **Teleport**: base cost `c0`, per‑tile cost `c1`, cooldown.
- **GA**: `P`, `G`, `\mu`, elites `E`, tournament size, crossover type.
- **Logging**: path to constraint JSON, sampling period.

---

## Experiments & Tips
- **Fitness scaling**: normalize states before dot‑products to keep margins comparable.
- **Class imbalance**: if some actions are rare in logs, apply per‑action weights in fitness.
- **Exploration vs exploitation**: keep a small **\(\varepsilon\)-greedy** during self‑play to avoid overfitting.
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

# AnyLogic Primer

## What Is AnyLogic?

AnyLogic is a simulation modeling tool. You build a virtual version of a real-world system — a factory, a supply chain, a hospital, a port — and run it forward in time to see what happens. Instead of guessing whether a new process will work, you simulate it first.

AnyLogic supports three simulation paradigms:

- **Discrete Event Simulation (DES)** — models processes as a sequence of events happening to entities (trucks arriving, silos filling, ships departing). This is what the grain terminal uses.
- **Agent-Based Modeling (ABM)** — models individual actors with their own rules and behaviors, useful for crowds, markets, ecosystems.
- **System Dynamics (SD)** — models feedback loops and stocks/flows at a high level, useful for strategy and policy.

Most tools force you to pick one. AnyLogic lets you mix all three in a single model.

## What Is Discrete Event Simulation?

Think of it like a board game. Entities (trucks, trains, ships) move through a series of steps:

```
[Arrive] → [Queue] → [Seize Resource] → [Process] → [Release] → [Exit]
```

At each step, something happens: a delay, a resource is seized, a decision is made. The simulation engine advances time from event to event — it doesn't tick through every second, it jumps to the next meaningful moment.

This makes DES efficient for modeling logistics, manufacturing, healthcare, and any system where things wait, compete for resources, and flow through processes.

## AnyLogic Learning Edition

The Learning Edition is **free** and fully functional for learning. What you get:

- All three modeling paradigms (DES, ABM, SD)
- The full Process Modeling Library (Source, Queue, Delay, Seize, Release, Sink, etc.)
- Road Traffic, Rail, Fluid, Pedestrian, and Material Handling libraries
- 3D animation and visualization
- Experiment frameworks (parameter variation, optimization, Monte Carlo)
- Java-based custom logic — you can write arbitrary code in your models

The only limitation vs. the Professional edition: you can't export standalone model executables. But for learning, exploring, and prototyping, it's everything you need.

**Download it here:** [anylogic.com/downloads](https://www.anylogic.com/downloads/)

## Key Concepts

### Entities
The things that flow through your model. In our case: trucks, trains, ships. Entities carry data (grain type, capacity, appointment time) and move through process flowcharts.

### Resources
Things that entities compete for. In our model: auto silo rows (4 of them) and pier berths (2 of them). When a truck needs a row and all 4 are busy, it waits.

### Flowcharts
The visual process logic. You drag blocks like **Source** (creates entities), **Queue** (holds them), **Seize** (grabs a resource), **Delay** (takes time), **Release** (frees a resource), and **Sink** (removes entities). Connect them with arrows.

### Agents
Everything in AnyLogic is an agent. A truck is an agent. A silo is an agent. Even `Main` (the top-level model) is an agent. Agents have parameters, variables, functions, state charts, and embedded objects.

### Parameters vs. Variables
- **Parameters** are inputs you set before or during the run (silo capacity = 5000, truck capacity = 125). They appear in the experiment UI.
- **Variables** are internal state that changes during the run (current silo fill level, number of trucks processed).

### The .alp File
AnyLogic models are stored as a single XML file (`.alp`). Everything — agents, flowcharts, functions, UI elements, 3D shapes — lives in this file. You can edit it in AnyLogic's GUI or directly in a text editor. This repo shows you both approaches.

## What This Repo Demonstrates

1. **Reading a model** — understanding what an existing simulation does by examining its structure and logic
2. **Extending a model** — adding new features (truck appointment scheduling) by editing the `.alp` XML directly
3. **Visualizing a model** — building a standalone Three.js web app that makes the simulation intuitive to non-technical audiences

Next: [Understanding the Model →](02-understanding.md)

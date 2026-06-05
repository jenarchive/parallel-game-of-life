# Go-Parallel-GameOfLife (Conway's Game of Life Parallel Implementation)

This project implements **Conway's Game of Life** cellular automaton using **Go (Golang)**. The core focus is on achieving **high-performance parallel simulation** by utilising Go's built-in concurrency primitives: **Goroutines** and **Channels**. This approach ensures thread-safe operations and enables efficient simulation on large-scale grids.

## Overview

Conway's Game of Life is an iteration-driven simulation. The grid evolves based on simple rules:

1. Any live cell with fewer than two live neighbours dies, as if by underpopulation.
2. Any live cell with two or three live neighbours lives on to the next generation.
3. Any live cell with more than three live neighbours dies, as if by overpopulation.
4. Any dead cell with exactly three live neighbours becomes a live cell, as if by reproduction.

Our implementation parallelises these computations across multiple Go routines, significantly enhancing performance compared to serial execution.

## Final Coursework Report

The full detailed analysis of the parallel implementation, including performance benchmarks and design rationale, is available in the final report.

[![Report Preview](https://github.com/Jen0821/Parallel-GameOfLife/blob/main/report.jpg)](https://github.com/Jen0821/Parallel-GameOfLife/blob/main/report.pdf)

**Click the image to view the full report.**

## Key Technologies and Implementation

The simulation utilises a **Distributor/Worker model** to divide the grid (board) into sections, distributing the computation load across multiple Goroutines.

| Feature | Technology / Concept | Description |
| :--- | :--- | :--- |
| **Primary Language** | Go (Golang) | Used for entire application logic. |
| **Concurrency** | **Goroutines** & **Channels** | Implements the **Distributor/Worker** pattern for parallel updates. |
| **Synchronisation** | **`sync.Mutex`** (`mu`) | Employed to safeguard shared variables (`world` and `turn`) from **data races**. |
| **Boundary Handling** | **`worldCopyForWorkers`** | The Distributor passes a duplicate world state, including adjacent edge cells, to Workers to restore **task isolation** for local computation. |
| **Visualization** | **SDL (Simple DirectMedia Layer)** | Handles the real-time graphical display of the grid state. |
| **Data I/O** | **PGM (Portable Graymap)** | Used for loading initial states and saving final/intermediate states. |
| **Domain** | **Closed Domain (Toroidal)** | Implements toric boundary conditions (pixels on opposite edges are connected). |

## Parallel Implementation Architecture

The project employs a robust multi-goroutine architecture, orchestrated by the central **Distributor Goroutine**.

### Architecture Diagrams

#### Distributor, Workers, and Alive Cells Count

This diagram shows the core parallel structure, where the **Distributor** manages the **Workers** and communicates state reporting via the `AliveCellsCount` event to the testing component (`TestAlive`).

![Preview](https://github.com/Jen0821/Parallel-GameOfLife/blob/main/diagrams1.jpg)

#### I/O Goroutine Communication

This diagram illustrates how the main process (`Distributor`) interacts with the dedicated **I/O Goroutine** for file handling and control signals, including the final PGM output file for testing (`TestPgm`).

![Preview](https://github.com/Jen0821/Parallel-GameOfLife/blob/main/diagrams2.jpg)

### Goroutine Roles and Communication

* **Distributor Goroutine (Main):** Acts as the **orchestrator**, managing the simulation loop (`for turn < p.Turns`) and the pool of Worker Goroutines. It aggregates results and listens for user keypresses via a `select` statement.
* **Worker Goroutines:** A pool of `p.Threads` workers compute the next state for their assigned **sub-region (slice)** of the world grid in parallel. They return results to the Distributor via the dedicated `results[w]` channel.
* **IO Goroutine:** Manages all PGM image reading and writing operations. It communicates with the Distributor using the `ioCommand` channel.
* **Ticker Goroutine:** Triggered by `time.Ticker`, it periodically calculates the total number of **Alive Cells**. It uses **`mu.Lock()`** to safely access the shared world state.

### Synchronisation and Data Safety

* **Mutex Protection:** Shared variables like the `world` and `turn` are protected by a **`sync.Mutex`** (`mu`) to prevent **data races**.
* **Exclusive Access:** Goroutines must call `mu.Lock()` before accessing `world` or `turn` and `mu.Unlock()` afterwards to guarantee exclusive access.

## Performance Analysis

Benchmark tests were executed on a Linux machine with **20 physical cores**, running the simulation for **1000 turns** on a **$512 \times 512$** grid.

| Implementation | Threads | Average Runtime (s) | Improvement |
| :--- | :--- | :--- | :--- |
| **Serial** | 1 | 83.5 | N/A |
| **Parallel (Optimal)** | 15 | 12.6 | 85% |

### Scaling and Constraints

* **Significant Speedup:** The parallel implementation reduced the mean runtime from **83.5 seconds** (serial) to **12.6 seconds** (parallel with 15 threads), achieving an **85%** performance improvement.
* **Limiting Factors:** Performance stabilisation occurred after about 13 threads. This is due to:
    1. The machine's hardware limit (**20 physical CPU cores**).
    2. The **I/O part** (PGM file handling) remains a **serial execution bottleneck**, limiting maximum achievable speedup.

## Initial State Example

The simulation starts from an initial PGM image representing the live and dead cells on the grid.

Here's an example of an initial board state:

![Preview](https://github.com/Jen0821/Parallel-GameOfLife/blob/main/Initial-State.jpg)

## ⚙️ Implemented Features (Step-by-Step)

The project integrates serial implementation with progressive parallel features, I/O, and user control.

### 1. Parallel Core Logic (Steps 1 & 2)

* **Serial Baseline:** Initial single-threaded implementation of the Game of Life rules.
* **Parallelization:** Implementation of the **Distributor** and **Worker** model. The Distributor divides the board into stripes and assigns them to a pool of **Worker Goroutines** (`gol.Params.Threads`) to calculate the next state in parallel.

### 2. State Reporting and Events (Step 3)

* **Alive Count Ticker:** Uses a **Ticker** to report the total number of **Alive Cells** via the `AliveCellsCount` event every **2 seconds**. This provides real-time feedback on the simulation's activity.

### 3. Image Output (Step 4)

* Implements logic to output the final state of the board as a **PGM image** after all turns have been completed.

### 4. User Control Rules (Step 5)

Interactive keyboard controls are processed by the main event loop:

* **`s` (Save):** Saves the current board state as a PGM image (protected by `mu`) and sends the `ImageOutputComplete` event.
* **`q` (Quit):** Completes the current turn, saves the final state as a PGM image, and terminates the program.
* **`p` (Pause/Resume):** Toggles the simulation state between running and paused (`StateChange` event).

### 5. Real-Time Visualisation (Step 6)

* Integration with **SDL** to display the simulation in real-time within a dedicated window.
* Utilizes **`CellFlipped`** (sent by Workers) and **`TurnComplete`** events to manage graphical updates efficiently.

Here's an example of the real-time visualisation:

![Preview](https://github.com/Jen0821/Parallel-GameOfLife/blob/main/SDL-Visualisation.jpg)

## Setup

### **Prerequisites**

Install Go and SDL development libraries.

#### Example of Successful JDK Setup (General Technical Prerequisite)

The project requires a proper development environment. This image shows an example of a correct JDK installation (`homebrew OpenJDK 17.0.14`) being registered in a project setting.

![Preview](https://github.com/Jen0821/Parallel-GameOfLife/blob/main/diagrams3.jpg)

### macOS (Homebrew)

```bash
brew install sdl2
```

### Linux (Ubuntu/Debian)

```bash
sudo apt-get install libsdl2-dev
```

### Windows

```bash
# Requires MinGW installation and manual SDL2 linking
# Refer to the Go SDL documentation for detailed platform-specific setup
```

## Running and Testing

To run the implementation and tests:

```bash
# Run the program with SDL visualisation (Step 6)
go run .

# Run tests with the SDL window to test visualisation and keyboard control (Step 5 & 6)
# Ensure SDL development libraries are installed on your system.
go test ./tests -v -run TestKeyboard -sdl

# Run tests with race detector for thread safety checks (Step 6)
go test ./tests -v -race
```

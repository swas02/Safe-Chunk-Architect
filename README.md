# Safe-Chunk Architecture & TaskGuard Module

## Professional Technical Documentation

**Version 1.0**

---

# 1. Executive Summary

The **Safe-Chunk Project Manager** is a professional-grade, fault-tolerant documentation and structured data-entry system built with PyQt.

It is engineered for:

* High-volume structured form input
* Long user sessions
* Zero-tolerance for data loss
* Guaranteed UI responsiveness
* Defensive runtime execution

The architecture combines:

1. **Modular JSON sharding**
2. **Atomic write protection**
3. **Instance locking**
4. **Autosave debouncing**
5. **Checkpoint versioning**
6. **Process-isolated execution safety (TaskGuard)**

The result is a layered defensive system that protects:

* Data integrity
* File integrity
* Runtime execution stability
* User experience continuity

---

# 2. Architectural Philosophy

The system is built around three non-negotiable principles:

 i. Never lose user data

 ii. Never freeze the UI

 iii. Never corrupt project files

All design decisions trace back to these constraints.

---

# 3. Modular Storage Model (Safe-Chunk Method)

## 3.1 Problem

Large monolithic JSON files:

* Slow writes
* Increased corruption risk
* Large data-loss surface
* Poor scalability

## 3.2 Solution: JSON Sharding

Projects are divided into independent **chunks**, typically grouped by form or logical section.

### Example Scenario:

* 10 forms
* 20 questions per form
* 200 total data points

Instead of writing 200 fields in one file:

* Each form writes to its own chunk
* Only modified sections are saved

---

## 3.3 Benefits

| Feature                  | Impact                 |
| ------------------------ | ---------------------- |
| Localized writes         | Faster saves           |
| Reduced corruption scope | Only one chunk at risk |
| Parallel scalability     | Modular growth         |
| Performance isolation    | No heavy rewrites      |

---

# 4. Project File Structure

```
projects/
└── {uniqueId}/
    ├── .lock
    ├── manifest.json
    ├── data_chunks/
    │   ├── form_1.json
    │   ├── form_2.json
    ├── data_chunks_bak/
    │   ├── form_1.bak
    │   ├── form_2.bak
    └── checkpoints/
        ├── full_project_2026_02_22.json
```

---

## 4.1 Component Breakdown

### `.lock`

Prevents multi-instance write collisions.

### `manifest.json`

Project metadata:

```json
{
  "project_name": "Bridge Inspection A1",
  "version": "1.0"
}
```

### `data_chunks/`

Live JSON shards.

### `data_chunks_bak/`

Automatic fallback backups.

### `checkpoints/`

Full project snapshots.

---

# 5. Write Safety Mechanisms

## 5.1 Atomic Swap Strategy

Files are never overwritten directly.

### Write Process:

1. Save to `chunk_01.tmp`
2. Rename existing `chunk_01.json` → `chunk_01.bak`
3. Rename `chunk_01.tmp` → `chunk_01.json`

### Guarantees:

* No partial writes
* No zero-byte files
* Power-loss protection
* Always recoverable state

---

## 5.2 Corruption Recovery

If a chunk fails to load:

1. Attempt primary `.json`
2. Fallback to `.bak`
3. Log integrity warning

Worst-case data loss:

> Only the most recent edit cycle

---

# 6. State Management Strategy

## 6.1 Single Source of Truth

The system uses Python's object referencing model:

* UI widgets reference mutable dictionaries
* Manager references the same dictionaries
* Changes propagate instantly

No manual synchronization required.

### Conceptual Example:

```python
form_1 = {"material_name": "Glass"}
forms = [form_1]

for form in forms:
    form["is_recyclable"] = True
```

All references update automatically.

---

## 6.2 Architectural Benefit

* No event bus required
* No state duplication
* No serialization layer between UI and model
* Reduced complexity

---

# 7. Autosave Strategy

## 7.1 Debounced Save

A timer resets on every user input.

Save triggers only when:

> User pauses for defined duration (e.g., 3 seconds)

### Advantages:

* Reduced disk wear
* Avoids redundant writes
* Preserves responsiveness
* Maintains data safety

---

# 8. Instance Locking Mechanism

## 8.1 Problem

Two active writers on same project = corruption risk.

## 8.2 Solution

Creation of `.lock` file upon project open.

If another instance attempts open:

* Lock detected
* User warned
* Access denied

Lock removed on clean shutdown.

---

# 9. TaskGuard Module (Execution Safety Layer)

## 9.1 Purpose

TaskGuard protects the application from:

* Infinite loops
* Heavy CPU-bound hangs
* Blocking operations
* Faulty business logic
* Third-party computation failures

Its purpose:

> Ensure the GUI never freezes due to a bad function.

---

## 9.2 Why Process Isolation?

Threads cannot safely kill:

* Infinite loops
* C-extension deadlocks
* CPU-bound hangs

TaskGuard uses:

```python
multiprocessing.Process
```

Processes can be terminated at the OS level.

---

## 9.3 Execution Model

### Step 1 — Spawn Child Process

Function runs in isolated environment.

### Step 2 — Result Communication

Child sends result via:

```python
multiprocessing.Queue
```

### Step 3 — Timed Wait

```python
process.join(timeout)
```

### Step 4 — Timeout Handling

If still alive after timeout:

```python
process.terminate()
```

Process is forcefully killed.

Main application remains responsive.

---

## 9.4 Return Contract

```python
(True, result)
(False, error_message)
```

Clear success/failure semantics.

---

## 9.5 Example Failure Case

```python
def engineering_calc(length):
    if length < 0:
        while True:
            pass
```

Without TaskGuard:

* Application freezes indefinitely.

With TaskGuard:

* Process terminated after timeout.
* Error message returned.
* UI remains responsive.

---

# 10. Layered Defensive Model

The system protects against failure at multiple levels:

| Layer         | Protection                  |
| ------------- | --------------------------- |
| Chunking      | Limits data-loss scope      |
| Atomic Writes | Prevents partial corruption |
| Backups       | Enables recovery            |
| Locking       | Prevents concurrent writes  |
| Debounce      | Optimizes safe persistence  |
| TaskGuard     | Prevents runtime freeze     |

This forms a complete defensive stack.

---

# 11. Failure Scenarios & Outcomes

| Scenario                     | Result                                               |
| ---------------------------- | ---------------------------------------------------- |
| Power loss during save       | Backup remains                                       |
| Corrupted chunk              | Fallback to `.bak`                                   |
| Infinite loop in calculation | Process killed                                       |
| Double project open          | Lock prevents write                                  |
| Unexpected crash             | Lock cleared on next safe startup (policy dependent) |

---

# 12. Scalability Characteristics

### Horizontal Scalability

* Add new forms → add new chunks
* No architecture change required

### Performance Scalability

* Writes scale with active section only
* No global rewrite bottleneck

### Maintenance Scalability

* Chunk isolation simplifies debugging
* Clear file boundaries

---

# 13. Data Export Model

To share projects:

System performs **Flattened Export**:

1. Load all chunks
2. Merge into single JSON structure
3. Output portable file

Internal modularity remains invisible to end user.

---

# 14. Security & Stability Considerations

* File operations atomic
* Runtime execution isolated
* Corruption fallback built-in
* No direct user exposure to internal folder structure
* Deterministic failure behavior

---

# 15. Architectural Assessment

### Reliability: ★★★★★

### Data Safety: ★★★★★

### Runtime Stability: ★★★★★

### Scalability: ★★★★☆

### Maintainability: ★★★★☆

This architecture reflects:

* Defensive engineering mindset
* Production-grade data handling
* Professional desktop software design patterns
* Enterprise reliability principles adapted to PySide

---

# 17. Practical Q&A – Simple Thinking Behind the Architecture

This section explains the system using straightforward reasoning.
Each answer reflects the core mindset behind the design.

---

## Q1: Why don’t we just save everything into one big file?

**Simple Answer:**
Because big files break big.

If one giant JSON file gets corrupted:

* Everything is at risk.
* Recovery becomes complex.
* Save operations become heavy and slow.

By splitting into chunks:

* Only one small section is affected if something goes wrong.
* Saves are faster.
* Risk is localized.

Think of it like storing documents in folders instead of one massive binder.

---

## Q2: Why do we create backup files every time?

**Simple Answer:**
Because computers can fail at any moment.

Power cuts.
Disk hiccups.
OS crashes.

If we overwrite directly and the system stops halfway:

* The file may become empty.
* Data may become unreadable.

By using the swap method:

* The old version is always preserved.
* There is always something to recover.

We assume failure is possible — and design for it.

---

## Q3: Why use a `.lock` file?

**Simple Answer:**
Two writers at the same time equals corruption.

If two app instances write to the same file:

* They overwrite each other.
* JSON structure can break.
* Data becomes inconsistent.

The `.lock` file acts like:

> “This project is currently in use. Do not enter.”

It prevents silent damage.

---

## Q4: Why autosave after a pause instead of every keystroke?

**Simple Answer:**
Because saving too often wastes resources.

If we save every keypress:

* Disk wear increases.
* CPU usage increases.
* System feels heavy.

If we save only when the user pauses:

* We capture meaningful progress.
* We reduce unnecessary writes.
* Performance stays smooth.

We balance safety with efficiency.

---

## Q5: Why use Python’s object referencing instead of syncing manually?

**Simple Answer:**
Because simpler systems fail less.

If UI and Manager share the same dictionary:

* Change in UI = change in memory
* No synchronization layer required
* No duplicate state bugs

More syncing logic means more failure points.

We eliminate complexity where possible.

---

## Q6: Why create a separate process for risky calculations (TaskGuard)?

**Simple Answer:**
Because some mistakes never return.

If a function enters an infinite loop:

* The GUI freezes.
* The user must force-close the app.
* Unsaved work may be lost.

Threads cannot safely stop infinite loops.

Processes can.

By isolating risky logic in another process:

* If it hangs, we kill it.
* The main application survives.
* The user continues working.

We isolate danger instead of trusting perfect code.

---

## Q7: Why not just trust developers to write safe functions?

**Simple Answer:**
Because edge cases always exist.

Even experienced developers:

* Miss rare conditions
* Encounter unexpected input
* Integrate third-party code

Professional systems assume:

> Something will eventually go wrong.

The architecture absorbs failure instead of assuming perfection.

---

## Q8: What happens if a chunk file gets corrupted?

**Simple Answer:**
We step back one save.

The system:

1. Detects failure.
2. Loads the `.bak` version.
3. Continues safely.

Worst-case loss:

* A few minutes of edits.

Not the entire project.

---

## Q9: Why hide the internal folder structure from the user?

**Simple Answer:**
Because accidental tampering causes damage.

If users manually:

* Delete chunk files
* Rename folders
* Move internal JSON

They can break the project.

We treat the project folder as a managed black box.

Users interact only through the application.

---

## Q10: What is the overall philosophy behind this system?

**Simple Answer:**
Expect failure. Prevent catastrophe.

We assume:

* Power loss can happen.
* Code can hang.
* Files can corrupt.
* Users can make mistakes.

Instead of hoping these never occur,
we design so that when they do:

* Damage is limited.
* Recovery is automatic.
* The application remains stable.

---

# 18. Conclusion

The **Safe-Chunk Project Manager Architecture**, combined with the **TaskGuard execution safety layer**, delivers a structured, defensive, and production-ready desktop system designed for high-trust professional environments.

The architecture ensures:

* Modular data isolation through JSON sharding
* Minimal data-loss exposure via chunk-based storage
* Atomic write guarantees with swap-based persistence
* Automatic corruption recovery through backup mirroring
* Concurrency protection using instance locking
* Resource-efficient autosave through debounced timing
* Runtime execution isolation using process-based TaskGuard

The system is built around a clear principle:

> Anticipate failure. Limit its impact. Preserve user trust.

Each layer addresses a specific class of risk:

| Risk Type                | Mitigation Layer                |
| ------------------------ | ------------------------------- |
| File corruption          | Atomic swap + backups           |
| Power loss               | Write staging + fallback        |
| Concurrent writes        | `.lock` mechanism               |
| Infinite loops           | Process termination (TaskGuard) |
| Long-running computation | Timeout enforcement             |
| Large project scaling    | Chunk-based modular storage     |

The result is a cohesive reliability stack that protects:

* Data integrity
* File system consistency
* Application responsiveness
* Long-session user workflows

This architecture is particularly suited for:

* Engineering documentation systems
* Structured inspection tools
* Compliance and audit platforms
* Large-form enterprise data entry systems
* Professional environments requiring deterministic behavior

It prioritizes:

* Predictable failure handling
* Isolation of risk domains
* Maintainable modular design
* Clear separation between storage, state, and execution

In practical terms, the system transforms:

* Potential freeze states → Controlled timeouts
* Partial writes → Recoverable versions
* Corruption events → Localized restoration
* Heavy saves → Lightweight modular updates

The Safe-Chunk + TaskGuard model represents a disciplined, reliability-first approach to desktop application architecture.

It is not optimized merely for functionality —
it is optimized for **stability under stress**.

# Coherence Assistant Lab

**Coherence-based control for Siri-style assistants, tested in small worlds.**

This repo contains a set of Colab/Notebook experiments where a small local LLM (Phi-3) is used as a **language front-end**, and a separate **coherence core** maintains structured state and enforces constraints for assistant-like tasks.

The goal: show that many “classic Siri failures” are not about the model being too small, but about the **architecture**. When we:

- treat the LLM as *language cortex* (parsing & suggestions),
- and let a small **coherence engine** control the actual actions,

we get assistants that are **more reliable, cheaper, and easier to interpret**.

---

## 1. High-level idea

Typical assistant stack (today):

> speech → LLM → directly calls Calendar/Reminders/Clock/Maps

Your proposal:

> speech → LLM (NLU + hints) →  
> → **Coherence Core** (world state + constraints + energy minimization) →  
> → Calendar / Reminders / Clock / Shortcuts / Maps

The **Coherence Core** holds explicit state:

- Trips (home → station → train → destination)
- Alarms (weekday patterns)
- Persona/preferences (vegetarian, hates loud, budget, etc.)
- Habits/shortcuts (repeated sequences)
- Safety info (trusted contacts, location-sharing rules)

For each new request, the core:

1. Adds constraints (from the LLM-parsed schema),
2. Finds a **low-energy / high-coherence state** that satisfies all the rules,
3. Only then emits concrete actions (times, alarms, events, shortcuts, allow/deny).

The LLM never directly owns the world state; it just helps turn messy language into constraints.

---

## 2. Experiments

All experiments run in a single notebook:

- `notebooks/coherence_assistant_lab.ipynb`

The model: `microsoft/Phi-3-mini-4k-instruct` running locally via `transformers`.

For each task, we compare:

- **LLM-only baseline**: Phi is asked to do everything end-to-end.
- **LLM + Coherence**: Phi only parses language; a tiny Python coherence engine enforces constraints.

### 2.1 Multi-step trip planning (Failure Mode 1)

Task (toy world):

> “Book me a trip from home in Kraków to Warsaw tomorrow morning.  
> I must arrive by 09:00/10:00/11:00.  
> Create ONE calendar event for the whole trip door-to-door,  
> and set a reminder 1 hour before I leave home.  
> I need 30 minutes from home to the station, and 2 hours on the train.”

Constraints:

- `arrival_time ≤ latest_allowed_arrival`
- `train_departure = arrival_time − 2h`
- `home_departure = train_departure − 30min`
- `calendar_event = [home_departure, arrival_time]`
- `reminder_time = home_departure − 60min`
- (in a variant) `home_departure ≥ 07:00`

**Results (toy benchmark):**

| Task                                         | LLM-only (Phi) | LLM + Coherence Basin |
|---------------------------------------------|----------------|------------------------|
| Trip planning (Level 2/3 constraints)       | ~0% success    | **100% success**       |

The coherence version treats times as a small vector and snaps them onto the constraint manifold, instead of trusting the LLM’s raw JSON.

---

### 2.2 Alarm revision & consistency (Failure Mode 2)

Task:

1. “Set an alarm for 6:30 on weekdays.”  
2. “Actually, make Friday 7:00.”

Desired final pattern:

- Mon–Thu: 06:30  
- Fri: 07:00  
- Weekend: no alarm  
- No duplicates, minimal change.

**LLM-only baseline:**

- Phi gets:
  - current alarm list as JSON,
  - user command,
  - and is asked to output an updated JSON list.
- In practice: unstable, duplicates, wrong days, or JSON issues.

**Coherence core:**

- Represent as `AlarmRoutine = {day → time_or_None}`.
- New commands become simple constraint schemas, e.g.:

  ```python
  {"type": "set_days_time", "days": [...], "time": ...}

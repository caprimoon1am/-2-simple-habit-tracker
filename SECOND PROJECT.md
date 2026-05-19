# Daily Habit Tracker

A minimal, pastel-styled habit tracker built with React + TypeScript + Vite. No backend, no database — everything lives in `localStorage`. Three habits, seven days, one cycle at a time.

---

## What it does

- **3 habits per cycle** — names and emojis are editable between cycles.
- **7-day cycle** — one checkbox row per day. The week bar at the bottom shows all 7 days colour-coded by completion count.
- **Setup screen** — after day 7 ends the app pauses and asks you to confirm or change your habits before starting a new cycle.
- **One-time past-day edit** — you can fix one past day per day (each slot can only be edited once after the fact).
- **History tab** — completed cycles are archived automatically so you can see your past weeks.
- **Test Controls panel** — a collapsible debug panel (bottom-right) lets you simulate a cycle end, populate past days, seed fake history, and reset everything.

---

## Tech stack

| Layer | Choice |
|---|---|
| Framework | React 19 + TypeScript |
| Build tool | Vite 7 |
| Styling | Plain CSS (no UI library, no Tailwind in the main app) |
| Persistence | `localStorage` only |
| Fonts | Inter (Google Fonts, loaded in `index.html`) |

---

## How to replicate from scratch

### 1 — Scaffold a Vite + React + TypeScript project

```bash
npm create vite@latest habit-tracker -- --template react-ts
cd habit-tracker
npm install
```

### 2 — Replace the source files

Delete everything inside `src/` and replace with the four files below.  
Replace `index.html` with the one below.  
You do **not** need any extra npm packages beyond the standard Vite + React + TypeScript scaffold.

---

## Source files

### `index.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1" />
    <title>Daily Habit Tracker</title>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

### `src/main.tsx`

```tsx
import { createRoot } from "react-dom/client";
import App from "./App";
import "./index.css";

createRoot(document.getElementById("root")!).render(<App />);
```

---

### `src/DemoControls.tsx`

```tsx
import { useState } from "react";

interface DemoControlsProps {
  onSimulateCycleEnd: () => void;
  onPopulatePastDays: () => void;
  onResetDemo: () => void;
  onSeedHistory: () => void;
}

export default function DemoControls({
  onSimulateCycleEnd,
  onPopulatePastDays,
  onResetDemo,
  onSeedHistory,
}: DemoControlsProps) {
  const [open, setOpen] = useState(false);

  return (
    <div className="demo-controls">
      <button
        className="demo-toggle"
        onClick={() => setOpen((o) => !o)}
        aria-expanded={open}
        title="Demo / test controls — not part of normal use"
      >
        🧪 Test Controls {open ? "▲" : "▼"}
      </button>

      {open && (
        <div className="demo-panel">
          <p className="demo-label">Demo only — does not affect normal use</p>
          <div className="demo-buttons">
            <button className="demo-btn" onClick={onSimulateCycleEnd}>
              Simulate Cycle End
            </button>
            <button className="demo-btn" onClick={onPopulatePastDays}>
              Populate Past Days
            </button>
            <button className="demo-btn" onClick={onSeedHistory}>
              Seed 3 History Cycles
            </button>
            <button className="demo-btn demo-btn--danger" onClick={onResetDemo}>
              Reset Demo
            </button>
          </div>
        </div>
      )}
    </div>
  );
}
```

---

### `src/App.tsx`

```tsx
import { useState, useEffect, useCallback } from "react";
import DemoControls from "./DemoControls";

// ─── Constants ────────────────────────────────────────────────────────────────

const DEFAULT_HABIT_NAMES: [string, string, string] = [
  "Manifest 2 mins",
  "Log wins",
  "Write 3 lines of affirmation",
];

const DEFAULT_HABIT_EMOJIS: [string, string, string] = ["✨", "🏆", "💫"];

const WEEKDAYS = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];

const CYCLE_KEY = "habitTrackerCycle";
const NAMES_KEY = "habitTrackerNames";
const EMOJIS_KEY = "habitTrackerEmojis";
const HISTORY_KEY = "habitTrackerHistory";

// ─── Types ────────────────────────────────────────────────────────────────────

interface DayData {
  date: string;
  habits: [boolean, boolean, boolean];
  editUsed?: boolean;
}

interface CycleData {
  cycleStartDate: string;
  days: DayData[];
}

interface HistoryEntry {
  id: string;
  startDate: string;
  endDate: string;
  habitNames: [string, string, string];
  days: [boolean, boolean, boolean][];
  perDayCompleted: number[];
  totalCompleted: number;
  totalPossible: 21;
  completionPct: number;
}

type AppMode = "tracking" | "setup";

// ─── Habit name helpers ───────────────────────────────────────────────────────

function loadHabitNames(): [string, string, string] {
  try {
    const raw = localStorage.getItem(NAMES_KEY);
    if (!raw) return [...DEFAULT_HABIT_NAMES];
    const parsed = JSON.parse(raw);
    if (
      Array.isArray(parsed) &&
      parsed.length === 3 &&
      parsed.every((n) => typeof n === "string" && n.trim().length > 0)
    ) {
      return parsed as [string, string, string];
    }
  } catch {}
  return [...DEFAULT_HABIT_NAMES];
}

function saveHabitNames(names: [string, string, string]): void {
  localStorage.setItem(NAMES_KEY, JSON.stringify(names));
}

// ─── Habit emoji helpers ──────────────────────────────────────────────────────

function loadHabitEmojis(): [string, string, string] {
  try {
    const raw = localStorage.getItem(EMOJIS_KEY);
    if (raw === null) return [...DEFAULT_HABIT_EMOJIS];
    const parsed = JSON.parse(raw);
    if (
      Array.isArray(parsed) &&
      parsed.length === 3 &&
      parsed.every((e) => typeof e === "string")
    ) {
      return parsed as [string, string, string];
    }
  } catch {}
  return [...DEFAULT_HABIT_EMOJIS];
}

function saveHabitEmojis(emojis: [string, string, string]): void {
  localStorage.setItem(EMOJIS_KEY, JSON.stringify(emojis));
}

// ─── History helpers ──────────────────────────────────────────────────────────

function loadHistory(): HistoryEntry[] {
  try {
    const raw = localStorage.getItem(HISTORY_KEY);
    if (!raw) {
      localStorage.setItem(HISTORY_KEY, "[]");
      return [];
    }
    const parsed = JSON.parse(raw);
    if (Array.isArray(parsed)) return parsed as HistoryEntry[];
  } catch {}
  return [];
}

function saveHistory(history: HistoryEntry[]): void {
  localStorage.setItem(HISTORY_KEY, JSON.stringify(history));
}

function snapshotCycleToHistory(): HistoryEntry[] | null {
  try {
    const raw = localStorage.getItem(CYCLE_KEY);
    if (!raw) return null;
    const cycle = JSON.parse(raw) as CycleData;
    if (
      !cycle ||
      typeof cycle.cycleStartDate !== "string" ||
      !Array.isArray(cycle.days) ||
      cycle.days.length !== 7
    ) {
      return null;
    }

    const existing = loadHistory();
    if (existing.some((e) => e.id === cycle.cycleStartDate)) {
      return null;
    }

    const names = loadHabitNames();
    const days = cycle.days.map((d) => d.habits as [boolean, boolean, boolean]);
    const perDayCompleted = days.map((h) => h.filter(Boolean).length);
    const totalCompleted = perDayCompleted.reduce((a, b) => a + b, 0);
    const totalPossible = 21 as const;
    const completionPct = Math.round((totalCompleted / totalPossible) * 100);

    const entry: HistoryEntry = {
      id: cycle.cycleStartDate,
      startDate: cycle.cycleStartDate,
      endDate: cycle.days[6].date,
      habitNames: names,
      days,
      perDayCompleted,
      totalCompleted,
      totalPossible,
      completionPct,
    };

    const updated = [entry, ...existing];
    saveHistory(updated);
    return updated;
  } catch {
    return null;
  }
}

// ─── Date helpers ─────────────────────────────────────────────────────────────

function getTodayString(): string {
  const d = new Date();
  const y = d.getFullYear();
  const m = String(d.getMonth() + 1).padStart(2, "0");
  const dd = String(d.getDate()).padStart(2, "0");
  return `${y}-${m}-${dd}`;
}

function getWeekdayLabel(dateStr: string): string {
  const [y, m, d] = dateStr.split("-").map(Number);
  return WEEKDAYS[new Date(y, m - 1, d).getDay()];
}

function addDays(dateStr: string, n: number): string {
  const [y, m, d] = dateStr.split("-").map(Number);
  const result = new Date(y, m - 1, d + n);
  const ry = result.getFullYear();
  const rm = String(result.getMonth() + 1).padStart(2, "0");
  const rd = String(result.getDate()).padStart(2, "0");
  return `${ry}-${rm}-${rd}`;
}

function daysSinceStart(cycleStartDate: string): number {
  const today = getTodayString();
  const [sy, sm, sd] = cycleStartDate.split("-").map(Number);
  const [ty, tm, td] = today.split("-").map(Number);
  const startMs = new Date(sy, sm - 1, sd).getTime();
  const todayMs = new Date(ty, tm - 1, td).getTime();
  return Math.floor((todayMs - startMs) / (1000 * 60 * 60 * 24));
}

function formatDateRange(startDate: string, endDate: string): string {
  const months = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"];
  const [, sm, sd] = startDate.split("-").map(Number);
  const [, em, ed] = endDate.split("-").map(Number);
  return `${months[sm - 1]} ${sd} – ${months[em - 1]} ${ed}`;
}

// ─── Cycle helpers ────────────────────────────────────────────────────────────

function createCycle(startDate: string): CycleData {
  const days: DayData[] = Array.from({ length: 7 }, (_, i) => ({
    date: addDays(startDate, i),
    habits: [false, false, false],
    editUsed: false,
  }));
  return { cycleStartDate: startDate, days };
}

function loadCycle(): CycleData | null {
  try {
    const raw = localStorage.getItem(CYCLE_KEY);
    if (!raw) return null;
    const parsed = JSON.parse(raw) as CycleData;
    if (
      typeof parsed.cycleStartDate !== "string" ||
      !Array.isArray(parsed.days) ||
      parsed.days.length !== 7
    ) {
      return null;
    }
    parsed.days = parsed.days.map((day) => ({
      ...day,
      editUsed: day.editUsed ?? false,
    }));
    return parsed;
  } catch {
    return null;
  }
}

function saveCycle(cycle: CycleData): void {
  localStorage.setItem(CYCLE_KEY, JSON.stringify(cycle));
}

function getInitResult():
  | { mode: "tracking"; cycle: CycleData; todayIndex: number }
  | { mode: "setup" } {
  const today = getTodayString();
  const cycle = loadCycle();

  if (cycle) {
    const elapsed = daysSinceStart(cycle.cycleStartDate);
    if (elapsed >= 7) {
      return { mode: "setup" };
    }
    return { mode: "tracking", cycle, todayIndex: elapsed };
  }

  const newCycle = createCycle(today);
  saveCycle(newCycle);
  return { mode: "tracking", cycle: newCycle, todayIndex: 0 };
}

// ─── Mini dot color map (matches week-slot completion colours) ────────────────

const DOT_COLORS: Record<number, { bg: string; border: string }> = {
  0: { bg: "#fdf0f0", border: "#f2c4c4" },
  1: { bg: "#fdf7ee", border: "#D9A96B" },
  2: { bg: "#eaf3f7", border: "#C2D7E1" },
  3: { bg: "#f2eef1", border: "#56394F" },
};

// ─── Component ────────────────────────────────────────────────────────────────

export default function App() {
  const [appMode, setAppMode] = useState<AppMode>(() => getInitResult().mode);

  const [{ cycle, todayIndex }, setHabitState] = useState(() => {
    const result = getInitResult();
    if (result.mode === "tracking") {
      return { cycle: result.cycle, todayIndex: result.todayIndex };
    }
    return { cycle: createCycle(getTodayString()), todayIndex: 0 };
  });

  const [habitNames, setHabitNames] = useState<[string, string, string]>(() =>
    loadHabitNames()
  );
  const [habitEmojis, setHabitEmojis] = useState<[string, string, string]>(() =>
    loadHabitEmojis()
  );

  const [setupNames, setSetupNames] = useState<[string, string, string]>(() =>
    loadHabitNames()
  );
  const [setupEmojis, setSetupEmojis] = useState<[string, string, string]>(() =>
    loadHabitEmojis()
  );
  const [setupError, setSetupError] = useState<string>("");

  const [viewingDayIndex, setViewingDayIndex] = useState(() => {
    const result = getInitResult();
    return result.mode === "tracking" ? result.todayIndex : 0;
  });

  const [activeEditingPastDayIndex, setActiveEditingPastDayIndex] = useState<number | null>(null);
  const [lockedMsg, setLockedMsg] = useState<string>("");

  const [history, setHistory] = useState<HistoryEntry[]>(() => loadHistory());
  const [showHistory, setShowHistory] = useState(false);

  useEffect(() => {
    if (appMode === "setup") {
      const updated = snapshotCycleToHistory();
      if (updated) setHistory(updated);
    }
  }, [appMode]);

  const refreshDate = useCallback(() => {
    const result = getInitResult();
    if (result.mode === "setup") {
      setAppMode("setup");
      setSetupNames(loadHabitNames());
    } else {
      setAppMode("tracking");
      setHabitState({ cycle: result.cycle, todayIndex: result.todayIndex });
      setViewingDayIndex(result.todayIndex);
      setActiveEditingPastDayIndex(null);
      setLockedMsg("");
    }
    setSetupNames(loadHabitNames());
    setSetupEmojis(loadHabitEmojis());
  }, []);

  useEffect(() => {
    window.addEventListener("focus", refreshDate);
    return () => window.removeEventListener("focus", refreshDate);
  }, [refreshDate]);

  useEffect(() => {
    setViewingDayIndex(todayIndex);
    setActiveEditingPastDayIndex(null);
    setLockedMsg("");
  }, [todayIndex]);

  const handleStartNextCycle = (
    names: [string, string, string],
    emojis: [string, string, string],
  ) => {
    const trimmedNames = names.map((n) => n.trim()) as [string, string, string];
    if (trimmedNames.some((n) => n.length === 0)) {
      setSetupError("Please fill in all 3 habits.");
      return;
    }
    const trimmedEmojis = emojis.map((e) => e.trim()) as [string, string, string];
    setSetupError("");

    saveHabitNames(trimmedNames);
    setHabitNames(trimmedNames);
    saveHabitEmojis(trimmedEmojis);
    setHabitEmojis(trimmedEmojis);

    const newCycle = createCycle(getTodayString());
    saveCycle(newCycle);

    setHabitState({ cycle: newCycle, todayIndex: 0 });
    setViewingDayIndex(0);
    setLockedMsg("");
    setShowHistory(false);
    setAppMode("tracking");
  };

  const handleKeepSameHabits = () => {
    handleStartNextCycle(habitNames, habitEmojis);
  };

  // ── Demo controls ────────────────────────────────────────────────────────

  const handleSeedHistory = useCallback(() => {
    const today = getTodayString();
    const seedCycles: Array<{
      startDate: string;
      habits: [boolean, boolean, boolean][];
      names: [string, string, string];
    }> = [
      {
        startDate: addDays(today, -21),
        names: ["Morning meditation", "Drink 8 glasses of water", "Read 10 pages"],
        habits: [
          [true, true, true],[true, true, false],[false, false, false],
          [true, true, true],[true, false, true],[true, true, true],[true, true, false],
        ],
      },
      {
        startDate: addDays(today, -14),
        names: ["Morning meditation", "Drink 8 glasses of water", "Read 10 pages"],
        habits: [
          [true, false, true],[true, true, true],[true, true, true],
          [false, false, true],[true, true, true],[true, true, false],[true, true, true],
        ],
      },
      {
        startDate: addDays(today, -7),
        names: habitNames,
        habits: [
          [true, true, true],[true, true, true],[false, true, true],
          [true, true, true],[true, false, false],[true, true, true],[true, true, true],
        ],
      },
    ];

    const entries: HistoryEntry[] = seedCycles.map(({ startDate, names, habits }) => {
      const endDate = addDays(startDate, 6);
      const perDayCompleted = habits.map((h) => h.filter(Boolean).length);
      const totalCompleted = perDayCompleted.reduce((a, b) => a + b, 0);
      const totalPossible = 21 as const;
      const completionPct = Math.round((totalCompleted / totalPossible) * 100);
      return { id: startDate, startDate, endDate, habitNames: names, days: habits,
               perDayCompleted, totalCompleted, totalPossible, completionPct };
    });

    const reversed = [...entries].reverse();
    saveHistory(reversed);
    setHistory(reversed);
    setShowHistory(true);
  }, [habitNames]);

  const handleSimulateCycleEnd = useCallback(() => {
    setSetupNames(loadHabitNames());
    setSetupEmojis(loadHabitEmojis());
    setSetupError("");
    setActiveEditingPastDayIndex(null);
    setLockedMsg("");
    setAppMode("setup");
  }, []);

  const handlePopulatePastDays = useCallback(() => {
    const today = getTodayString();
    const startDate = addDays(today, -6);
    const sampleHabits: [boolean, boolean, boolean][] = [
      [true, true, true],[true, false, true],[false, false, false],
      [true, true, false],[true, false, false],[true, true, true],
    ];
    const days: DayData[] = Array.from({ length: 7 }, (_, i) => ({
      date: addDays(startDate, i),
      habits: i < 6 ? sampleHabits[i] : [false, false, false],
      editUsed: false,
    }));
    const newCycle: CycleData = { cycleStartDate: startDate, days };
    saveCycle(newCycle);
    setHabitState({ cycle: newCycle, todayIndex: 6 });
    setViewingDayIndex(6);
    setActiveEditingPastDayIndex(null);
    setLockedMsg("");
    setAppMode("tracking");
  }, []);

  const handleResetDemo = useCallback(() => {
    localStorage.removeItem(CYCLE_KEY);
    localStorage.removeItem(NAMES_KEY);
    localStorage.removeItem(EMOJIS_KEY);
    localStorage.removeItem(HISTORY_KEY);
    setHistory([]);
    const result = getInitResult();
    const names = loadHabitNames();
    const emojis = loadHabitEmojis();
    setSetupNames(names);
    setHabitNames(names);
    setSetupEmojis(emojis);
    setHabitEmojis(emojis);
    setSetupError("");
    setActiveEditingPastDayIndex(null);
    setLockedMsg("");
    setShowHistory(false);
    if (result.mode === "tracking") {
      setHabitState({ cycle: result.cycle, todayIndex: result.todayIndex });
      setViewingDayIndex(result.todayIndex);
      setAppMode("tracking");
    } else {
      setAppMode("setup");
    }
  }, []);

  // ── Tracking actions ─────────────────────────────────────────────────────

  const toggleHabit = (habitIndex: number) => {
    const viewedState = getDayState(viewingDayIndex);
    const isViewingPastEdit = activeEditingPastDayIndex === viewingDayIndex;
    if (viewedState !== "today" && !isViewingPastEdit) return;

    setHabitState((prev) => {
      const initResult = getInitResult();
      const { cycle: freshCycle, todayIndex: freshTodayIndex } =
        initResult.mode === "tracking"
          ? initResult
          : { cycle: prev.cycle, todayIndex: prev.todayIndex };

      const writeIndex =
        viewingDayIndex === prev.todayIndex ? freshTodayIndex : viewingDayIndex;

      const updatedDays = freshCycle.days.map((day, i) => {
        if (i !== writeIndex) return day;
        const newHabits: [boolean, boolean, boolean] = [...day.habits] as [boolean, boolean, boolean];
        newHabits[habitIndex] = !newHabits[habitIndex];
        return { ...day, habits: newHabits };
      });

      const updatedCycle = { ...freshCycle, days: updatedDays };
      saveCycle(updatedCycle);
      return { cycle: updatedCycle, todayIndex: freshTodayIndex };
    });
  };

  const handleSlotClick = (i: number) => {
    const state = getDayState(i);
    if (state === "future") return;

    if (state === "today") {
      setViewingDayIndex(todayIndex);
      setActiveEditingPastDayIndex(null);
      setLockedMsg("");
      return;
    }

    const day = cycle.days[i];
    if (day.editUsed) {
      setViewingDayIndex(i);
      setActiveEditingPastDayIndex(null);
      setLockedMsg(`${getWeekdayLabel(day.date)} has already been edited and is locked.`);
      return;
    }

    setLockedMsg("");
    setViewingDayIndex(i);
    setActiveEditingPastDayIndex(i);
    setHabitState((prev) => {
      const updatedDays = prev.cycle.days.map((d, idx) => {
        if (idx !== i) return d;
        return { ...d, editUsed: true };
      });
      const updatedCycle = { ...prev.cycle, days: updatedDays };
      saveCycle(updatedCycle);
      return { cycle: updatedCycle, todayIndex: prev.todayIndex };
    });
  };

  // ── Derived values ───────────────────────────────────────────────────────

  const getDayState = (i: number): "future" | "today" | "past" => {
    if (i > todayIndex) return "future";
    if (i === todayIndex) return "today";
    return "past";
  };

  const getCompletedForDay = (i: number): number =>
    cycle.days[i]?.habits.filter(Boolean).length ?? 0;

  const viewedDayData = cycle.days[viewingDayIndex];
  const isViewingToday = viewingDayIndex === todayIndex;
  const isViewingEditable = isViewingToday || activeEditingPastDayIndex === viewingDayIndex;
  const completedToday = cycle.days[todayIndex]?.habits.filter(Boolean).length ?? 0;
  const completedViewed = viewedDayData?.habits.filter(Boolean).length ?? 0;
  const allDoneToday = completedToday === 3;

  // ─── Render ─────────────────────────────────────────────────────────────────

  const demoControls = (
    <DemoControls
      onSimulateCycleEnd={handleSimulateCycleEnd}
      onPopulatePastDays={handlePopulatePastDays}
      onResetDemo={handleResetDemo}
      onSeedHistory={handleSeedHistory}
    />
  );

  // ── Setup screen ──────────────────────────────────────────────────────────

  if (appMode === "setup") {
    return (
      <div className="page">
        {demoControls}
        <div className="card">
          <div className="header">
            <h1 className="title">Daily Check-In 🤍</h1>
            <p className="subtitle">Nice work finishing the week!</p>
          </div>

          <p className="setup-heading">What are your habits for the next 7 days?</p>

          <div className="setup-fields">
            {([0, 1, 2] as const).map((i) => (
              <div key={i} className="setup-field-row">
                <input
                  className="setup-emoji-input"
                  type="text"
                  value={setupEmojis[i]}
                  onChange={(e) => {
                    const updated = [...setupEmojis] as [string, string, string];
                    updated[i] = e.target.value;
                    setSetupEmojis(updated);
                  }}
                  maxLength={4}
                  aria-label={`Emoji for habit ${i + 1}`}
                />
                <input
                  className="setup-input"
                  type="text"
                  value={setupNames[i]}
                  onChange={(e) => {
                    const updated = [...setupNames] as [string, string, string];
                    updated[i] = e.target.value;
                    setSetupNames(updated);
                  }}
                  placeholder={`Habit ${i + 1}`}
                  maxLength={60}
                />
              </div>
            ))}
          </div>

          {setupError && <p className="setup-error">{setupError}</p>}

          <button
            className="btn-primary"
            onClick={() => handleStartNextCycle(setupNames, setupEmojis)}
          >
            Start new cycle
          </button>
          <button className="btn-secondary" onClick={handleKeepSameHabits}>
            Keep same habits
          </button>
        </div>
      </div>
    );
  }

  // ── Tracking screen ───────────────────────────────────────────────────────

  const progressClass = lockedMsg
    ? "progress progress-locked"
    : activeEditingPastDayIndex === viewingDayIndex
      ? "progress progress-editing"
      : allDoneToday && isViewingToday
        ? "progress progress-done"
        : "progress";

  const progressText = lockedMsg ? (
    lockedMsg
  ) : activeEditingPastDayIndex === viewingDayIndex ? (
    <>
      Editing {getWeekdayLabel(viewedDayData.date)} — {completedViewed}/3 done
      <span className="edit-note">One-time edit · changes save instantly</span>
    </>
  ) : isViewingToday ? (
    allDoneToday ? "All done today! 🎉" : `${completedToday}/3 completed today`
  ) : (
    `${getWeekdayLabel(viewedDayData.date)} — ${completedViewed}/3 (read-only)`
  );

  return (
    <div className="page">
      {demoControls}
      <div className="card">
        <div className="header">
          <h1 className="title">Daily Check-In 🤍</h1>
          <p className="subtitle">see i vibe coded this thing in like 10 minutes hAhAhA</p>
        </div>

        <div className="tab-toggle">
          <button
            className={`tab-btn${!showHistory ? " tab-btn--active" : ""}`}
            onClick={() => setShowHistory(false)}
          >
            Today
          </button>
          <button
            className={`tab-btn${showHistory ? " tab-btn--active" : ""}`}
            onClick={() => setShowHistory(true)}
          >
            History
          </button>
        </div>

        {showHistory ? (
          <div className="history-list">
            {history.length === 0 ? (
              <p className="history-empty">No completed cycles yet. Keep going!</p>
            ) : (
              history.map((entry) => (
                <div key={entry.id} className="history-card">
                  <div className="history-card-top">
                    <span className="history-date-range">
                      {formatDateRange(entry.startDate, entry.endDate)}
                    </span>
                    <span className="history-pct">{entry.completionPct}% complete</span>
                  </div>

                  <div className="history-dots">
                    {entry.perDayCompleted.map((count, di) => {
                      const colors = DOT_COLORS[count] ?? DOT_COLORS[0];
                      return (
                        <span
                          key={di}
                          className="history-dot"
                          style={{ background: colors.bg, borderColor: colors.border }}
                          title={`${WEEKDAYS[(new Date(entry.startDate).getDay() + di) % 7]}: ${count}/3`}
                        >
                          {count}/3
                        </span>
                      );
                    })}
                  </div>

                  <div className="history-habits">
                    {entry.habitNames.map((name, ni) => (
                      <span key={ni} className="history-habit-name">{name}</span>
                    ))}
                  </div>
                </div>
              ))
            )}
          </div>
        ) : (
          <>
            <ul className="habit-list">
              {habitNames.map((name, i) => {
                const checked = viewedDayData?.habits[i] ?? false;
                const isReadonly = !isViewingEditable;
                return (
                  <li
                    key={i}
                    className={[
                      "habit-item",
                      checked ? "done" : "",
                      isReadonly ? "habit-item--readonly" : "",
                    ]
                      .filter(Boolean)
                      .join(" ")}
                    onClick={() => toggleHabit(i)}
                  >
                    <span className="checkbox-wrap">
                      <input
                        type="checkbox"
                        className="checkbox"
                        checked={checked}
                        readOnly
                        tabIndex={-1}
                      />
                    </span>
                    <span className="habit-label">
                      {habitEmojis[i]} {name}
                    </span>
                  </li>
                );
              })}
            </ul>

            <div className={progressClass}>{progressText}</div>

            <div className="week-bar">
              {cycle.days.map((day, i) => {
                const state = getDayState(i);
                const completed = getCompletedForDay(i);
                const isToday = state === "today";
                const isFuture = state === "future";
                const isPast = state === "past";
                const isEditable = isPast && !day.editUsed;
                const isLocked = isPast && day.editUsed;
                const isViewing = viewingDayIndex === i;

                const classes = [
                  "week-slot",
                  `week-slot--${state}`,
                  !isFuture ? `week-slot--c${completed}` : "",
                  isEditable ? "week-slot--editable" : "",
                  isLocked ? "week-slot--locked" : "",
                  isViewing ? "week-slot--viewing" : "",
                ]
                  .filter(Boolean)
                  .join(" ");

                return (
                  <button
                    key={i}
                    className={classes}
                    onClick={() => handleSlotClick(i)}
                    disabled={isFuture}
                    aria-label={`${getWeekdayLabel(day.date)}: ${isFuture ? "future" : `${completed}/3`}`}
                  >
                    <span className="week-slot-label">{getWeekdayLabel(day.date)}</span>
                    <span className="week-slot-count">
                      {isFuture ? "—" : `${completed}/3`}
                    </span>
                    {isToday && <span className="week-slot-today-dot" />}
                    {isEditable && <span className="week-slot-edit-hint">✏️</span>}
                    {isLocked && <span className="week-slot-lock-hint">🔒</span>}
                  </button>
                );
              })}
            </div>
          </>
        )}
      </div>
    </div>
  );
}
```

---

### `src/index.css`

```css
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: Arial, Helvetica, sans-serif;
  background-color: #EADBDD;
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 24px;
  color: #3a2030;
}

.page {
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
}

.card {
  background: #ffffff;
  border-radius: 16px;
  padding: 40px 36px;
  width: 100%;
  max-width: 420px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.07);
}

.header {
  margin-bottom: 32px;
  text-align: center;
}

.title {
  font-size: 26px;
  font-weight: 700;
  color: #3a1f2a;
  letter-spacing: -0.3px;
}

.subtitle {
  margin-top: 6px;
  font-size: 14px;
  color: #9a7a82;
}

/* ── Habit list ───────────────────────────────────────── */

.habit-list {
  list-style: none;
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 28px;
}

.habit-item {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 14px 16px;
  border-radius: 10px;
  border: 1.5px solid #e8d5d8;
  cursor: pointer;
  background: #fdf8f9;
  transition: border-color 0.15s ease, background 0.15s ease;
  user-select: none;
}

.habit-item:hover {
  border-color: #c9a0a8;
  background: #fdf0f2;
}

.habit-item.done {
  background: #fdf0f2;
  border-color: #c9a0a8;
}

.checkbox-wrap {
  flex-shrink: 0;
  display: flex;
  align-items: center;
}

.checkbox {
  width: 20px;
  height: 20px;
  cursor: pointer;
  accent-color: #b07080;
}

.habit-label {
  font-size: 16px;
  color: #3a2030;
  transition: text-decoration 0.1s, color 0.1s;
  line-height: 1.4;
}

.habit-item.done .habit-label {
  text-decoration: line-through;
  color: #c0a0a8;
}

.habit-item--readonly {
  opacity: 0.65;
  pointer-events: none;
}

.habit-item--readonly .checkbox {
  cursor: not-allowed;
}

/* ── Progress statement ───────────────────────────────── */

.progress {
  text-align: center;
  font-size: 15px;
  font-weight: 600;
  color: #8a5060;
  background: #fdf0f2;
  border-radius: 8px;
  padding: 12px 16px;
  margin-bottom: 20px;
  transition: background 0.2s, color 0.2s;
}

.progress.progress-done {
  background: #f5d6db;
  color: #8a3050;
}

/* ── 7-day history bar ──────────────────────────────────────────────────────── */

.week-bar {
  display: flex;
  gap: 6px;
  justify-content: space-between;
  margin-top: 4px;
}

.week-slot {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 8px 4px 6px;
  border-radius: 8px;
  border: 1.5px solid #e3ebe3;
  background: #fafffe;
  font-size: 11px;
  position: relative;
  min-width: 0;
}

.week-slot-label {
  font-weight: 700;
  color: #5a6e5b;
  font-size: 11px;
}

.week-slot-count {
  color: #7a8f7b;
  font-size: 11px;
}

.week-slot-today-dot {
  display: block;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: #4a9e4f;
  margin-top: 2px;
}

.week-slot--future {
  opacity: 0.38;
  border-style: dashed;
  background: #f5f5f5;
}

.week-slot--future .week-slot-label { color: #aaa; }
.week-slot--future .week-slot-count { color: #bbb; }

.week-slot--today {
  border-width: 2px;
}

/* 0/3 — soft blush */
.week-slot--c0:not(.week-slot--future) { border-color: #f2c4c4; background: #fdf0f0; }
.week-slot--c0:not(.week-slot--future) .week-slot-label { color: #c0736e; }
.week-slot--c0:not(.week-slot--future) .week-slot-count { color: #d4908a; }
.week-slot--c0:not(.week-slot--future) .week-slot-today-dot { background: #d4908a; }

/* 1/3 — warm amber */
.week-slot--c1:not(.week-slot--future) { border-color: #D9A96B; background: #fdf7ee; }
.week-slot--c1:not(.week-slot--future) .week-slot-label { color: #7A5A20; }
.week-slot--c1:not(.week-slot--future) .week-slot-count { color: #9A7A40; }
.week-slot--c1:not(.week-slot--future) .week-slot-today-dot { background: #9A7A40; }

/* 2/3 — soft blue-grey */
.week-slot--c2:not(.week-slot--future) { border-color: #C2D7E1; background: #eaf3f7; }
.week-slot--c2:not(.week-slot--future) .week-slot-label { color: #3a6070; }
.week-slot--c2:not(.week-slot--future) .week-slot-count { color: #5a8090; }
.week-slot--c2:not(.week-slot--future) .week-slot-today-dot { background: #5a8090; }

/* 3/3 — deep mauve */
.week-slot--c3:not(.week-slot--future) { border-color: #56394F; background: #f2eef1; }
.week-slot--c3:not(.week-slot--future) .week-slot-label { color: #56394F; }
.week-slot--c3:not(.week-slot--future) .week-slot-count { color: #7a5570; }
.week-slot--c3:not(.week-slot--future) .week-slot-today-dot { background: #56394F; }

/* ── Past-day editing states ────────────────────────────────────────────────── */

.week-slot--editable:hover {
  filter: brightness(0.93);
  transform: translateY(-1px);
  transition: filter 0.12s, transform 0.12s;
}

.week-slot--locked { opacity: 0.7; }

.week-slot-edit-hint,
.week-slot-lock-hint {
  font-size: 9px;
  line-height: 1;
  margin-top: 1px;
}

/* ── Progress bar variants ─────────────────────────────────────────────────── */

.progress-editing { background: #eef3fe; color: #3a55a0; }
.progress-editing .edit-note {
  display: block;
  font-size: 12px;
  font-weight: 400;
  color: #6a80c0;
  margin-top: 4px;
}

.progress-locked { background: #fef3e2; color: #8a6020; }

/* ── Setup screen ──────────────────────────────────────────────────────────── */

.setup-heading {
  font-size: 15px;
  font-weight: 600;
  color: #5a3040;
  text-align: center;
  margin-bottom: 20px;
}

.setup-fields {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 16px;
}

.setup-field-row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.setup-emoji-input {
  width: 46px;
  flex-shrink: 0;
  padding: 10px 4px;
  border: 1.5px solid #e8d5d8;
  border-radius: 10px;
  font-size: 18px;
  font-family: Arial, Helvetica, sans-serif;
  color: #3a2030;
  background: #fdf8f9;
  outline: none;
  text-align: center;
  transition: border-color 0.15s;
}

.setup-emoji-input:focus { border-color: #b07080; background: #fff; }

.setup-input {
  flex: 1;
  padding: 10px 14px;
  border: 1.5px solid #e8d5d8;
  border-radius: 10px;
  font-size: 15px;
  font-family: Arial, Helvetica, sans-serif;
  color: #3a2030;
  background: #fdf8f9;
  outline: none;
  transition: border-color 0.15s;
}

.setup-input:focus { border-color: #b07080; background: #fff; }

.setup-error {
  font-size: 13px;
  color: #c0304a;
  text-align: center;
  margin-bottom: 12px;
}

.btn-primary {
  display: block;
  width: 100%;
  padding: 13px 16px;
  border: none;
  border-radius: 10px;
  background: #b07080;
  color: #fff;
  font-size: 15px;
  font-weight: 700;
  font-family: Arial, Helvetica, sans-serif;
  cursor: pointer;
  margin-bottom: 10px;
  transition: background 0.15s;
}

.btn-primary:hover { background: #9a5f6e; }

.btn-secondary {
  display: block;
  width: 100%;
  padding: 11px 16px;
  border: 1.5px solid #c9a0a8;
  border-radius: 10px;
  background: transparent;
  color: #8a5060;
  font-size: 14px;
  font-weight: 600;
  font-family: Arial, Helvetica, sans-serif;
  cursor: pointer;
  transition: background 0.15s, border-color 0.15s;
}

.btn-secondary:hover { background: #fdf0f2; border-color: #b07080; }

/* ── Demo Controls panel ───────────────────────────────────────────────────── */

.demo-controls {
  position: fixed;
  bottom: 16px;
  right: 16px;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 6px;
  font-family: Arial, Helvetica, sans-serif;
}

.demo-toggle {
  padding: 7px 12px;
  border-radius: 20px;
  border: 1.5px solid #b0a090;
  background: #f5f0eb;
  color: #6a5540;
  font-size: 12px;
  font-weight: 700;
  cursor: pointer;
  opacity: 0.85;
  transition: opacity 0.15s, background 0.15s;
  white-space: nowrap;
}

.demo-toggle:hover { opacity: 1; background: #ede6df; }

.demo-panel {
  background: #faf7f4;
  border: 1.5px solid #c9b8a8;
  border-radius: 12px;
  padding: 12px 14px;
  min-width: 200px;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
}

.demo-label {
  font-size: 11px;
  color: #9a8070;
  text-align: center;
  margin-bottom: 10px;
  font-style: italic;
}

.demo-buttons {
  display: flex;
  flex-direction: column;
  gap: 7px;
}

.demo-btn {
  width: 100%;
  padding: 8px 12px;
  border-radius: 8px;
  border: 1.5px solid #c9b8a8;
  background: #fff;
  color: #4a3828;
  font-size: 13px;
  font-weight: 600;
  font-family: Arial, Helvetica, sans-serif;
  cursor: pointer;
  transition: background 0.13s, border-color 0.13s;
  text-align: left;
}

.demo-btn:hover { background: #f0e8e0; border-color: #a89080; }
.demo-btn--danger { border-color: #d4a0a0; color: #802020; }
.demo-btn--danger:hover { background: #fdf0f0; border-color: #c07070; }

/* ── Tab toggle ────────────────────────────────────────────────────────────── */

.tab-toggle {
  display: flex;
  gap: 0;
  margin-bottom: 24px;
  border-radius: 10px;
  overflow: hidden;
  border: 1.5px solid #e8d5d8;
}

.tab-btn {
  flex: 1;
  padding: 10px 0;
  border: none;
  background: #fdf8f9;
  color: #9a7a82;
  font-size: 14px;
  font-weight: 600;
  font-family: Arial, Helvetica, sans-serif;
  cursor: pointer;
  transition: background 0.15s, color 0.15s;
}

.tab-btn + .tab-btn { border-left: 1.5px solid #e8d5d8; }
.tab-btn--active { background: #b07080; color: #fff; }
.tab-btn:not(.tab-btn--active):hover { background: #fdf0f2; color: #7a5060; }

/* ── History list ──────────────────────────────────────────────────────────── */

.history-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.history-empty {
  text-align: center;
  font-size: 14px;
  color: #9a7a82;
  padding: 24px 0;
}

.history-card {
  border: 1.5px solid #e8d5d8;
  border-radius: 10px;
  padding: 12px 14px;
  background: #fdf8f9;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.history-card-top {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  gap: 8px;
}

.history-date-range {
  font-size: 13px;
  font-weight: 700;
  color: #3a2030;
}

.history-pct {
  font-size: 12px;
  font-weight: 600;
  color: #8a5060;
  white-space: nowrap;
}

.history-dots {
  display: flex;
  gap: 5px;
  align-items: center;
}

.history-dot {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 34px;
  height: 34px;
  border-radius: 6px;
  border: 1.5px solid transparent;
  flex-shrink: 0;
  font-size: 10px;
  font-weight: 700;
  color: inherit;
}

.history-habits {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.history-habit-name {
  font-size: 11px;
  color: #6a4050;
  background: #fdf0f2;
  border: 1px solid #e8d5d8;
  border-radius: 20px;
  padding: 2px 8px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 100%;
}
```

---

## localStorage keys

| Key | Contents |
|---|---|
| `habitTrackerCycle` | Current 7-day cycle JSON (`CycleData`) |
| `habitTrackerNames` | Array of 3 habit name strings |
| `habitTrackerEmojis` | Array of 3 emoji strings |
| `habitTrackerHistory` | Array of `HistoryEntry` objects, newest first |

To clear all data: open the browser console and run:

```js
["habitTrackerCycle","habitTrackerNames","habitTrackerEmojis","habitTrackerHistory"]
  .forEach(k => localStorage.removeItem(k));
location.reload();
```

---

## Colour palette

| Role | Hex |
|---|---|
| Page background | `#EADBDD` |
| Card background | `#FFFFFF` |
| Primary text | `#3a2030` |
| Muted text | `#9a7a82` |
| Accent / buttons | `#b07080` |
| 0/3 slot | border `#f2c4c4` · bg `#fdf0f0` |
| 1/3 slot | border `#D9A96B` · bg `#fdf7ee` |
| 2/3 slot | border `#C2D7E1` · bg `#eaf3f7` |
| 3/3 slot | border `#56394F` · bg `#f2eef1` |

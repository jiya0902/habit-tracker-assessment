import { useState, useRef, useEffect, useCallback } from "react";

// ─── Utilities ────────────────────────────────────────────────────────────────

function today() {
  const d = new Date();
  d.setHours(0, 0, 0, 0);
  return d;
}

function addDays(date, n) {
  const d = new Date(date);
  d.setDate(d.getDate() + n);
  return d;
}

function startOfWeek(date) {
  const d = new Date(date);
  const day = d.getDay(); // 0=Sun
  d.setDate(d.getDate() - day);
  d.setHours(0, 0, 0, 0);
  return d;
}

function getWeekDays(refDate) {
  const start = startOfWeek(refDate);
  return Array.from({ length: 7 }, (_, i) => addDays(start, i));
}

function formatDateKey(date) {
  return date.toISOString().slice(0, 10);
}

function isToday(date) {
  return formatDateKey(date) === formatDateKey(today());
}

function isFuture(date) {
  return date > today();
}

function getWeekLabel(refDate) {
  const days = getWeekDays(refDate);
  const fmt = (d) =>
    d.toLocaleDateString("en-US", { month: "short", day: "numeric" });
  const startFmt = fmt(days[0]);
  const endFmt = fmt(days[6]);
  const year = days[6].getFullYear();
  return `${startFmt} – ${endFmt}, ${year}`;
}

function isCurrentWeek(refDate) {
  const t = today();
  return formatDateKey(startOfWeek(refDate)) === formatDateKey(startOfWeek(t));
}

function calcStreak(habitId, checkmarks) {
  let streak = 0;
  let cur = new Date(today());
  while (true) {
    const key = formatDateKey(cur);
    if (checkmarks[`${habitId}::${key}`]) {
      streak++;
      cur = addDays(cur, -1);
    } else {
      break;
    }
  }
  return streak;
}

// ─── Sample data ───────────────────────────────────────────────────────────────

function seedCheckmarks(habits) {
  const marks = {};
  const t = today();
  habits.forEach((h) => {
    for (let i = 1; i <= 14; i++) {
      const d = addDays(t, -i);
      if (Math.random() > 0.35) {
        marks[`${h.id}::${formatDateKey(d)}`] = true;
      }
    }
  });
  return marks;
}

const INITIAL_HABITS = [
  { id: "h1", name: "Morning run" },
  { id: "h2", name: "Read 30 min" },
  { id: "h3", name: "Drink 2L water" },
  { id: "h4", name: "Meditate" },
];

// ─── Sub-components ────────────────────────────────────────────────────────────

function CheckCell({ checked, onToggle, disabled, todayCell }) {
  const [pop, setPop] = useState(false);

  function handleClick() {
    if (disabled) return;
    if (!checked) {
      setPop(true);
      setTimeout(() => setPop(false), 300);
    }
    onToggle();
  }

  return (
    <button
      onClick={handleClick}
      disabled={disabled}
      aria-checked={checked}
      role="checkbox"
      aria-label={checked ? "Mark incomplete" : "Mark complete"}
      style={{
        width: 34,
        height: 34,
        borderRadius: 8,
        border: disabled
          ? "1.5px solid var(--border-faint)"
          : checked
          ? "1.5px solid var(--moss)"
          : todayCell
          ? "1.5px solid var(--moss-light)"
          : "1.5px solid var(--border)",
        background: disabled
          ? "var(--surface-dim)"
          : checked
          ? "var(--moss)"
          : todayCell
          ? "var(--moss-tint)"
          : "var(--surface)",
        cursor: disabled ? "not-allowed" : "pointer",
        opacity: disabled ? 0.35 : 1,
        display: "flex",
        alignItems: "center",
        justifyContent: "center",
        transition: "all 0.15s ease",
        transform: pop ? "scale(1.2)" : "scale(1)",
        outline: "none",
        flexShrink: 0,
      }}
    >
      {checked && (
        <svg
          width="15"
          height="15"
          viewBox="0 0 16 16"
          fill="none"
          stroke="white"
          strokeWidth="2.5"
          strokeLinecap="round"
          strokeLinejoin="round"
          aria-hidden="true"
        >
          <polyline points="3,8 6.5,11.5 13,4.5" />
        </svg>
      )}
      {!checked && todayCell && !disabled && (
        <div
          aria-hidden="true"
          style={{
            width: 6,
            height: 6,
            borderRadius: "50%",
            background: "var(--moss)",
            opacity: 0.5,
          }}
        />
      )}
    </button>
  );
}

function StreakBadge({ streak }) {
  if (streak < 2) return null;
  return (
    <span
      title={`${streak}-day streak`}
      style={{
        display: "inline-flex",
        alignItems: "center",
        gap: 3,
        fontSize: 11,
        fontWeight: 600,
        color: streak >= 7 ? "var(--amber)" : "var(--moss-dark)",
        background: streak >= 7 ? "var(--amber-tint)" : "var(--moss-tint)",
        padding: "2px 7px",
        borderRadius: 99,
        letterSpacing: "0.01em",
        flexShrink: 0,
        fontFamily: "var(--mono)",
      }}
    >
      {streak >= 7 ? "🔥" : "✦"} {streak}
    </span>
  );
}

function HabitRow({ habit, weekDays, checkmarks, onToggle, onRename, onDelete, index }) {
  const [editing, setEditing] = useState(false);
  const [editValue, setEditValue] = useState(habit.name);
  const [showMenu, setShowMenu] = useState(false);
  const [deleting, setDeleting] = useState(false);
  const inputRef = useRef(null);
  const menuRef = useRef(null);
  const streak = calcStreak(habit.id, checkmarks);

  useEffect(() => {
    if (editing && inputRef.current) {
      inputRef.current.focus();
      inputRef.current.select();
    }
  }, [editing]);

  useEffect(() => {
    if (!showMenu) return;
    function handler(e) {
      if (menuRef.current && !menuRef.current.contains(e.target)) setShowMenu(false);
    }
    document.addEventListener("mousedown", handler);
    return () => document.removeEventListener("mousedown", handler);
  }, [showMenu]);

  function commitRename() {
    const v = editValue.trim();
    if (v && v !== habit.name) onRename(habit.id, v);
    else setEditValue(habit.name);
    setEditing(false);
  }

  function handleKeyDown(e) {
    if (e.key === "Enter") commitRename();
    if (e.key === "Escape") { setEditValue(habit.name); setEditing(false); }
  }

  function handleDelete() {
    setDeleting(true);
    setShowMenu(false);
    setTimeout(() => onDelete(habit.id), 260);
  }

  return (
    <div
      role="row"
      style={{
        display: "flex",
        alignItems: "center",
        gap: 8,
        padding: "9px 0",
        borderBottom: "1px solid var(--divider)",
        opacity: deleting ? 0 : 1,
        transform: deleting ? "translateX(-12px) scale(0.97)" : "translateX(0) scale(1)",
        transition: "opacity 0.26s, transform 0.26s",
        animationDelay: `${index * 40}ms`,
      }}
    >
      {/* Color dot */}
      <div
        aria-hidden="true"
        style={{
          width: 8,
          height: 8,
          borderRadius: "50%",
          background: "var(--moss)",
          opacity: 0.55,
          flexShrink: 0,
        }}
      />

      {/* Name */}
      <div style={{ flex: 1, minWidth: 0, display: "flex", alignItems: "center", gap: 6 }}>
        {editing ? (
          <input
            ref={inputRef}
            value={editValue}
            onChange={(e) => setEditValue(e.target.value)}
            onBlur={commitRename}
            onKeyDown={handleKeyDown}
            aria-label="Edit habit name"
            style={{
              flex: 1,
              minWidth: 0,
              fontSize: 14,
              fontWeight: 500,
              color: "var(--text-primary)",
              background: "var(--surface-dim)",
              border: "1.5px solid var(--moss-light)",
              borderRadius: 6,
              padding: "2px 8px",
              outline: "none",
              fontFamily: "inherit",
            }}
          />
        ) : (
          <button
            onDoubleClick={() => setEditing(true)}
            title={`${habit.name} — double-click to rename`}
            style={{
              flex: 1,
              minWidth: 0,
              textAlign: "left",
              fontSize: 14,
              fontWeight: 500,
              color: "var(--text-primary)",
              background: "none",
              border: "none",
              cursor: "text",
              padding: 0,
              overflow: "hidden",
              textOverflow: "ellipsis",
              whiteSpace: "nowrap",
              fontFamily: "inherit",
            }}
          >
            {habit.name}
          </button>
        )}
        <StreakBadge streak={streak} />
      </div>

      {/* Check cells */}
      <div
        role="group"
        aria-label={`${habit.name} completions`}
        style={{ display: "flex", alignItems: "center", gap: 4, flexShrink: 0 }}
      >
        {weekDays.map((day) => {
          const key = formatDateKey(day);
          return (
            <CheckCell
              key={key}
              checked={!!checkmarks[`${habit.id}::${key}`]}
              onToggle={() => onToggle(habit.id, key)}
              disabled={isFuture(day)}
              todayCell={isToday(day)}
            />
          );
        })}
      </div>

      {/* Actions menu */}
      <div ref={menuRef} style={{ position: "relative", flexShrink: 0 }}>
        <button
          onClick={() => setShowMenu((v) => !v)}
          aria-label={`Options for ${habit.name}`}
          aria-haspopup="true"
          aria-expanded={showMenu}
          style={{
            width: 28,
            height: 28,
            borderRadius: 6,
            display: "flex",
            alignItems: "center",
            justifyContent: "center",
            background: showMenu ? "var(--surface-dim)" : "none",
            border: "none",
            cursor: "pointer",
            color: "var(--text-secondary)",
            opacity: showMenu ? 1 : 0.4,
            transition: "opacity 0.15s, background 0.15s",
            outline: "none",
          }}
          onMouseEnter={(e) => (e.currentTarget.style.opacity = "1")}
          onMouseLeave={(e) => { if (!showMenu) e.currentTarget.style.opacity = "0.4"; }}
        >
          <svg viewBox="0 0 16 16" width="14" height="14" fill="currentColor" aria-hidden="true">
            <circle cx="8" cy="3" r="1.2" />
            <circle cx="8" cy="8" r="1.2" />
            <circle cx="8" cy="13" r="1.2" />
          </svg>
        </button>

        {showMenu && (
          <div
            role="menu"
            style={{
              position: "absolute",
              right: 0,
              top: 32,
              zIndex: 100,
              width: 140,
              background: "var(--surface-elevated)",
              border: "1px solid var(--border)",
              borderRadius: 10,
              boxShadow: "0 4px 16px rgba(0,0,0,0.10)",
              overflow: "hidden",
            }}
          >
            <button
              role="menuitem"
              onClick={() => { setEditing(true); setShowMenu(false); }}
              style={menuItemStyle}
            >
              <svg viewBox="0 0 16 16" width="13" height="13" fill="none" stroke="currentColor" strokeWidth="1.5" aria-hidden="true">
                <path d="M11.5 2.5l2 2L5 13H3v-2L11.5 2.5z" strokeLinejoin="round" />
              </svg>
              Rename
            </button>
            <button
              role="menuitem"
              onClick={handleDelete}
              style={{ ...menuItemStyle, color: "var(--danger)" }}
            >
              <svg viewBox="0 0 16 16" width="13" height="13" fill="none" stroke="currentColor" strokeWidth="1.5" aria-hidden="true">
                <path d="M3 4h10M6 4V2h4v2M5 4v9h6V4" strokeLinejoin="round" strokeLinecap="round" />
              </svg>
              Delete
            </button>
          </div>
        )}
      </div>
    </div>
  );
}

const menuItemStyle = {
  width: "100%",
  display: "flex",
  alignItems: "center",
  gap: 8,
  padding: "9px 12px",
  fontSize: 13,
  fontWeight: 500,
  color: "var(--text-primary)",
  background: "none",
  border: "none",
  cursor: "pointer",
  textAlign: "left",
  fontFamily: "inherit",
  transition: "background 0.12s",
};

function WeekHeader({ weekDays }) {
  return (
    <div
      role="row"
      style={{ display: "flex", alignItems: "center", gap: 8, paddingBottom: 4 }}
    >
      <div style={{ flex: 1, minWidth: 0 }} aria-hidden="true" />
      <div style={{ display: "flex", alignItems: "center", gap: 4, flexShrink: 0 }}>
        {weekDays.map((day) => {
          const t = isToday(day);
          const f = isFuture(day);
          return (
            <div
              key={day.toISOString()}
              style={{ width: 34, display: "flex", flexDirection: "column", alignItems: "center", gap: 2 }}
              aria-label={day.toLocaleDateString("en-US", { weekday: "long", month: "short", day: "numeric" })}
            >
              <span
                style={{
                  fontSize: 9,
                  fontFamily: "var(--mono)",
                  fontWeight: 600,
                  letterSpacing: "0.08em",
                  textTransform: "uppercase",
                  color: t ? "var(--moss)" : f ? "var(--text-faint)" : "var(--text-secondary)",
                }}
              >
                {day.toLocaleDateString("en-US", { weekday: "short" }).slice(0, 2)}
              </span>
              <span
                style={{
                  width: 22,
                  height: 22,
                  display: "flex",
                  alignItems: "center",
                  justifyContent: "center",
                  borderRadius: "50%",
                  background: t ? "var(--moss)" : "transparent",
                  fontSize: 11,
                  fontFamily: "var(--mono)",
                  fontWeight: 600,
                  color: t ? "white" : f ? "var(--text-faint)" : "var(--text-secondary)",
                }}
              >
                {day.getDate()}
              </span>
            </div>
          );
        })}
      </div>
      <div style={{ width: 28, flexShrink: 0 }} aria-hidden="true" />
    </div>
  );
}

function WeekNav({ refDate, onPrev, onNext, onGoToday, canGoNext }) {
  const label = getWeekLabel(refDate);
  const onCurrent = isCurrentWeek(refDate);

  return (
    <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", gap: 8 }}>
      <div style={{ display: "flex", alignItems: "center", gap: 4 }}>
        <NavBtn onClick={onPrev} label="Previous week">
          <svg viewBox="0 0 16 16" width="14" height="14" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" aria-hidden="true">
            <polyline points="10 4 6 8 10 12" />
          </svg>
        </NavBtn>
        <span
          style={{
            fontSize: 13,
            fontWeight: 600,
            color: "var(--text-primary)",
            fontFamily: "var(--mono)",
            minWidth: 190,
            textAlign: "center",
            letterSpacing: "0.01em",
          }}
        >
          {label}
        </span>
        <NavBtn onClick={onNext} label="Next week" disabled={!canGoNext}>
          <svg viewBox="0 0 16 16" width="14" height="14" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" aria-hidden="true">
            <polyline points="6 4 10 8 6 12" />
          </svg>
        </NavBtn>
      </div>
      {!onCurrent && (
        <button
          onClick={onGoToday}
          style={{
            display: "flex",
            alignItems: "center",
            gap: 5,
            fontSize: 12,
            fontWeight: 600,
            color: "var(--moss-dark)",
            background: "var(--moss-tint)",
            border: "none",
            borderRadius: 8,
            padding: "5px 10px",
            cursor: "pointer",
            fontFamily: "inherit",
            transition: "background 0.15s",
          }}
        >
          <svg viewBox="0 0 16 16" width="12" height="12" fill="none" stroke="currentColor" strokeWidth="1.8" strokeLinecap="round" aria-hidden="true">
            <rect x="2" y="3" width="12" height="11" rx="2" />
            <line x1="2" y1="7" x2="14" y2="7" />
            <line x1="5.5" y1="1" x2="5.5" y2="4" />
            <line x1="10.5" y1="1" x2="10.5" y2="4" />
          </svg>
          Today
        </button>
      )}
    </div>
  );
}

function NavBtn({ onClick, label, disabled, children }) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      aria-label={label}
      style={{
        width: 30,
        height: 30,
        borderRadius: 7,
        display: "flex",
        alignItems: "center",
        justifyContent: "center",
        background: "none",
        border: "none",
        cursor: disabled ? "not-allowed" : "pointer",
        color: disabled ? "var(--text-faint)" : "var(--text-secondary)",
        transition: "background 0.15s, color 0.15s",
        outline: "none",
      }}
    >
      {children}
    </button>
  );
}

function EmptyState({ onAddSample }) {
  const suggestions = ["Exercise", "Read 30 min", "Drink water", "Meditate", "Journaling"];
  return (
    <div
      style={{
        display: "flex",
        flexDirection: "column",
        alignItems: "center",
        justifyContent: "center",
        padding: "48px 24px",
        textAlign: "center",
      }}
    >
      <div
        style={{
          width: 72,
          height: 72,
          borderRadius: 20,
          background: "var(--moss-tint)",
          display: "flex",
          alignItems: "center",
          justifyContent: "center",
          marginBottom: 20,
          fontSize: 32,
        }}
        aria-hidden="true"
      >
        📋
      </div>
      <h2 style={{ fontSize: 20, fontWeight: 700, color: "var(--text-primary)", margin: "0 0 8px", fontFamily: "var(--display)" }}>
        Start building habits
      </h2>
      <p style={{ fontSize: 13, color: "var(--text-secondary)", maxWidth: 260, lineHeight: 1.6, margin: "0 0 24px" }}>
        Track daily habits with a weekly grid. Add your first habit or pick a suggestion to get started.
      </p>
      <div style={{ display: "flex", flexWrap: "wrap", gap: 8, justifyContent: "center", maxWidth: 320 }}>
        {suggestions.map((s) => (
          <button
            key={s}
            onClick={() => onAddSample(s)}
            style={{
              padding: "6px 14px",
              borderRadius: 99,
              background: "var(--surface-dim)",
              border: "1px solid var(--border)",
              fontSize: 12,
              fontWeight: 500,
              color: "var(--text-secondary)",
              cursor: "pointer",
              fontFamily: "inherit",
              transition: "all 0.15s",
            }}
          >
            + {s}
          </button>
        ))}
      </div>
    </div>
  );
}

function AddHabitInput({ onAdd }) {
  const [value, setValue] = useState("");
  const inputRef = useRef(null);

  function submit() {
    const v = value.trim();
    if (!v) return;
    onAdd(v);
    setValue("");
  }

  function handleKey(e) {
    if (e.key === "Enter") submit();
    if (e.key === "Escape") setValue("");
  }

  return (
    <div style={{ display: "flex", gap: 8, paddingTop: 14 }}>
      <input
        ref={inputRef}
        value={value}
        onChange={(e) => setValue(e.target.value)}
        onKeyDown={handleKey}
        placeholder="Add a new habit…"
        aria-label="New habit name"
        style={{
          flex: 1,
          height: 38,
          borderRadius: 9,
          border: "1.5px solid var(--border)",
          background: "var(--surface)",
          fontSize: 13,
          fontWeight: 500,
          color: "var(--text-primary)",
          padding: "0 12px",
          outline: "none",
          fontFamily: "inherit",
          transition: "border-color 0.15s",
        }}
        onFocus={(e) => (e.target.style.borderColor = "var(--moss)")}
        onBlur={(e) => (e.target.style.borderColor = "var(--border)")}
      />
      <button
        onClick={submit}
        disabled={!value.trim()}
        style={{
          height: 38,
          padding: "0 16px",
          borderRadius: 9,
          background: value.trim() ? "var(--moss)" : "var(--surface-dim)",
          border: "none",
          color: value.trim() ? "white" : "var(--text-faint)",
          fontSize: 13,
          fontWeight: 600,
          cursor: value.trim() ? "pointer" : "not-allowed",
          fontFamily: "inherit",
          transition: "all 0.15s",
          flexShrink: 0,
        }}
      >
        Add
      </button>
    </div>
  );
}

function StatsBar({ habits, weekDays, checkmarks }) {
  const total = habits.length * weekDays.filter((d) => !isFuture(d)).length;
  const done = habits.reduce((acc, h) => {
    return (
      acc +
      weekDays.filter((d) => !isFuture(d) && checkmarks[`${h.id}::${formatDateKey(d)}`]).length
    );
  }, 0);
  const pct = total === 0 ? 0 : Math.round((done / total) * 100);

  return (
    <div
      style={{
        display: "flex",
        alignItems: "center",
        gap: 12,
        padding: "10px 0 14px",
        borderBottom: "1px solid var(--divider)",
        marginBottom: 4,
      }}
    >
      <div style={{ flex: 1, position: "relative", height: 5, borderRadius: 99, background: "var(--surface-dim)", overflow: "hidden" }}>
        <div
          style={{
            position: "absolute",
            left: 0,
            top: 0,
            height: "100%",
            width: `${pct}%`,
            background: pct === 100 ? "var(--amber)" : "var(--moss)",
            borderRadius: 99,
            transition: "width 0.4s ease",
          }}
        />
      </div>
      <span style={{ fontSize: 12, fontWeight: 600, fontFamily: "var(--mono)", color: "var(--moss-dark)", flexShrink: 0 }}>
        {done}/{total} · {pct}%
      </span>
    </div>
  );
}

// ─── Main App ──────────────────────────────────────────────────────────────────

export default function HabitTracker() {
  const [habits, setHabits] = useState(INITIAL_HABITS);
  const [checkmarks, setCheckmarks] = useState(() => seedCheckmarks(INITIAL_HABITS));
  const [refDate, setRefDate] = useState(() => today());

  const weekDays = getWeekDays(refDate);
  const canGoNext = !isCurrentWeek(refDate);

  function toggle(habitId, dateKey) {
    const k = `${habitId}::${dateKey}`;
    setCheckmarks((prev) => ({ ...prev, [k]: !prev[k] }));
  }

  function addHabit(name) {
    const id = `h${Date.now()}`;
    setHabits((prev) => [...prev, { id, name }]);
  }

  function renameHabit(id, name) {
    setHabits((prev) => prev.map((h) => (h.id === id ? { ...h, name } : h)));
  }

  function deleteHabit(id) {
    setHabits((prev) => prev.filter((h) => h.id !== id));
  }

  function addSample(name) {
    addHabit(name);
  }

  return (
    <>
      <style>{`
        :root {
          --moss: #5a8a4a;
          --moss-dark: #3d6134;
          --moss-light: #8ab87a;
          --moss-tint: #edf4e9;
          --amber: #c8891a;
          --amber-tint: #fdf3e3;
          --danger: #c0392b;
          --surface: #ffffff;
          --surface-dim: #f3f4f2;
          --surface-elevated: #ffffff;
          --border: rgba(0,0,0,0.12);
          --border-faint: rgba(0,0,0,0.07);
          --divider: rgba(0,0,0,0.07);
          --text-primary: #1a2015;
          --text-secondary: #546050;
          --text-faint: #a0a89a;
          --mono: 'JetBrains Mono', 'Fira Mono', monospace;
          --display: Georgia, 'Times New Roman', serif;
        }
        @media (prefers-color-scheme: dark) {
          :root {
            --moss: #7ab86a;
            --moss-dark: #9dd08d;
            --moss-light: #4d7a3d;
            --moss-tint: #1c2a18;
            --amber: #e8a840;
            --amber-tint: #2a2010;
            --danger: #e05a4a;
            --surface: #1e231c;
            --surface-dim: #262c23;
            --surface-elevated: #2a3028;
            --border: rgba(255,255,255,0.12);
            --border-faint: rgba(255,255,255,0.06);
            --divider: rgba(255,255,255,0.07);
            --text-primary: #e8ede4;
            --text-secondary: #9db890;
            --text-faint: #5a7050;
          }
        }
        * { box-sizing: border-box; }
        button:focus-visible { outline: 2px solid var(--moss); outline-offset: 2px; }
        input:focus-visible { outline: none; }
      `}</style>

      <div
        style={{
          maxWidth: 620,
          margin: "0 auto",
          padding: "28px 20px 40px",
          fontFamily: "system-ui, -apple-system, sans-serif",
          color: "var(--text-primary)",
        }}
      >
        {/* Header */}
        <div style={{ marginBottom: 24 }}>
          <div style={{ display: "flex", alignItems: "center", gap: 10, marginBottom: 4 }}>
            <span aria-hidden="true" style={{ fontSize: 22 }}>🌿</span>
            <h1
              style={{
                fontSize: 22,
                fontWeight: 700,
                margin: 0,
                fontFamily: "var(--display)",
                color: "var(--text-primary)",
                letterSpacing: "-0.02em",
              }}
            >
              Habits
            </h1>
          </div>
          <p style={{ fontSize: 13, color: "var(--text-secondary)", margin: 0, paddingLeft: 32 }}>
            {new Date().toLocaleDateString("en-US", { weekday: "long", month: "long", day: "numeric" })}
          </p>
        </div>

        {/* Card */}
        <div
          style={{
            background: "var(--surface)",
            border: "1px solid var(--border)",
            borderRadius: 16,
            padding: "20px 20px 16px",
          }}
        >
          {/* Week navigation */}
          <WeekNav
            refDate={refDate}
            onPrev={() => setRefDate((d) => addDays(d, -7))}
            onNext={() => setRefDate((d) => addDays(d, 7))}
            onGoToday={() => setRefDate(today())}
            canGoNext={canGoNext}
          />

          <div style={{ height: 14 }} />

          {/* Stats bar */}
          {habits.length > 0 && (
            <StatsBar habits={habits} weekDays={weekDays} checkmarks={checkmarks} />
          )}

          {/* Week header */}
          {habits.length > 0 && <WeekHeader weekDays={weekDays} />}

          {/* Habit rows */}
          {habits.length === 0 ? (
            <EmptyState onAddSample={addSample} />
          ) : (
            <div role="table" aria-label="Habit tracker">
              {habits.map((h, i) => (
                <HabitRow
                  key={h.id}
                  habit={h}
                  weekDays={weekDays}
                  checkmarks={checkmarks}
                  onToggle={toggle}
                  onRename={renameHabit}
                  onDelete={deleteHabit}
                  index={i}
                />
              ))}
            </div>
          )}

          {/* Add input */}
          <AddHabitInput onAdd={addHabit} />
        </div>
      </div>
    </>
  );
}

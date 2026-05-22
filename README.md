# habit-tracker-assessment
import { useState } from 'react';

export function CheckCell({ checked, onToggle, disabled, isToday }) {
  const [animating, setAnimating] = useState(false);

  function handleClick() {
    if (disabled) return;
    if (!checked) {
      setAnimating(true);
      setTimeout(() => setAnimating(false), 300);
    }
    onToggle();
  }

  function handleKeyDown(e) {
    if (e.key === ' ' || e.key === 'Enter') {
      e.preventDefault();
      handleClick();
    }
  }

  return (
    <div className="flex items-center justify-center">
      <button
        onClick={handleClick}
        onKeyDown={handleKeyDown}
        disabled={disabled}
        aria-label={checked ? 'Mark incomplete' : 'Mark complete'}
        aria-checked={checked}
        role="checkbox"
        className={[
          'relative w-8 h-8 rounded-lg border-2 transition-all duration-200 flex items-center justify-center',
          'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-moss focus-visible:ring-offset-1',
          disabled
            ? 'border-cream-200 bg-cream-50 cursor-not-allowed opacity-40'
            : checked
            ? 'border-moss bg-moss shadow-check cursor-pointer hover:bg-moss-dark hover:border-moss-dark'
            : isToday
            ? 'border-moss/40 bg-cream-50 cursor-pointer hover:border-moss hover:bg-moss/10'
            : 'border-ink/10 bg-cream-50 cursor-pointer hover:border-moss/50 hover:bg-cream-100',
        ].join(' ')}
      >
        {checked && (
          <svg
            className={`w-4 h-4 text-white ${animating ? 'animate-check-pop' : ''}`}
            viewBox="0 0 16 16"
            fill="none"
            stroke="currentColor"
            strokeWidth="2.5"
            strokeLinecap="round"
            strokeLinejoin="round"
            aria-hidden="true"
          >
            <polyline points="3,8 6.5,11.5 13,4.5" />
          </svg>
        )}
        {!checked && isToday && !disabled && (
          <div className="w-1.5 h-1.5 rounded-full bg-moss/40" aria-hidden="true" />
        )}
      </button>
    </div>
  );
}
import { useState, useRef, useEffect } from 'react';
import { CheckCell } from './CheckCell';
import { StreakBadge } from './StreakBadge';
import { formatDateKey, isFutureDay, calculateStreak } from '../utils/dateUtils';
import { format, isToday } from 'date-fns';

export function HabitRow({ habit, weekDays, checkmarks, isChecked, onToggle, onRename, onDelete, index }) {
  const [editing, setEditing] = useState(false);
  const [editValue, setEditValue] = useState(habit.name);
  const [showMenu, setShowMenu] = useState(false);
  const [deleting, setDeleting] = useState(false);
  const inputRef = useRef(null);
  const menuRef = useRef(null);
  const streak = calculateStreak(habit.id, checkmarks);

  useEffect(() => {
    if (editing && inputRef.current) {
      inputRef.current.focus();
      inputRef.current.select();
    }
  }, [editing]);

  // Close menu on outside click
  useEffect(() => {
    if (!showMenu) return;
    function handleClick(e) {
      if (menuRef.current && !menuRef.current.contains(e.target)) {
        setShowMenu(false);
      }
    }
    document.addEventListener('mousedown', handleClick);
    return () => document.removeEventListener('mousedown', handleClick);
  }, [showMenu]);

  function commitRename() {
    const trimmed = editValue.trim();
    if (trimmed && trimmed !== habit.name) {
      onRename(habit.id, trimmed);
    } else {
      setEditValue(habit.name);
    }
    setEditing(false);
  }

  function handleKeyDown(e) {
    if (e.key === 'Enter') commitRename();
    if (e.key === 'Escape') {
      setEditValue(habit.name);
      setEditing(false);
    }
  }

  function handleDelete() {
    setDeleting(true);
    setShowMenu(false);
    setTimeout(() => onDelete(habit.id), 280);
  }

  return (
    <div
      className={[
        'group flex items-center gap-2 transition-all duration-300',
        deleting ? 'opacity-0 -translate-x-4 scale-95' : 'opacity-100 translate-x-0',
        'animate-slide-in',
      ].join(' ')}
      style={{ animationDelay: `${index * 40}ms`, animationFillMode: 'both' }}
      role="row"
    >
      {/* Habit name + controls */}
      <div className="flex items-center gap-2 min-w-0 flex-1" style={{ minWidth: 0 }}>
        {/* Color dot */}
        <div
          className="w-2 h-2 rounded-full flex-shrink-0 bg-moss opacity-60 group-hover:opacity-100 transition-opacity"
          aria-hidden="true"
        />

        {editing ? (
          <input
            ref={inputRef}
            value={editValue}
            onChange={(e) => setEditValue(e.target.value)}
            onBlur={commitRename}
            onKeyDown={handleKeyDown}
            className="flex-1 min-w-0 text-sm font-body font-medium text-ink bg-cream-100 border border-moss/40 rounded-md px-2 py-0.5 focus:outline-none focus:border-moss"
            aria-label="Edit habit name"
          />
        ) : (
          <button
            onDoubleClick={() => setEditing(true)}
            className="flex-1 min-w-0 text-left text-sm font-medium text-ink truncate hover:text-moss-dark transition-colors focus-visible:text-moss-dark"
            title={`${habit.name} — double-click to rename`}
            aria-label={`${habit.name}, double-click to rename`}
          >
            {habit.name}
          </button>
        )}

        <StreakBadge streak={streak} />
      </div>

      {/* Checkboxes */}
      <div className="flex items-center gap-1 flex-shrink-0" role="group" aria-label={`${habit.name} completions for the week`}>
        {weekDays.map((day) => {
          const key = formatDateKey(day);
          const future = isFutureDay(day);
          const today = isToday(day);
          return (
            <CheckCell
              key={key}
              checked={isChecked(habit.id, key)}
              onToggle={() => onToggle(habit.id, key)}
              disabled={future}
              isToday={today}
              aria-label={`${habit.name} on ${format(day, 'EEEE, MMM d')}`}
            />
          );
        })}
      </div>

      {/* Actions menu */}
      <div className="relative flex-shrink-0" ref={menuRef}>
        <button
          onClick={() => setShowMenu((v) => !v)}
          aria-label={`Options for ${habit.name}`}
          aria-haspopup="true"
          aria-expanded={showMenu}
          className={[
            'w-7 h-7 rounded-md flex items-center justify-center text-ink-faint transition-all duration-150',
            'hover:bg-cream-200 hover:text-ink focus-visible:bg-cream-200',
            'opacity-0 group-hover:opacity-100 focus-visible:opacity-100',
            showMenu ? 'opacity-100 bg-cream-200 text-ink' : '',
          ].join(' ')}
        >
          <svg viewBox="0 0 16 16" className="w-4 h-4" fill="currentColor" aria-hidden="true">
            <circle cx="8" cy="3" r="1.2" />
            <circle cx="8" cy="8" r="1.2" />
            <circle cx="8" cy="13" r="1.2" />
          </svg>
        </button>

        {showMenu && (
          <div
            className="absolute right-0 top-8 z-10 w-36 bg-white rounded-xl shadow-card-hover border border-cream-200 overflow-hidden animate-slide-in"
            role="menu"
          >
            <button
              onClick={() => { setEditing(true); setShowMenu(false); }}
              className="w-full flex items-center gap-2 px-3 py-2 text-sm text-ink hover:bg-cream-100 transition-colors text-left"
              role="menuitem"
            >
              <svg viewBox="0 0 16 16" className="w-3.5 h-3.5 text-ink-muted" fill="none" stroke="currentColor" strokeWidth="1.5" aria-hidden="true">
                <path d="M11.5 2.5l2 2L5 13H3v-2L11.5 2.5z" strokeLinejoin="round" />
              </svg>
              Rename
            </button>
            <button
              onClick={handleDelete}
              className="w-full flex items-center gap-2 px-3 py-2 text-sm text-rose hover:bg-rose-soft transition-colors text-left"
              role="menuitem"
            >
              <svg viewBox="0 0 16 16" className="w-3.5 h-3.5" fill="none" stroke="currentColor" strokeWidth="1.5" aria-hidden="true">
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
import { format, isToday } from 'date-fns';
import { isFutureDay } from '../utils/dateUtils';

export function WeekHeader({ weekDays }) {
  return (
    <div className="flex items-center gap-2" role="row">
      {/* Spacer to align with habit name column */}
      <div className="flex-1 min-w-0" aria-hidden="true" />

      {/* Day columns */}
      <div className="flex items-center gap-1 flex-shrink-0">
        {weekDays.map((day) => {
          const today = isToday(day);
          const future = isFutureDay(day);
          return (
            <div
              key={day.toISOString()}
              className="w-8 flex flex-col items-center gap-0.5"
              aria-label={format(day, 'EEEE, MMM d')}
            >
              <span
                className={[
                  'text-[10px] font-mono uppercase tracking-widest transition-colors',
                  today
                    ? 'text-moss font-semibold'
                    : future
                    ? 'text-ink-faint/50'
                    : 'text-ink-faint',
                ].join(' ')}
              >
                {format(day, 'EEE').slice(0, 2)}
              </span>
              <span
                className={[
                  'w-6 h-6 flex items-center justify-center rounded-full text-xs font-mono font-medium transition-all',
                  today
                    ? 'bg-moss text-white'
                    : future
                    ? 'text-ink-faint/50'
                    : 'text-ink-soft',
                ].join(' ')}
              >
                {format(day, 'd')}
              </span>
            </div>
          );
        })}
      </div>

      {/* Spacer to align with actions column */}
      <div className="w-7 flex-shrink-0" aria-hidden="true" />
    </div>
  );
}
import { getWeekLabel } from '../utils/dateUtils';

export function WeekNavigation({ referenceDate, onPrev, onNext, onToday, onCurrentWeek, canGoNext }) {
  const label = getWeekLabel(referenceDate);

  return (
    <div className="flex items-center justify-between gap-2 flex-wrap">
      <div className="flex items-center gap-1">
        <button
          onClick={onPrev}
          aria-label="Previous week"
          className="w-8 h-8 rounded-lg flex items-center justify-center text-ink-muted hover:bg-cream-200 hover:text-ink transition-all duration-150 focus-visible:ring-2 focus-visible:ring-moss focus-visible:ring-offset-1 active:scale-95"
        >
          <svg viewBox="0 0 16 16" className="w-4 h-4" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" aria-hidden="true">
            <polyline points="10 4 6 8 10 12" />
          </svg>
        </button>

        <span className="text-sm font-medium text-ink px-1 min-w-[180px] text-center tabular-nums">
          {label}
        </span>

        <button
          onClick={onNext}
          disabled={!canGoNext}
          aria-label="Next week"
          className={[
            'w-8 h-8 rounded-lg flex items-center justify-center transition-all duration-150',
            'focus-visible:ring-2 focus-visible:ring-moss focus-visible:ring-offset-1',
            canGoNext
              ? 'text-ink-muted hover:bg-cream-200 hover:text-ink active:scale-95'
              : 'text-ink-faint/30 cursor-not-allowed',
          ].join(' ')}
        >
          <svg viewBox="0 0 16 16" className="w-4 h-4" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round" aria-hidden="true">
            <polyline points="6 4 10 8 6 12" />
          </svg>
        </button>
      </div>

      {!onCurrentWeek && (
        <button
          onClick={onToday}
          className="flex items-center gap-1.5 px-3 py-1.5 rounded-lg bg-moss/10 text-moss-dark text-sm font-medium hover:bg-moss/20 transition-all duration-150 focus-visible:ring-2 focus-visible:ring-moss focus-visible:ring-offset-1 animate-slide-in active:scale-95"
          aria-label="Go to current week"
        >
          <svg viewBox="0 0 16 16" className="w-3.5 h-3.5" fill="none" stroke="currentColor" strokeWidth="1.8" strokeLinecap="round" aria-hidden="true">
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
export function EmptyState({ onAddSample }) {
  const suggestions = ['Exercise', 'Read 30 min', 'Drink water', 'Meditate'];

  return (
    <div className="flex flex-col items-center justify-center py-16 px-6 animate-fade-up text-center">
      {/* Illustration */}
      <div className="relative mb-6">
        <div className="w-20 h-20 rounded-2xl bg-cream-200 flex items-center justify-center">
          <svg viewBox="0 0 48 48" className="w-10 h-10" fill="none" aria-hidden="true">
            <rect x="6" y="10" width="36" height="30" rx="4" fill="#c8d5b9" />
            <rect x="6" y="10" width="36" height="8" rx="4" fill="#7fa564" />
            <rect x="14" y="24" width="6" height="6" rx="1.5" fill="#7fa564" opacity="0.4" />
            <rect x="24" y="24" width="6" height="6" rx="1.5" fill="#7fa564" opacity="0.4" />
            <rect x="14" y="32" width="6" height="4" rx="1.5" fill="#7fa564" opacity="0.2" />
            <line x1="14" y1="16" x2="14" y2="10" stroke="white" strokeWidth="2" strokeLinecap="round" />
            <line x1="34" y1="16" x2="34" y2="10" stroke="white" strokeWidth="2" strokeLinecap="round" />
          </svg>
        </div>
        <div className="absolute -top-1 -right-1 w-6 h-6 rounded-full bg-amber-warm flex items-center justify-center text-xs">
          ✦
        </div>
      </div>

      <h2 className="font-display text-2xl text-ink mb-2">
        Start building habits
      </h2>
      <p className="text-ink-muted text-sm max-w-xs mb-8 leading-relaxed">
        Track daily habits with a weekly grid. Add your first habit and watch your streaks grow.
      </p>

      {/* Suggestion chips */}
      <div className="flex flex-wrap gap-2 justify-center max-w-xs">
        {suggestions.map((s) => (
          <button
            key={s}
            onClick={() => onAddSample(s)}
            className="px-3 py-1.5 rounded-full bg-cream-100 border border-cream-200 text-sm text-ink-soft hover:bg-moss/10 hover:border-moss/30 hover:text-moss-dark transition-all duration-150 focus-visible:ring-2 focus-visible:ring-moss active:scale-95"
          >
            + {s}
          </button>
        ))}
      </div>
    </div>
  );
}

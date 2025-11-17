# Champion's Forge - AI Development Guide

## Project Overview
Champion's Forge is a **single-file PWA** (Progressive Web App) for habit tracking, focus timer, and win logging. The entire application logic lives in `index.html` with inline CSS and JavaScript - there is no build process or external dependencies.

## Architecture Essentials

### Single-File Structure
- **`index.html`**: Complete app (~2000+ lines) - HTML, CSS (`<style>`), and JavaScript (`<script>`)
- **`manifest.json`**: PWA metadata for installability
- **`sw.js`**: Service worker for offline caching
- **`images/`**: PWA icons (192x192, 512x512 variants)

### Data Storage Strategy
- **IndexedDB** (`EternalForge` database): Primary persistent storage via `DB` module
- **LocalStorage**: Sound settings and volume preferences only
- **State synchronization**: All changes call `State.save()` which writes to IndexedDB
- **Auto-save pattern**: Critical user actions (habit logging, win entries) save immediately - never rely on user clicking "Save"

### Core Modules (JavaScript Objects)
All modules are defined as singleton objects in global scope:

1. **`State`**: Central data store
   - `data.events[]`: Time-series log of all activities (habits, focus sessions, wins)
   - `data.habits{}`: Habit configurations (thresholds, icons, tracking options)
   - `data.habitLog{}`: Timestamps of last habit occurrences
   - `data.timer`: Active timer state for session persistence
   - Always call `State.save()` after mutations

2. **`UI`**: Rendering engine
   - Uses DOM manipulation (createElement, appendChild) - **never innerHTML for user data** (XSS prevention)
   - `renderHabits()`: Main habit display with real-time color coding (danger/warning/safe)
   - `updateHabitTimers()`: Called every 60s to refresh elapsed time displays
   - Event delegation for dynamically created elements

3. **`Timer`**: Focus/Pomodoro timer
   - Uses `requestAnimationFrame` (not `setInterval`) for efficiency
   - Supports modes: timer, stopwatch, pomodoro with rest cycles
   - Wake Lock API integration to prevent screen sleep
   - Browser notifications with permission check

4. **`Sound`**: Audio feedback
   - Web Audio API (OscillatorNode) - no external files
   - Settings: type (beep/success/alert/soft), volume, repeat count
   - Never auto-plays on settings save (only on timer/habit events)

5. **`Actions`**: User action handlers
   - `logHabit()`: Multi-step prompts (note → spending → duration)
   - `createCustomHabit()`: Validates uniqueness, generates keys as `custom_<sanitized_name>`
   - Always validates input before state mutations

## Key Conventions

### Habit System
- **Default habits**: Defined in `DEFAULT_HABITS` constant, marked with `isDefault: true`
- **Custom habits**: User-created, marked with `isCustom: true`, deletable
- **Time thresholds**: `[danger, warning, safe]` hours - determines color coding
- **Money tracking**: Habits with `isMoneyOnly: true` skip time tracking, show monthly totals
- **Spending tracking**: Optional `trackSpending` flag adds spending prompts to non-money habits

### Timer Modes
- **Regular timer**: Countdown from specified duration
- **Stopwatch**: Count up indefinitely
- **Pomodoro**: Work/rest cycles with optional loop count
- **Session persistence**: Timer state survives page refresh via `Timer.resume()`

### Event Log Structure
All events pushed to `State.data.events[]` with consistent schema:
```javascript
{ type: 'habit'|'focus'|'win', time: timestamp, ...specific_fields }
```

### Security Patterns
1. **XSS prevention**: Use `textContent` or `createElement` for user data, never `innerHTML +=`
2. **Input validation**: Check prompts for `null` (user cancelled), validate numbers/lengths
3. **Safe rendering**: Create DOM nodes programmatically, append sanitized text nodes

## Development Workflows

### Testing
- **No automated tests** - manual testing in browser
- **Key test scenarios**: Habit logging with/without spending, timer completion notifications, data export/import
- **PWA testing**: Use Chrome DevTools > Application > Service Workers
- **Offline mode**: Throttle network, verify caching works

### Debugging
- Use `console.log()` statements (see existing patterns in `logHabit()`)
- Check browser console for errors (IndexedDB failures, notification permissions)
- Inspect `State.data` in console: `State.data.events.filter(e => e.type === 'habit')`

### Making Changes
1. **Edit `index.html`** directly - it's the only source file
2. **Update cache version** in `sw.js` (CACHE constant) if static assets change
3. **Test data persistence**: Open DevTools > Application > IndexedDB > EternalForge
4. **Validate PWA**: Run Lighthouse audit for PWA compliance

## Common Tasks

### Adding a New Habit Field
1. Update `DEFAULT_HABITS` object with new property
2. Modify `State.load()` to handle migration for existing users
3. Update `Actions.logHabit()` if new field requires user input
4. Adjust `UI.renderHistory()` to display the field

### Modifying Timer Behavior
- Timer state in `State.data.timer` object
- Animation loop in `Timer.animationLoop()` (RAF pattern)
- Notification logic in `Timer.notify()` - always check permission first

### Changing UI Layout
- CSS variables defined in `:root` for theming
- Responsive breakpoint at `768px` in media query
- Grid layouts use `auto-fit` for flexibility

## Critical Files Reference
- **`index.html:600-800`**: Habit configuration system
- **`index.html:1200-1400`**: Timer engine implementation
- **`index.html:1600-1800`**: Event rendering (XSS-safe patterns)
- **`sw.js`**: Cache invalidation - increment `CACHE` version on updates

## Performance Notes
- **Event pruning**: History limited to 5000 entries (see `State.save()`)
- **RAF timing**: Timer updates throttled to 200ms for CPU efficiency
- **Habit refresh**: 60-second interval for time display updates (not real-time)

## Known Patterns to Preserve
- Inline styles for dynamic colors (avoid class switching overhead)
- Event listener cleanup via `replaceWith(cloneNode())` to prevent duplicates
- Wake Lock fallback handling (feature detection)
- Data export uses ISO timestamp in filename for uniqueness

# Plan: Idle/Tap Clicker Game — Telegram Mini App

**Created:** 2026-03-20
**Mode:** Fast
**File:** `.ai-factory/PLAN.md`

## Settings

- **Testing:** No
- **Logging:** Verbose (console.debug/log throughout — state changes, upgrades, prestige, offline calc)
- **Docs:** No mandatory checkpoint

## Description

Build a fully-featured idle/tap clicker game as a single `index.html` file (all CSS and JS inline).
Dark theme, all text in Russian, Telegram WebApp SDK integration.
Covers Stories 1–6 from STORIES.md (Story 7 / E2E tests excluded per requirements).

**Deliverable:** `/opt/clicker-game-tma/index.html`

---

## Tasks

### Phase 1 — Foundation

#### Task 1: HTML scaffold + dark theme CSS
**File:** `index.html`

Create the base `index.html` with:
- `<!DOCTYPE html>` + viewport meta, Telegram WebApp SDK `<script>` tag (CDN)
- CSS custom properties for dark theme: `--bg`, `--surface`, `--accent`, `--text`, `--text-muted`, `--coin-color`, etc.
- Layout sections: `#header` (score + per-sec), `#coin-area` (tap zone), `#shop-panel` (upgrades)
- Google Fonts or system font stack (no external CDN beyond Telegram SDK and optional font)
- Responsive full-viewport layout (flex column), no horizontal scroll
- CSS keyframe stubs: `@keyframes tap-bounce`, `@keyframes float-up`, `@keyframes particle`

**Logging:**
- `console.log('[INIT] DOM loaded')` on DOMContentLoaded

---

#### Task 2: Game state model + localStorage persistence
**File:** `index.html` — `<script>` section

Define the central `state` object:
```js
const state = {
  coins: 0,
  totalCoins: 0,       // всего заработано (для prestige)
  totalTaps: 0,
  tapPower: 1,
  perSecond: 0,
  crystals: 0,         // prestige currency
  prestigeMultiplier: 1,
  upgrades: { /* see Task 4 */ },
  stage: 'bronze',
  sessionStart: Date.now(),
  lastSave: Date.now(),
}
```

Functions:
- `saveState()` — serialize to `localStorage['clicker_save']`, log `console.debug('[SAVE] saved at', Date.now())`
- `loadState()` — deserialize, deep-merge with defaults, log `console.debug('[LOAD] loaded state', state)`
- `resetState()` — factory reset (used by prestige)
- Auto-save every 10 seconds via `setInterval`

---

### Phase 2 — Core Mechanics

#### Task 3: Tap mechanic + floating text animation
**File:** `index.html`

- Large coin/crystal `#coin` element in `#coin-area` (SVG circle or emoji + CSS styled div)
- `handleTap(e)` on `touchstart` + `click`:
  - Compute `gain = state.tapPower * state.prestigeMultiplier`
  - Apply Critical Tap chance (from upgrade) → `gain *= 10`
  - Apply Mega Tap (every 50th tap) → `gain *= 100`
  - Apply Gold Rush active multiplier (`x5`) if active
  - `state.coins += gain; state.totalCoins += gain; state.totalTaps++`
  - Log `console.debug('[TAP] gain=%d totalCoins=%d', gain, state.totalCoins)`
  - Trigger scale bounce animation on `#coin`
  - Spawn floating `+N` text at tap coordinates, animate float-up + fade-out, then remove DOM node
  - Spawn 3–5 particle divs at tap coords (gold sparks CSS animation)
  - Call `updateUI()`
- `formatNumber(n)` utility: `1200 → "1.2K"`, `3500000 → "3.5M"`, `1200000000 → "1.2B"`

**Logging:**
- `console.debug('[TAP] critical hit!')` when critical
- `console.debug('[TAP] mega tap!')` when mega tap triggers
- `console.debug('[TAP] gold rush active, multiplier=5')`

---

#### Task 4: Upgrade shop — 6 upgrades
**File:** `index.html`

Define `UPGRADES` config array:
```js
const UPGRADES = [
  { id: 'tapPower',    name: 'Сила тапа',        desc: '+%v за тап',      baseCost: 10,   baseEffect: 1,  type: 'tap'      },
  { id: 'autoClicker', name: 'Автокликер',        desc: '+%v/сек',         baseCost: 50,   baseEffect: 1,  type: 'passive'  },
  { id: 'multiplier',  name: 'Множитель',         desc: 'x%v к доходу',   baseCost: 200,  baseEffect: 2,  type: 'multiplier'},
  { id: 'critTap',     name: 'Критический тап',   desc: '%v% шанс x10',   baseCost: 150,  baseEffect: 5,  type: 'crit'     },
  { id: 'megaTap',     name: 'Мега тап',          desc: 'каждый 50-й x100',baseCost: 500,  baseEffect: 1,  type: 'mega'     },
  { id: 'goldRush',    name: 'Золотая лихорадка', desc: '10 сек всё x5',  baseCost: 1000, baseEffect: 1,  type: 'rush'     },
]
```

State per upgrade: `state.upgrades[id] = { level: 0 }`

Functions:
- `getUpgradeCost(id)` → `baseCost * Math.pow(1.15, level)`
- `buyUpgrade(id)` — check affordability, deduct coins, increment level, recalculate derived stats, log `console.log('[UPGRADE] bought %s level=%d cost=%d', id, level, cost)`, call `updateUI()`
- `recalcStats()` — recompute `state.tapPower`, `state.perSecond`, `state.critChance` from upgrade levels
- Render `#shop-panel`: scrollable list of upgrade cards (name, level badge, cost, effect description, Buy button)
- Button disabled + dimmed when `coins < cost`

**Logging:**
- `console.debug('[UPGRADE] recalcStats tapPower=%d perSec=%f critChance=%d%', ...)`

---

### Phase 3 — Progression Systems

#### Task 5: Passive income + offline progress
**File:** `index.html`

- `startAutoClicker()` — `setInterval` every 1000ms:
  - `gain = state.perSecond * state.prestigeMultiplier * (goldRushActive ? 5 : 1)`
  - `state.coins += gain; state.totalCoins += gain`
  - Log `console.debug('[AUTO] tick gain=%f perSec=%f', gain, state.perSecond)`
  - Call `updateUI()`
- On load, compute offline earnings:
  - `elapsed = Math.min(Date.now() - state.lastSave, 2 * 3600 * 1000)` (cap 2h)
  - `offlineCoins = state.perSecond * (elapsed / 1000) * state.prestigeMultiplier`
  - If `offlineCoins > 0`: add to `state.coins`, show `#offline-modal` popup "Вы заработали X монет пока отсутствовали"
  - Log `console.log('[OFFLINE] elapsed=%ds earned=%f', elapsed/1000, offlineCoins)`
- `#offline-modal` — centered overlay, dismiss button "Забрать"

---

#### Task 6: Visual progression + prestige system
**File:** `index.html`

**Visual stages** (change `--coin-color`, `--bg-gradient`, coin label):
| Stage    | Threshold | Color   |
|----------|-----------|---------|
| Бронза   | 0         | #cd7f32 |
| Серебро  | 1 000     | #c0c0c0 |
| Золото   | 100 000   | #ffd700 |
| Платина  | 10 000 000| #e5e4e2 |
| Алмаз    | 1 000 000 000 | #b9f2ff |

- `updateStage()` — compute stage from `state.totalCoins`, update CSS vars + `#stage-label`, log `console.log('[STAGE] changed to %s', stage)`
- Progress bar `#stage-progress` — fills toward next threshold

**Prestige:**
- `#prestige-btn` visible when `state.totalCoins >= 1_000_000`
- `calcPrestigeCrystals()` → `Math.floor(Math.log10(state.totalCoins))` crystals to earn
- `doPrestige()`:
  - Prompt confirm modal "Сбросить прогресс? Вы получите N кристаллов"
  - `state.crystals += earned`
  - `state.prestigeMultiplier = 1 + state.crystals * 0.1`
  - `resetState()` (keep crystals, prestigeMultiplier)
  - Log `console.log('[PRESTIGE] crystals=%d multiplier=%f', state.crystals, state.prestigeMultiplier)`

---

### Phase 4 — Integration & Polish

#### Task 7: Telegram Mini App integration
**File:** `index.html`

- Init: `const tg = window.Telegram?.WebApp; tg?.ready(); tg?.expand()`
- Haptic:
  - `hapticTap()` → `tg?.HapticFeedback.impactOccurred('light')`
  - `hapticBuy()` → `tg?.HapticFeedback.impactOccurred('medium')`
  - Call on each tap + on each upgrade purchase
- Theme: read `tg?.colorScheme` (`'dark'`/`'light'`), apply CSS class `document.body.classList.add(colorScheme)`; default dark
- Share button `#share-btn`: `tg?.switchInlineQuery(...)` or `tg?.sendData(...)` with formatted score string "Я набрал X монет в Кликере! 🎮"
- Fallback: all `tg?.` calls are optional-chained; game works fully in browser without SDK

**Logging:**
- `console.log('[TG] WebApp detected, version=%s', tg?.version)`
- `console.log('[TG] colorScheme=%s', tg?.colorScheme)`
- `console.debug('[TG] haptic tap')`

---

#### Task 8: Polish — particles, stats panel, UI refinements
**File:** `index.html`

- **Particles:** `spawnParticles(x, y, count=5)` — create `<div class="particle">` elements at (x,y), random direction CSS vars, remove after animation ends
- **Stats panel** `#stats`:
  - Всего тапов: `state.totalTaps`
  - Всего заработано: `formatNumber(state.totalCoins)`
  - Время в игре: formatted `hh:mm:ss` from `sessionStart`
  - Кристаллы: `state.crystals` (if > 0)
  - Обновляется каждую секунду
- **`updateUI()`** — single function that refreshes all DOM: score header, per-sec, shop buttons, stage, prestige button visibility, stats
- **CSS polish:**
  - Shop card hover/active transitions (scale 0.98, brightness)
  - Coin glow matching current stage color
  - Mobile-friendly touch targets (min 44px)
  - Prevent double-tap zoom (`touch-action: manipulation`)
  - Smooth coin color transition on stage change

**Logging:**
- `console.debug('[UI] updateUI called, coins=%f', state.coins)`

---

## Commit Plan

| Checkpoint | After Task | Commit Message |
|---|---|---|
| 1 | Task 2 | `feat: scaffold HTML + dark theme + state persistence` |
| 2 | Task 4 | `feat: tap mechanic + upgrade shop` |
| 3 | Task 6 | `feat: passive income, offline progress, visual stages + prestige` |
| 4 | Task 8 | `feat: TMA integration + polish (particles, stats, animations)` |

---

## Notes

- All in one `index.html` — no build tools, no external assets except Telegram SDK CDN
- `formatNumber` must handle 0–trillions gracefully
- Gold Rush: set a flag + timer when purchased (activates immediately), not a passive upgrade
- Multiplier upgrade: multiply `tapPower` and `perSecond` by `1 + 0.5 * level` (tweak for balance)
- Auto-save on `visibilitychange` (tab hide / app minimize) to ensure `lastSave` is accurate for offline calc

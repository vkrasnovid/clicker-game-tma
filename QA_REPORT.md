# QA Report — Idle/Tap Clicker TMA
**Date:** 2026-03-20
**File reviewed:** `index.html` (943 lines, single-file SPA)
**Reviewer:** QA Tester (static analysis)

---

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 0     |
| HIGH     | 3     |
| MEDIUM   | 6     |
| LOW      | 7     |

No issues block launch entirely, but three HIGH-severity bugs affect core mobile gameplay.

---

## CRITICAL

_None found._

---

## HIGH

### H-1 — Missing `touchend` listener on shop cards made affordable via `updateUI()`

**Location:** `updateUI()` lines 875–882, vs `renderShop()` lines 834–837

**Description:**
`renderShop()` correctly attaches **both** `click` and `touchend` to purchasable cards:
```js
card.addEventListener('click', () => buyUpgrade(upg.id));
card.addEventListener('touchend', (e) => { e.preventDefault(); buyUpgrade(upg.id); });
```
`updateUI()` has a lightweight affordability-update path that, when a card transitions from `disabled → enabled`, only adds a `click` listener — no `touchend`:
```js
if (canBuy && card.classList.contains('disabled')) {
  card.classList.remove('disabled');
  card.addEventListener('click', () => buyUpgrade(upg.id)); // ← touchend never added
}
```
On mobile, `touchend` without `preventDefault()` will eventually generate a synthetic `click`, so purchases may still go through — but with a ~300 ms ghost-click delay and inconsistent behaviour across Telegram WebApp versions and Android webviews. The experience degrades noticeably compared to the immediate `touchend` path in `renderShop()`.

**Impact:** Mobile users experience delayed/unreliable taps on newly-affordable shop cards.
**Fix:** Either always call `renderShop()` when affordability changes, or add the `touchend` handler in the same branch.

---

### H-2 — `autoClicker` passive income accumulates fractional coins

**Location:** `startAutoClicker()` lines 778–786

**Description:**
```js
const gain = state.perSecond * state.prestigeMultiplier * (goldRushActive ? 5 : 1);
state.coins += gain;        // float added directly
state.totalCoins += gain;
```
`state.perSecond` and `state.prestigeMultiplier` can produce non-integer results (e.g., `1.5 * 1.3 = 1.95`). These floats accumulate in `state.coins`. `formatNumber()` calls `Math.floor(n)` for values < 1 000, so the displayed score will silently lag behind the real value. After enough ticks the discrepancy becomes visible (e.g., 0.95 coins display as "0" while the player's passive-income bar shows a non-zero rate).

**Impact:** Score display is incorrect; player cannot tell how many coins they actually have.
**Fix:** Apply `Math.floor()` or `Math.ceil()` to `gain` before adding to state.

---

### H-3 — `tapCount` is not reset on Prestige

**Location:** `confirmPrestige()` → `resetState()`, line 684 (`let tapCount = 0` is module-level)

**Description:**
`tapCount` drives the Mega Tap mechanic (every 50th tap gives ×100). It is a module-level variable and is **not** reset in `resetState()`:
```js
function resetState() {
  // ... resets state object ...
  // tapCount is never touched
}
```
After a prestige reset a player who was at tap #48 will trigger a Mega Tap on their 2nd tap of the new run, completely bypassing the intended 50-tap cooldown.

**Impact:** Mega Tap fires at wrong cadence post-prestige; breaks game balance.
**Fix:** Add `tapCount = 0;` inside `resetState()`.

---

## MEDIUM

### M-1 — Double-event registration on coin element (potential double-tap)

**Location:** Lines 916–917

```js
coinEl.addEventListener('touchstart', handleTap, { passive: false });
coinEl.addEventListener('click',      handleTap);
```
`handleTap` calls `e.preventDefault()` on `touchstart`, which should suppress the subsequent synthetic `click` on most browsers. However, this behaviour is not guaranteed in all Telegram WebApp webviews (especially older Android). If `click` fires after `touchstart`, one physical tap triggers two scoring events.

**Impact:** Score may double on certain devices/OS versions.
**Fix:** Check `e.type === 'touchstart'` early in `handleTap` and guard the `click` handler with a touch-detection flag.

---

### M-2 — `megaTap` upgrade level has no effect on bonus magnitude

**Location:** `handleTap()` lines 706–710, `UPGRADES` config line 429

**Description:**
The Mega Tap bonus is hardcoded:
```js
if (tapCount % 50 === 0) {
  gain *= 100;  // always ×100, regardless of megaTap level
}
```
`getUpgradeEffect('megaTap')` returns `1 * level` but this value is never used in the tap logic. Every level beyond level 1 is wasted money with zero additional benefit. The shop card correctly describes "кждый 50-й ×100" but that description never changes with level — it just charges more.

**Impact:** Players are misled into purchasing an upgrade with no scaling effect.
**Fix:** Either use `getUpgradeEffect('megaTap')` as a multiplier (`gain *= 100 * effect`) or cap megaTap at level 1.

---

### M-3 — Share uses `tg.switchInlineQuery` (incorrect for most bots)

**Location:** `shareScore()` lines 898–899

```js
if (tg?.switchInlineQuery) {
  tg.switchInlineQuery(msg, ['users', 'groups']);
}
```
`switchInlineQuery` opens an inline query in a chat — it requires the bot to be registered as an inline bot. A standard game TMA bot typically is not. The call will silently fail or produce an unexpected UI. `tg.openTelegramLink` with a pre-composed share URL or `tg.shareToStory` (for supported clients) are the correct alternatives.

**Impact:** Share button may do nothing or behave unexpectedly.
**Fix:** Use `tg.openTelegramLink('https://t.me/share/url?url=...&text=...')` as primary share path.

---

### M-4 — `goldRushActive` state not persisted to `localStorage`

**Location:** `saveState()` line 473, `goldRushEndTime` line 470

**Description:**
`goldRushActive` and `goldRushEndTime` are module-level variables never written to `state` and therefore never serialised. Reloading the page during an active Gold Rush silently cancels the 10-second bonus without refunding the coin cost.

**Impact:** Players lose a paid Gold Rush on any reload/backgrounding event.
**Fix:** Persist `goldRushEndTime` in `state`. On `loadState()`, if `goldRushEndTime > Date.now()`, call `activateGoldRush()` with the remaining duration.

---

### M-5 — Multiplier upgrade description shows misleading "×1.0" at level 0

**Location:** `renderShop()` line 812–814

```js
upg.id === 'multiplier'
  ? 'x' + (1 + 0.5 * level).toFixed(1)  // level=0 → "x1.0" (no effect)
  : ...
```
At level 0 the card reads "x доходу: ×1.0", which implies the upgrade is active when it has not been purchased. Players may skip it believing they already have a multiplier.

**Impact:** Minor UX confusion; could deter purchases of the multiplier upgrade.
**Fix:** At level 0 show the post-purchase value ("×1.5 after purchase") or hide the description until level ≥ 1.

---

### M-6 — No state validation on `loadState()` (corrupted save crashes game silently)

**Location:** `loadState()` lines 483–503

**Description:**
The deep merge does not validate that loaded numeric fields are finite numbers:
```js
state = { ...def, ...saved };
```
A corrupted or manually edited `localStorage` value (e.g. `coins: "NaN"` or `prestigeMultiplier: null`) will propagate into `state`. `formatNumber()` guards against `NaN` in display, but arithmetic (e.g. `gain *= state.prestigeMultiplier`) will silently propagate `NaN` through the entire game state.

**Impact:** Single bad save can permanently break a game session until `localStorage` is cleared.
**Fix:** After merging, clamp numeric fields: `state.coins = Math.max(0, Number(state.coins) || 0)`, etc.

---

## LOW

### L-1 — Three `setInterval` references never stored (no cleanup path)

**Location:** Lines 518, 778–786, 891–893

```js
setInterval(saveState, 10000);          // auto-save
setInterval(() => { /* auto-clicker */ }, 1000);
setInterval(() => { /* time display */ }, 1000);
```
None of the interval IDs are stored. The intervals cannot be cleared if the game needs to reset, pause (e.g. for a tutorial overlay), or if a future test harness needs to control time. In practice, for a page-lifetime single-instance game this is survivable, but it makes testing and future feature work difficult.

**Fix:** `const autoSaveId = setInterval(...)` — store and expose via module API.

---

### L-2 — Gold Rush can be refreshed/extended by re-purchasing

**Location:** `activateGoldRush()` lines 604–617

```js
if (goldRushTimer) clearTimeout(goldRushTimer); // previous timer cancelled
goldRushTimer = setTimeout(() => { ... }, 10000); // full 10s restart
```
Each Gold Rush purchase clears the running timer and restarts the full 10 seconds. Wealthy players can permanently maintain the ×5 multiplier by purchasing repeatedly before the timer expires.

**Impact:** Unintended infinite Gold Rush is possible at higher coin counts; breaks late-game balance.
**Fix:** Either prevent purchase while active, or apply the new duration additively (`goldRushEndTime += 10000` and update a single managing interval rather than resetting).

---

### L-3 — Prestige button threshold (1 M) not aligned with stage thresholds

**Location:** `updateUI()` line 863

```js
const prestigeVisible = state.totalCoins >= 1_000_000;
```
The Gold stage threshold is 100 K and Platinum is 10 M; prestige appears at 1 M. This is between two stages, offering no landmark feedback to the player ("reach Gold to unlock Prestige" or "reach Platinum to unlock Prestige" would be clearer).

**Impact:** Minor UX confusion; not a functional bug.
**Fix:** Align prestige unlock threshold with a stage boundary (e.g. `>= STAGES[3].threshold` = 10 M).

---

### L-4 — `crystals-display` shows total crystals, not gain preview during regular play

**Location:** `updateUI()` lines 854–860

The header shows current crystals + multiplier, but the player has no in-game indicator of how many crystals they would earn if they prestiged now without opening the modal. This creates unnecessary friction.

**Impact:** Low discoverability of prestige timing.
**Fix:** Add a small hint below the prestige button: "Prestige now → +N crystals".

---

### L-5 — `stat-time` counter also updated by a second `setInterval` in addition to auto-clicker tick

**Location:** Lines 891–893 and `updateUI()` line 887

`updateUI()` already updates `stat-time` on every call (including every auto-tick). A dedicated 1-second interval also updates it. This causes redundant DOM writes and an effective update rate of ~2× per second.

**Impact:** Negligible performance cost; harmless double write.
**Fix:** Remove the dedicated `stat-time` interval; rely on `updateUI()` calls.

---

### L-6 — `coin-bounce` animation class removal added as listener on every tap

**Location:** `handleTap()` lines 733–737

```js
coin.classList.remove('coin-bounce');
void coin.offsetWidth;
coin.classList.add('coin-bounce');
coin.addEventListener('animationend', () => coin.classList.remove('coin-bounce'), { once: true });
```
`{ once: true }` ensures the listener self-removes, so there is no permanent leak. However, if the player taps faster than the 250 ms animation, the previous `animationend` listener fires after the class is re-added, removing it prematurely. The `void coin.offsetWidth` reflow trick mitigates this partially but not completely under rapid tapping.

**Impact:** Visual jitter (coin bounce animation cut short) under fast tapping.
**Fix:** Track whether a bounce is already in progress and skip re-adding the class, or use a CSS `animation-play-state` toggle instead.

---

### L-7 — No `tg.enableClosingConfirmation()` called

**Location:** Telegram SDK init block, lines 402–413

Auto-save fires on `visibilitychange`, but there is no closing confirmation to warn users who accidentally close the app before the 10-second autosave fires (e.g. if they tapped rapidly and have unsaved progress).

**Impact:** Players can lose up to 10 seconds of progress on accidental close.
**Fix:** Call `tg.enableClosingConfirmation()` in the SDK init block.

---

## Checklist Summary

| Area               | Status   | Notes |
|--------------------|----------|-------|
| Tap mechanic       | ⚠️ MEDIUM | Potential double-fire on some Android webviews (M-1) |
| +N float animation | ✅ Pass   | `spawnFloatText` with `animationend` cleanup is correct |
| Upgrade shop (×6)  | ⚠️ HIGH   | Mobile affordability transition missing `touchend` (H-1); megaTap scaling broken (M-2) |
| Price scaling      | ✅ Pass   | `baseCost * 1.15^level` correctly implemented |
| Auto-clicker       | ⚠️ HIGH   | Float accumulation causes display lag (H-2) |
| Offline earnings   | ✅ Pass   | 2-hour cap enforced; `perSecond` recalculated before use |
| Visual stages      | ✅ Pass   | Thresholds at 1K/100K/10M/1B match spec; CSS vars updated |
| Stage progress bar | ✅ Pass   | Linear interpolation correct; max stage shows 100% |
| Prestige reset     | ⚠️ HIGH   | `tapCount` not reset (H-3); crystals/multiplier preserved correctly |
| Crystal formula    | ✅ Pass   | `floor(log10(totalCoins))` correct; modal preview correct |
| Gold Rush 10s ×5   | ⚠️ MEDIUM | State lost on reload (M-4); infinite refresh exploit (L-2) |
| Critical Tap ×10   | ✅ Pass   | Random check correct; 80% cap enforced; precedence below mega |
| localStorage       | ⚠️ MEDIUM | No value validation on load (M-6) |
| Telegram haptic    | ✅ Pass   | Optional chaining guards all `tg.*` calls |
| Telegram theme     | ✅ Pass   | `colorScheme === 'light'` applies `.light` class |
| Telegram share     | ⚠️ MEDIUM | `switchInlineQuery` likely wrong API (M-3) |
| Responsive 320px   | ✅ Pass   | `max-width:480px`, `100dvh`, `touch-action:manipulation` all correct |
| Memory leaks       | ⚠️ LOW    | `setInterval` IDs not stored (L-1); float/particle cleanup correct |

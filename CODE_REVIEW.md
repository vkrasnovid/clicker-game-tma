# Code Review — index.html (Clicker Game TMA)

**Verdict: REQUEST_CHANGES**

Reviewed: 942-line single-file idle/tap clicker Telegram Mini App.
Prior fix annotations (H-1…M-6) reflect an earlier review pass — this review validates residual issues.

---

## 1. Code Quality & Structure

**Good:**
- `'use strict'` at module top.
- Config objects (`UPGRADES`, `STAGES`) are cleanly separated and immutable at runtime.
- `DEFAULT_STATE()` factory avoids shared-reference mutations.
- Consistent `[TAG]` log prefixes make debugging readable.
- CSS custom properties enable clean theme switching.
- `{ once: true }` on animation-end listeners prevents leaks.

**Issues:**

### CQ-1 — `updateUI()` triggers `renderShop()` on unafford transition (Medium)
**Location:** `index.html:906`

```js
} else if (!canBuy && !card.classList.contains('disabled')) {
  renderShop();   // full DOM rebuild
}
```

`renderShop()` is a full shop rebuild (wipes `innerHTML`, recreates all DOM nodes). This branch fires every time any card transitions from affordable to unaffordable. Since `updateUI()` is called on every tap and every auto-tick (1/sec), if the player's coin balance oscillates around any upgrade's cost threshold — which is common early-game when auto income trickles in — a full shop rebuild triggers on every second. This causes visible flicker and unnecessary GC pressure.

**Fix:** track affordability as a data attribute and only add/remove classes without rebuilding:
```js
card.dataset.canBuy = canBuy ? '1' : '0';
card.classList.toggle('disabled', !canBuy);
// manage listeners once via a single delegated listener on #shop-panel
```

### CQ-2 — Inline `onclick` globals in HTML (Low)
**Location:** `index.html:369–371, 393–394`

`onclick="openPrestigeModal()"` etc. requires these functions to be globally scoped. In strict mode inside a `<script>` tag these are window-global anyway, but it tightly couples HTML to JS internals and is fragile if the file is ever wrapped in a module.

**Fix:** bind in JS after DOM ready (already done for the coin element — apply the same pattern here).

### CQ-3 — `UPGRADES.find()` on every cost/effect call (Low)
**Location:** `index.html:564, 570`

`getUpgradeCost` and `getUpgradeEffect` each call `UPGRADES.find(u => u.id === id)` — O(n) scan with no memoisation. Called frequently during every `updateUI()` shop affordability pass (6 upgrades × 2 calls = 12 scans/tick). With only 6 entries this is negligible today, but a simple `Map` lookup would be cleaner:
```js
const UPGRADES_MAP = Object.fromEntries(UPGRADES.map(u => [u.id, u]));
```

### CQ-4 — `getComputedStyle` inside particle loop (Low)
**Location:** `index.html:788`

```js
el.style.background = getComputedStyle(document.documentElement)
  .getPropertyValue('--coin-color').trim();
```

Called 4 times per tap (once per particle). `getComputedStyle` forces a style recalculation. Cache the result outside the loop or once per stage change.

---

## 2. Security

**Overall: No XSS vectors found.** All `innerHTML` assignments use data from:
- Compile-time constants (`UPGRADES`, `STAGES` config) — not user-controlled.
- `formatNumber()` output — returns only digits, letters `TMBK`, and `.` characters.
- `Math.*` operations returning numbers.

`loadState()` has thorough numeric clamping for all game-play-critical fields.

### SEC-1 — `state.lastSave` not sanitized (Medium)
**Location:** `index.html:501–509, 810`

The numeric-clamping block in `loadState()` sanitizes `coins`, `totalCoins`, `tapPower`, `perSecond`, `critChance`, `crystals`, `prestigeMultiplier`, and `goldRushEndTime` — but **not `lastSave`**.

`calcOfflineEarnings()` computes:
```js
const elapsed = Math.min(Date.now() - state.lastSave, 2 * 3600 * 1000);
const coins = Math.floor(state.perSecond * (elapsed / 1000) * state.prestigeMultiplier);
```

If `state.lastSave` is set to a **future timestamp** (by a player editing their own localStorage), `Date.now() - state.lastSave` is **negative**. `Math.min(negative, 7_200_000)` = negative. `Math.floor(perSecond * negative)` = negative. The check `if (offlineCoins > 0)` gates the modal but `state.coins += offlineCoins` is **not guarded** — negative offline earnings silently deduct coins from the player.

Although this is a self-inflicted attack on a single user's own save, it creates a surprising outcome (coins go down on startup) that could be perceived as a bug.

**Fix:**
```js
state.lastSave = Math.min(Date.now(), Math.max(0, Number(state.lastSave) || 0));
```
And/or guard in `calcOfflineEarnings()`:
```js
const elapsed = Math.max(0, Math.min(Date.now() - state.lastSave, 2 * 3600 * 1000));
```

### SEC-2 — localStorage tamper affects prestige crystal count (Informational)
`state.crystals` is clamped to `Math.max(0, ...)` so it cannot go negative. But it is not capped at an upper bound, so a tampered save with `crystals: 9999` gives `prestigeMultiplier = 1 + 9999 * 0.1 = 1000.9`. Acceptable for a single-player idle game with no server backend.

---

## 3. Performance & Memory Leaks

### PERF-1 — `setInterval` IDs discarded, cannot be cleaned up (Low)
**Location:** `index.html:533, 795–804, 918`

Three `setInterval` calls store no ID:
```js
setInterval(saveState, 10000);   // line 533
// ...inside startAutoClicker():
setInterval(() => { ... }, 1000); // line 795
// stats timer:
setInterval(() => { ... }, 1000); // line 918
```

`startAutoClicker()` is called once inside `DOMContentLoaded` so there is no double-registration risk today. However, if the init path is ever re-entered (e.g., hot-reload in dev tools, or a future SPA refactor), all three intervals stack. Store the IDs:
```js
let autoClickerInterval = null;
// ...
if (autoClickerInterval) clearInterval(autoClickerInterval);
autoClickerInterval = setInterval(...);
```

### PERF-2 — Float text and particle elements are correctly cleaned up (Pass)
`animationend` with `{ once: true }` removes elements from the DOM. No accumulation.

### PERF-3 — Coin bounce listener is correctly one-shot (Pass)
`{ once: true }` on `animationend` for coin-bounce. No listener accumulation.

### PERF-4 — `updateUI()` affordability listener accumulation (Low)
**Location:** `index.html:900–904`

When a card transitions from disabled to affordable:
```js
card.addEventListener('click', () => buyUpgrade(upg.id));
card.addEventListener('touchend', (e) => { ... });
```

These add new closures. In practice, `buyUpgrade()` immediately calls `renderShop()` which replaces the entire card DOM, so the stale listeners are GC'd. The accumulation only happens if `updateUI()` fires the transition multiple times before a `renderShop()` — which cannot occur because once `disabled` is removed on first transition, subsequent `updateUI()` calls take neither branch. **No real leak**, but the pattern is fragile and relies on implicit ordering guarantees.

**Recommendation:** Use event delegation on `#shop-panel` instead:
```js
shopPanel.addEventListener('click', (e) => {
  const card = e.target.closest('.shop-card:not(.disabled)');
  if (card) buyUpgrade(card.dataset.id);
});
```

---

## 4. Edge Cases

### EDGE-1 — `calcOfflineEarnings` with future `lastSave` (covered under SEC-1)

### EDGE-2 — `calcPrestigeCrystals()` with zero `totalCoins` (Pass)
```js
Math.max(1, Math.floor(Math.log10(0)))
// = Math.max(1, Math.floor(-Infinity))
// = Math.max(1, -Infinity) = 1
```
Returns 1, which is correct minimum. ✓

### EDGE-3 — `getUpgradeCost()` at extreme levels overflows to `Infinity` (Informational)
`Math.ceil(baseCost * Math.pow(1.15, level))` → at level ≈ 900, result exceeds `Number.MAX_VALUE` and becomes `Infinity`. `state.coins < Infinity` is always true so the card would always appear unaffordable — correct degenerate behaviour. Not a bug for a typical session.

### EDGE-4 — Double-confirm prestige race (Low)
`confirmPrestige()` is called from an HTML button. If a user double-taps the confirm button faster than the modal's `display = 'none'` takes effect, `confirmPrestige()` fires twice. The second call operates on the already-reset state (where `totalCoins = 0`), awarding `max(1, floor(log10(0))) = 1` extra crystal and re-triggering a reset of an already-reset state. Net effect: 1 free crystal.

**Fix:** Disable the confirm button inside `confirmPrestige()` before the first operation completes, or guard with a flag.

### EDGE-5 — NaN propagation in `state.tapPower` (Pass)
`recalcStats()` uses `Math.max(1, ...)` so `tapPower` is always ≥ 1. ✓

### EDGE-6 — `tapCount` rollover (Pass)
`tapCount` is a JS float64. `Number.MAX_SAFE_INTEGER % 50` is well-defined. Not a practical concern.

---

## 5. Telegram SDK Usage

### TG-1 — `tg.sendData()` closes the WebApp (Medium)
**Location:** `index.html:929`

```js
} else if (tg?.sendData) {
  tg.sendData(msg);
}
```

`Telegram.WebApp.sendData()` sends a message to the bot **and immediately closes the Mini App**. This is the correct method when the WebApp is opened via a keyboard button and the intent is to return data to the bot. For a share action initiated by the user in the middle of a game session, closing the app is unexpected and destructive. The user loses their game context.

This branch executes when `tg.openTelegramLink` is unavailable (older SDK versions). It should be replaced with a graceful fallback (e.g., copy to clipboard) rather than closing the app.

**Fix:**
```js
} else {
  navigator.clipboard?.writeText(msg).catch(() => {});
}
```

### TG-2 — Share URL has no meaningful bot deep link (Low)
**Location:** `index.html:927`

```js
tg.openTelegramLink('https://t.me/share/url?url=https%3A%2F%2Ft.me%2F&text=' + encodeURIComponent(msg));
```

The `url` parameter is `https://t.me/` — the Telegram homepage — not the bot's link. The resulting share only contains the text. If the intent is viral growth, replace with the actual bot/game deep link.

### TG-3 — SDK init before `DOMContentLoaded` (Pass)
`tg.ready()` and `tg.expand()` are called synchronously at script parse time. This is correct per the Telegram WebApp SDK contract — `ready()` must be called as early as possible to signal the app has loaded. ✓

### TG-4 — `tg.colorScheme` theme detection (Pass)
Correctly applies `body.light` before first render. ✓

### TG-5 — `HapticFeedback` guarded (Pass)
`tg?.HapticFeedback?.impactOccurred(...)` — double optional chain handles both missing `tg` and missing `HapticFeedback`. ✓

---

## Summary Table

| ID | Area | Severity | Description |
|----|------|----------|-------------|
| SEC-1 | Security | **Medium** | `state.lastSave` unsanitized — negative offline earnings possible |
| TG-1 | Telegram SDK | **Medium** | `sendData()` closes the app during share — destructive fallback |
| CQ-1 | Code Quality | **Medium** | `renderShop()` full rebuild called inside `updateUI()` affordability loop |
| EDGE-4 | Edge Case | Low | Double-tap confirm prestige awards 1 free crystal |
| PERF-1 | Performance | Low | `setInterval` IDs discarded — leak risk if init runs more than once |
| PERF-4 | Performance | Low | Per-tick listener registration pattern is fragile |
| CQ-2 | Code Quality | Low | Inline `onclick` globals couple HTML to JS |
| CQ-3 | Code Quality | Low | `UPGRADES.find()` on every tick — use a Map |
| CQ-4 | Code Quality | Low | `getComputedStyle` inside particle loop |
| TG-2 | Telegram SDK | Low | Share URL has no bot deep link |
| SEC-2 | Security | Info | Uncapped `crystals` in localStorage (single-player, acceptable) |

**Required before approval:**
1. Fix `SEC-1` — clamp or guard `state.lastSave` to prevent negative offline earnings.
2. Fix `TG-1` — remove `sendData()` fallback or replace with a non-destructive alternative.
3. Fix `CQ-1` — avoid full `renderShop()` inside the `updateUI()` affordability loop; the current code can cause a full DOM rebuild every second when coins oscillate near an upgrade threshold.

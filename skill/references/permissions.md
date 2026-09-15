# Admin / user view separation and access codes

## Why a screen-privacy model at all

The request that led here was: regular staff should be able to run the reconciliation (upload files, click run) but only see pass/fail counts, while only an admin (or someone the admin specifically authorizes) can see which counterparty/item failed and why, and the underlying amounts. There's no server, so this can only ever be a **UI-level visibility gate**, not real access control. Say this plainly to the user before building it, and bake the same disclosure into the tool's own UI copy (not just the chat) — the tool will outlive this conversation and future viewers need to know its limits too.

## Two tiers

1. **Admin** — one shared PIN (first use sets it, stored in `localStorage`; never store it anywhere it'd sync across machines, since that would be a false promise of shared access control). Admin can: configure tolerance rules, manage the user-access-code list, and see full detail.
2. **Granted user** — admin registers a name + auto-generated 4-digit access code per person (`{name, code, active}` list in `localStorage`). A granted user enters name+code to unlock detail viewing for that browser session only (`sessionStorage`, cleared on tab close) — they cannot reach rules configuration or user management, only detail viewing.

```js
function isAdminUnlocked(){ return sessionStorage.getItem('admin_unlocked') === '1'; }
function isUserUnlocked(){ return sessionStorage.getItem('user_unlocked') === '1'; }
function isDetailUnlocked(){ return isAdminUnlocked() || isUserUnlocked(); }
```

Gate detail-rendering sections on `isDetailUnlocked()`; gate admin-only panels (rules, user management) on `isAdminUnlocked()` specifically — a granted user must not be able to reach those just because they unlocked something.

## View switching (why one file, one `currentView` state)

`currentView` is `'admin'` or `'user'`, driving `document.body.dataset.view` and CSS rules like `body:not([data-view="admin"]) #adminOnlyPanels{display:none;}`. Switching views is a left-side hamburger button that opens a slide-in drawer (not a small dropdown — see SKILL.md design notes) listing "사용자 화면" / "관리자 화면", with a "잠그고 나가기" (lock and exit) item that appears only while in admin view, to fully clear the admin session rather than just visually switching away from it.

Do not split this into two physical HTML files thinking it's cleaner. It isn't — see SKILL.md's "two views, one file" lesson. The `localStorage`-backed rules/mappings/user-list must be visible from whichever view is active, and that only works reliably if it's the same document.

## Access log

Every successful granted-user unlock appends `{name, time}` to a capped `localStorage` list (last 50), shown to the admin as "최근 열람 기록" (recent access log) — cheap, useful accountability trail even though the gate itself isn't real security.

## Cross-machine sync (there isn't one, by design)

`localStorage` is per-browser-profile-per-file. If people use the tool on different PCs, the admin's user list set up on PC-A won't exist on PC-B. The template includes an explicit "내보내기/가져오기" (export/import) pair: export serializes the user list to JSON and shows it via `prompt()` for the admin to copy; import parses pasted JSON back into the list. This is a manual, occasional sync step — say so, don't imply it's automatic.

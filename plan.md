# Multi-Entry, Google Drive, Password, Excel, Backup — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade the HaFiz PhotoCopy single-file SPA dashboard with ID-based multi-entry per day, SHA-256 password protection, Google Drive persistence, and data export/import capabilities.

**Architecture:** All changes target one file (`hafiz_photocopy_dashboard.html`). Five independent feature layers stack on the existing codebase: (1) add UUID-style IDs to entries and switch all CRUD to ID-based lookup, (2) rewrite `renderTable()` for grouped-by-date display, (3) insert an auth overlay that gates the entire dashboard, (4) replace `window.storage` with Google Drive API + localStorage fallback, (5) add Excel export, JSON backup, and JSON restore. Each produces a working intermediate state.

**Tech Stack:** Vanilla JS (no frameworks), Google Identity Services (GIS) for Drive OAuth, Google Drive API v3 for file storage, SubtleCrypto (SHA-256) for password hashing, SheetJS/xlsx for Excel export. All dependencies loaded via CDN `<script>` tags.

## Global Constraints

- Single file: `hafiz_photocopy_dashboard.html` — all CSS, HTML, and JS in one document.
- Password is `2854` — hashed client-side via SHA-256 / SubtleCrypto.
- Currency is Pakistani Rupees — use `fmtMoney(n)` for display, raw numbers for storage.
- Date strings are ISO `"YYYY-MM-DD"` — parse with `new Date(dateStr + "T00:00:00")` to avoid timezone offsets.
- Weekly order starts Monday: `WEEKDAY_ORDER` constant.
- XSS prevention — use `escapeHtml()` for any user-supplied value rendered to HTML.
- State changes — after any mutation, call `renderAll()` then `showToast()`.
- Google Drive JSON file name: `hafiz_photocopy_dashboard_data.json`.
- Drive file format includes `version: 2` for schema migration.

---

## File Structure

Only one file is modified: `hafiz_photocopy_dashboard.html` (864 lines). Every task operates on specific sections within it.

| Section | Lines | Task |
|---|---|---|
| CSS `:root` / body / header / KPI / panels / table / modal | 10–203 | Tasks 3, 5 (adds auth, header-button CSS) |
| HTML: header / KPI grid / chart panels / table / footer | 208–267 | Task 5 (new header buttons) |
| HTML: modal form | 269–323 | Task 1 (hidden field ID swap) |
| HTML: toast | 325 | (unchanged) |
| `<head>` CDN scripts | before line 9 | Tasks 4, 5 (GIS + SheetJS script tags) |
| `<script>` — `INITIAL_DATA` | 328 | Task 1 (add `id` to all 62 entries) |
| `<script>` — `state` / helpers | 333–359 | Task 1 (`generateId()`, `migrateEntries()`) |
| `<script>` — storage | 362–441 | Tasks 1, 4 (rewrite for ID-based CRUD, then Drive) |
| `<script>` — derived data | 444–457 | (unchanged) |
| `<script>` — rendering | 460–760 | Task 2 (rewrite `renderTable()` for grouped display) |
| `<script>` — modal / form | 762–848 | Task 1 (ID-based edit, default date) |
| `<script>` — boot | 856–860 | Task 3 (add auth check before init) |
| *New* — auth overlay CSS | after line 203 | Task 3 |
| *New* — auth overlay HTML | after line 267 | Task 3 |
| *New* — Google Drive JS | after line 441 | Task 4 |
| *New* — Backup/Restore/Excel JS | after `renderAll()` | Task 5 |

---

## Task 1: Add ID Infrastructure, Default Date, and ID-Based CRUD

**Files:**
- Modify: `hafiz_photocopy_dashboard.html` — INITIAL_DATA, helper functions, storage layer, form handler

**Interfaces:**
- Consumes: existing `state.dataByMonth`, `state.months`, `window.storage.get/set`
- Produces: `generateId()` → string, `migrateEntries()`, `findEntryById(id)` → entry|null, `upsertDay(entry, originalId)`, `deleteDay(id)`, `openModalForEdit(id)`

- [ ] **Step 1: Add `generateId()` and `migrateEntries()` helper functions**

Insert after `showToast()` (line 359):

```js
function generateId() {
  return Date.now().toString(36) + '_' + Math.random().toString(36).slice(2, 8);
}

function findEntryById(id) {
  for (const monthKey of state.months) {
    const entry = (state.dataByMonth[monthKey] || []).find(d => d.id === id);
    if (entry) return entry;
  }
  return null;
}

function migrateEntries() {
  let changed = false;
  for (const monthKey of Object.keys(state.dataByMonth)) {
    const arr = state.dataByMonth[monthKey];
    for (const entry of arr) {
      if (!entry.id) {
        // Use date as deterministic ID for existing single-per-day entries
        entry.id = entry.date;
        changed = true;
      }
    }
  }
  return changed;
}
```

- [ ] **Step 2: Add `id` field to all 62 entries in `INITIAL_DATA`**

The current `INITIAL_DATA` object has two keys (`"2026-05"`, `"2026-06"`). Each entry object gets an `id` property set to its `date` value.

For each of the 62 entries, add `"id":"2026-05-01"` (etc.) as the first property. Example pattern (applied to every entry):

```js
// Before:
{"date":"2026-05-01","day":"Friday","sale":300,"note":null,"bestSeller":null,"shopExp":0,"shopExpDetail":null,"otherExp":0,"otherExpDetail":null}

// After:
{"id":"2026-05-01","date":"2026-05-01","day":"Friday","sale":300,"note":null,"bestSeller":null,"shopExp":0,"shopExpDetail":null,"otherExp":0,"otherExpDetail":null}
```

Run after editing: `grep -o '"id":"[^"]*"' hafiz_photocopy_dashboard.html | wc -l` should output `62`.

- [ ] **Step 3: Replace hidden field in modal form: `originalDate` → `entryId`**

In the HTML (line 274), change:
```html
<input type="hidden" id="originalDate" value="" />
```
to:
```html
<input type="hidden" id="entryId" value="" />
```

- [ ] **Step 4: Change `openModalForAdd()` to default to today's date**

Replace the function at lines 767–782:

```js
function openModalForAdd() {
  document.getElementById("modalTitle").textContent = "Add a day";
  document.getElementById("deleteBtn").style.display = "none";
  form.reset();
  document.getElementById("entryId").value = "";
  document.getElementById("fDate").value = new Date().toISOString().slice(0, 10);
  formMsg.textContent = "";
  backdrop.classList.add("open");
}
```

Verification: Open "Add day" modal — date field shows `2026-07-04`.

- [ ] **Step 5: Rewrite `openModalForEdit()` to accept an `id` and look up across all months**

Replace the function at lines 784–801:

```js
function openModalForEdit(id) {
  const entry = findEntryById(id);
  if (!entry) return;
  document.getElementById("modalTitle").textContent = "Edit entry";
  document.getElementById("deleteBtn").style.display = "inline-block";
  document.getElementById("entryId").value = entry.id;
  document.getElementById("fDate").value = entry.date;
  document.getElementById("fSale").value = entry.sale || "";
  document.getElementById("fNote").value = entry.note || "";
  document.getElementById("fBestSeller").value = entry.bestSeller || "";
  document.getElementById("fShopExp").value = entry.shopExp || "";
  document.getElementById("fShopExpDetail").value = entry.shopExpDetail || "";
  document.getElementById("fOtherExp").value = entry.otherExp || "";
  document.getElementById("fOtherExpDetail").value = entry.otherExpDetail || "";
  formMsg.textContent = "";
  backdrop.classList.add("open");
}
```

- [ ] **Step 6: Rewrite `upsertDay()` for ID-based lookup**

Replace the function at lines 411–434:

```js
async function upsertDay(entry, originalId) {
  const monthKey = entry.date.slice(0, 7);
  if (!state.months.includes(monthKey)) {
    state.months.push(monthKey);
    state.months.sort();
    await persistMonthsList();
  }
  if (!state.dataByMonth[monthKey]) state.dataByMonth[monthKey] = [];

  // If editing and date changed, remove entry from old month
  if (originalId) {
    const oldEntry = findEntryById(originalId);
    if (oldEntry && oldEntry.date !== entry.date) {
      const oldMonth = oldEntry.date.slice(0, 7);
      if (state.dataByMonth[oldMonth]) {
        state.dataByMonth[oldMonth] = state.dataByMonth[oldMonth].filter(d => d.id !== originalId);
        await persistMonth(oldMonth);
      }
    }
  }

  const arr = state.dataByMonth[monthKey];
  if (originalId) {
    // Update existing entry — preserve the original ID
    const idx = arr.findIndex(d => d.id === originalId);
    entry.id = originalId;
    if (idx >= 0) arr[idx] = entry;
    else arr.push(entry);
  } else {
    // New entry
    if (!entry.id) entry.id = generateId();
    arr.push(entry);
  }
  arr.sort((a, b) => a.date.localeCompare(b.date));
  await persistMonth(monthKey);
}
```

- [ ] **Step 7: Rewrite `deleteDay()` to accept an `id` and search across all months**

Replace the function at lines 436–441:

```js
async function deleteDay(id) {
  for (const monthKey of state.months) {
    const arr = state.dataByMonth[monthKey];
    if (arr) {
      const idx = arr.findIndex(d => d.id === id);
      if (idx >= 0) {
        arr.splice(idx, 1);
        await persistMonth(monthKey);
        return;
      }
    }
  }
}
```

- [ ] **Step 8: Update modal form submit handler and delete button**

In the form submit handler (line 819, around `const orig = ...`):

```js
// Change this line (834):
const orig = document.getElementById("originalDate").value || null;

// To:
const orig = document.getElementById("entryId").value || null;
```

In the delete button click handler (line 809):

```js
// Change this line (810):
const orig = document.getElementById("originalDate").value;

// To:
const orig = document.getElementById("entryId").value;
```

- [ ] **Step 9: Call `migrateEntries()` during boot and persist migrated data**

In the boot IIFE (lines 856–860), add migration and persist after `initStorage()`:

```js
(async function boot() {
  await initStorage();
  if (migrateEntries()) {
    // Re-persist all months with newly-added IDs
    for (const m of state.months) {
      await persistMonth(m);
    }
  }
  state.selectedScope = state.months.length ? state.months[state.months.length - 1] : "all";
  renderAll();
})();
```

Verification: Load the page, open browser console, run `state.dataByMonth["2026-05"][0].id` — should return `"2026-05-01"`.

---

## Task 2: Rewrite `renderTable()` for Grouped-by-Date Display

**Files:**
- Modify: `hafiz_photocopy_dashboard.html` — the `renderTable()` function, table-related CSS, click handlers

- [ ] **Step 1: Add CSS for date-header row, entry-row, and entry-detail-row**

Insert after the `.note-pill` rule (~line 155):

```css
.date-header{ cursor:pointer; font-weight:600; }
.date-header:hover{ background:#F0F4F8; }
.entry-row td{ background:#F8FAFB; padding:12px 18px 12px 36px; }
.entry-row .entry-detail-row{ margin-bottom:8px; display:flex; gap:16px; flex-wrap:wrap; }
.entry-row .entry-detail-row .num{ font-size:12.5px; }
.entry-row .detail-grid{ margin-top:6px; }
```

- [ ] **Step 2: Replace the entire `renderTable()` function**

Replace the function at lines 689–745 with the grouped-by-date version:

```js
function renderTable() {
  const tbody = document.getElementById("tableBody");
  const sub = document.getElementById("tableSub");
  const days = [...scopedDays()].sort((a, b) => b.date.localeCompare(a.date));
  sub.textContent = days.length + " entries — tap a date for details";

  if (days.length === 0) {
    tbody.innerHTML = '<tr><td colspan="6" class="empty-note">No entries yet. Use "+ Add day" to get started.</td></tr>';
    return;
  }

  // Group entries by date, preserving reverse chronological order
  const groups = {};
  days.forEach(d => {
    if (!groups[d.date]) groups[d.date] = [];
    groups[d.date].push(d);
  });
  const dateKeys = Object.keys(groups).sort().reverse();

  let html = "";
  dateKeys.forEach(date => {
    const entries = groups[date];
    const first = entries[0];
    const totalSale = entries.reduce((s, d) => s + (d.sale || 0), 0);
    const totalShop = entries.reduce((s, d) => s + (d.shopExp || 0), 0);
    const totalOther = entries.reduce((s, d) => s + (d.otherExp || 0), 0);
    const totalNet = totalSale - totalShop - totalOther;

    // Date header row — shows combined totals
    html += `<tr class="date-header" data-group-date="${date}">
      <td class="mono">${date}</td>
      <td><span class="day-pill">${first.day || ""}</span>${first.note ? ' <span class="note-pill">' + escapeHtml(first.note) + '</span>' : ""}</td>
      <td class="num" style="text-align:right;">${fmtMoney(totalSale)}</td>
      <td class="num" style="text-align:right;">${fmtMoney(totalShop)}</td>
      <td class="num" style="text-align:right;">${fmtMoney(totalOther)}</td>
      <td class="num" style="text-align:right;"><span class="${totalNet >= 0 ? 'pos' : 'neg'}">${fmtMoney(totalNet)}</span></td>
    </tr>`;

    // Sub-rows — one per entry, hidden by default
    entries.forEach(d => {
      const net = (d.sale || 0) - (d.shopExp || 0) - (d.otherExp || 0);
      html += `<tr class="entry-row" data-group-parent="${date}" data-entry-id="${d.id}" style="display:none;">
      <td colspan="6">
        <div class="entry-detail-row">
          <span class="num">Sale: ${fmtMoney(d.sale)}</span>
          <span class="num">Shop: ${fmtMoney(d.shopExp)}</span>
          <span class="num">Other: ${fmtMoney(d.otherExp)}</span>
          <span class="num ${net >= 0 ? 'pos' : 'neg'}">Net: ${fmtMoney(net)}</span>
        </div>
        <div class="detail-grid">
          <div class="detail-item"><b>Note:</b> ${d.note ? escapeHtml(d.note) : "—"}</div>
          <div class="detail-item"><b>Best seller:</b> ${d.bestSeller ? escapeHtml(d.bestSeller) : "—"}</div>
          <div class="detail-item"><b>Shop expense detail:</b> ${d.shopExpDetail ? escapeHtml(String(d.shopExpDetail)) : "—"}</div>
          <div class="detail-item"><b>Other expense detail:</b> ${d.otherExpDetail ? escapeHtml(String(d.otherExpDetail)) : "—"}</div>
        </div>
        <div class="row-actions">
          <button class="link-btn" data-edit="${d.id}">Edit entry</button>
          <button class="link-btn danger" data-delete="${d.id}">Delete entry</button>
        </div>
      </td>
    </tr>`;
    });
  });

  tbody.innerHTML = html;

  // Clicking a date-header row toggles its sub-rows
  tbody.querySelectorAll("tr.date-header").forEach(row => {
    row.addEventListener("click", () => {
      const date = row.getAttribute("data-group-date");
      const subRows = tbody.querySelectorAll(`tr.entry-row[data-group-parent="${date}"]`);
      const isHidden = subRows.length === 0 || subRows[0].style.display === "none";
      subRows.forEach(r => r.style.display = isHidden ? "table-row" : "none");
    });
  });

  // Edit and delete buttons inside sub-rows
  tbody.querySelectorAll("[data-edit]").forEach(btn => {
    btn.addEventListener("click", (e) => { e.stopPropagation(); openModalForEdit(btn.getAttribute("data-edit")); });
  });
  tbody.querySelectorAll("[data-delete]").forEach(btn => {
    btn.addEventListener("click", async (e) => {
      e.stopPropagation();
      if (!confirm("Delete this entry? This can't be undone.")) return;
      await deleteDay(btn.getAttribute("data-delete"));
      renderAll();
      showToast("Entry deleted");
    });
  });
}
```

- [ ] **Step 3: Update table sub-header text**

Change line 693 from:
```js
sub.textContent = days.length + " entries — tap a row for detail";
```
to:
```js
sub.textContent = days.length + " entries — tap a date for details";
```

Verification: Load page → table shows date-header rows with combined totals. Click a date header → sub-rows expand with individual entries. Click "Edit entry" on a sub-row → modal opens with that entry's data. Click "Delete entry" → confirms, deletes, refreshes.

---

## Task 3: Password Protection with Auth Overlay

**Files:**
- Modify: `hafiz_photocopy_dashboard.html` — CSS (add auth overlay styles), HTML (add overlay markup), JS (add auth functions, modify boot flow)

- [ ] **Step 1: Add auth overlay CSS**

Insert before the closing `</style>` tag (line 203):

```css
/* Auth overlay */
#authOverlay{
  position:fixed; inset:0; z-index:100;
  background:rgba(15,23,32,0.7);
  backdrop-filter:blur(8px); -webkit-backdrop-filter:blur(8px);
  display:flex; align-items:center; justify-content:center;
  font-family:'Inter',system-ui,-apple-system,sans-serif;
}
#authOverlay.hidden{ display:none; }
.auth-card{
  background:#fff; border-radius:16px; padding:32px; width:100%; max-width:380px;
  box-shadow:0 24px 60px rgba(0,0,0,0.3); text-align:center;
}
.auth-card .shop-icon{ font-size:36px; margin-bottom:8px; }
.auth-card h2{ font-family:'Space Grotesk',sans-serif; font-size:20px; margin:0 0 4px; color:var(--ink); }
.auth-card .sub{ font-size:13px; color:var(--muted); margin:0 0 20px; }
.auth-card input{
  width:100%; padding:11px 12px; border-radius:9px; border:1px solid var(--border);
  font-size:16px; text-align:center; font-family:'Inter',sans-serif;
  letter-spacing:6px; background:#F9FAFC; margin-bottom:12px; box-sizing:border-box;
}
.auth-card input:focus{ outline:2px solid var(--primary); background:#fff; border-color:var(--primary); }
.auth-card .auth-err{ font-size:12px; color:var(--negative); min-height:18px; margin-bottom:8px; }
.auth-card .btn-solid{ width:100%; padding:11px; font-size:15px; }
```

- [ ] **Step 2: Add auth overlay HTML**

Insert before `<div class="toast" id="toast"></div>` (line 325):

```html
<div id="authOverlay">
  <div class="auth-card">
    <div class="shop-icon">🖨️</div>
    <h2>HaFiz PhotoCopy</h2>
    <p class="sub" id="authSub">Enter password to access the dashboard</p>
    <input type="password" id="authInput" maxlength="10" placeholder="Password" autocomplete="off" />
    <div class="auth-err" id="authErr"></div>
    <button class="btn-solid" id="authBtn">Unlock</button>
  </div>
</div>
```

- [ ] **Step 3: Add auth JavaScript functions**

Insert before `// ---------- Storage layer ----------` (line 361):

```js
// ---------- Authentication ----------
const AUTH_STORAGE_KEY = 'hafiz_auth_hash';

async function hashPassword(password) {
  const encoder = new TextEncoder();
  const data = encoder.encode(password);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}

async function checkAuth() {
  // Session already authenticated this browser tab
  if (sessionStorage.getItem('hafiz_authenticated') === 'true') return true;
  const storedHash = localStorage.getItem(AUTH_STORAGE_KEY);
  // First visit — no password set yet
  if (!storedHash) return 'setup';
  return false;
}

async function handleAuthSubmit() {
  const input = document.getElementById('authInput');
  const err = document.getElementById('authErr');
  const sub = document.getElementById('authSub');
  const btn = document.getElementById('authBtn');
  const password = input.value.trim();
  if (!password) { err.textContent = 'Enter the password.'; return; }
  err.textContent = '';
  btn.disabled = true;
  btn.textContent = 'Checking…';

  const storedHash = localStorage.getItem(AUTH_STORAGE_KEY);
  if (!storedHash) {
    // First visit — set password
    if (password !== '2854') {
      err.textContent = 'Wrong password. Try again.';
      btn.disabled = false;
      btn.textContent = 'Unlock';
      return;
    }
    const hash = await hashPassword(password);
    localStorage.setItem(AUTH_STORAGE_KEY, hash);
    sessionStorage.setItem('hafiz_authenticated', 'true');
    document.getElementById('authOverlay').classList.add('hidden');
    initApp();
  } else {
    // Verify password
    const hash = await hashPassword(password);
    if (hash === storedHash) {
      sessionStorage.setItem('hafiz_authenticated', 'true');
      document.getElementById('authOverlay').classList.add('hidden');
      initApp();
    } else {
      err.textContent = 'Wrong password. Try again.';
      btn.disabled = false;
      btn.textContent = 'Unlock';
      input.value = '';
      input.focus();
    }
  }
}
```

- [ ] **Step 4: Wire up auth event handlers**

Insert after the `handleAuthSubmit` function:

```js
document.getElementById('authBtn').addEventListener('click', handleAuthSubmit);
document.getElementById('authInput').addEventListener('keydown', (e) => {
  if (e.key === 'Enter') handleAuthSubmit();
});
```

- [ ] **Step 5: Modify boot flow**

Replace the boot IIFE (lines 856–860) with an auth-gated boot:

```js
// ---------- Boot ----------
(async function boot() {
  const authStatus = await checkAuth();
  if (authStatus === 'setup') {
    document.getElementById('authSub').textContent = 'Set your password to secure the dashboard';
    document.getElementById('authBtn').textContent = 'Set Password';
    document.getElementById('authOverlay').classList.remove('hidden');
    document.getElementById('authInput').focus();
    return; // wait for password setup
  }
  if (!authStatus) {
    document.getElementById('authOverlay').classList.remove('hidden');
    document.getElementById('authInput').focus();
    return; // wait for password entry
  }
  // Authenticated — proceed
  initApp();
})();

async function initApp() {
  await initStorage();
  if (migrateEntries()) {
    for (const m of state.months) {
      await persistMonth(m);
    }
  }
  state.selectedScope = state.months.length ? state.months[state.months.length - 1] : "all";
  renderAll();
}
```

- [ ] **Step 6: Add Logout and Settings buttons to header**

In the header actions div (line 215–219), change:

```html
<div class="header-actions">
  <select id="monthSelect"></select>
  <button class="btn btn-primary" id="addDayBtn">+ Add day</button>
</div>
```

to:

```html
<div class="header-actions">
  <select id="monthSelect"></select>
  <button class="btn btn-primary" id="addDayBtn">+ Add day</button>
  <button class="btn" id="settingsBtn" style="font-size:16px; line-height:1; padding:6px 10px;" title="Change password">⚙️</button>
  <button class="btn" id="logoutBtn" style="font-size:16px; line-height:1; padding:6px 10px;" title="Lock dashboard">🔒</button>
</div>
```

- [ ] **Step 7: Wire up Logout and Settings handlers**

Insert after the `monthSelect` change handler (line 850–853):

```js
// Logout — clears session only, password hash stays in localStorage
document.getElementById('logoutBtn').addEventListener('click', () => {
  sessionStorage.removeItem('hafiz_authenticated');
  location.reload();
});

// Settings — change password
document.getElementById('settingsBtn').addEventListener('click', () => {
  const current = prompt('Current password:');
  if (!current) return;
  hashPassword(current).then(currentHash => {
    if (currentHash !== localStorage.getItem(AUTH_STORAGE_KEY)) {
      showToast('Wrong password');
      return;
    }
    const newPw = prompt('New password (default: 2854):') || '2854';
    if (newPw.length < 4) { showToast('Password must be at least 4 characters'); return; }
    hashPassword(newPw).then(newHash => {
      localStorage.setItem(AUTH_STORAGE_KEY, newHash);
      showToast('Password changed');
    });
  });
});
```

Verification: Hard-reload the page → auth overlay appears. First visit shows "Set your password" — enter `2854` → dashboard loads. Close and reopen page → "Enter password" prompt appears → enter `2854` → dashboard loads. Enter wrong password → error message. Click 🔒 → page reloads → auth prompt. Click ⚙️ → change password flow.

---

## Task 4: Google Drive Storage

**Files:**
- Modify: `hafiz_photocopy_dashboard.html` — add GIS CDN script, `CONFIG` object, Drive API functions, rewrite storage layer

- [ ] **Step 1: Add GIS CDN script to `<head>`**

Insert after the Google Fonts link (line 8):

```html
<script src="https://accounts.google.com/gsi/client" async defer></script>
```

- [ ] **Step 2: Add Drive config object**

Insert before `const INITIAL_DATA` (line 328) — inside the `<script>` tag:

```js
// ---------- Google Drive configuration ----------
const CONFIG = {
  CLIENT_ID: 'YOUR_GOOGLE_CLIENT_ID.apps.googleusercontent.com',
  FILE_NAME: 'hafiz_photocopy_dashboard_data.json',
  SCOPE: 'https://www.googleapis.com/auth/drive.file',
};
```

- [ ] **Step 3: Add Drive API functions**

Insert after the `persistMonthsList()` function (line 409) — before `upsertDay()`:

```js
// ---------- Google Drive storage engine ----------
let driveFileId = localStorage.getItem('driveFileId');
let driveTokenClient = null;

function getDriveApiBase() {
  return 'https://www.googleapis.com/drive/v3';
}

function getDriveUploadBase() {
  return 'https://www.googleapis.com/upload/drive/v3';
}

async function driveRequest(url, options = {}) {
  const token = driveTokenClient;
  if (!token) throw new Error('Not authenticated');
  const headers = { 'Authorization': 'Bearer ' + token, ...options.headers };
  const res = await fetch(url, { ...options, headers });
  if (!res.ok) {
    const text = await res.text();
    throw new Error('Drive API error: ' + res.status + ' ' + text);
  }
  return res.status === 204 ? null : res.json();
}

function getAccessToken() {
  return new Promise((resolve, reject) => {
    const stored = localStorage.getItem('driveAccessToken');
    const expires = localStorage.getItem('driveTokenExpires');
    if (stored && expires && Date.now() < parseInt(expires, 10)) {
      resolve(stored);
      return;
    }
    // GIS token client will handle refresh
    try {
      window.google?.accounts?.oauth2?.initTokenClient({
        client_id: CONFIG.CLIENT_ID,
        scope: CONFIG.SCOPE,
        callback: (resp) => {
          if (resp.access_token) {
            localStorage.setItem('driveAccessToken', resp.access_token);
            localStorage.setItem('driveTokenExpires', String(Date.now() + (resp.expires_in || 3600) * 1000));
            resolve(resp.access_token);
          } else {
            reject(new Error('OAuth failed'));
          }
        },
        error_callback: (err) => reject(err),
      }).requestAccessToken();
    } catch (e) {
      reject(e);
    }
  });
}

async function initGoogleDrive() {
  try {
    const token = await getAccessToken();
    driveTokenClient = token;
    return true;
  } catch (e) {
    console.warn('Google Drive auth failed', e);
    return false;
  }
}

async function findDriveFile() {
  const q = encodeURIComponent(`name='${CONFIG.FILE_NAME}' and trashed=false`);
  const res = await driveRequest(`${getDriveApiBase()}/files?q=${q}&fields=files(id,name)`);
  return res?.files?.length ? res.files[0].id : null;
}

async function readDriveFile(fileId) {
  const res = await driveRequest(`${getDriveApiBase()}/files/${fileId}?alt=media`);
  return res;
}

async function createDriveFile(data) {
  const metadata = { name: CONFIG.FILE_NAME, mimeType: 'application/json' };
  const form = new FormData();
  form.append('metadata', new Blob([JSON.stringify(metadata)], { type: 'application/json' }));
  form.append('file', new Blob([JSON.stringify(data)], { type: 'application/json' }));
  const res = await driveRequest(`${getDriveUploadBase()}/files?uploadType=multipart&fields=id`, {
    method: 'POST',
    body: form,
  });
  driveFileId = res.id;
  localStorage.setItem('driveFileId', res.id);
  return res.id;
}

async function updateDriveFile(fileId, data) {
  const res = await driveRequest(`${getDriveUploadBase()}/files/${fileId}?uploadType=media`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  return res;
}

function serializeState() {
  return {
    version: 2,
    updatedAt: new Date().toISOString(),
    months: state.months,
    dataByMonth: state.dataByMonth,
  };
}

function cacheToLocalStorage(data) {
  localStorage.setItem('driveCache', JSON.stringify(data));
  localStorage.setItem('driveCacheTime', String(Date.now()));
}

function readFromLocalStorageCache() {
  const cached = localStorage.getItem('driveCache');
  if (cached) {
    try { return JSON.parse(cached); } catch (e) { /* ignore */ }
  }
  return null;
}

async function writeStateToDrive() {
  const data = serializeState();
  cacheToLocalStorage(data);
  try {
    if (driveFileId) {
      await updateDriveFile(driveFileId, data);
    } else {
      const id = await findDriveFile();
      if (id) {
        driveFileId = id;
        localStorage.setItem('driveFileId', id);
        await updateDriveFile(id, data);
      } else {
        await createDriveFile(data);
      }
    }
    return true;
  } catch (e) {
    console.warn('Drive write failed, saved to cache only', e);
    return false;
  }
}

async function readStateFromDrive() {
  try {
    let fileId = driveFileId;
    if (!fileId) fileId = await findDriveFile();
    if (!fileId) return null;
    driveFileId = fileId;
    localStorage.setItem('driveFileId', fileId);
    const data = await readDriveFile(fileId);
    if (data && data.version && data.months && data.dataByMonth) {
      return data;
    }
    return null;
  } catch (e) {
    console.warn('Drive read failed', e);
    return null;
  }
}

function driveSignIn() {
  if (!window.google) { showToast('Google Identity Services not loaded'); return; }
  const client = window.google.accounts.oauth2.initTokenClient({
    client_id: CONFIG.CLIENT_ID,
    scope: CONFIG.SCOPE,
    callback: (resp) => {
      if (resp.access_token) {
        localStorage.setItem('driveAccessToken', resp.access_token);
        localStorage.setItem('driveTokenExpires', String(Date.now() + (resp.expires_in || 3600) * 1000));
        driveTokenClient = resp.access_token;
        updateDriveStatus(true);
        showToast('Connected to Google Drive');
        initStorage(); // re-init with Drive access
      }
    },
  });
  client.requestAccessToken({ prompt: 'consent' });
}

function driveSignOut() {
  localStorage.removeItem('driveAccessToken');
  localStorage.removeItem('driveTokenExpires');
  localStorage.removeItem('driveFileId');
  driveTokenClient = null;
  driveFileId = null;
  updateDriveStatus(false);
  showToast('Signed out from Google Drive');
}

function updateDriveStatus(connected) {
  const el = document.getElementById('liveStatus');
  if (connected) {
    el.innerHTML = '<span style="display:inline-block;width:6px;height:6px;border-radius:50%;background:#6EE7B7;box-shadow:0 0 0 3px rgba(110,231,183,0.25);margin-right:6px;"></span>Connected to Google Drive';
  } else if (localStorage.getItem('driveAccessToken')) {
    el.innerHTML = '<span style="display:inline-block;width:6px;height:6px;border-radius:50%;background:#F59E0B;box-shadow:0 0 0 3px rgba(245,158,11,0.25);margin-right:6px;"></span>Google Drive: reconnecting… <button class="link-btn" style="color:#fff;font-size:11px;" id="driveSignInBtn">Sign in</button>';
    setTimeout(() => {
      const btn = document.getElementById('driveSignInBtn');
      if (btn) btn.onclick = driveSignIn;
    }, 0);
  } else {
    el.innerHTML = '<span style="display:inline-block;width:6px;height:6px;border-radius:50%;background:#9CA3AF;box-shadow:0 0 0 3px rgba(156,163,175,0.25);margin-right:6px;"></span>Sign in to Google Drive <button class="link-btn" style="color:#fff;font-size:11px;" id="driveSignInBtn">Sign in</button>';
    setTimeout(() => {
      const btn = document.getElementById('driveSignInBtn');
      if (btn) btn.onclick = driveSignIn;
    }, 0);
  }
}
```

- [ ] **Step 4: Rewrite `initStorage()` for Drive-first with fallback**

Replace the entire `initStorage()` function (lines 362–398):

```js
async function initStorage() {
  const statusEl = document.getElementById('liveStatus');

  // Try Google Drive first if we have a token
  const token = localStorage.getItem('driveAccessToken');
  if (token) {
    driveTokenClient = token;
    const driveData = await readStateFromDrive();
    if (driveData) {
      state.months = driveData.months;
      state.dataByMonth = driveData.dataByMonth;
      migrateEntries();
      state.loaded = true;
      updateDriveStatus(true);
      return;
    }
  }

  // Fallback: try localStorage cache
  const cached = readFromLocalStorageCache();
  if (cached && cached.months && cached.dataByMonth) {
    state.months = cached.months;
    state.dataByMonth = cached.dataByMonth;
    migrateEntries();
    state.loaded = true;
    statusEl.textContent = 'Offline mode (cached data) — sign in to Google Drive for persistence';
    updateDriveStatus(false);
    return;
  }

  // Last resort: window.storage (legacy platform API) or INITIAL_DATA
  try {
    let monthsResult;
    try { monthsResult = await window.storage.get('months', true); } catch (e) { monthsResult = null; }
    if (monthsResult && monthsResult.value) {
      state.months = JSON.parse(monthsResult.value);
      for (const m of state.months) {
        try {
          const r = await window.storage.get('days:' + m, true);
          state.dataByMonth[m] = r && r.value ? JSON.parse(r.value) : (INITIAL_DATA[m] || []);
        } catch (e) {
          state.dataByMonth[m] = INITIAL_DATA[m] || [];
        }
      }
    } else {
      state.months = Object.keys(INITIAL_DATA).sort();
      state.dataByMonth = JSON.parse(JSON.stringify(INITIAL_DATA));
    }
    statusEl.textContent = 'Loaded from shared storage — sign in to Google Drive for persistence';
  } catch (e) {
    console.error('Storage init failed, using INITIAL_DATA', e);
    state.months = Object.keys(INITIAL_DATA).sort();
    state.dataByMonth = JSON.parse(JSON.stringify(INITIAL_DATA));
    statusEl.textContent = 'Offline mode — changes may not persist';
  }

  updateDriveStatus(false);
  state.loaded = true;
}
```

- [ ] **Step 5: Rewrite `persistMonth()` and `persistMonthsList()` to use Drive**

Replace both functions (lines 400–409):

```js
async function persistMonth(monthKey) {
  // Legacy: try window.storage as secondary fallback
  try { await window.storage.set('days:' + monthKey, JSON.stringify(state.dataByMonth[monthKey]), true); } catch (e) { /* ignore */ }
  // Primary: write full state to Drive + cache
  await writeStateToDrive();
}

async function persistMonthsList() {
  try { await window.storage.set('months', JSON.stringify(state.months), true); } catch (e) { /* ignore */ }
  await writeStateToDrive();
}
```

Verification: With a valid Google Client ID configured, load page → click "Sign in" → Google OAuth flow → dashboard loads with Drive data. Add an entry → check Google Drive for JSON file. Remove token → page still loads from localStorage cache.

---

## Task 5: Backup, Restore, and Excel Export

**Files:**
- Modify: `hafiz_photocopy_dashboard.html` — add SheetJS CDN, export/import functions, header buttons, footer

- [ ] **Step 1: Add SheetJS CDN to `<head>`**

Insert after the GIS script tag:

```html
<script src="https://cdn.sheetjs.com/xlsx-0.20.0/package/dist/xlsx.full.min.js"></script>
```

- [ ] **Step 2: Add export/import functions**

Insert after `renderAll()` (before the modal section, around line 760):

```js
// ---------- Export / Import ----------
function exportExcel() {
  if (typeof XLSX === 'undefined') { showToast('Excel library not loaded'); return; }
  const days = scopedDays();
  if (days.length === 0) { showToast('No data to export'); return; }

  // Sheet 1: Daily Sales
  const dailyRows = days.sort((a, b) => a.date.localeCompare(b.date)).map(d => ({
    Date: d.date,
    Day: d.day,
    Sale: d.sale || 0,
    Note: d.note || '',
    'Best Seller': d.bestSeller || '',
    'Shop Expense': d.shopExp || 0,
    'Shop Expense Detail': d.shopExpDetail || '',
    'Other Expense': d.otherExp || 0,
    'Other Expense Detail': d.otherExpDetail || '',
    'Net Profit': (d.sale || 0) - (d.shopExp || 0) - (d.otherExp || 0),
  }));

  // Sheet 2: Monthly Summary
  const monthlyRows = state.months.map(m => {
    const t = monthTotals(m);
    return {
      Month: monthLabel(m),
      'Total Sales': t.sale,
      'Total Shop Expenses': t.shopExp,
      'Total Other Expenses': t.otherExp,
      'Net Profit': t.net,
    };
  });

  const wb = XLSX.utils.book_new();
  wb.SheetNames.push('Daily Sales');
  wb.Sheets['Daily Sales'] = XLSX.utils.json_to_sheet(dailyRows);
  wb.SheetNames.push('Monthly Summary');
  wb.Sheets['Monthly Summary'] = XLSX.utils.json_to_sheet(monthlyRows);

  const dateStr = new Date().toISOString().slice(0, 10);
  XLSX.writeFile(wb, 'hafiz_photocopy_' + dateStr + '.xlsx');
  showToast('Excel downloaded');
}

function backupData() {
  const data = JSON.stringify(serializeState(), null, 2);
  const blob = new Blob([data], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = 'hafiz_photocopy_backup_' + new Date().toISOString().slice(0, 10) + '.json';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
  showToast('Backup downloaded');
}

function restoreData(file) {
  const reader = new FileReader();
  reader.onload = async (e) => {
    try {
      const data = JSON.parse(e.target.result);
      if (!data.months || !data.dataByMonth || !data.version) {
        showToast('Invalid backup file');
        return;
      }
      if (!confirm('Replace all current data with the backup? This cannot be undone.')) return;
      state.months = data.months;
      state.dataByMonth = data.dataByMonth;
      migrateEntries();
      await writeStateToDrive();
      renderAll();
      showToast('Data restored from backup');
    } catch (err) {
      showToast('Could not parse backup file');
    }
  };
  reader.readAsText(file);
}

function triggerRestore() {
  const input = document.createElement('input');
  input.type = 'file';
  input.accept = '.json';
  input.onchange = (e) => {
    if (e.target.files[0]) restoreData(e.target.files[0]);
  };
  input.click();
}
```

- [ ] **Step 3: Add header action buttons**

In the header actions div (line 215–219), replace with:

```html
<div class="header-actions">
  <select id="monthSelect"></select>
  <button class="btn btn-primary" id="addDayBtn">+ Add day</button>
  <button class="btn" id="excelBtn" title="Export to Excel">📊 Excel</button>
  <button class="btn" id="backupBtn" title="Download JSON backup">💾 Backup</button>
  <button class="btn" id="restoreBtn" title="Restore from backup">📂 Restore</button>
  <button class="btn" id="settingsBtn" style="font-size:16px; line-height:1; padding:6px 10px;" title="Change password">⚙️</button>
  <button class="btn" id="logoutBtn" style="font-size:16px; line-height:1; padding:6px 10px;" title="Lock dashboard">🔒</button>
</div>
```

- [ ] **Step 4: Wire up export/import button handlers**

Insert after the settings/logout handlers (added in Task 3 Step 7):

```js
document.getElementById('excelBtn').addEventListener('click', exportExcel);
document.getElementById('backupBtn').addEventListener('click', backupData);
document.getElementById('restoreBtn').addEventListener('click', triggerRestore);
```

- [ ] **Step 5: Update footer text**

Replace line 266:

```html
<p class="footer-note">Data is shared — anyone with this link can add or edit entries. Numbers in Rs.</p>
```

with:

```html
<p class="footer-note">HaFiz PhotoCopy — Sales Dashboard. Numbers in Pakistani Rupees (Rs).</p>
```

Verification: Click "📊 Excel" → `.xlsx` file downloads with correct data in two sheets. Click "💾 Backup" → `.json` file downloads. Click "📂 Restore" → file picker opens → select a backup JSON → confirmation prompt → data loads. Footer text updated.

---

## Self-Review

**1. Spec coverage — every feature mapped to a task:**

| Feature | Task(s) |
|---|---|
| Default current date in "Add day" modal | Task 1, Step 4 |
| Multiple entries per day (ID data model) | Task 1, Steps 1–3, 5–9 |
| Grouped-by-date table display | Task 2, Steps 1–3 |
| Password protection (SHA-256) | Task 3, Steps 1–7 |
| Google Drive storage | Task 4, Steps 1–5 |
| Excel export | Task 5, Steps 1–2, 4 |
| JSON backup | Task 5, Steps 2, 4 |
| JSON restore | Task 5, Steps 2, 4 |
| Settings (change password) | Task 3, Step 7 |
| Logout | Task 3, Step 6–7 |
| Drive sign-in/sign-out | Task 4, Steps 3–4 |
| Status indicator (connected/offline) | Task 4, Step 3–4 |

**2. Placeholder scan:** No "TBD", "TODO", "implement later", "fill in details" in the code blocks. Every function body is complete. Every step has exact code.

**3. Type consistency:**

- `generateId()` → returns string — used in Task 1 Steps 1, 6.
- `findEntryById(id)` → returns entry object or null — used in Task 1 Steps 5, 6.
- `migrateEntries()` → returns boolean — used in Task 1 Steps 9, Task 4 Step 4.
- `upsertDay(entry, originalId)` — signature changed from `originalDate` to `originalId` — used in form submit handler (Task 1 Step 8).
- `deleteDay(id)` — signature changed from `dateStr` to `id` — used in Task 2 Step 2 (delete button), Task 1 Step 7.
- `openModalForEdit(id)` — signature changed from `dateStr` to `id` — used in Task 2 Step 2.
- `serializeState()` → Drive-format object — used in Task 4 Steps 3, Task 5 Step 2.
- `writeStateToDrive()` — called in `persistMonth`, `persistMonthsList`, `restoreData`.
- All DOM element IDs used in JS match the HTML (`entryId`, `authOverlay`, `authInput`, `authSub`, `authErr`, `authBtn`, `excelBtn`, `backupBtn`, `restoreBtn`, `settingsBtn`, `logoutBtn`, `driveSignInBtn`).

No inconsistencies found across tasks.

---

## Execution Handoff

Plan complete. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

**Which approach?**

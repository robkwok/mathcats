# MathCats: Player Accounts & Firebase Backend

## Context
MathCats is a single-file (`index.html`) kindergarten math game currently using localStorage for star persistence. We need to add player profiles so multiple kids can share a classroom iPad, with data synced to Firebase so stars survive browser data clears. The game will be deployed to GitHub Pages (static hosting, no server).

## Approach Summary
- **Firebase Anonymous Auth** signs in silently per device (no kid-facing login)
- **Firestore** stores profiles as `devices/{uid}/profiles/{id}`
- **Profile picker screen** is the new entry point — kids tap their avatar to play
- **Create profile screen** lets kids pick a name + cat avatar
- **localStorage is the source of truth for UI**; Firestore syncs in the background
- Stars only go up, so merge conflicts resolve with `max(local, remote)`
- **Game works fully offline** — if Firebase fails to load, localStorage-only mode keeps everything playable

---

## Step 1: Firebase Project Setup (manual — user does this)
1. Create project at console.firebase.google.com ("mathcats", no Analytics)
2. Enable **Authentication > Anonymous** sign-in
3. Create **Firestore Database** in production mode
4. Register a **Web App** and copy the `firebaseConfig` object
5. Add GitHub Pages domain to **Auth > Authorized domains**
6. Paste security rules (provided below)

## Step 2: Add Firebase SDK to `index.html`
**Critical: load order must guarantee `firebase` is defined before the inline game script runs.**

- Add three `<script>` tags (NO `defer`) immediately before the inline `<script>` block at the bottom of `<body>`, in this order:
  1. `firebase-app-compat.js`
  2. `firebase-auth-compat.js`
  3. `firebase-firestore-compat.js`
- Add Firebase config + `firebase.initializeApp()` at the top of the main inline `<script>` block
- Wrap all Firebase access in a `safeDb()` helper that returns `null` if `typeof firebase === 'undefined'` (handles ad-blockers, offline CDN, etc.)
- Call `firebase.auth().signInAnonymously()` on load; store the UID in `deviceUid` and in `localStorage['mathcats_device_uid']`

**Loading overlay**: show a full-screen "Cats are waking up..." spinner on initial app load; dismiss once `onAuthStateChanged` fires OR after a 2-second timeout (whichever comes first) — don't let slow/blocked Firebase block the UI.

## Step 3: Profile Data Layer

### State variables (replaces the global `totalStars`)
```js
var currentProfile = null;   // { id, name, avatar, totalStars }
var profiles = [];           // array of profile objects
var db = null;               // Firestore handle or null
var deviceUid = null;        // Firebase UID or localStorage-generated UUID fallback
```

**Key refactor**: every read/write of `totalStars` must go through `currentProfile.totalStars`. Touchpoints in existing code:
- `index.html:820` — initial load (remove, now done via `selectProfile`)
- `index.html:952-953` — streak bonus in `checkAnswer()`
- `index.html:1049-1050` — `endGame()` earned stars
- `index.html:1032-1035` — `updateScoreDisplay()`
- `index.html:1081-1109` — `renderCollection()` unlock check
- `index.html:1169-1171` — `saveStars()` (full rewrite)

Add a helper `getCurrentStars()` and `setCurrentStars(n)` so we have one place to update both the in-memory profile and trigger saves.

### Firestore data model
```
devices/{deviceUid}/profiles/{profileId}
  name: "Emma"        (string, 1-12 chars)
  avatar: "😻"        (string, must be in allowed CAT_EMOJIS subset)
  totalStars: 42      (int, can only increase)
  createdAt: timestamp
  lastPlayedAt: timestamp
```

### New functions
- `loadProfiles()` — fetch from Firestore, merge with localStorage using `max(localStars, remoteStars)`, cache result
- `createProfile(name, avatar)` — generate `crypto.randomUUID()` id, write to Firestore + localStorage. **Debounce submit** to prevent double-tap duplicates.
- `selectProfile(id)` — set `currentProfile`, load stars, **reset `score`, `streak`, `questionNum`, `answering`**, go to title screen. Must call `updateScoreDisplay()` and `updateGreeting()`.
- `deleteProfile(id)` — remove from Firestore + localStorage. **Rules**:
  - Cannot delete the last profile (UI disables button, rule not enforced server-side)
  - If deleting the active profile, exit to profile picker first
  - Requires long-press (hold 1s) confirmation — no kid-friendly accidental delete
- `saveProfilesToLocalStorage()` / `loadProfilesFromLocalStorage()` — JSON serialize/deserialize
- `safeDb()` — returns Firestore instance or null if Firebase unavailable
- `migrateOldData()` — runs once on first load; documented below

### localStorage schema (explicit)
| Key | Value | Purpose |
|---|---|---|
| `mathcats_profiles_v1` | JSON array of profile objects | cached profile list |
| `mathcats_current_profile_id` | string | last-selected profile ID (auto-select on return) |
| `mathcats_device_uid` | string | stable device ID (Firebase UID or fallback UUID) |
| `mathcats_migrated_v1` | `"true"` | one-shot flag so migration doesn't re-run |
| `mathcats_stars` | *deleted after migration* | legacy key from v0 |

### `saveStars()` rewrite
```js
function saveStars() {
  if (!currentProfile) return;  // guard — shouldn't happen, but safety
  // totalStars is now currentProfile.totalStars (updated at call site)
  saveProfilesToLocalStorage();
  var fs = safeDb();
  if (fs && deviceUid) {
    fs.collection('devices').doc(deviceUid)
      .collection('profiles').doc(currentProfile.id)
      .update({ 
        totalStars: currentProfile.totalStars,
        lastPlayedAt: firebase.firestore.FieldValue.serverTimestamp() 
      })
      .catch(function(err) { console.warn('Firestore save failed:', err); });
  }
}
```
Every caller of `saveStars()` must first update `currentProfile.totalStars` (not a dangling global).

### Migration from v0
On first load, if `mathcats_migrated_v1` is not set:
1. Read `mathcats_stars` (if exists)
2. Create a single profile `{ id: randomUUID(), name: "Player 1", avatar: "🐱", totalStars: <old value> }`
3. Write to localStorage and Firestore
4. Delete `mathcats_stars` key
5. Set `mathcats_migrated_v1 = "true"`
6. Auto-select the new profile and skip profile picker for a seamless first-run

## Step 4: New UI — Profile Picker Screen
**New entry point** (replaces `titleScreen` as the initial `.active` screen).

- Title: "Who's Playing?" in existing gradient style
- Grid of large tappable profile cards (~120x140px min for kindergartener fingers): avatar emoji (~3rem) + name + star count badge
- "+ New Friend" card at the end with dashed border
- Max 6 profiles per device — when reached, hide/disable the "New Friend" card and show "Ask a grown-up to make space!" message
- If zero profiles exist (and no migration), auto-redirect to create screen
- Reuses existing `.screen` container pattern and `showScreen()` helper
- Long-press on a profile card (1s) reveals a small red "X" for deletion (prevents accidental deletes)

## Step 5: New UI — Create Profile Screen
- Title: "Make Your Cat Card!"
- Subtitle: **"Use your first name!"** (privacy nudge — no PII encouragement)
- Large text input for name (maxlength 12, font-size ~2rem, touch-friendly)
- Horizontal scrollable row of **8 cat avatar options** drawn from existing `CAT_EMOJIS` subset (indices 0-7 for consistency); selected avatar has thick colored border
- "Let's Go!" button (disabled until name is non-empty after trim)
- **Debounce submit**: set `answering = true` on first tap to prevent double-submit duplicate profiles
- Back arrow to return to profile picker (hidden if no profiles exist yet — can't back out of first-run)

## Step 6: Title Screen Changes
- Add greeting at top: `<avatar> Hi, <name>!` in the subtitle area
- Add small "Switch Player" button (outline style, not prominent) that returns to profile picker
- `updateGreeting()` is called from `showScreen('titleScreen')` to refresh when returning
- **Gate `startGame()`**: if `!currentProfile`, redirect to profile picker instead of starting

## Step 7: Navigation Matrix
Define explicit screen transitions to avoid inconsistency:

| From | "Back" / "Home" goes to |
|---|---|
| gameScreen | titleScreen (current profile preserved) |
| resultsScreen (Home button) | titleScreen |
| collectionScreen | titleScreen |
| titleScreen (Switch Player) | profilePickerScreen |
| profilePickerScreen | (exit only — no back) |
| createProfileScreen | profilePickerScreen (unless first-run: no back) |

Extend `showScreen()` to call `updateGreeting()` when entering `titleScreen`, and `renderProfilePicker()` when entering `profilePickerScreen`.

## Step 8: Offline Resilience
- **localStorage is the UI source of truth** — reads from in-memory `profiles`/`currentProfile` which are populated from localStorage first, then reconciled with Firestore
- If Firebase SDK fails to load: `safeDb()` returns null, all reads/writes bypass Firestore, game is 100% functional in local-only mode
- If `signInAnonymously()` fails (ad-blocker, etc.): use a localStorage-generated UUID as `deviceUid` fallback so profile IDs remain stable
- If `deviceUid` from Firebase changes from the cached one (browser data partially cleared): re-upload the cached profiles under the new UID (don't wipe the local cache)
- Loading overlay has a 2-second timeout — if Firebase isn't ready by then, proceed with local-only mode and attempt to sync in the background

## Step 9: Firestore Security Rules
**Fixed**: separate the monotonic stars check from metadata updates so renaming or changing avatar doesn't require a star increase.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} { allow read, write: if false; }
    
    match /devices/{deviceId} {
      allow read, write: if request.auth != null && request.auth.uid == deviceId;
      
      match /profiles/{profileId} {
        allow read, list: if request.auth != null && request.auth.uid == deviceId;
        
        allow create: if request.auth != null 
                      && request.auth.uid == deviceId
                      && request.resource.data.name is string
                      && request.resource.data.name.size() > 0
                      && request.resource.data.name.size() <= 12
                      && request.resource.data.avatar is string
                      && request.resource.data.avatar.size() > 0
                      && request.resource.data.avatar.size() <= 8
                      && request.resource.data.totalStars is int
                      && request.resource.data.totalStars >= 0;
        
        allow update: if request.auth != null 
                      && request.auth.uid == deviceId
                      // totalStars must not decrease (if present in update)
                      && (!('totalStars' in request.resource.data) 
                          || request.resource.data.totalStars >= resource.data.totalStars)
                      // name if updated must still be valid
                      && (!('name' in request.resource.data)
                          || (request.resource.data.name is string 
                              && request.resource.data.name.size() > 0
                              && request.resource.data.name.size() <= 12));
        
        allow delete: if request.auth != null && request.auth.uid == deviceId;
      }
    }
  }
}
```

Note: Max-6-profile cap is enforced client-side only — Firestore rules can't easily count subcollection size without a counter field. For a kindergarten classroom this is acceptable.

## Step 10: Deploy to GitHub Pages
1. `git init` + push to public GitHub repo
2. Settings > Pages > deploy from `main` branch, root
3. Add `<username>.github.io` to Firebase authorized domains

---

## Verification Checklist

### Critical path
- [ ] Open in browser — anonymous auth succeeds (check console)
- [ ] Loading overlay appears briefly and dismisses
- [ ] First run: migration from `mathcats_stars` creates a "Player 1" profile automatically
- [ ] Create a new profile — appears in Firestore Console
- [ ] Play a round — stars update in Firestore (check console + doc)
- [ ] Refresh page — profile and stars persist
- [ ] Create second profile — profile picker shows both
- [ ] Switch between profiles — stars are independent; score/streak reset on switch
- [ ] Collection screen unlocks match per-profile star count, not bleeding between profiles

### Offline / failure paths
- [ ] Block `firestore.googleapis.com` in devtools — game still creates profiles and saves stars via localStorage only
- [ ] Block Firebase CDN entirely — game still works, no JS errors in console
- [ ] Clear localStorage — on next auth, profiles reload from Firestore
- [ ] Simulate slow network — loading overlay resolves within 2s, game proceeds
- [ ] Open two tabs, play in both — after refresh, both show `max(localStars, remoteStars)`

### Edge cases
- [ ] Rapid double-tap "Let's Go!" on create — only one profile created
- [ ] Try to create 7th profile — "New Friend" card is disabled/hidden
- [ ] Tap "Count the Cats" before selecting profile — redirects to profile picker (shouldn't be possible from UI, but verify via console)
- [ ] Name validation: empty, whitespace-only, 13 chars — all rejected with disabled button
- [ ] Long-press to delete active profile — exits to profile picker first
- [ ] Try to delete last profile — button disabled
- [ ] Change profile name/avatar — Firestore update succeeds (rules allow metadata edits)

### Platform
- [ ] Test on iPad Safari (primary target)
- [ ] Test on iPhone Safari
- [ ] Test on desktop Chrome

### Security (manual console tests)
- [ ] Try to write `totalStars: -1` — rejected
- [ ] Try to decrease `totalStars` — rejected
- [ ] Try to read another `deviceId`'s profiles — rejected
- [ ] Try to write a 13-char name — rejected

---

## File to modify
`/Users/robkwok/Hobbies/mathcat/index.html` (single file — all changes here)

---

## Plan Review Findings (resolved)

### Consensus issues fixed
1. **Script load order** (Codex, Gemini, Cursor — HIGH) — Removed `defer`, Firebase SDK loads synchronously immediately before the inline game script.
2. **Firestore rules blocked metadata updates** (Codex, Cursor — MEDIUM) — Rewrote `update` rule to only check `totalStars` monotonicity when it's present in the update, so renames/avatar changes work.
3. **Global `totalStars` needs per-profile refactor** (Codex, Cursor, Gemini — HIGH) — Added explicit touchpoint list in Step 3, introduced `currentProfile.totalStars` as single source of truth, added `getCurrentStars`/`setCurrentStars` helpers.
4. **Firebase load failure guard** (Gemini, Cursor — HIGH) — Added `safeDb()` helper and localStorage-first architecture; game works 100% offline.

### Critical individual findings addressed
- **`startGame()` gating** (Codex) — added guard redirecting to picker if no current profile
- **Double-tap duplicate profiles** (Gemini, Cursor) — debounce on "Let's Go!" submit
- **State reset on profile switch** (Gemini) — `selectProfile()` resets `score`, `streak`, `questionNum`, `answering`
- **Delete profile behavior** (Codex) — long-press confirmation, can't delete last profile, active-profile deletion exits to picker first
- **localStorage schema** (Codex) — explicit table in Step 3 with versioned keys
- **Navigation matrix** (Cursor) — explicit table in Step 7
- **Privacy nudge** (Gemini, Cursor) — "Use your first name!" subtitle on create screen
- **Loading overlay** (Gemini) — 2-second timeout, proceeds in offline mode if Firebase slow

### Testing gaps
Testing is manual via the verification checklist above. This is a hobby project with a single HTML file — adding Jest/Playwright would be disproportionate to scope. The checklist explicitly covers critical path, offline failure modes, edge cases, and adversarial scenarios (monotonic stars, unauthorized access, malformed inputs).

### Verdict
**Plan approved for implementation.** The high-risk technical issues (script ordering, star state management, rule correctness) are resolved. The localStorage-first architecture ensures graceful degradation if Firebase is unavailable.

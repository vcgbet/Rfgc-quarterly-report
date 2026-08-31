# RFGC Quarterly Report — Security Hardening + Render Deployment

This package prepares your app for production hosting on **Render**.
Copy these two files into your GitHub repo `vcgbet/Rfgc-quarterly-report`:

| File | Action |
|---|---|
| `index.html` | **Replace** the existing one (this is your app, secured) |
| `render.yaml` | **Add** (Render deployment blueprint) |

`firebase-rules.json` and `DEPLOY.md` are reference files — you don't have to commit them (but it doesn't hurt).

---

## 🔐 What was changed in `index.html`

1. **Removed the default-admin hint** — nothing on the login screen reveals credentials.
2. **All passwords are now stored as SHA-256 hashes** (`passHash`), never in plain text:
   - Admin, Pastor, and Secretary passwords are hashed on every device.
   - **Automatic migration:** the first time a device loads the new version, existing accounts found with plain-text passwords (in the browser or in Firebase) are hashed in place — **every existing login keeps working** (same username, same password).
   - Passwords are never shown in the Pastors/Secretaries tables anymore. Instead, a **"Login Credentials" popup shows the password ONCE** when an account is created or reset — share it with the pastor/secretary immediately (e.g. WhatsApp/SMS).
   - New **"Reset Pass"** button on each pastor & secretary row.
3. **New default admin password.** Fresh installs (new browsers) use:
   - Username: `admin`  •  Password: `Rfgc@Admin2026`
   - ⚠️ **Log in and change it in Settings → Update Password** as your first action after deploying.
   - Browsers that already used the app keep whatever admin password they had (it is migrated automatically).
   - Note: the admin password was always stored per-device (it's not synced through Firebase) — set the same strong password on each device you administer from.
4. **Firebase Anonymous sign-in** — the app now signs in to Firebase anonymously, so the database rules can *require* authentication (see step 2 below).
5. **Backups no longer contain plain-text passwords.** Old backup files still restore fine (their passwords get hashed on import).
6. Well-known `password123` no longer works on fresh devices.
7. **Reports can no longer be lost** (see "Data persistence" below).

---

## ☁️ Deploy on Render

1. Push the updated `index.html` + `render.yaml` to the `main` branch of your repo.
2. Go to <https://dashboard.render.com> → **New + → Blueprint** → select `vcgbet/Rfgc-quarterly-report` → **Apply**.
   *(Or: **New + → Static Site** → connect the repo → leave build command empty/default → publish directory `.`, and the settings in `render.yaml` are applied automatically when the file is in the repo.)*
3. Render gives you a URL like `https://rfgc-quarterly-report.onrender.com` — free HTTPS included.
4. (Optional) Later you can point a custom domain at it, and remove/keep the GitHub Pages version. If you keep GitHub Pages, it serves the same secured app — both are fine.

Render free static sites spin down when idle; first load after inactivity may take a few seconds. Your data is unaffected (it lives in Firebase, not on Render).

---

## 🔥 Firebase console (important — do this before/with the deploy)

1. Open <https://console.firebase.google.com> → project **rfgc-report-system**.
2. **Authentication → Sign-in method → Add new → Anonymous → Enable.**
   *(The app signs in anonymously so the rules below can require an authenticated client.)*
3. **Realtime Database → Rules** tab → replace everything with the contents of **`firebase-rules.json`** → **Publish**.
4. First device to load the new app will automatically hash any plain-text passwords currently stored in Firebase.

> Rollout tip: after deploying, refresh (hard-refresh: Ctrl+Shift+R) the app on phones/PCs that already use it, so every device runs the secured version. Old open tabs still running the previous version should be closed.

---

## 💾 Data persistence — submitted reports are never lost

The app is **local-first**: every submission is written to the device immediately, then synced to Firebase. Protections added:

| Scenario | Before | Now |
|---|---|---|
| Refresh / restart browser / idle device | Reports only survived if the cloud write had succeeded | Reports always reload from the device (localStorage) and re-sync |
| Weak internet / offline submission | Write could silently fail — report never reached the pastor/admin | Saved on device, auto-retried on reconnect, with a "Saved on this device" notice |
| Another device (or a stale connection) overwrites data | A cloud snapshot could **wipe** reports saved locally | Snapshots are **merged** — a local submission can never be clobbered |
| Deleting a branch/pastor/report while offline | Delete could "come back" from a stale snapshot | Deletes are tombstoned and re-pushed until the cloud confirms |
| Refresh while filling the report form | **All typed entries lost** | Draft autosaves every few seconds and is restored (with a "Discard draft" option) |
| Login lost on every refresh | Had to sign in again each time | Stays logged in for 14 days per device |

Also: a live cloud update no longer kicks you out of a form — the dashboard only refreshes when no modal is open and no unsaved input exists.

> Note: if a device is offline for a long time, the data is still safe on that device — it syncs automatically once internet returns. Regular **Export All Data** backups (Settings) remain recommended as the final safety net.

---

## ✅ Post-deploy checklist

- [ ] Log in as admin with `Rfgc@Admin2026` (or your existing password on old devices) — works
- [ ] **Change the admin password** in Settings → Update Password
- [ ] Log in as one branch pastor — works with the same credentials as before
- [ ] Log in as one branch secretary — works
- [ ] **Persistence test:** submit a report → refresh the browser / close & reopen → report still in "My Submissions", pastor's and admin's dashboards
- [ ] **Draft test:** start a new report, type something, refresh → draft restored notice appears
- [ ] Create/reset a secretary → password popup appears once → login with it works
- [ ] Submit a test report → sign as pastor → see it in Admin → Endorsed Reports
- [ ] Export DOCX and PDF once (needs internet for the export libraries)
- [ ] Check Firebase console → Realtime Database → data still updating

---

## 🛡️ Honest security notes (what this does and doesn't fix)

- **Fixed:** public display of admin credentials; well-known default password; plain-text passwords in Firebase/localStorage/backups/GitHub; wide-open database (once rules are published).
- **Understood limitation:** this is a client-side app — login checks happen in the browser and the Firebase API config is public by design (that's normal for Firebase; the *rules* are the real gate). Anonymous-auth rules stop casual abusers and scripts, but a determined attacker who understands the app can still write to Firebase while pretending to be the app. Bullet-proof security would require real server-side authentication (e.g. Firebase Auth email/password per user + per-user rules) — a bigger refactor that can be done later if needed.
- Recommendation: keep regular **Export All Data** backups (Settings page) — they now contain hashes, not passwords.

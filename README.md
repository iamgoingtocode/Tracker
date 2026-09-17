# Divinheal Partnership Tracker

A single-file, zero-dependency tracker for B2B partnerships, partners, meeting notes and open items.
Runs entirely in the browser — designed to be hosted free on GitHub Pages.

---

## Deploy to GitHub Pages

1. Create a new repository on GitHub (it can be **private** — Pages still works on private repos for
   personal accounts on paid plans; on a free account make it **public**. The repo being public does
   **not** expose your data, because your data never leaves your browser — see *Where the data lives*).

2. From this folder:

```bash
git init && git add . && git commit -m "Divinheal partnership tracker" && git branch -M main && git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPO.git && git push -u origin main
```

3. On GitHub: **Settings → Pages → Source: Deploy from a branch → Branch: `main` / `(root)` → Save.**

4. After a minute your tracker is live at:
   `https://YOUR-USERNAME.github.io/YOUR-REPO/`

Bookmark that URL. It must be `https://` — the tracker uses the browser's Web Crypto API, which
browsers only expose on `https://` or `localhost`.

To update later: edit `index.html`, then `git commit -am "..." && git push`.

---

## How it works

### Login and privacy

- Multiple accounts can exist on one browser. Each account has its **own separate vault**.
- Your password is never stored. It is run through **PBKDF2-SHA256 (250,000 iterations)** to derive an
  **AES-256-GCM** key, and that key encrypts your whole vault in `localStorage`.
- Signing in with a different username shows a completely different, isolated dataset.
- **There is no password reset.** Nobody — including you — can decrypt the vault without the password.

### Where the data lives

Everything is stored **encrypted**, in the browser you are using. GitHub only serves `index.html`;
it never sees your data — unless you turn on **cloud sync**, and even then it only ever receives
ciphertext (see below).

---

## Cloud sync — using the tracker on more than one device

**Off by default.** With sync off, your data lives in one browser on one machine. Signing in on your
phone with the same password shows an *empty* tracker, because the password is not a lookup key to a
server — it is the decryption key for data sitting on that one device.

**Turn it on** in *Settings → Cloud sync*. From then on, every save also pushes your vault to a
**private GitHub Gist**.

### What actually crosses the network

Only the encrypted blob. The vault is sealed with AES-256-GCM *before* it is sent, using a key
derived from your password by PBKDF2. GitHub stores something like:

```json
{ "app":"divinheal-tracker", "device":"Windows-ab1e",
  "kdf": { "salt":"9Fk2…", "iter":250000 },
  "blob": { "iv":"7Qx…", "ct":"k7Jx9fQ2mVp8Lz…" } }
```

GitHub cannot read `ct`. Your password never leaves the browser, and is never sent to GitHub in any
form. The `salt` is not secret — it is what lets a *second* device derive the same key from the same
password.

### Setting up a second device

1. On the new device, open your Pages URL and click **☁ Sign in from sync** on the login screen.
2. Paste a GitHub token and enter **the same password** you use on your first device.
3. It finds your vault, decrypts it locally, and sets the device up.

The token must be a **classic** token with only the `gist` scope —
[create one here](https://github.com/settings/tokens/new?scopes=gist&description=Divinheal%20Tracker%20sync).
Fine-grained tokens cannot access gists. The token is stored **encrypted with your password** on each
device and is never written into exported backups. Revoke it any time from GitHub settings.

### If two devices both make changes

The tracker does not silently pick a winner. It tells you which device changed what and when, and
asks. Whichever side you discard is snapshotted first, so it stays recoverable under
*Backups → Snapshots*.

### Turning sync off

*Settings → Cloud sync → Turn off sync* stops this device syncing and forgets the token. Your data
stays local. The gist stays on GitHub until you delete it yourself at
[gist.github.com](https://gist.github.com).

### Without sync

Moving devices means *Backups → Export*, then *Restore from a backup file* on the new device.

### Auto-save

Every change writes to the encrypted vault ~0.4s after you stop typing. The header shows
`Saved just now`. If you try to close the tab mid-save, the browser warns you.

---

## Structure

```
Profile  ─ the person whose work you are tracking
  ├── Tab 1: B2B Partnerships   NGOs, insurers, corporates, government bodies, embassies…
  └── Tab 2: Partners           facilitators, agents, consultants, referral partners…

Click any name  →  full record, with:
  · Overview      country, type, stage, contacts, commission, institutions tied
  · Meeting notes one entry per meeting — date, attendees, what happened, decisions, next steps
  · Open items    everything outstanding on that relationship
  · History       every change ever made to it, with one-click undo
```

Open Items also appear as their own **top-level tab**, aggregated across every profile, so the
start page always shows what is outstanding — filterable by profile, status, priority, owner and overdue.

---

## The things worth knowing

**Global dropdowns.** Any dropdown with a `＋ New` button writes back to a global list. Add
"Cooperative society" once as a partnership type and it is in that dropdown forever, everywhere.
Manage all the lists in **Settings → Global dropdown lists**.

**Action items from meetings.** The meeting form has an *Action items* box. One item per line —
each line becomes an open item against that record and appears on the Open Items tab immediately.
This is the fastest way to log a meeting: notes + actions in one pass.

**Linking partners to institutions.** On a partner, *Institutions we are tying this partner with*
ticks off the B2B partnerships in the same profile. The link shows on both records.

**Nothing deletes without a popup.** Deletes that cascade (a profile, or a record with several
meetings) additionally require typing `DELETE`. Every delete keeps a full copy in
**Backups → Change history**, so it can be undone.

**Two kinds of backup, both automatic:**
- *Change history* — every add, edit and delete, individually reversible (last 900).
- *Snapshots* — a full copy of everything, taken automatically every 6 hours of activity (last 20).

**Sync backups too.** With sync on, the gist keeps its own full revision history — every push is a
GitHub revision you can browse at `gist.github.com/<id>/revisions`.

**Export.**
- `.html` — the entire tracker **plus your data** in one file. Double-click it and it opens as a
  fully browsable read-only copy: no internet, no login, no server. This is the real backup.
- `.json` — data only, for importing elsewhere.

Both exports are **unencrypted**. That is deliberate — a backup you cannot open is not a backup —
but it means you should treat the files like any confidential document.

---

## Browser storage limit

`localStorage` caps out around 5 MB per site. Settings shows current usage. Thousands of meeting
notes would be needed to get near it; if you ever do, export a backup and trim old snapshots.

---

## Local development

```bash
python -m http.server 8777
```

Then open `http://localhost:8777`. Opening `index.html` directly from disk will **not** work —
browsers disable Web Crypto on `file://`. (Exported `.html` backups *do* open fine from disk,
because they contain their data already and never need to decrypt anything.)

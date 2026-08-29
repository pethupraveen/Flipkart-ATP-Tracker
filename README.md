# Flipkart ATP Tracker

A single-file stock dashboard that reads a Google Sheet in the visitor's own browser, using their
own Google account. It is a static page — no server, no build step, no dependencies.

**No data lives in this repository.** The page ships empty and fills itself at load time. A visitor
whose Google account can't open the spreadsheet sees a sign-in screen and nothing else, even though
the page itself is public.

---

## Setup

### 1. Create the OAuth client

You need one credential from Google so the page is allowed to ask for access.

1. Go to <https://console.cloud.google.com/> and create a project (any name).
2. **APIs & Services → Library** → search *Google Sheets API* → **Enable**.
3. **APIs & Services → OAuth consent screen**:
   - If your Google account is a Workspace account and you can see the option, choose **Internal**.
     This is the smoother path — no test-user list, no warning screen.
   - Otherwise choose **External**, leave it in **Testing**, and under *Test users* add every Google
     address that will open the dashboard. Only those addresses can sign in.
   - Scopes: you don't need to add any here. The page requests `spreadsheets.readonly` at sign-in.
4. **APIs & Services → Credentials → Create credentials → OAuth client ID**:
   - Application type: **Web application**
   - **Authorised JavaScript origins** — add your Pages origin, scheme and host only, no path:
     ```
     https://YOUR-USERNAME.github.io
     ```
     Add `http://localhost:8000` too if you want to test locally.
   - Leave *Authorised redirect URIs* empty. This page uses the token flow, not a redirect.
5. Copy the client ID. It looks like `1234567890-abc...xyz.apps.googleusercontent.com`.

A client ID is **not a secret** — it is designed to sit in public front-end code. What protects it is
the origin allowlist above: Google will refuse the request if it comes from any other domain.

### 2. Configure the page

Open `index.html`, find the `CONFIG` block near the top of the script, and set `clientId`. While
you're there, check the sheet ID and tab names match your spreadsheet:

```js
const CONFIG = {
  clientId: "PASTE_YOUR_CLIENT_ID_HERE.apps.googleusercontent.com",
  sheetId:  "10Z3k5TydjwR-16FXU-D5fqy0t6W2oWZTIARmfaAiXOc",
  sheetTitle: "Sprig/Motorola ATP and sellout",
  tabs: ["Sprig", "Motorola", "Minutes Motorola"],
  refreshMinutes: 15,
  criticalDays: 14,
  lowDays: 30,
  overstockDays: 90,
  leadTimeDays: 14,
  targetCoverDays: 45,
};
```

`leadTimeDays` and `targetCoverDays` drive the action list — set them to your real supplier lead time
and the cover you want a replenishment to restore.

The sheet ID is the long string in the spreadsheet URL between `/d/` and `/edit`.

### 3. Publish

1. Create a GitHub repository and commit `index.html` at the root.
2. **Settings → Pages → Source: Deploy from a branch**, branch `main`, folder `/ (root)`.
3. Wait a minute, then open `https://YOUR-USERNAME.github.io/REPO-NAME/` and sign in.

After the first sign-in, the page reloads data without a click for the rest of that browser session.
It re-reads the sheet every 15 minutes, when you return to the tab, and whenever you press
**Refresh now**.

---

## How it reads the sheet

The page finds the row whose first cell is `FSN`, treats columns A–E as FSN, SKU, ATP, DRR and DOH,
and reads every column after that whose heading parses as a date. So you can add SKUs, add days, or
reorder tabs without touching the code. New tabs only need their name added to `CONFIG.tabs`.

Numbers are recomputed rather than trusted:

- **Run rate** — units sold over the trailing 30 days, divided by 30. Where a SKU sold nothing in
  that window, the sheet's own DRR column is used instead.
- **Days of cover** — stock on hand divided by that run rate.
- **Bands** — under 14 days critical, under 30 low, 30–90 healthy, over 90 overstocked. Stock with no
  sales in 30 days is flagged separately as idle rather than healthy.

## The action list

**Do this today** is the top panel: one row per SKU that needs a decision, worst first, with the
quantity to act on. A SKU that is inside its bands doesn't appear at all.

| Action | When it fires | The number shown |
| --- | --- | --- |
| **Restock now** | Zero ATP but the SKU is still selling | Units to order |
| **Order today** | Cover is shorter than the lead time — it runs dry before a PO could land | Units to order |
| **Order this week** | Cover falls under the low band during the lead time | Units to order |
| **Promote** | Holding more than `overstockDays` of demand | Units above that line |
| **Clear** | Stock on hand, no sales in the trailing 30 days | Idle units |

Order quantity is `run rate × (leadTimeDays + targetCoverDays) − stock on hand`: enough to cover the
wait *and* land back on target cover. **Runs dry** is today plus the current days of cover.

Within an action, rows are ranked by what is at stake — units of demand that go unserved for the
reorder actions, units of idle capital for the others. Click an action chip to filter the list;
**Download list (CSV)** exports whatever is on screen, so filtering to *Order today* and exporting
gives you that purchase order and nothing else.

Trailing date columns with no data anywhere are ignored, so empty future dates don't drag the run
rate down. Cells holding fractional values in a daily-sales column are read as zero and counted in
the footnote.

---

## Using the page

The layout is one column of panels under a sticky header, so the catalogue tabs, the sheet's read
time, **Refresh now** and **Sign out** stay reachable however far down you scroll.

| | |
| --- | --- |
| **Light / dark** | Follows your operating system by default. The button in the top right overrides it, and the choice is remembered on that browser. |
| **`/`** | Jumps to the ledger search box. |
| **`Esc`** | Clears the search box and any action or status filter in one go. |
| **Filter chips** | Click an action or status chip to narrow the list; click it again, or **Clear filter**, to widen it. The count on each chip is live. |
| **Sorting** | Click a ledger column heading, or focus it with `Tab` and press `Enter`. The arrow shows the current sort. |

On a phone the ledger drops the Catalogue, 7d and Trend columns — the rest scrolls sideways — and each
action becomes a card with its on-hand, cover and runs-dry figures labelled underneath.

---

## Troubleshooting

| What you see | What it means |
| --- | --- |
| `Sign-in was blocked: popup_failed_to_open` | Browser blocked the popup. Allow popups for the site. |
| `redirect_uri_mismatch` or `origin mismatch` | The Pages origin isn't on the authorised origins list. It must be `https://you.github.io` — host only, no repository path, no trailing slash. |
| `Access blocked: this app is not verified` | The consent screen is External and your address isn't in the test-user list. Add it, or switch the project to Internal. |
| `Access blocked` on a work account | Your Workspace admin restricts unverified third-party apps. You'd need the app allowlisted, or sign in with an account that has sheet access and isn't under that policy. |
| `Your Google account can't read this spreadsheet` | The signed-in account has no access to the sheet, or the Sheets API isn't enabled on the Cloud project. |
| Data looks wrong after a sheet edit | Someone changed the column layout. Check that row still starts with `FSN` and the date headings are still in row order. |

---

## Things worth knowing

- **The page is public; the data is not.** Keep it that way — don't commit CSV exports, screenshots
  of the numbers, or a cached `data.json` into this repo. The moment you do, the numbers are on the
  open web.
- **Access is inherited, not granted.** This page can't show anyone data they couldn't already open
  in Google Sheets themselves. It doesn't widen access to the spreadsheet by a single person.
- **No data is stored.** No cookies, no local copy of the numbers, no analytics. The only thing the
  page keeps is your light/dark preference, in `localStorage` under `scc-theme`. The access token lives
  in memory and expires in about an hour; signing out revokes it.
- `<meta name="robots" content="noindex">` is set, which keeps the page out of search results but is
  not a security measure.

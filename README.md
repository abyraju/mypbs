# PBS — Personal Banking System
### v1.0.0

A self-contained personal finance tracker that runs entirely in a single HTML file. No server, no account, no install required. Your data stays in your browser and can optionally sync to a Google Sheet you own.

---

## Running the App

PBS is a single HTML file. There is nothing to install and no build step.

**Option A — Open directly in your browser**
1. Download `pbs_1_0_0.html`
2. Double-click it, or drag it into any browser window
3. The app opens immediately

**Option B — Save to your phone's home screen**
On iOS (Safari): open the file → tap the Share button → **Add to Home Screen**
On Android (Chrome): open the file → tap the menu (⋮) → **Add to Home Screen**
The app then behaves like a native app — full screen, no browser chrome.

**Option C — Serve it locally (optional)**
If you prefer a clean `localhost` URL:
```
npx serve .
# or
python3 -m http.server 8080
```
Then open `http://localhost:8080/pbs_1_0_0.html`

> No internet connection is needed to run PBS. All data is stored in your browser's `localStorage` under the key `ledger_db_v2`. The only network calls made are to Google Sheets (if you configure sync) and to the Google Fonts CDN for typography.

---

## First-Time Setup

When you open PBS for the first time with no data, a **Setup Guide** appears automatically. You can close it and start using the app immediately — cloud sync is optional.

### Step 1 — Set your budget and currency

1. Click **Settings** in the nav bar
2. Under **Budget & Currency**, enter your monthly budget amount and select your default currency
3. Click **Save Budget Settings**

### Step 2 — Add people

PBS tracks who each transaction is associated with (yourself, a family member, a business partner, etc.).

1. Click **Profiles**
2. Under **People**, click **+ Add Person**
3. Enter a name (and optionally a photo URL)
4. Save

### Step 3 — Add payment methods

1. Still in **Profiles**, find the **Payment Methods** section
2. Click **+ Add Payment Method**
3. Enter the card/account name, last 4 digits (optional), payment type, and currency
4. Save

### Step 4 — Add your first transaction

1. Click **Transactions**
2. Click **+ Add Transaction**
3. Fill in: name, amount, person, payment method, date, and category
4. Click **Save Transaction**

---

## Cloud Sync via Google Sheets (Optional)

PBS syncs to a Google Sheet you own through a small Google Apps Script relay. This is a one-time five-minute setup. Once done, your data is readable as a normal spreadsheet and syncs automatically across devices.

### Why Apps Script?

Google's Sheets API blocks API key-based writes from static HTML files — this is a hard Google policy. Apps Script runs as you inside Google's infrastructure, giving it full read/write access with no OAuth popup and no server required.

---

### Setup: Step by Step

**1. Create a Google Sheet**

Go to [sheets.google.com](https://sheets.google.com) and create a blank spreadsheet. Rename the first tab to `Transactions` (or whatever name you prefer — you'll enter it in PBS Settings later).

**2. Open the Apps Script editor**

In your Google Sheet: **Extensions → Apps Script**

**3. Paste the relay script**

Delete all existing code in the editor and paste the following:

```javascript
// PBS Relay Script v1.0.0
// Paste this into Extensions → Apps Script in your Google Sheet

const SHEET_NAME_DEFAULT = 'Transactions';

function doPost(e) {
  try {
    const body  = JSON.parse(e.postData.contents);
    const ss    = SpreadsheetApp.getActiveSpreadsheet();
    const tab   = body.tab || SHEET_NAME_DEFAULT;
    const sheet = ss.getSheetByName(tab) || ss.insertSheet(tab);

    if (body.action === 'write') {
      sheet.clearContents();
      if (body.values && body.values.length) {
        sheet.getRange(1, 1, body.values.length, body.values[0].length)
             .setValues(body.values);
      }
      return ok({ written: body.values ? body.values.length : 0 });
    }

    if (body.action === 'read') {
      const range  = body.range || 'A:Z';
      const values = sheet.getRange(range).getValues();
      return ok({ values });
    }

    return ok({ error: 'Unknown action: ' + body.action });
  } catch(err) {
    return ok({ error: err.message });
  }
}

function ok(data) {
  return ContentService
    .createTextOutput(JSON.stringify(data))
    .setMimeType(ContentService.MimeType.JSON);
}

function doGet(e) { return ok({ status: 'PBS relay online' }); }
```

Save the script (Ctrl+S / Cmd+S).

**4. Deploy the script as a Web App**

1. Click **Deploy → New deployment**
2. Click the ⚙️ gear icon next to "Select type" → choose **Web App**
3. Set **Execute as: Me**
4. Set **Who has access: Anyone**
5. Click **Deploy**
6. Authorise when prompted — this is a one-time Google permission grant
7. Copy the **Web App URL** — it looks like:
   `https://script.google.com/macros/s/AKfyc…/exec`

**5. Paste the URL into PBS**

1. In PBS, click **Settings**
2. Under **Google Sheets Sync**, paste the URL into the **Apps Script Web App URL** field
3. Confirm the **Sheet Tab Name** matches your spreadsheet tab (default: `Transactions`)
4. Click **Save Settings**
5. Click **▶ Run Test** to verify the connection end-to-end

A green "Script URL configured ✓" banner will appear when the connection is working.

---

### Sync Controls

| Control | What it does |
|---|---|
| ☁ Push (sync button) | Writes all local transactions to Google Sheets, overwriting the sheet |
| ⬇ Pull | Reads from Google Sheets and merges into local data |
| Auto-sync on change | Pushes automatically after every saved transaction (default: on) |
| Pull on startup | Fetches from Sheets when the app opens (default: on) |
| Sync interval | Pushes on a timer (0 = manual only) |

> **Note:** Sync is a full overwrite in both directions. PBS uses timestamps to merge pull data without losing local changes, but if you edit the sheet directly, always pull before adding new transactions locally.

### Re-deploying after script changes

If you ever edit the relay script, you must create a **new deployment** (not update the existing one) to get a new URL. Update the URL in PBS Settings after each new deployment.

---

## Admin PIN

Editing and deleting transactions, people, and payment methods requires an admin PIN. This prevents accidental changes on a shared device.

**Setting your PIN:**
1. Go to **Settings → Admin PIN**
2. Leave "Current PIN" blank if this is your first time
3. Enter a 4–6 digit PIN and confirm it
4. Click **Update PIN**

**Using admin mode:**
Click the **🔐 Admin** button in the nav bar (or the lock icon on mobile) and enter your PIN. Admin mode stays active for the session. You can lock it again manually.

If you forget your PIN, you can clear all local data from Settings → Danger Zone and start fresh (your Google Sheet data is unaffected).

---

## The Five Tabs

| Tab | Purpose |
|---|---|
| **Dashboard** | KPI cards, four chart views, recent transactions |
| **Transactions** | Full searchable/filterable transaction list |
| **Profiles** | Manage people and payment methods |
| **Recurring** | Set up repeating transactions (subscriptions, bills) |
| **Settings** | Budget, currency, sync, PIN, appearance, data export |

---

## Dashboard Charts

The chart section has four tabs:

- **📈 Over Time** — daily spending as a line chart; filter by person or payment method
- **🍩 By Category** — donut chart with a percentage breakdown table; filter by category
- **👤 By Person** — horizontal bar chart ranking each person by total spend
- **📅 Monthly** — grouped bar chart of your top 5 categories across the last 6 months

Use the **From** and **To** date inputs to narrow any chart to a date range. **✕ Clear** resets all filters.

---

## Recurring Transactions

Use the **Recurring** tab to set up templates that auto-generate transactions on a schedule (daily, weekly, monthly, or yearly). PBS processes pending recurring transactions each time the app opens. You can pause, edit, or delete any template without affecting already-generated transactions.

---

## Exporting and Importing Data

All export/import controls are in **Settings → Danger Zone**.

| Action | Format | Use case |
|---|---|---|
| Export Local Data (JSON) | `.json` | Full backup including all settings, people, payment methods |
| Import from JSON | `.json` | Restore a full backup |
| Export as CSV | `.csv` | Open in Excel, Numbers, or Google Sheets for analysis |

The CSV column order is: `ID, Date, Name, Amount, Currency, Person, Payment, Category, Comments, Created, Updated`

---

## Data and Privacy

- All data is stored locally in your browser (`localStorage`)
- Nothing is sent anywhere unless you configure Google Sheets sync
- If you configure sync, data goes to **your own Google Sheet** — PBS has no servers and no accounts
- Clearing your browser data or using a private/incognito window will erase local data — export a JSON backup regularly if this matters to you

---

## Browser Compatibility

PBS works in any modern browser. Tested in:
- Chrome / Chromium 100+
- Safari 15+
- Firefox 100+
- Edge 100+

JavaScript must be enabled. No extensions or plugins required.

---

## File Reference

| File | Description |
|---|---|
| `pbs_1_0_0.html` | The application — open this in a browser |
| `pbs_1_0_0_changelog.txt` | Full technical changelog for v1.0.0 |
| `README.md` | This file |

---

*PBS — Personal Banking System · v1.0.0*

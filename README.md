# 💊 Medicine Automation — n8n Telegram Bot

An n8n workflow automation that helps pharmacy shops manage their medicine lists through Telegram. Staff submit medicines for approval, admins review and approve them, and everything gets stored in a private Google Sheet per shop — no data shared between shops.

---

## 🧩 What It Does

This automation connects two Telegram bots — one for **staff** and one for **admins** — through an n8n workflow backed by Google Sheets.

- **Staff** submit medicine names via Telegram. The bot verifies the medicine using Gemini AI + Tavily search and sends an approval request to the admin.
- **Admins** receive the request and approve or reject it. Approved medicines are written directly into the shop's private Google Sheet.
- Every shop gets its **own isolated Google Spreadsheet**. No shop can read or modify another shop's data.

---

## 🏗️ Architecture Overview

```
Staff Bot (Telegram)
    └── Submit medicine name
         └── Gemini AI + Tavily verifies medicine & company
              └── If valid → Save to Pending Requests (shop spreadsheet)
                       └── Notify admin via Admin Bot

Admin Bot (Telegram)
    └── approve / reject / /add / /all / /all -a / /clear -a
         └── Read/Write shop's Google Spreadsheet
```

### Two Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| Staff Flow | Staff sends a message to Staff Bot | Verify medicine, submit for admin approval |
| Admin Flow | Admin sends a command to Admin Bot | Register shop, approve/reject, manage medicine list |

---

## 📦 Tech Stack

| Tool | Role |
|---|---|
| [n8n](https://n8n.io) | Workflow automation engine |
| Telegram Bot API | Staff & admin chat interface |
| Google Sheets API | Per-shop medicine database |
| Google Gemini AI | Medicine name verification |
| Tavily Search | Real-time web search for manufacturer verification |

---

## 🗄️ Data Structure

### Central Registry Spreadsheet

**`Registry` tab** — one row per shop

| shopCode | shopName | adminChatId | spreadsheetId | city | unionOrCityCorp | villageOrArea | roadName |
|---|---|---|---|---|---|---|---|

**`StaffRegistry` tab** — one row per staff member

| staffChatId | shopCode | staffName |
|---|---|---|

### Per-Shop Spreadsheet (auto-created on registration)

**`Medicine List` tab** — column per company, medicines listed below

```
Beximco Pharma    |  Square Pharma   |  Incepta Pharma
------------------+------------------+-----------------
Napa              |  Seclo           |  Amdocal
Napa Extend       |  Losectil        |  Ciplox
```

**`Pending Requests` tab**

| status | medicine | company | newCompany | requestId | userChatId | shopCode | staffName |
|---|---|---|---|---|---|---|---|

---

## 🚀 Getting Started

### Prerequisites

- A running [n8n](https://n8n.io) instance
- Two Telegram bots (one for staff, one for admin) — create via [@BotFather](https://t.me/BotFather)
- A Google account with Sheets API enabled
- A Gemini API key
- A [Tavily](https://tavily.com) API key

### Setup Steps

1. **Import the workflow** — Import `medicine_automations.json` into your n8n instance.

2. **Configure credentials** in n8n:
   - `telegramApi` (staff bot token) → used in staff-side nodes
   - `telegramApi` (admin bot token) → used in admin-side nodes
   - `googleSheetsOAuth2Api` → for all Sheets read/write operations
   - `googlePalmApi` → for Gemini AI verification
   - `tavilyApi` → for web search

3. **Create the central Registry spreadsheet** in Google Sheets with two tabs:
   - `Registry` — columns: `shopCode`, `shopName`, `adminChatId`, `spreadsheetId`, `city`, `unionOrCityCorp`, `villageOrArea`, `roadName`
   - `StaffRegistry` — columns: `staffChatId`, `shopCode`, `staffName`

4. **Update the spreadsheet ID** in the workflow nodes that reference the central registry (`Lookup Shop in Registry`, `Fetch Staff Registry Join`, etc.) with your actual spreadsheet ID.

5. **Activate the workflow** in n8n.

---

## 🤖 Bot Commands

### Admin Bot

| Command | Description |
|---|---|
| `/start` | Show welcome message and registration instructions |
| `/register ShopName City,Union,Village,Road` | Register your shop (one time) |
| `/add MedicineName` | Directly add a medicine (AI verifies, no approval needed) |
| `approve MedicineName` | Approve a pending staff request |
| `reject MedicineName` | Reject a pending staff request |
| `/all` | View the full medicine list grouped by company |
| `/all -a` | Approve all pending requests at once |
| `/clear -a` | Show confirmation prompt to clear the medicine list |
| `/clear -a confirm` | Confirm and clear all medicine entries |

**Registration format:**
```
/register GreenMedical Dhaka,DNCC,Dhanmondi,Road27
```
No spaces between commas. All 4 location parts are required.

---

### Staff Bot

| Command | Description |
|---|---|
| `/start` | Show welcome message and join instructions |
| `/join SHOPCODE YourName` | Link yourself to a shop (one time per device) |
| `[any medicine name]` | Submit a medicine for admin approval |

**Join format:**
```
/join SHOP-1234567890 Rahim
```

---

## 🔄 Workflow Flows

### Staff: Submit a Medicine

```
Staff types medicine name
  → Gemini AI + Tavily verifies the medicine and identifies the company
  → Checks if medicine already exists in the shop's sheet
  → If new → saves to Pending Requests → notifies admin
  → If already exists → tells staff it's already in the list
  → If invalid → tells staff the medicine could not be verified
```

### Admin: Approve / Reject

```
Admin types "approve Napa" or "reject Napa"
  → Looks up pending request in shop's spreadsheet
  → Approve: writes medicine to Medicine List under the correct company column
             updates status to "approved" in Pending Requests
             notifies staff of the approval
  → Reject:  updates status to "rejected"
             notifies staff of the rejection
```

### Admin: Bulk Approve All

```
Admin types "/all -a"
  → Fetches all pending requests
  → Batch-writes all medicines to the Medicine List
  → Marks all as "approved" in Pending Requests
  → Sends summary to admin
  → Notifies each staff member whose medicines were approved
```

### Shop Registration

```
Admin types "/register ShopName City,Union,Village,Road"
  → Checks if admin already has a registered shop
  → Creates a new Google Spreadsheet for the shop
  → Writes headers to "Medicine List" and "Pending Requests" tabs
  → Saves shop entry to the central Registry
  → Sends admin their unique shop code to share with staff
```

---

## 🔒 Data Isolation

Each shop gets a dedicated Google Spreadsheet created automatically on registration. The spreadsheet ID is stored in the central `Registry` tab and looked up per request. Staff and admins can only read and write their own shop's spreadsheet — there is no cross-shop data access.

---

## 🙏 Acknowledgements

Built with [n8n](https://n8n.io), [Telegram Bot API](https://core.telegram.org/bots/api), [Google Sheets API](https://developers.google.com/sheets/api), [Google Gemini](https://ai.google.dev/), and [Tavily](https://tavily.com).

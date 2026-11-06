# Google Sheets Synchronization Integration Guide

A complete, production-grade guide for synchronizing attendance sessions from the **Face Attendance Management System** directly into a **Google Sheet** in real time.

---

## 📋 Overview & Supported Methods

The system supports two independent, production-tested integration pathways:

| Feature | Method 1: Google Apps Script Webhook *(Recommended)* | Method 2: Google Cloud Service Account |
| :--- | :--- | :--- |
| **Complexity** | ⚡ **1 Minute (Easiest)** | ⏱️ 5–10 Minutes |
| **Cost** | 🆓 **100% Free** | 🆓 **100% Free** |
| **GCP / Console Setup** | ❌ None required | ✅ Required (Cloud Console) |
| **Org Policy Blocks** | 🛡️ **Immune** (Works on school/work accounts) | ⚠️ May be blocked by `iam.disableServiceAccountKeyCreation` |
| **Credential Files** | ❌ No `.json` file needed | ✅ Requires `service_account.json` |
| **Transport** | Direct HTTP POST | `gspread` + Google OAuth2 |

---

## 🚀 Method 1: Google Apps Script Webhook (Recommended)

This method connects your Google Sheet directly through an HTTP Webhook deployed as a lightweight Google Apps Script Web App. It requires **zero Google Cloud Console configuration**, **zero credential keys**, and works on **all Google accounts** (personal, institutional, or corporate).

### Step 1: Open Your Google Sheet
1. Open or create a Google Sheet at **[sheets.new](https://sheets.new)**.
2. Give the spreadsheet any name (e.g., `Class Attendance 2026`).

### Step 2: Open Apps Script Editor
1. In the top menu of your Google Sheet, click **Extensions** $\rightarrow$ **Apps Script**.
2. Delete any boilerplate code inside the editor.

### Step 3: Paste the Webhook Handler Code
Copy and paste the following production script into the editor:

```javascript
function doPost(e) {
  try {
    var data = JSON.parse(e.postData.contents);
    var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
    var dateStr = data.session_date; // Expected format: "DD-MM-YYYY"
    var attendance = data.attendance || {}; // e.g. {"STU001": "P", "STU002": "A"}
    var names = data.names || {}; // e.g. {"STU001": "Student Name"}

    var values = sheet.getDataRange().getValues();
    
    // 1. Initialize headers if sheet is empty
    if (values.length === 0 || values[0].length === 0 || values[0][0] === "") {
      sheet.appendRow(["Student ID", "Student Name", dateStr]);
      values = sheet.getDataRange().getValues();
    }

    var headers = values[0];
    var dateCol = headers.indexOf(dateStr) + 1; // 1-indexed

    // 2. Automatically create date column if missing
    if (dateCol === 0) {
      dateCol = headers.length + 1;
      sheet.getRange(1, dateCol).setValue(dateStr);
      headers.push(dateStr);
    }

    // 3. Map existing Student IDs to sheet rows
    var studentRowMap = {};
    for (var i = 1; i < values.length; i++) {
      if (values[i][0]) {
        studentRowMap[values[i][0].toString().trim()] = i + 1;
      }
    }

    // 4. Update or append student attendance records
    for (var studentId in attendance) {
      var status = attendance[studentId];
      var row = studentRowMap[studentId.toString().trim()];

      if (row) {
        var existingStatus = sheet.getRange(row, dateCol).getValue().toString().trim();
        // Mandatory Non-Downgrade Rule: Never overwrite 'P' with 'A'
        if (existingStatus !== "P") {
          sheet.getRange(row, dateCol).setValue(status);
        }
      } else {
        // New student not yet in sheet: append new row
        var studentName = names[studentId] || ("Student " + studentId);
        var newRow = [studentId, studentName];
        while (newRow.length < dateCol - 1) {
          newRow.push("");
        }
        newRow[dateCol - 1] = status;
        sheet.appendRow(newRow);
        studentRowMap[studentId] = sheet.getLastRow();
      }
    }

    return ContentService.createTextOutput(JSON.stringify({
      status: "success",
      message: "Attendance recorded successfully",
      date: dateStr,
      updated_count: Object.keys(attendance).length
    })).setMimeType(ContentService.MimeType.JSON);

  } catch (error) {
    return ContentService.createTextOutput(JSON.stringify({
      status: "error",
      message: error.toString()
    })).setMimeType(ContentService.MimeType.JSON);
  }
}
```

### Step 4: Deploy as Web App
1. Click the blue **Deploy** button in the top right $\rightarrow$ select **New deployment**.
2. Click the gear icon ⚙️ next to "Select type" $\rightarrow$ choose **Web app**.
3. Configure the deployment settings:
   - **Description**: `Attendance Webhook`
   - **Execute as**: `Me (<your-email>)`
   - **Who has access**: `Anyone` *(Crucial so your local server can POST data)*
4. Click **Deploy**.

### Step 5: Authorize the Script
1. When prompted with **Authorization required**, click **Authorize access**.
2. Select your Google Account.
3. If the warning screen *"Google hasn't verified this app"* appears:
   - Click the small gray link: **Advanced** (bottom left).
   - Click **Go to Untitled project (unsafe)**.
   - Click **Allow**.
4. Google will finalize the deployment and present you with your **Web App URL**:
   ```text
   https://script.google.com/macros/s/AKfycb.../exec
   ```

### Step 6: Configure `.env`
Open your local `.env` file in the project root and add:

```ini
GOOGLE_SHEET_WEBHOOK_URL=https://script.google.com/macros/s/your_deployment_id/exec
```

---

## 🔒 Method 2: Google Cloud Service Account (`gspread`)

If your deployment mandates traditional IAM service accounts, follow this setup:

### Step 1: Create a Google Cloud Project
1. Navigate to **[Google Cloud Console](https://console.cloud.google.com/)**.
2. Click **New Project** $\rightarrow$ Name it `Attendance-Sync` $\rightarrow$ Click **Create**.

### Step 2: Enable Required APIs
Enable the following free APIs:
- **[Google Sheets API](https://console.cloud.google.com/apis/library/sheets.googleapis.com)**
- **[Google Drive API](https://console.cloud.google.com/apis/library/drive.googleapis.com)**

### Step 3: Create Service Account & Export Key
1. Go to **APIs & Services** $\rightarrow$ **Credentials**.
2. Click **+ CREATE CREDENTIALS** $\rightarrow$ **Service account**.
3. Name it `attendance-sync` $\rightarrow$ Click **Done**.
4. Click on the newly created service account email.
5. Go to the **KEYS** tab $\rightarrow$ **ADD KEY** $\rightarrow$ **Create new key** $\rightarrow$ Choose **JSON** $\rightarrow$ **Create**.
6. Move the downloaded key file into your project:
   ```text
   c:\Projects\FCP-teamup\face-attendance\instance\service_account.json
   ```

> [!TIP]
> If Google blocks key creation with `iam.disableServiceAccountKeyCreation`, either switch to **Method 1 (Apps Script)** or run this in Google Cloud Shell:
> ```bash
> gcloud resource-manager org-policies disable-enforce iam.disableServiceAccountKeyCreation --project=YOUR_PROJECT_ID
> ```

### Step 4: Share Your Google Sheet
1. Open your Google Sheet $\rightarrow$ click the green **Share** button.
2. Paste the service account's email address (e.g., `attendance-sync@your-project.iam.gserviceaccount.com`).
3. Set role to **Editor**, uncheck "Notify people", and click **Share**.
4. Copy the **Sheet ID** from the URL:
   `https://docs.google.com/spreadsheets/d/`**`<SHEET_ID>`**`/edit`.

### Step 5: Configure `.env`
```ini
GOOGLE_SERVICE_ACCOUNT_JSON=instance/service_account.json
GOOGLE_SHEET_ID=your_extracted_sheet_id_here
```

---

## 📡 API Specification & Payload Structure

The sync engine dispatches a structured JSON payload to the Google Sheet backend upon teacher confirmation:

### HTTP Request
- **Method**: `POST`
- **Content-Type**: `application/json`

### JSON Request Schema
```json
{
  "session_date": "06-11-2026",
  "attendance": {
    "ASML24001": "P",
    "ASML24002": "A",
    "ASML24003": "P"
  },
  "names": {
    "ASML24001": "Praveen R",
    "ASML24002": "Alice Smith",
    "ASML24003": "Bob Jones"
  }
}
```

### JSON Response Schema
```json
{
  "status": "success",
  "message": "Attendance recorded successfully",
  "date": "06-11-2026",
  "updated_count": 3
}
```

---

## 🛡️ Core Business Logic & Safety Rules

1. **Automatic Column Provisioning**:
   - The engine checks for a column named after the current `DD-MM-YYYY` date.
   - If missing, it automatically creates the header column on the right without disturbing previous records.

2. **Mandatory Non-Downgrade Rule**:
   - If a student is already marked **`P`** (Present) on a given date, subsequent resubmissions or re-scans will **never downgrade them to `A`** (Absent).
   - Upgrading from **`A`** to **`P`** upon re-scan or manual correction is permitted.

3. **Dynamic Student Enrollment**:
   - If an enrolled student does not yet exist in the Google Sheet roster, a new row is appended with their `Student ID` and `Student Name`.

---

## 🧪 Verification & Testing

### 1. Direct Python Command-Line Test
You can test your live Webhook directly from PowerShell:

```powershell
.\venv\Scripts\python.exe -c "import requests; r = requests.post('https://script.google.com/macros/s/your_deployment_id/exec', json={'session_date': '06-11-2026', 'attendance': {'TEST001': 'P'}, 'names': {'TEST001': 'Test Student'}}, timeout=20); print(r.status_code, r.text)"
```
Expected output:
```text
200 {"status":"success","message":"Attendance recorded successfully",...}
```

### 2. Automated Test Suite
Run the Phase 7 Google Sheets unit and integration test suite:

```powershell
.\venv\Scripts\python.exe test_google_sheets_sync.py
```
Expected output:
```text
======================================================================
PHASE 7: GOOGLE SHEETS SYNCHRONIZATION TEST SUITE
======================================================================
[Step 1/5] Verifying Configuration Detection...       [PASS]
[Step 2/5] Testing Sheet Header & Row Synchronization... [PASS]
[Step 3/5] Testing Resubmission on Existing Date Column... [PASS]
[Step 4/5] Testing Mandatory Rule: NEVER Downgrade 'P' to 'A'... [PASS]
[Step 5/5] Testing Dynamic Multi-Date Expansion...     [PASS]
======================================================================
ALL PHASE 7 GOOGLE SHEETS TESTS PASSED (100%)
======================================================================
```

### 3. End-to-End Web Flow
1. Start the web application: `python run.py`.
2. Navigate to [http://127.0.0.1:5000/attendance/upload](http://127.0.0.1:5000/attendance/upload).
3. Upload classroom photo(s) and proceed to the Review screen.
4. Click **Confirm Attendance**.
5. Switch to your Google Sheet—the columns and **`P`** / **`A`** statuses will appear automatically!

---

## ❓ Troubleshooting FAQ

### 1. Error: `Google haven't verified this app`
- **Cause**: Google displays this warning for self-authored Apps Scripts.
- **Solution**: Click **Advanced** $\rightarrow$ Click **Go to Untitled project (unsafe)** $\rightarrow$ Click **Allow**.

### 2. Error: `HTTP 405 Method Not Allowed`
- **Cause**: You opened the Webhook URL in a regular browser tab (which sends a `GET` request instead of a `POST` request).
- **Solution**: The script defines `doPost(e)`. Test it using Python `requests.post()` or via the web application.

### 3. Status shows `Saved Locally (Sheets unconfigured)`
- **Cause**: The `.env` file does not contain a valid `GOOGLE_SHEET_WEBHOOK_URL` or `GOOGLE_SERVICE_ACCOUNT_JSON`.
- **Solution**: Check that your `.env` contains:
  ```ini
  GOOGLE_SHEET_WEBHOOK_URL=https://script.google.com/macros/s/.../exec
  ```
  Restart your Flask server after modifying `.env`.

### 4. Updates appear in sheet, but old 'P' turned to 'A'
- The Apps Script and `sync.py` enforce the non-downgrade rule. If this happens, verify that the `doPost` code contains:
  ```javascript
  if (existingStatus !== "P") {
    sheet.getRange(row, dateCol).setValue(status);
  }
  ```

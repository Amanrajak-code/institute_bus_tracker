# 🛠️ Bus Tracker Backend Setup Guide

The reason your app is not working is that it is trying to connect to a **private/restricted Google Apps Script URL**. To fix this, you need to set up your own backend using Google Sheets (free).

Follow these steps to get your app working!

## Step 1: Create a Google Sheet
1. Go to [sheets.google.com](https://sheets.google.com) and create a **New Spreadsheet**.
2. Name it **"Bus Tracker Database"**.
3. Rename 'Sheet1' to **"CurrentStatus"**.
4. In the first row of "CurrentStatus", add these headers:
   `bus_id`, `lat`, `lng`, `status`, `next_stop`, `speed`, `last_update`

## Step 2: Create the Script
1. In your Google Sheet, go to **Extensions > Apps Script**.
2. Delete any code in `Code.gs` and paste the following code:

```javascript
// CONFIGURATION
const SHEET_NAME = "CurrentStatus";

function doPost(e) {
  const lock = LockService.getScriptLock();
  lock.tryLock(10000);
  
  try {
    const doc = SpreadsheetApp.getActiveSpreadsheet();
    let sheet = doc.getSheetByName(SHEET_NAME);
    
    // Create sheet if missing
    if (!sheet) {
      sheet = doc.insertSheet(SHEET_NAME);
      sheet.appendRow(["bus_id", "lat", "lng", "status", "next_stop", "speed", "last_update"]);
    }
    
    // Parse data
    const data = JSON.parse(e.postData.contents);
    const busId = data.bus_id;
    const timestamp = new Date();
    
    // Search for existing bus row
    const range = sheet.getDataRange();
    const values = range.getValues();
    let rowIndex = -1;
    
    for (let i = 1; i < values.length; i++) {
      if (values[i][0] === busId) {
        rowIndex = i + 1; // 1-based index
        break;
      }
    }
    
    // Row data
    const rowData = [
      busId,
      data.lat,
      data.lng,
      data.status,
      data.next_stop,
      data.speed,
      timestamp.toISOString()
    ];
    
    // Update or Append
    if (rowIndex > 0) {
      sheet.getRange(rowIndex, 1, 1, rowData.length).setValues([rowData]);
    } else {
      sheet.appendRow(rowData);
    }
    
    return ContentService.createTextOutput(JSON.stringify({ 'result': 'success' }))
      .setMimeType(ContentService.MimeType.JSON);
      
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ 'result': 'error', 'error': err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
      
  } finally {
    lock.releaseLock();
  }
}

function doGet(e) {
  try {
    const doc = SpreadsheetApp.getActiveSpreadsheet();
    const sheet = doc.getSheetByName(SHEET_NAME);
    
    if (!sheet) {
      return ContentService.createTextOutput(JSON.stringify({ 
        'success': true, 
        'buses': [] 
      })).setMimeType(ContentService.MimeType.JSON);
    }
    
    const data = sheet.getDataRange().getValues();
    const headers = data[0];
    const buses = [];
    
    // Skip header row
    for (let i = 1; i < data.length; i++) {
      const row = data[i];
      buses.push({
        bus_id: row[0],
        lat: row[1],
        lng: row[2],
        status: row[3],
        next_stop: row[4],
        speed: row[5],
        last_update: row[6]
      });
    }
    
    return ContentService.createTextOutput(JSON.stringify({ 
      'success': true, 
      'buses': buses 
    })).setMimeType(ContentService.MimeType.JSON);
    
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ 'success': false, 'error': err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

## Step 3: Deploy
1. Click the blue **Deploy** button > **New deployment**.
2. Click the gear icon (Select type) > **Web app**.
3. Fill in:
   - **Description**: Bus Tracker API
   - **Execute as**: Me (your email)
   - **Who has access**: **Anyone** (IMPORTANT! Do not choose "Anyone with Google account")
4. Click **Deploy**.
5. Copy the **Web App URL** (it ends with `/exec`).

## Step 4: Connect to App
1. Open `public/driver.html` and `public/student.html`.
2. Find `const CONFIG` at the top of the script.
3. Replace the `GOOGLE_SCRIPT_URL` with your new Web App URL.

```javascript
const CONFIG = {
    GOOGLE_SCRIPT_URL: 'https://script.google.com/macros/s/YOUR_NEW_DEPLOYMENT_ID/exec',
    // ...
};
```

That's it! Your app will now send and receive data correctly.

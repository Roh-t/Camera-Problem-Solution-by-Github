# Camera-Problem-Solution-by-Github

# Complete Theory: Camera in Google Apps Script (GAS)

---

## Why Camera Fails in GAS

```
script.google.com  →  sends HTTP header:
Permissions-Policy: camera=()
```

This means **Google's server itself blocks camera** at the header level. No browser setting, no permission grant, no code trick can override a server-sent `Permissions-Policy` header. This is why even after setting Camera = Allow in Chrome, it still shows "Permission Denied".

---

## The Complete Solution Architecture

```
┌─────────────────────────────────────────────────────┐
│                    FLOW DIAGRAM                      │
│                                                      │
│  GAS App (script.google.com)                        │
│       │                                              │
│       │  1. User clicks Punch IN/OUT                │
│       │                                              │
│       │  2. window.open() → Opens Popup             │
│       │         │                                    │
│       │         ▼                                    │
│       │  GitHub Pages (roh-t.github.io)             │
│       │  ✅ Camera WORKS here                        │
│       │         │                                    │
│       │         │  3. Capture Photo                 │
│       │         │  4. Get Location                  │
│       │         │                                    │
│       │         │  5. postMessage / BroadcastChannel│
│       │         │         │                         │
│       │         ▼         ▼                         │
│       │  GAS App receives data                      │
│       │         │                                    │
│       │         ▼                                    │
│       │  6. handlePunchWithPhoto()                  │
│       │         │                                    │
│       │         ▼                                    │
│       │  7. Upload photo → Google Drive             │
│       │  8. Save URL → Attendance Sheet             │
│                                                      │
└─────────────────────────────────────────────────────┘
```

---

## Part 1: GitHub Pages Setup

### Why GitHub Pages?
- Free hosting
- No server needed
- Runs on `github.io` domain — no `Permissions-Policy` restriction
- Camera works perfectly

### Steps to Setup
```
1. Create GitHub account → github.com
2. New Repository → name it anything (e.g. punch-camera)
3. Add index.html → your camera capture page
4. Settings → Pages → Source: main branch → Save
5. Your URL: https://YOUR-USERNAME.github.io/REPO-NAME/
```

### Files Needed
```
punch-camera/
└── index.html   ← Only this one file needed
```

---

## Part 2: The Camera Page (index.html on GitHub Pages)

### What it does step by step:
```
1. Read URL params → action=IN or action=OUT
2. Call navigator.mediaDevices.getUserMedia()
   → Works because github.io allows camera
3. Show live video preview
4. User clicks Capture → canvas.drawImage(video)
5. Convert to base64 JPEG string
6. User clicks Confirm → get GPS location
7. Send data back to parent GAS window via postMessage
8. Close popup
```

### Key Code Concepts

**Reading URL params:**
```javascript
const params = new URLSearchParams(window.location.search);
const action = params.get('action'); // 'IN' or 'OUT'
```

**Starting camera:**
```javascript
stream = await navigator.mediaDevices.getUserMedia({
    video: { facingMode: 'user' },  // front camera
    audio: false
});
document.getElementById('video').srcObject = stream;
```

**Capturing photo:**
```javascript
const canvas = document.createElement('canvas');
canvas.width  = video.videoWidth;
canvas.height = video.videoHeight;
canvas.getContext('2d').drawImage(video, 0, 0);
const base64 = canvas.toDataURL('image/jpeg', 0.80).split(',')[1];
// split(',')[1] removes "data:image/jpeg;base64," prefix
```

**Sending to parent window:**
```javascript
// Method 1: postMessage (when opened as popup)
window.opener.postMessage({
    type: 'PUNCH_DATA',
    action: action,
    location: loc,
    imageBase64: base64
}, '*');

// Method 2: BroadcastChannel (when opened as new tab)
const bc = new BroadcastChannel('punch_channel');
bc.postMessage({ type: 'PUNCH_DATA', ... });
bc.close();
```

---

## Part 3: GAS Attendance.html Changes

### Step 1 — Define camera URL
```javascript
// Top of your script
const CAMERA_HOST = 'https://roh-t.github.io/punch-camera/';
```

### Step 2 — Open popup on button click
```javascript
function getLocationAndPunch(action) {
    const btn = document.getElementById(action === 'IN' ? 'btn-in' : 'btn-out');
    
    // Save references globally for later use
    window._punchBtn      = btn;
    window._punchOrigText = btn.innerHTML;
    
    btn.disabled  = true;
    btn.innerHTML = 'Opening Camera...';

    // Build URL with action param
    const popupUrl = CAMERA_HOST + '?action=' + action;
    
    // Open as popup window
    window.open(popupUrl, 'PunchCamera', 
        'width=440,height=560,top=80,left=300');
}
```

### Step 3 — Listen for data back from popup
```javascript
// Method 1: postMessage listener
window.addEventListener('message', function(e) {
    if (!e.data || e.data.type !== 'PUNCH_DATA') return;
    handlePunchPayload(e.data);
});

// Method 2: BroadcastChannel listener (tab fallback)
const bc = new BroadcastChannel('punch_channel');
bc.onmessage = function(e) {
    if (!e.data || e.data.type !== 'PUNCH_DATA') return;
    handlePunchPayload(e.data);
};

// Single handler for both methods
function handlePunchPayload(data) {
    const { action, location, imageBase64 } = data;
    sendPunchWithPhoto(action, location, 
        window._punchBtn, window._punchOrigText, imageBase64);
}
```

### Step 4 — Send to backend
```javascript
function sendPunchWithPhoto(action, loc, btn, origText, imageBase64) {
    google.script.run
        .withSuccessHandler(res => {
            btn.disabled  = false;
            btn.innerHTML = origText;
            if (res.success) {
                Swal.fire('Saved', res.message, 'success');
                loadAttendanceTable();
            }
        })
        .handlePunchWithPhoto(currentUser.email, action, loc, imageBase64);
}
```

---

## Part 4: Code.gs Backend

### handlePunchWithPhoto function — Full Flow
```javascript
function handlePunchWithPhoto(email, action, locationStr, imageBase64) {

    // STEP 1: Upload photo to Google Drive
    let photoUrl = "";
    if (imageBase64 && imageBase64.length > 100) {
        const timestamp = Utilities.formatDate(
            new Date(), Session.getScriptTimeZone(), "yyyyMMdd_HHmmss"
        );
        const fileName = "Punch_" + email.split('@')[0] 
                       + "_" + action + "_" + timestamp + ".jpg";
        photoUrl = uploadFileToDrive(imageBase64, fileName, "image/jpeg");
    }

    // STEP 2: Run existing punch logic (creates/updates sheet row)
    const punchResult = handlePunch(email, action, locationStr);

    // STEP 3: Save photo URL to correct column
    if (punchResult.success && photoUrl) {
        const ss    = SpreadsheetApp.openById(SS_ID);
        const sheet = ss.getSheetByName(SHEET_ATTENDANCE);
        const data  = sheet.getDataRange().getValues();
        const today = new Date().toLocaleDateString();
        const tEmail = email.trim().toLowerCase();

        for (let i = data.length - 1; i >= 1; i--) {
            const rowDate  = new Date(data[i][0]).toLocaleDateString();
            const rowEmail = String(data[i][3]).trim().toLowerCase();

            if (rowDate === today && rowEmail === tEmail) {
                if (action === 'IN') {
                    sheet.getRange(i + 1, 12).setValue(photoUrl); // Col L
                } else {
                    sheet.getRange(i + 1, 13).setValue(photoUrl); // Col M
                }
                break;
            }
        }
    }

    return punchResult;
}
```

---

## Part 5: Attendance Sheet Structure

```
Col A  → Date
Col B  → Employee ID
Col C  → Name
Col D  → Email
Col E  → In Time
Col F  → Out Time
Col G  → Hours
Col H  → Status
Col I  → Location
Col J  → Type of Leave
Col K  → Remark
Col L  → Punch In Photo   ← NEW (photo URL from Drive)
Col M  → Punch Out Photo  ← NEW (photo URL from Drive)
```

---

## Part 6: Data Flow Summary

```
PUNCH IN clicked
      │
      ▼
popup opens → github.io/punch-camera/?action=IN
      │
      ├── Camera starts (getUserMedia works on github.io)
      ├── User captures photo
      ├── Gets GPS coordinates
      ├── Converts photo to base64
      ├── postMessage({ action, location, imageBase64 }) → parent
      └── Popup closes
      │
      ▼
GAS window receives postMessage
      │
      ▼
google.script.run.handlePunchWithPhoto(email, 'IN', location, base64)
      │
      ├── uploadFileToDrive(base64) → returns Google Drive URL
      ├── handlePunch() → creates row in Attendance sheet
      └── sheet.getRange(row, 12).setValue(driveUrl) → Col L
```

---

## Part 7: Troubleshooting Reference

| Problem | Cause | Fix |
|---|---|---|
| `Permissions-Policy: camera=()` | GAS server header | Use external domain (GitHub Pages) |
| Popup blocked | Browser setting | Allow popups for script.google.com |
| postMessage not received | Popup vs Tab | Add BroadcastChannel as fallback |
| Photo saves to wrong column | IN/OUT not checked | Check `action === 'IN'` before saving |
| Drive upload fails | Wrong folder ID | Check `DRIVE_FOLDER_ID` in Code.gs |
| Camera black screen | Stream not attached | Check `video.srcObject = stream` |
| Base64 too large | High quality setting | Use `0.75` or `0.80` in `toDataURL` |

---

## Part 8: Reuse This Pattern For Anything Else

This same pattern works for **any browser API blocked by GAS**:

```
Blocked in GAS          →    Use GitHub Pages
─────────────────────────────────────────────
Camera (getUserMedia)   →    ✅ Works on github.io
Microphone recording    →    ✅ Works on github.io  
Bluetooth API           →    ✅ Works on github.io
USB API                 →    ✅ Works on github.io
File System Access API  →    ✅ Works on github.io

Communication back:
→ postMessage (popup)
→ BroadcastChannel (tab)
→ localStorage polling (last resort)
```

> **Golden Rule:** If a browser API is blocked in GAS, host that specific feature on GitHub Pages and communicate back via `postMessage` or `BroadcastChannel`. Your GAS app handles all data saving — GitHub Pages only handles the browser API interaction.

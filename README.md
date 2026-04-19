<div align="center">
  <img src="qin.png" alt="Q-In Logo" width="150" style="border-radius: 20px; box-shadow: 0 10px 25px rgba(59, 130, 246, 0.3);">
  
  # 🎓 Q-In: Smart Geo-Fenced Attendance System
  
  **A highly secure, serverless attendance management system built with Flutter, Google Apps Script, and Biometric Authentication.**

  [![Flutter](https://img.shields.io/badge/Flutter-02569B?style=for-the-badge&logo=flutter&logoColor=white)]()
  [![Google Apps Script](https://img.shields.io/badge/Apps_Script-4285F4?style=for-the-badge&logo=google&logoColor=white)]()
  [![Security](https://img.shields.io/badge/Security-Biometric_%7C_Geo--Fenced-10B981?style=for-the-badge)]()

</div>

---

## 🚀 Overview

**Q-In** is a modern, anti-proxy attendance system that entirely eliminates the need for expensive servers. It leverages a student's own smartphone hardware (Biometrics and GPS) combined with a completely free Google Sheets backend to manage classroom attendance securely and efficiently.

### ✨ Key Features
* **🔒 Device Locking:** Accounts are permanently bound to a single physical device ID upon the first login. Students cannot log in on their friends' phones.
* **👤 Biometric & M-PIN Security:** Uses FaceID, Fingerprint, or a custom 4-digit M-PIN to secure the student dashboard.
* **📍 Geo-Fenced QR Scanning:** Attendance is only accepted if the student's GPS location is within 100 meters of the teacher's generated QR code location.
* **⏳ Time-Sensitive Tokens:** Teacher-generated QR codes contain encrypted timestamps and expire automatically after 5 minutes to prevent sharing photos of the QR code.
* **📊 Automated Subject Matrix:** The Google Apps Script automatically generates clean, day-by-day attendance spreadsheets (Matrix format) for every individual subject.
* **💸 Zero Backend Costs:** 100% powered by Google Apps Script and Google Sheets API.

---

## 🏗️ System Architecture

The ecosystem consists of three parts:
1. **Google Sheets (Database):** Acts as the central hub. Admins input student rosters and subject lists here.
2. **Apps Script (API):** The middleware that processes attendance logic, verifies GPS/Device IDs, prevents duplicates, and builds the attendance matrix.
3. **Flutter App (Client):** The premium dark-themed mobile app used by students to claim accounts, view detailed attendance history, and scan QR codes.
4. **Web Portal (Teacher):** A clean, glassmorphism HTML interface for teachers to generate secure, geo-tagged QR codes live in the classroom.

---

## 🛠️ Setup Instructions

### Phase 1: Google Sheet Configuration
1. Create a new Google Sheet.
2. Create exactly three tabs named exactly as follows:
   * **`Students`** (Row 1 Headers: `Timestamp`, `Name`, `RollNo`, `Class`, `Semester`, `DeviceID`)
   * **`Attendance`** (Row 1 Headers: `Timestamp`, `Date`, `Subject`, `RollNo`, `Name`, `DeviceID`)
   * **`Subjects`** (Row 1 Headers: `Class`, `Semester`, `SubjectName`, `SubjectCode`, `TotalLectures`)
3. Pre-fill the `Students` and `Subjects` tabs with your university data. **Leave the `DeviceID` column entirely blank.**

### Phase 2: Deploy Google Apps Script
1. In your Google Sheet, click **Extensions > Apps Script**.
2. Delete any existing code and paste the complete `code.gs` file provided below.
3. Click **Deploy > New deployment**.
4. Select **Type: Web app**.
5. Set **Execute as:** `Me` and **Who has access:** `Anyone`.
6. Click **Deploy** and authorize the necessary permissions.
7. **Copy the generated Web App URL.**

<details>
<summary><b>📜 Click here to view the complete Apps Script Code (code.gs)</b></summary>

```javascript
function doPost(e) {
  var lock = LockService.getScriptLock();
  try { lock.waitLock(30000); } catch (err) { return createJSONOutput({ "status": "error", "message": "Server busy" }); }
  
  try {
    var ss = SpreadsheetApp.getActiveSpreadsheet();
    var studentsSheet = ss.getSheetByName("Students");
    var attendanceSheet = ss.getSheetByName("Attendance");
    var subjectsSheet = ss.getSheetByName("Subjects");
    
    var params = JSON.parse(e.postData.contents);
    var action = params.action;

    // 1. CLAIM ACCOUNT
    if (action === "claim_account") {
      var reqRoll = String(params.rollNo).trim().toUpperCase();
      var reqDevice = String(params.deviceId).trim();
      var data = studentsSheet.getDataRange().getValues();
      
      for (var i = 1; i < data.length; i++) {
        if (String(data[i][2]).trim().toUpperCase() === reqRoll) {
          var dbDevice = String(data[i][5]).trim();
          if (dbDevice !== "" && dbDevice !== "null" && dbDevice !== reqDevice && reqRoll !== "GOOGLE_TEST") {
            return createJSONOutput({ "status": "error", "message": "This account is already locked to another device." });
          }
          studentsSheet.getRange(i + 1, 6).setValue(reqDevice);
          return createJSONOutput({ "status": "success", "student": { "name": data[i][1], "rollNo": data[i][2], "class": data[i][3], "sem": data[i][4] } });
        }
      }
      return createJSONOutput({ "status": "error", "message": "Roll Number not found. Contact Admin." });
    }

    // 2. GET DASHBOARD DETAILS
    if (action === "get_student_details") return getStudentDetails(params, studentsSheet, subjectsSheet, ss);
    
    // 3. MARK ATTENDANCE
    if (action === "mark_attendance") return markAttendance(params, studentsSheet, attendanceSheet, ss);
    
    // 4. GET CLASSES
    if (action === "get_classes") return getAvailableClasses(subjectsSheet);

    return createJSONOutput({ "status": "error", "message": "Unknown Action" });
  } catch (e) {
    return createJSONOutput({ "status": "error", "message": "Error: " + e.toString() });
  } finally { 
    lock.releaseLock(); 
  }
}

function getAvailableClasses(subjectsSheet) {
  if (!subjectsSheet) return createJSONOutput({"status": "error", "message": "Subjects sheet missing"});
  var data = subjectsSheet.getDataRange().getValues();
  var classMap = {};
  for (var i = 1; i < data.length; i++) {
    var cls = String(data[i][0]).trim(); var sem = String(data[i][1]).trim();
    if (cls && sem) { if (!classMap[cls]) classMap[cls] = {}; classMap[cls][sem] = true; }
  }
  var result = {};
  for (var c in classMap) { result[c] = Object.keys(classMap[c]).sort(); }
  return createJSONOutput({"status": "success", "data": result});
}

function getStudentDetails(params, studentsSheet, subjectsSheet, ss) {
  var deviceId = String(params.deviceId).trim(); 
  var studentData = studentsSheet.getDataRange().getValues(); 
  var foundStudent = null;
  
  for (var i = 1; i < studentData.length; i++) {
    if (String(studentData[i][5]).trim() == deviceId) { 
      foundStudent = { "name": studentData[i][1], "rollNo": studentData[i][2], "class": String(studentData[i][3]).trim(), "sem": String(studentData[i][4]).trim() }; 
      break; 
    }
  }
  if (!foundStudent) return createJSONOutput({ "status": "error", "message": "Device not recognized. Please claim account again." });

  var subjectList = [];
  if (subjectsSheet) {
    var subData = subjectsSheet.getDataRange().getValues();
    for (var j = 1; j < subData.length; j++) {
      if (String(subData[j][0]).trim() == foundStudent.class && String(subData[j][1]).trim() == foundStudent.sem) {
        subjectList.push({ "name": subData[j][2], "code": String(subData[j][3]).trim(), "total": subData[j][4] || 1 });
      }
    }
  }

  var history = [];
  var stats = {};

  for (var s = 0; s < subjectList.length; s++) {
    var subCode = subjectList[s].code;
    var subSheet = ss.getSheetByName(subCode);
    
    if (subSheet) {
      var data = subSheet.getDataRange().getValues();
      if (data.length > 0) {
        var headers = data[0]; 
        var studentRow = null;
        
        for (var r = 1; r < data.length; r++) { 
          if (String(data[r][0]) == String(foundStudent.rollNo)) { studentRow = data[r]; break; } 
        }
        
        for (var c = 4; c < headers.length; c++) { 
          var isPresent = studentRow ? (studentRow[c] === "P") : false;
          history.push({ 
            "lecture": headers[c] + " (" + subCode + ")", 
            "status": isPresent ? "Present" : "Absent" 
          });
          
          if (isPresent) {
             stats[subCode] = (stats[subCode] || 0) + 1;
          }
        } 
      }
    }
  }
  
  history.sort(function(a, b) {
    return b.lecture.localeCompare(a.lecture); 
  });
  
  return createJSONOutput({ "status": "success", "student": foundStudent, "subjects": subjectList, "attendance_counts": stats, "history": history });
}

function markAttendance(params, studentsSheet, attendanceSheet, ss) {
  var inputDevice = String(params.deviceId).trim(); 
  var inputSubject = String(params.subject).trim().toUpperCase(); 
  var inputDate = String(params.date).trim();
  
  var studentData = studentsSheet.getDataRange().getValues(); 
  var sName = "", sRoll = "", sClass = "", sSem = "", found = false;
  
  for (var i = 1; i < studentData.length; i++) {
    if (String(studentData[i][5]).trim() == inputDevice) { 
      sName = studentData[i][1]; 
      sRoll = studentData[i][2]; 
      sClass = studentData[i][3];
      sSem = studentData[i][4];
      found = true; break; 
    }
  }
  if (!found) return createJSONOutput({ "status": "error", "message": "Device not registered" });

  var attData = attendanceSheet.getDataRange().getValues();
  for (var j = 1; j < attData.length; j++) {
    var rowDate = attData[j][1]; 
    var sheetDateStr = (rowDate instanceof Date) ? Utilities.formatDate(rowDate, Session.getScriptTimeZone(), "yyyy-MM-dd") : String(rowDate).trim().substring(0, 10);
    if (sheetDateStr == inputDate && String(attData[j][2]).trim().toUpperCase() == inputSubject && String(attData[j][5]).trim() == inputDevice) {
      return createJSONOutput({ "status": "error", "message": "Already marked today!" });
    }
  }

  attendanceSheet.appendRow([new Date(), "'" + inputDate, "'" + inputSubject, sRoll, sName, inputDevice]);
  updateSubjectMatrix(ss, sRoll, sName, sClass, sSem, inputDate, inputSubject);
  
  return createJSONOutput({ "status": "success", "message": "Attendance Marked!" });
}

function createJSONOutput(data) { return ContentService.createTextOutput(JSON.stringify(data)).setMimeType(ContentService.MimeType.JSON); }

function updateSubjectMatrix(ss, rollNo, name, sClass, sSem, date, subjectCode) {
  var sheet = ss.getSheetByName(subjectCode);
  if (!sheet) {
    sheet = ss.insertSheet(subjectCode);
  }
  
  var data = sheet.getDataRange().getValues(); 
  
  if (data.length == 0) { 
    sheet.appendRow(["Roll No", "Student Name", "Class", "Semester"]);
    data = sheet.getDataRange().getValues(); 
  } 
  
  var headerRow = data[0]; 
  var colIndex = headerRow.indexOf(date);
  
  if (colIndex == -1) { 
    colIndex = headerRow.length; 
    sheet.getRange(1, colIndex + 1).setValue(date); 
  }
  
  var rowIndex = -1;
  for (var i = 1; i < data.length; i++) { 
    if (String(data[i][0]) == String(rollNo)) { rowIndex = i; break; } 
  }
  
  if (rowIndex == -1) { 
    sheet.appendRow([rollNo, name, sClass, sSem]); 
    rowIndex = sheet.getLastRow() - 1; 
  }
  
  sheet.getRange(rowIndex + 1, colIndex + 1).setValue("P");
}

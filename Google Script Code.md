function doPost(e) {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var sheet = ss.getSheetByName("Sheet1");

  var data = JSON.parse(e.postData.contents);

  sheet.appendRow([
    new Date(),
    data.roll,
    data.name,
    data.dept,
    data.course,
    data.rfid,
    data.date,
    data.time
  ]);

  return ContentService.createTextOutput("Success");
}

// ======================================================
//  ESP32 RFID ATTENDANCE SYSTEM (FULL WORKING CODE)
//  LCD + MFRC522 + GOOGLE SHEET + LED + BUZZER
// ======================================================

#include <WiFi.h>
#include <HTTPClient.h>
#include <SPI.h>
#include <MFRC522.h>
#include <LiquidCrystal_I2C.h>

// ================= RFID Pins =================
#define SS_PIN 5
#define RST_PIN 2

// ================= LEDs & BUZZER =================
#define GREEN_LED 13
#define RED_LED 27
#define BUZZER 14

// ================= LCD =================
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ================= RFID Reader =================
MFRC522 rfid(SS_PIN, RST_PIN);

// ================= WiFi =================
const char* ssid = "INFINIX";
const char* password = "1234567890";

// ================= Google Script URL =================
String GOOGLE_SCRIPT_URL = "https://script.google.com/macros/s/AKfycbwXWVrpjuvCLyJbxOn_6AsuTDrQgokLAImt7XchEIE1nizIZ7mlcOn-w5NBLK-hNQf7/exec";

// ------------------------------------------------------
// STUDENT DATABASE
// ------------------------------------------------------
struct Student {
  String roll;
  String name;
  String dept;
  String course;
  String rfid;
};

// Add students here (RFID UID must be EXACT)
Student students[] = {
  {"101", "Rohit Singh", "CSE", "B.Tech", "93 F2 3B DA"},
  {"102", "Aman Verma", "ECE", "B.Tech", "7E 8C 33 2"},
};

int totalStudents = sizeof(students) / sizeof(students[0]);

bool findStudent(String uid, Student &found) {
  for (int i = 0; i < totalStudents; i++) {
    if (students[i].rfid == uid) {
      found = students[i];
      return true;
    }
  }
  return false;
}

// ------------------------------------------------------
// LCD Home Screen
// ------------------------------------------------------
void showHomeScreen() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("RFID Attendance");
  lcd.setCursor(0, 1);
  lcd.print("Scan Your Card...");
}

// ------------------------------------------------------
// SETUP
// ------------------------------------------------------
void setup() {
  Serial.begin(115200);

  lcd.init();
  lcd.backlight();

  pinMode(GREEN_LED, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  pinMode(BUZZER, OUTPUT);

  SPI.begin();
  rfid.PCD_Init();

  WiFi.begin(ssid, password);
  lcd.clear();
  lcd.print("Connecting WiFi");

  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }

  showHomeScreen();
  delay(1000);
}

// ------------------------------------------------------
// SEND DATA TO GOOGLE SHEETS
// ------------------------------------------------------
void sendToGoogle(String roll, String name, String dept, String course, String rfid) {
  if (WiFi.status() == WL_CONNECTED) {

    HTTPClient http;

    // Date & time from compile-time (you can enable NTP later)
    String date = __DATE__;
    String time = __TIME__;

    String jsonData = "{\"roll\":\"" + roll + 
                      "\", \"name\":\"" + name +
                      "\", \"dept\":\"" + dept +
                      "\", \"course\":\"" + course +
                      "\", \"rfid\":\"" + rfid +
                      "\", \"date\":\"" + date +
                      "\", \"time\":\"" + time + "\"}";

    http.begin(GOOGLE_SCRIPT_URL);
    http.addHeader("Content-Type", "application/json");

    int httpCode = http.POST(jsonData);
    String payload = http.getString();

    Serial.println("Response: " + payload);

    http.end();
  }
}

// ------------------------------------------------------
// LOOP
// ------------------------------------------------------
void loop() {
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial())
    return;

  // GET RFID UID
  String uid = "";
  for (byte i = 0; i < rfid.uid.size; i++) {
    uid += String(rfid.uid.uidByte[i], HEX);
    if (i < rfid.uid.size - 1) uid += " ";
  }
  uid.toUpperCase();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Card Detected!");
  lcd.setCursor(0, 1);
  lcd.print(uid);
  delay(1200);

  Student found;

  if (findStudent(uid, found)) {

    // VALID CARD → GREEN LED + BUZZER
    digitalWrite(GREEN_LED, HIGH);
    tone(BUZZER, 1200, 150);

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Welcome:");
    lcd.setCursor(0, 1);
    lcd.print(found.name);
    delay(1500);

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Attendance");
    lcd.setCursor(0, 1);
    lcd.print("Marked!");
    delay(1500);

    sendToGoogle(found.roll, found.name, found.dept, found.course, uid);

    digitalWrite(GREEN_LED, LOW);
  }
  else {
    // INVALID CARD → RED LED + BUZZER
    digitalWrite(RED_LED, HIGH);
    tone(BUZZER, 600, 300);

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Invalid Card!");
    lcd.setCursor(0, 1);
    lcd.print("Try Again");
    delay(2000);

    digitalWrite(RED_LED, LOW);
  }

  rfid.PICC_HaltA();
  showHomeScreen();
}

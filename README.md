#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// =================================================
// WIFI
// =================================================

const char* ssid = "YOUR_HOTSPOT_NAME";
const char* password = "YOUR_HOTSPOT_PASSWORD";

WebServer server(80);

// =================================================
// PINS
// =================================================

#define VOLTAGE_PIN 34
#define TEMP_PIN 32

#define MOTOR_IN1 26
#define MOTOR_IN2 27

// =================================================
// LCD
// =================================================

LiquidCrystal_I2C lcd(0x27, 16, 2);

// =================================================
// TEMPERATURE SENSOR
// =================================================

OneWire oneWire(TEMP_PIN);
DallasTemperature sensors(&oneWire);

// =================================================
// FIXED MOTOR CURRENT
// =================================================

float fixedCurrent = 0.50;
float currentLimit = 1.00;

// =================================================
// SOH
// =================================================

float SOH = 90.0;

// =================================================
// SOC CALCULATION
// =================================================

float calculateSOC(float batteryVoltage)
{
  float SOC;

  if (batteryVoltage >= 11.1)
  {
    SOC = 100.0;
  }
  else
  {
    SOC = 100.0 - ((11.1 - batteryVoltage) * 9.09);
  }

  if (SOC > 100) SOC = 100;
  if (SOC < 0) SOC = 0;

  return SOC;
}

// =================================================
// BATTERY STATUS
// =================================================

const char* getBatteryStatus(float SOC)
{
  if (SOC >= 80)
    return "NORMAL";

  else if (SOC >= 50)
    return "MEDIUM";

  else if (SOC > 20)
    return "LOW";

  else
    return "CRITICAL";
}

// =================================================
// WEB PAGE
// =================================================

void handleRoot()
{
  int voltageADC = analogRead(VOLTAGE_PIN);

  float sensorVoltage =
    (voltageADC * 3.3) / 4095.0;

  float batteryVoltage = sensorVoltage * 5.0;

  sensors.requestTemperatures();

  float temperature = sensors.getTempCByIndex(0);

  float SOC = calculateSOC(batteryVoltage);

  float current = fixedCurrent;

  if (current > currentLimit)
    current = currentLimit;

  String html = "";

  html += "<!DOCTYPE html>";
  html += "<html><head>";

  html += "<meta name='viewport' ";
  html += "content='width=device-width, initial-scale=1'>";

  html += "<title>EV Battery Monitoring</title>";

  html += "<style>";

  html += "body{font-family:Arial;";
  html += "background:#f2f2f2;";
  html += "text-align:center;padding:15px;}";

  html += "h1{color:#222;}";

  html += ".box{background:white;padding:18px;";
  html += "margin:12px auto;max-width:400px;";
  html += "border-radius:15px;";
  html += "box-shadow:0 2px 8px #aaa;}";

  html += ".value{font-size:25px;font-weight:bold;}";

  html += ".button{display:inline-block;";
  html += "padding:15px 20px;";
  html += "background:#4285F4;color:white;";
  html += "text-decoration:none;border-radius:10px;";
  html += "font-weight:bold;}";

  html += "</style></head><body>";

  html += "<h1>EV Battery Monitoring</h1>";

  // VOLTAGE

  html += "<div class='box'>";
  html += "<h2>Battery Voltage</h2>";
  html += "<div class='value'>";
  html += String(batteryVoltage, 2);
  html += " V</div></div>";

  // CURRENT

  html += "<div class='box'>";
  html += "<h2>Motor Current</h2>";
  html += "<div class='value'>";
  html += String(current, 2);
  html += " A</div>";

  html += "<p>Maximum Limit: ";
  html += String(currentLimit, 2);
  html += " A</p></div>";

  // TEMPERATURE

  html += "<div class='box'>";
  html += "<h2>Temperature</h2>";
  html += "<div class='value'>";

  if (temperature == -127.0)
  {
    html += "ERROR";
  }
  else
  {
    html += String(temperature, 1);
    html += " &deg;C";
  }

  html += "</div></div>";

  // SOC

  html += "<div class='box'>";
  html += "<h2>Battery SOC</h2>";
  html += "<div class='value'>";
  html += String(SOC, 1);
  html += " %</div>";

  html += "<p>Battery Status: ";
  html += getBatteryStatus(SOC);
  html += "</p></div>";

  // SOH

  html += "<div class='box'>";
  html += "<h2>Battery SOH</h2>";
  html += "<div class='value'>";
  html += String(SOH, 1);
  html += " %</div></div>";

  // CHARGING STATION

  html += "<div class='box'>";
  html += "<h2>Nearby Charging Station</h2>";

  html += "<a class='button' ";
  html += "href='https://www.google.com/maps/search/?api=1&query=EV+charging+station+near+me' ";
  html += "target='_blank'>";

  html += "Find Nearby Charging Station";
  html += "</a></div>";

  html += "</body></html>";

  server.send(200, "text/html", html);
}

// =================================================
// SETUP
// =================================================

void setup()
{
  Serial.begin(115200);

  analogReadResolution(12);

  // MOTOR RUNNING

  pinMode(MOTOR_IN1, OUTPUT);
  pinMode(MOTOR_IN2, OUTPUT);

  digitalWrite(MOTOR_IN1, HIGH);
  digitalWrite(MOTOR_IN2, LOW);

  // LCD

  lcd.init();
  lcd.backlight();

  // TEMPERATURE

  sensors.begin();

  // WIFI

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Connecting WiFi");

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.println("WiFi Connected");

  Serial.print("ESP32 IP: ");
  Serial.println(WiFi.localIP());

  // WEB SERVER

  server.on("/", handleRoot);
  server.begin();

  Serial.println("Web Server Started");

  // LCD WIFI SCREEN

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("WiFi Connected");

  delay(1000);

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("IP:");

  lcd.setCursor(0, 1);
  lcd.print(WiFi.localIP());

  delay(3000);
}

// =================================================
// LOOP
// =================================================

void loop()
{
  server.handleClient();

  // VOLTAGE

  int voltageADC = analogRead(VOLTAGE_PIN);

  float sensorVoltage =
    (voltageADC * 3.3) / 4095.0;

  float batteryVoltage = sensorVoltage * 5.0;

  // TEMPERATURE

  sensors.requestTemperatures();

  float temperature = sensors.getTempCByIndex(0);

  // SOC

  float SOC = calculateSOC(batteryVoltage);

  // CURRENT

  float current = fixedCurrent;

  if (current > currentLimit)
    current = currentLimit;

  // BATTERY STATUS

  const char* batteryStatus = getBatteryStatus(SOC);

  // SERIAL MONITOR

  Serial.println("-------------------------");

  Serial.print("Voltage: ");
  Serial.print(batteryVoltage, 2);
  Serial.println(" V");

  Serial.print("Current: ");
  Serial.print(current, 2);
  Serial.println(" A");

  Serial.print("Temperature: ");

  if (temperature == -127.0)
  {
    Serial.println("ERROR");
  }
  else
  {
    Serial.print(temperature, 1);
    Serial.println(" C");
  }

  Serial.print("SOC: ");
  Serial.print(SOC, 1);
  Serial.println(" %");

  Serial.print("Battery Status: ");
  Serial.println(batteryStatus);

  Serial.print("SOH: ");
  Serial.print(SOH, 1);
  Serial.println(" %");

  // =================================================
  // LCD SCREEN
  // =================================================

  lcd.clear();

  // FIRST LINE: SOC + STATUS

  lcd.setCursor(0, 0);

  lcd.print("SOC:");
  lcd.print((int)SOC);
  lcd.print("% ");
  lcd.print(batteryStatus);

  // SECOND LINE: TEMPERATURE

  lcd.setCursor(0, 1);

  lcd.print("Temp:");

  if (temperature == -127.0)
  {
    lcd.print("ERROR");
  }
  else
  {
    lcd.print(temperature, 1);
    lcd.print((char)223);
    lcd.print("C");
  }

  delay(2000);

  // LOW BATTERY WARNING

  if (SOC <= 20)
  {
    lcd.clear();

    lcd.setCursor(0, 0);
    lcd.print("LOW BATTERY!");

    lcd.setCursor(0, 1);
    lcd.print("Find Charger");

    delay(3000);
  }
}#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <OneWire.h>
#include <DallasTemperature.h>

// =================================================
// WIFI
// =================================================

const char* ssid = "YOUR_HOTSPOT_NAME";
const char* password = "YOUR_HOTSPOT_PASSWORD";

WebServer server(80);

// =================================================
// PINS
// =================================================

#define VOLTAGE_PIN 34
#define TEMP_PIN 32

#define MOTOR_IN1 26
#define MOTOR_IN2 27

// =================================================
// LCD
// =================================================

LiquidCrystal_I2C lcd(0x27, 16, 2);

// =================================================
// TEMPERATURE SENSOR
// =================================================

OneWire oneWire(TEMP_PIN);
DallasTemperature sensors(&oneWire);

// =================================================
// FIXED MOTOR CURRENT
// =================================================

float fixedCurrent = 0.50;
float currentLimit = 1.00;

// =================================================
// SOH
// =================================================

float SOH = 90.0;

// =================================================
// SOC CALCULATION
// =================================================

float calculateSOC(float batteryVoltage)
{
  float SOC;

  if (batteryVoltage >= 11.1)
  {
    SOC = 100.0;
  }
  else
  {
    SOC = 100.0 - ((11.1 - batteryVoltage) * 9.09);
  }

  if (SOC > 100) SOC = 100;
  if (SOC < 0) SOC = 0;

  return SOC;
}

// =================================================
// BATTERY STATUS
// =================================================

const char* getBatteryStatus(float SOC)
{
  if (SOC >= 80)
    return "NORMAL";

  else if (SOC >= 50)
    return "MEDIUM";

  else if (SOC > 20)
    return "LOW";

  else
    return "CRITICAL";
}

// =================================================
// WEB PAGE
// =================================================

void handleRoot()
{
  int voltageADC = analogRead(VOLTAGE_PIN);

  float sensorVoltage =
    (voltageADC * 3.3) / 4095.0;

  float batteryVoltage = sensorVoltage * 5.0;

  sensors.requestTemperatures();

  float temperature = sensors.getTempCByIndex(0);

  float SOC = calculateSOC(batteryVoltage);

  float current = fixedCurrent;

  if (current > currentLimit)
    current = currentLimit;

  String html = "";

  html += "<!DOCTYPE html>";
  html += "<html><head>";

  html += "<meta name='viewport' ";
  html += "content='width=device-width, initial-scale=1'>";

  html += "<title>EV Battery Monitoring</title>";

  html += "<style>";

  html += "body{font-family:Arial;";
  html += "background:#f2f2f2;";
  html += "text-align:center;padding:15px;}";

  html += "h1{color:#222;}";

  html += ".box{background:white;padding:18px;";
  html += "margin:12px auto;max-width:400px;";
  html += "border-radius:15px;";
  html += "box-shadow:0 2px 8px #aaa;}";

  html += ".value{font-size:25px;font-weight:bold;}";

  html += ".button{display:inline-block;";
  html += "padding:15px 20px;";
  html += "background:#4285F4;color:white;";
  html += "text-decoration:none;border-radius:10px;";
  html += "font-weight:bold;}";

  html += "</style></head><body>";

  html += "<h1>EV Battery Monitoring</h1>";

  // VOLTAGE

  html += "<div class='box'>";
  html += "<h2>Battery Voltage</h2>";
  html += "<div class='value'>";
  html += String(batteryVoltage, 2);
  html += " V</div></div>";

  // CURRENT

  html += "<div class='box'>";
  html += "<h2>Motor Current</h2>";
  html += "<div class='value'>";
  html += String(current, 2);
  html += " A</div>";

  html += "<p>Maximum Limit: ";
  html += String(currentLimit, 2);
  html += " A</p></div>";

  // TEMPERATURE

  html += "<div class='box'>";
  html += "<h2>Temperature</h2>";
  html += "<div class='value'>";

  if (temperature == -127.0)
  {
    html += "ERROR";
  }
  else
  {
    html += String(temperature, 1);
    html += " &deg;C";
  }

  html += "</div></div>";

  // SOC

  html += "<div class='box'>";
  html += "<h2>Battery SOC</h2>";
  html += "<div class='value'>";
  html += String(SOC, 1);
  html += " %</div>";

  html += "<p>Battery Status: ";
  html += getBatteryStatus(SOC);
  html += "</p></div>";

  // SOH

  html += "<div class='box'>";
  html += "<h2>Battery SOH</h2>";
  html += "<div class='value'>";
  html += String(SOH, 1);
  html += " %</div></div>";

  // CHARGING STATION

  html += "<div class='box'>";
  html += "<h2>Nearby Charging Station</h2>";

  html += "<a class='button' ";
  html += "href='https://www.google.com/maps/search/?api=1&query=EV+charging+station+near+me' ";
  html += "target='_blank'>";

  html += "Find Nearby Charging Station";
  html += "</a></div>";

  html += "</body></html>";

  server.send(200, "text/html", html);
}

// =================================================
// SETUP
// =================================================

void setup()
{
  Serial.begin(115200);

  analogReadResolution(12);

  // MOTOR RUNNING

  pinMode(MOTOR_IN1, OUTPUT);
  pinMode(MOTOR_IN2, OUTPUT);

  digitalWrite(MOTOR_IN1, HIGH);
  digitalWrite(MOTOR_IN2, LOW);

  // LCD

  lcd.init();
  lcd.backlight();

  // TEMPERATURE

  sensors.begin();

  // WIFI

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Connecting WiFi");

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.println("WiFi Connected");

  Serial.print("ESP32 IP: ");
  Serial.println(WiFi.localIP());

  // WEB SERVER

  server.on("/", handleRoot);
  server.begin();

  Serial.println("Web Server Started");

  // LCD WIFI SCREEN

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("WiFi Connected");

  delay(1000);

  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("IP:");

  lcd.setCursor(0, 1);
  lcd.print(WiFi.localIP());

  delay(3000);
}

// =================================================
// LOOP
// =================================================

void loop()
{
  server.handleClient();

  // VOLTAGE

  int voltageADC = analogRead(VOLTAGE_PIN);

  float sensorVoltage =
    (voltageADC * 3.3) / 4095.0;

  float batteryVoltage = sensorVoltage * 5.0;

  // TEMPERATURE

  sensors.requestTemperatures();

  float temperature = sensors.getTempCByIndex(0);

  // SOC

  float SOC = calculateSOC(batteryVoltage);

  // CURRENT

  float current = fixedCurrent;

  if (current > currentLimit)
    current = currentLimit;

  // BATTERY STATUS

  const char* batteryStatus = getBatteryStatus(SOC);

  // SERIAL MONITOR

  Serial.println("-------------------------");

  Serial.print("Voltage: ");
  Serial.print(batteryVoltage, 2);
  Serial.println(" V");

  Serial.print("Current: ");
  Serial.print(current, 2);
  Serial.println(" A");

  Serial.print("Temperature: ");

  if (temperature == -127.0)
  {
    Serial.println("ERROR");
  }
  else
  {
    Serial.print(temperature, 1);
    Serial.println(" C");
  }

  Serial.print("SOC: ");
  Serial.print(SOC, 1);
  Serial.println(" %");

  Serial.print("Battery Status: ");
  Serial.println(batteryStatus);

  Serial.print("SOH: ");
  Serial.print(SOH, 1);
  Serial.println(" %");

  // =================================================
  // LCD SCREEN
  // =================================================

  lcd.clear();

  // FIRST LINE: SOC + STATUS

  lcd.setCursor(0, 0);

  lcd.print("SOC:");
  lcd.print((int)SOC);
  lcd.print("% ");
  lcd.print(batteryStatus);

  // SECOND LINE: TEMPERATURE

  lcd.setCursor(0, 1);

  lcd.print("Temp:");

  if (temperature == -127.0)
  {
    lcd.print("ERROR");
  }
  else
  {
    lcd.print(temperature, 1);
    lcd.print((char)223);
    lcd.print("C");
  }

  delay(2000);

  // LOW BATTERY WARNING

  if (SOC <= 20)
  {
    lcd.clear();

    lcd.setCursor(0, 0);
    lcd.print("LOW BATTERY!");

    lcd.setCursor(0, 1);
    lcd.print("Find Charger");

    delay(3000);
  }
}

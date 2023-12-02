# IoT project for portfolio
Iot

#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>
#include <DHT.h>

#define DHTPIN D4          // pin for DHT11
#define DHTTYPE DHT11      // using DHT

#define motionSensorPin D1 // pin for motion detector
#define ledPin D2          // Pin for LED
#define fanPin1 D5         // Pin for DC motor (direction 1 )
#define fanPin2 D6         // Pin for DC motor (Pin for DC motor)

DHT dht(DHTPIN, DHTTYPE);
ESP8266WebServer server(80);

bool manualControl = false;
float setTemperature = 25.0; // Default temperature to turn on the DC motor

void handleRoot() {
  String page = "<html><body>";
  page += "<h1>ESP8266 Control</h1>";

  // Information about sensors and device status
  page += "<p>Motion Sensor: ";
  page += digitalRead(motionSensorPin) ? "Motion Detected" : "No Motion";
  page += "</p>";
  page += "<p>Temperature: ";
  page += dht.readTemperature();
  page += " °C</p>";

  // Manual control
  page += "<p>Manual Control: <input type=\"checkbox\" onchange=\"location.href='/manual?state='+(this.checked?1:0)\" ";
  page += manualControl ? "checked" : "";
  page += "></p>";

  // Set temperature for the DC motor
  page += "<form action='/temperature' method='GET'>";
  page += "<label for='temp'>Set Temperature:</label>";
  page += "<input type='text' id='temp' name='temperature'>";
  page += "<input type='submit' value='Set'>";
  page += "</form>";

  // Buttons for manual control
  page += "<form action='/led' method='GET'>";
  page += "<input type='submit' name='led' value='Turn LED On'>";
  page += "<input type='submit' name='led' value='Turn LED Off'>";
  page += "</form>";

  page += "<form action='/fan' method='GET'>";
  page += "<input type='submit' name='fan' value='Turn Fan On'>";
  page += "<input type='submit' name='fan' value='Turn Fan Off'>";
  page += "</form>";

  page += "</body></html>";
  server.send(200, "text/html", page);
}

void handleControl() {
  if (server.hasArg("led")) {
    if (server.arg("led") == "Turn LED On") {
      digitalWrite(ledPin, HIGH);
    } else if (server.arg("led") == "Turn LED Off") {
      digitalWrite(ledPin, LOW);
    }
  }

  if (server.hasArg("fan")) {
    if (server.arg("fan") == "Turn Fan On") {
      digitalWrite(fanPin1, HIGH);
      digitalWrite(fanPin2, LOW);
    } else if (server.arg("fan") == "Turn Fan Off") {
      digitalWrite(fanPin1, LOW);
      digitalWrite(fanPin2, LOW);
    }
  }

  server.sendHeader("Location", String("/"), true);
  server.send(303);
}

void handleManualControl() {
  if (server.arg("state") == "1") {
    manualControl = true;
  } else {
    manualControl = false;
  }
  server.send(200, "text/plain", "Manual Control Set");
}

void handleSetTemperature() {
  if (server.hasArg("temperature")) {
    setTemperature = server.arg("temperature").toFloat();
  }
  server.send(200, "text/plain", "Set Temperature");
}

void setup() {
  pinMode(motionSensorPin, INPUT);
  pinMode(ledPin, OUTPUT);
  pinMode(fanPin1, OUTPUT);
  pinMode(fanPin2, OUTPUT);

  digitalWrite(ledPin, LOW);
  digitalWrite(fanPin1, LOW);
  digitalWrite(fanPin2, LOW);

  dht.begin();
  WiFi.begin("Xiaomi", "13941394"); // enter SSID and password
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi..");
  }
  Serial.println("Connected to WiFi");

  server.on("/", handleRoot);
  server.on("/led", handleControl);
  server.on("/fan", handleControl);
  server.on("/manual", handleManualControl);
  server.on("/temperature", handleSetTemperature);

  server.begin();
}

void loop() {
  server.handleClient();
  if (!manualControl) {
    //Read data from sensors and automatic control
    if (digitalRead(motionSensorPin)) {
      digitalWrite(ledPin, HIGH);
    } else {
      digitalWrite(ledPin, LOW);
    }
    float temperature = dht.readTemperature();
    if (!isnan(temperature) && temperature > setTemperature) {
      digitalWrite(fanPin1, HIGH);
      digitalWrite(fanPin2, LOW);
    } else {
      digitalWrite(fanPin1, LOW);
      digitalWrite(fanPin2, LOW);
    }
  }
}

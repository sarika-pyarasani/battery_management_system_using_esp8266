# battery_management_system_using_esp8266
Automatically charges the battery and avoid power loss and fire accidents update the battery status in things board.

//code
#include <ESP8266WiFi.h>

// ThingSpeak API Configuration
String apiKey = "1IRRBSTB3X6EPKW4";  // Replace with your ThingSpeak API Key
const char* ssid = "thiruma";         // Replace with your WiFi SSID
const char* pass = "thiruma@31";      // Replace with your WiFi Password
const char* server = "api.thingspeak.com"; // ThingSpeak server

WiFiClient client;

// Battery Voltage Sensing Variables
const int analogInPin = A0;   // Analog input pin for battery voltage
float r1 = 1000.0;           // Resistor R1 (99kΩ)
float r2 = 1000.0;           // Resistor R2 (99kΩ)
float calibration = 0.36;     // Calibration value for voltage divider
float referenceVoltage = 3.3; // Reference voltage for ADC
int sensorValue;              // Analog output of sensor

// Battery State Variables
float battery_voltage = 0.0;
float volt = 0.0;
int bat_percentage = 0;
int flag = 1;                 // 0 = Charging, 1 = Discharging
int battery_connected = 0;    // 1 = Battery connected, 0 = Battery not connected

// Noise Filtering Variables
const int noiseThreshold = 30; // Noise threshold for ADC values
const float voltageThreshold = 0.5; // Minimum voltage to detect battery (0.5V)
const int numReadings = 10;    // Number of samples for averaging
int readings[numReadings];     // Array to store sensor readings
int readIndex = 0;             // Index of current reading
int total = 0;                 // Total of all readings
int average = 0;               // Average value of readings

void setup() {
  Serial.begin(115200);
  Serial.println("Connecting to WiFi...");
  WiFi.begin(ssid, pass);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected!");

  // Initialize readings array to 0
  for (int i = 0; i < numReadings; i++) {
    readings[i] = 0;
  }
}

void loop() {
  // Read analog value from battery
  sensorValue = analogRead(analogInPin);

  // Add the new reading to the total
  total -= readings[readIndex];
  readings[readIndex] = sensorValue;
  total += readings[readIndex];

  // Advance to the next position in the array
  readIndex = (readIndex + 1) % numReadings;

  // Calculate the average reading
  average = total / numReadings;

  // If the average is below the noise threshold, treat it as no battery connected
  if (average < noiseThreshold) {
    battery_voltage = 0.0;
    bat_percentage = 0;
    volt = 0.0;
    battery_connected = 0; // Battery not connected
    flag = 1; // Assume discharging state since no valid reading
  } else {
    // Calculate battery voltage when connected
    battery_voltage = (((average * referenceVoltage) / 1024) * 2 + calibration); // Multiply by 2 for voltage divider

    // Check if battery voltage exceeds the minimum threshold
    if (battery_voltage > voltageThreshold) {
      battery_connected = 1; // Battery is connected

      // Calculate battery percentage
      bat_percentage = mapFloat(battery_voltage, 2.8, 4.2, 0, 100); // Battery cutoff at 2.8V, max voltage at 4.2V
      bat_percentage = constrain(bat_percentage, 0, 100);

      // Calculate input voltage using voltage divider
      int analogvalue = analogRead(A0);
      float temp = (analogvalue * referenceVoltage) / 1024.0;
      volt = temp / (r2 / (r1 + r2));
      if (volt < 0.1) {
        volt = 0.0;
      }

      // Determine charging or discharging state
      if (volt >= 4.1) {
        flag = 0; // Charging
      } else if (volt <= 3.2) {
        flag = 1; // Discharging
      }
    } else {
      battery_connected = 0; // Treat as not connected
    }
  }

  // Send data to ThingSpeak
  if (client.connect(server, 80)) {
    String postStr = apiKey;

    // Field 1: Charging voltage
    postStr += "&field1=" + String(flag == 0 ? volt : 0.0);

    // Field 2: Discharging voltage
    postStr += "&field2=" + String(flag == 1 ? volt : 0.0);

    // Field 3: Battery voltage
    postStr += "&field3=" + String(battery_voltage);

    // Field 4: Battery percentage
    postStr += "&field4=" + String(bat_percentage);

    // Field 5: Battery connected (1 = connected, 0 = not connected)
    postStr += "&field5=" + String(battery_connected);

    postStr += "\r\n\r\n";

    client.print("POST /update HTTP/1.1\n");
    client.print("Host: api.thingspeak.com\n");
    client.print("Connection: close\n");
    client.print("X-THINGSPEAKAPIKEY: " + apiKey + "\n");
    client.print("Content-Type: application/x-www-form-urlencoded\n");
    client.print("Content-Length: " + String(postStr.length()) + "\n\n");
    client.print(postStr);

    Serial.println("Data sent to ThingSpeak.");
  }

  client.stop();

  // Print values to Serial Monitor for debugging
  Serial.print("Battery Voltage = ");
  Serial.print(battery_voltage);
  Serial.print(" V\tBattery Percentage = ");
  Serial.print(bat_percentage);
  Serial.print("%\tVoltage = ");
  Serial.print(volt);
  Serial.print(" V\tState = ");
  Serial.print(flag == 0 ? "Charging" : "Discharging");
  Serial.print("\tBattery Connected = ");
  Serial.println(battery_connected == 1 ? "Yes" : "No");

  delay(15000); // Update data every 15 seconds
}

// Helper function to map floating-point values
float mapFloat(float x, float in_min, float in_max, float out_min, float out_max) {
  return (x - in_min) * (out_max - out_min) / (in_max - in_min) + out_min;
}

#define BLYNK_TEMPLATE_ID "TMPL3XrFgp3XX"
#define BLYNK_TEMPLATE_NAME "Smart Waste Dustbin"
#define BLYNK_AUTH_TOKEN "M6JmZrFX-O0IcOcUDBRrP__koepEmBLP"
#define BLYNK_PRINT Serial

#include <ESP32Servo.h>
#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
#include <TinyGPS++.h>
#include <HardwareSerial.h>

// Blynk credentials
char auth[] = "M6JmZrFX-O0IcOcUDBRrP__koepEmBLP";
const char* ssid = "spi";
const char* password = "12345678";

// GPS

TinyGPSPlus gps;

// Pin Definitions
const int irSensorPin = 4;       // IR sensor pin for detecting dry waste
const int trigPin = 18;           // Ultrasonic sensor trig pin for waste level detection
const int echoPin = 19;           // Ultrasonic sensor echo pin for waste level detection
const int rainSensorPin = 15;     // Rain sensor pin (wet waste)
const int buzzer = 21;
// Define servo positions for waste sorting
const int wetWastePosition = 30;   // Servo position for wet waste
const int dryWastePosition = 150;  // Servo position for dry waste
const int defaultPosition = 90;   // Default servo position when no waste is detected

Servo sortingServo;  // Create a servo object

void setup() {
  // Start serial communication for debugging
 Serial.begin(115200);      // Serial Monitor for debugging
  Serial1.begin(9600, SERIAL_8N1, 16, 17);  // Initialize Serial1 for GPS (RX=GPIO16, TX=GPIO17)
  // Connect to Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("WiFi connected");
  
  // Start Blynk
  Blynk.begin(auth, ssid, password);

  // Attach the servo to pin 5
  sortingServo.attach(5);

  // Set the IR sensor pin and ultrasonic sensor pins as input/output
  pinMode(irSensorPin, INPUT);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(rainSensorPin, INPUT);
  pinMode(buzzer, OUTPUT);
  digitalWrite(buzzer, LOW);
 }

void loop() {
  // Run Blynk to process incoming messages
  Blynk.run();

  // Detect the type of waste and update the servo position
  if (isWetWasteDetected()) {
    sortingServo.write(wetWastePosition);  // Move servo to wet waste compartment
    Blynk.virtualWrite(V0, "Wet Waste Detected");
    Serial.println("Wet Waste Detected");
  }
  else if (isDryWasteDetected()) {
    sortingServo.write(dryWastePosition); // Move servo to dry waste compartment
    Blynk.virtualWrite(V0, "Dry Waste Detected");
    Serial.println("Dry Waste Detected");
  }
  else {
    sortingServo.write(defaultPosition);  // Move servo to default position
    Blynk.virtualWrite(V0, "No Waste Detected");
    Serial.println("No Waste Detected");
  }

  // Read the ultrasonic sensor value to determine the waste bin fullness level
  long distance = measureDistance(); 

  // Update the waste bin fullness status on Blynk
  updateWasteBinFullness(distance);

  // Read and update GPS coordinates
  updateGPSCoordinates();

  // Short delay before repeating the loop
  delay(500);
}

// Function to measure the distance for bin fullness
long measureDistance() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  // Measure the pulse duration from echo pin
  long duration = pulseIn(echoPin, HIGH);

  // Calculate distance in cm
  long distance = duration * 0.0344 / 2;
  return distance;
}

// Function to check if wet waste is detected using the rain sensor
bool isWetWasteDetected() {
  return digitalRead(rainSensorPin) == LOW;  // If rain sensor is LOW, it detects wet waste
}

// Function to check if dry waste is detected using the IR sensor
bool isDryWasteDetected() {
  // If the IR sensor is triggered (object detected)
  return digitalRead(irSensorPin) == LOW; // LOW signal when object is detected (IR beam interrupted)
}

// Function to update the waste bin fullness status on Blynk
void updateWasteBinFullness(long distance) {
  if (distance < 5) {  // Less than 20 cm: Full
    Blynk.virtualWrite(V1, "Full");
    Serial.println("Bin is Full");
    digitalWrite(buzzer, HIGH);
  } else if (distance < 15) {  // 20-50 cm: Half-Filled
    Blynk.virtualWrite(V1, "Half-Filled");
    Serial.println("Bin is Half-Filled");
  } else {  // More than 15 cm: Empty
    Blynk.virtualWrite(V1, "Empty");
    Serial.println("Bin is Empty");
    digitalWrite(buzzer, LOW);
  }
}

// Function to update GPS coordinates to Blynk
void updateGPSCoordinates() {
  // Process GPS data to get location
    while (Serial1.available() > 0) {
      gps.encode(Serial1.read());
    }

    // If a GPS fix is available, display the location
    if (gps.location.isUpdated()) {
      float latitude = gps.location.lat();
      float longitude = gps.location.lng();

      // Display GPS data on the Serial Monitor for debugging
      Serial.print("Latitude= "); 
      Serial.print(latitude, 6);
      Serial.print(" Longitude= "); 
      Serial.println(longitude, 6);

      // Update Blynk with GPS data
      Blynk.virtualWrite(V2, latitude);    // Field for latitude in Blynk
      Blynk.virtualWrite(V3, longitude);   // Field for longitude in Blynk
    }
}

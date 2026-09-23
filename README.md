# Seismic-Intrusion-Detection-System-
Detecting intrusions through ground vibrations where cameras fail. A low-cost embedded security system using ESP32 and two ADXL345 accelerometers to detect footsteps and ground disturbances, estimate their location along a 10-meter monitoring line, and send alerts to a Blynk 2.0 dashboard.




// --- CRITICAL BLYNK CONFIGURATION ---
#define BLYNK_TEMPLATE_ID   "TMPL40oSfYtgv"          
#define BLYNK_TEMPLATE_NAME "Seismic Intrusion Detector"
#define BLYNK_AUTH_TOKEN    "u4ilekJHNYPsQ1Tp77CiLiwzpxWO4x3u" 

#define BLYNK_PRINT Serial

#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>
#include <Wire.h>
#include <Adafruit_Sensor.h>
#include <Adafruit_ADXL345_U.h>

// --- Network Specifications ---
char auth[] = BLYNK_AUTH_TOKEN;
char ssid[] = "iotresearchlab";
char pass[] = "iotlab2023";

// --- Hardware Initialization ---
Adafruit_ADXL345_Unified sensorA = Adafruit_ADXL345_Unified(0x53); // Sensor A (0 meters)
Adafruit_ADXL345_Unified sensorB = Adafruit_ADXL345_Unified(0x1D); // Sensor B (10 meters)

BlynkTimer timer;

// --- Signal Processing Variables ---
const float totalDistance = 10.0;    
const float alertThreshold = 3.5;   // RECTIFIED: Increased from 1.2 to 3.5 to clear the noisy baseline spikes

// Dynamic Auto-Calibration Baselines
float baselineA = 0.0;
float baselineB = 0.0;

unsigned long lastAlertTime = 0;
const unsigned long cooldown = 5000; // 5-second notification lock

void checkSeismicArray() {
  sensors_event_t eventA, eventB;
  sensorA.getEvent(&eventA);
  sensorB.getEvent(&eventB);

  // Subtract the unique ambient baseline calculated during setup to read pure kinetic impacts
  float vibA = abs(eventA.acceleration.z - baselineA);
  float vibB = abs(eventB.acceleration.z - baselineB);

  // Isolate the highest current spike energy
  float currentMaxVibration = max(vibA, vibB);

  // 1. Send the clean, baseline-subtracted vibration value to your app graph (V0)
  Blynk.virtualWrite(V0, currentMaxVibration); 

  // 2. DYNAMIC DETECTION LOGIC
  if (vibA > alertThreshold || vibB > alertThreshold) {
    
    // Compute the ratio location exclusively during a threshold crossing event
    float totalVibration = vibA + vibB;
    float calculatedLocation = totalDistance / 2.0; 
    if (totalVibration > 0.05) {
      calculatedLocation = (vibB / totalVibration) * totalDistance;
    }

    // Boundary clamping
    if(calculatedLocation < 0.0) calculatedLocation = 0.0;
    if(calculatedLocation > totalDistance) calculatedLocation = totalDistance;

    // Display the specific breach parameters ONLY on a real threshold breach
    Serial.print("Vibration_Spike:");
    Serial.print(currentMaxVibration);
    Serial.print(" -> [!!! BREACH DETECTED !!! Distance: ");
    Serial.print(calculatedLocation);
    Serial.println("m]");

    // Update phone interface parameters
    Blynk.virtualWrite(V2, 1);                  // Set Status to Breach (1)
    Blynk.virtualWrite(V1, calculatedLocation);  // Send coordinates to V1

    if (millis() - lastAlertTime > cooldown) {
      lastAlertTime = millis();
      String alertMsg = "BREACH! Intruder detected at " + String(calculatedLocation, 1) + "m along perimeter!";
      Blynk.logEvent("intruder_alert", alertMsg);
    }
  } else {
    // Normal operations output: Shows clean ambient tracking without triggering alarms
    Serial.print("Vibration_Value:");
    Serial.print(currentMaxVibration);
    Serial.println(" -> [PERIMETER CLEAR]");
    
    Blynk.virtualWrite(V2, 0); // Set Status to Safe (0)
  }
}

void setup() {
  Serial.begin(115200);
  
  Blynk.begin(auth, ssid, pass);

  if(!sensorA.begin()) {
    Serial.println("Sensor A (0x53) failed to initialize! Check SDO to GND.");
    while(1);
  }
  
  if(!sensorB.begin()) {
    Serial.println("Sensor B (0x1D) failed to initialize! Check SDO to 3V3.");
    while(1);
  }

  sensorA.setRange(ADXL345_RANGE_2_G);
  sensorB.setRange(ADXL345_RANGE_2_G);

  // --- AUTO-CALIBRATION ROUTINE ---
  Serial.println("Calibrating sensors. Do not touch the desk...");
  float sumA = 0, sumB = 0;
  sensors_event_t event;
  
  for(int i = 0; i < 20; i++) {
    sensorA.getEvent(&event); sumA += event.acceleration.z;
    sensorB.getEvent(&event); sumB += event.acceleration.z;
    delay(50);
  }
  baselineA = sumA / 20.0;
  baselineB = sumB / 20.0;
  
  Serial.print("Calibration complete! Baselines -> A: ");
  Serial.print(baselineA);
  Serial.print(" | B: ");
  Serial.println(baselineB);

  timer.setInterval(50L, checkSeismicArray);
}

void loop() {
  Blynk.run();  
  timer.run();  
}
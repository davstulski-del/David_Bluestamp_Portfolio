# Routine Reinforcement Armband
This project is a wearable armband that uses an ESP32, sensors, and a vibration motor to monitor movement and conditions like pressure or temperature, then give feedback through haptic and sound alerts. It combines inputs from an accelerometer, FSR, and temperature sensor to detect events and trigger different alert patterns. Overall, it demonstrates how a microcontroller can sense user behavior and provide real-time reminders or assistance.

| **Engineer** | **School** | **Interest** | **Year** |
|:--:|:--:|:--:|:--:|
| David S. | Los Gatos High School | Electrical Engineering | Incoming Senior

![Headstone Image](<img width="1214" height="591" alt="image" src="https://github.com/user-attachments/assets/f52b9128-16da-466d-b755-3d0423b936eb" />
)
  
# Final Milestone

<iframe width="315" height="560"
  src="https://www.youtube.com/embed/zo8FEhIY_fE"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
  allowfullscreen>
</iframe>

# Second Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/IVqLNLeHLR0?si=VjFMG_JSA3n3SJ5q" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# First Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/6Q3Q7nljpsE?si=vUZ1TnC5lamlJ1i0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# Schematics 
<img width="1408" height="692" alt="image" src="https://github.com/user-attachments/assets/6f15fa93-3cdc-400d-b3ca-5b0652d648ba" />


# Code

```c++
#include <Wire.h>

// -------------------- PINS --------------------
const int motorPin = D9;
const int buzzerPin = D8;
const int buttonPin = D2;
const int tempPin = A0;

// -------------------- MPU6050 --------------------
const int MPU_ADDR = 0x68;
bool accelFound = false;
int16_t baseX, baseY, baseZ;

// -------------------- ALERT STATES --------------------
bool motionAlert = false;
bool tempAlert = false;
bool alertsMuted = false;

// -------------------- THRESHOLDS --------------------
const float motionThresholdG = 0.73;

// Temperature limits (°C)
const float lowTempC = 35.0;
const float highTempC = 38.0;

// -------------------- SETUP --------------------
void setup() {
    Serial.begin(115200);
    delay(2000);

    analogReadResolution(12);

    pinMode(motorPin, OUTPUT);
    pinMode(buzzerPin, OUTPUT);
    pinMode(buttonPin, INPUT_PULLUP);

    digitalWrite(motorPin, LOW);
    noTone(buzzerPin);

    Serial.println("Patient alert project starting...");

    Wire.begin();

    Wire.beginTransmission(MPU_ADDR);
    byte error = Wire.endTransmission();

    if (error == 0) {
        Serial.println("MPU6050 found!");
        accelFound = true;

        Wire.beginTransmission(MPU_ADDR);
        Wire.write(0x6B);
        Wire.write(0);
        Wire.endTransmission();

        delay(500);

        readAccelerometer(baseX, baseY, baseZ);
    } else {
        Serial.println("MPU6050 not found. Motion alert disabled.");
        accelFound = false;
    }
}

// -------------------- MAIN LOOP --------------------
void loop() {

    if (buttonPressed()) {
        clearAlerts();
        alertsMuted = true;
        Serial.println("Alerts muted.");
        delay(500);
        return;
    }

    checkMotion();
    checkTemperature();

    if (!motionAlert && !tempAlert) {
        alertsMuted = false;
    }

    if (alertsMuted) {
        stopOutputs();
        delay(100);
        return;
    }

    if (motionAlert && tempAlert) {
        playBothAlert();
    }
    else if (motionAlert) {
        playMotionAlert();
    }
    else if (tempAlert) {
        playTempAlert();
    }
    else {
        stopOutputs();
    }

    delay(100);
}

// -------------------- MOTION CHECK --------------------
void checkMotion() {

    if (!accelFound)
        return;

    int16_t x, y, z;

    readAccelerometer(x, y, z);

    int rawMovement = abs(x - baseX) +
                      abs(y - baseY) +
                      abs(z - baseZ);

    float movementG = rawMovement / 16384.0;

    Serial.print("Movement G: ");
    Serial.println(movementG);

    motionAlert = (movementG > motionThresholdG);
}

// -------------------- TEMPERATURE CHECK --------------------
void checkTemperature() {

    int raw = analogRead(tempPin);

    float voltage = raw * (3.3 / 4095.0);

    // TMP36 with your calibration
    float tempC = (voltage - 0.5) * 100.0 + 22.5;

    // Print exactly like before
    Serial.print("Raw ADC: ");
    Serial.print(raw);

    Serial.print("  Voltage: ");
    Serial.print(voltage, 3);

    Serial.print(" V  Temperature: ");
    Serial.print(tempC, 1);
    Serial.println(" C");

    // Alert only above 35°C
    if (tempC > 35.0) {
        tempAlert = true;
    } else {
        tempAlert = false;
    }
}

// -------------------- ALERT PATTERNS --------------------
void playMotionAlert() {

    digitalWrite(motorPin, HIGH);
    tone(buzzerPin, 1000);
    delay(600);

    digitalWrite(motorPin, LOW);
    noTone(buzzerPin);
    delay(400);
}

void playTempAlert() {

    for (int i = 0; i < 2; i++) {
        digitalWrite(motorPin, HIGH);
        tone(buzzerPin, 1500);
        delay(150);

        digitalWrite(motorPin, LOW);
        noTone(buzzerPin);
        delay(150);
    }

    delay(500);
}

void playBothAlert() {

    digitalWrite(motorPin, HIGH);
    tone(buzzerPin, 2000);
}

// -------------------- HELPER FUNCTIONS --------------------
void clearAlerts() {

    motionAlert = false;
    tempAlert = false;

    stopOutputs();
}

void stopOutputs() {

    digitalWrite(motorPin, LOW);
    noTone(buzzerPin);
}

bool buttonPressed() {

    return digitalRead(buttonPin) == LOW;
}

void readAccelerometer(int16_t &x, int16_t &y, int16_t &z) {

    Wire.beginTransmission(MPU_ADDR);
    Wire.write(0x3B);
    Wire.endTransmission(false);

    Wire.requestFrom(MPU_ADDR, 6, true);

    x = Wire.read() << 8 | Wire.read();
    y = Wire.read() << 8 | Wire.read();
    z = Wire.read() << 8 | Wire.read();
}

```

# Bill of Materials

| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| Arduino ESP32 | Small programmable board used to run code and connect sensors to Wi‑Fi/Bluetooth for electronics projects.r | $20 | <a href="https://a.co/d/0b9IyXSm/"> Link </a> |
| Resistive Force Sensor | Measures pressure or force by changing its electrical resistance when pressed. | $11.99 | <a href="https://a.co/d/0ilOJpfW/"> Link </a> |
| Vibrating Mini motor | Produces vibration for haptic feedback or alerts in small devices. | $5.99 | <a href="https://a.co/d/0dE3pviE/"> Link </a> |
| Acceleromater | Measures acceleration and orientation (motion/tilt) of an object. | $11.25 | <a href="https://a.co/d/02l1g7ch/"> Link </a> |
| USBC | Supplies power and transfers data between devices using a USB‑C connector. | $3.88 | <a href="https://a.co/d/0abhMgcf/"> Link </a> |
| Analog Temperature Sensor | Outputs a voltage that changes with temperature so a microcontroller can read temperature. | $12 | <a href="https://a.co/d/0feTFzoj/"> Link </a> |
| Armband | Holds sensors or a device on your arm securely during activity. | $5.5 | <a href="https://a.co/d/03qCI75b/"> Link </a> |
| Electronics Kit | A starter collection of components (LEDs, resistors, jumper wires, etc.) used to build and learn circuits. | $14 | <a href="https://a.co/d/07xxwG7f/"> Link </a> |
| 9V barrel jack | Connector used to plug a 9V power supply into a device or project. | $6 | <a href="https://a.co/d/0iUhgdwP/"> Link </a> |
| DMM | Measures voltage, current, and resistance to test and troubleshoot electronics. | $9.99 | <a href="https://a.co/d/056W56EX/"> Link </a> |
| 9V Batteries | Portable power source for small electronics and devices that accept 9V cells. | $12.37 | <a href="https://a.co/d/0iGcJrKB/"> Link </a> |

# Other Resources/Examples
- [Armband Notes](https://docs.google.com/document/d/1VrPdPcL-Z65wnyhI12oAp2Bm2EwKIptNxDfZ0WaQimU/edit?tab=t.0)
- [Temperature Sensor Wiring Guide](https://saliterman.umn.edu/sites/saliterman.umn.edu/files/files/general/tmp36_temperature_sensor_arduino_tutorial_2_examples.pdf)

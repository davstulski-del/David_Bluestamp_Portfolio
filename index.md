# Routine Reinforcement Armband
Replace this text with a brief description (2-3 sentences) of your project. This description should draw the reader in and make them interested in what you've built. You can include what the biggest challenges, takeaways, and triumphs from completing the project were. As you complete your portfolio, remember your audience is less familiar than you are with all that your project entails!

You should comment out all portions of your portfolio that you have not completed yet, as well as any instructions:
```HTML 
<!--- This is an HTML comment in Markdown -->
<!--- Anything between these symbols will not render on the published site -->
```

| **Engineer** | **School** | **Interest** | **Year** |
|:--:|:--:|:--:|:--:|
| David S. | Los Gatos High School | Electrical Engineering | Incoming Senior

**Replace the BlueStamp logo below with an image of yourself and your completed project. Follow the guide [here](https://tomcam.github.io/least-github-pages/adding-images-github-pages-site.html) if you need help.**

![Headstone Image](logo.svg)
  
# Final Milestone

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/F7M7imOVGug" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your final milestone, explain the outcome of your project. Key details to include are:
- What you've accomplished since your previous milestone
- What your biggest challenges and triumphs were at BSE
- A summary of key topics you learned about
- What you hope to learn in the future after everything you've learned at BSE



# Second Milestone

**Don't forget to replace the text below with the embedding for your milestone video. Go to Youtube, click Share -> Embed, and copy and paste the code to replace what's below.**

<iframe width="560" height="315" src="https://www.youtube.com/embed/y3VAmNlER5Y" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

For your second milestone, explain what you've worked on since your previous milestone. You can highlight:
- Technical details of what you've accomplished and how they contribute to the final goal
- What has been surprising about the project so far
- Previous challenges you faced that you overcame
- What needs to be completed before your final milestone 

# First Milestone

<iframe width="560" height="315" src="https://www.youtube.com/embed/6Q3Q7nljpsE?si=vUZ1TnC5lamlJ1i0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# Schematics 
Here's where you'll put images of your schematics. [Tinkercad](https://www.tinkercad.com/blog/official-guide-to-tinkercad-circuits) and [Fritzing](https://fritzing.org/learning/) are both great resoruces to create professional schematic diagrams, though BSE recommends Tinkercad becuase it can be done easily and for free in the browser. 

# Code
Here's where you'll put your code. The syntax below places it into a block of code. Follow the guide [here]([url](https://www.markdownguide.org/extended-syntax/)) to learn how to customize it to your project needs. 

```c++
#include <Wire.h>                 // For MPU6050 accelerometer communication

#include <OneWire.h>              // For DS18B20 temperature sensor

#include <DallasTemperature.h>    // Easier DS18B20 temperature readings
 // -------------------- PINS --------------------
const int motorPin = D9; // Vibration motor pin
const int buzzerPin = D8; // Piezo buzzer pin
const int buttonPin = D2; // Button pin
const int tempPin = D3; // DS18B20 data pin
// -------------------- MPU6050 --------------------
const int MPU_ADDR = 0x68; // MPU6050 default I2C address
bool accelFound = false; // Tracks if MPU6050 was detected
int16_t baseX, baseY, baseZ; // Starting/normal position
// -------------------- DS18B20 TEMP SENSOR --------------------
OneWire oneWire(tempPin); // OneWire connection on D3
DallasTemperature tempSensor( & oneWire); // Temperature sensor object
// -------------------- ALERT STATES --------------------
bool motionAlert = false; // True when motion alert is active
bool tempAlert = false; // True when temp alert is active
bool alertsMuted = false; // True after button press mutes alert
// -------------------- THRESHOLDS --------------------
// Same idea as raw threshold of 12000.
// 12000 / 16384 = about 0.73g
const float motionThresholdG = 0.73;
// Temperature range
const float lowTempC = 35.0;
const float highTempC = 38.0;
// -------------------- SETUP --------------------
void setup() {
    Serial.begin(115200); // Start Serial Monitor
    delay(2000); // Wait before starting
    pinMode(motorPin, OUTPUT); // Motor output
    pinMode(buzzerPin, OUTPUT); // Buzzer output
    pinMode(buttonPin, INPUT_PULLUP); // Button uses internal pull-up
    digitalWrite(motorPin, LOW); // Motor off
    noTone(buzzerPin); // Buzzer off
    Serial.println("Patient alert project starting...");
    // Start temperature sensor
    tempSensor.begin();
    Serial.print("DS18B20 sensors found: ");
    Serial.println(tempSensor.getDeviceCount());
    // Start accelerometer
    Wire.begin();
    // Check if MPU6050 is connected
    Wire.beginTransmission(MPU_ADDR);
    byte error = Wire.endTransmission();
    if (error == 0) {
        Serial.println("MPU6050 found!");
        accelFound = true;
        // Wake up MPU6050
        Wire.beginTransmission(MPU_ADDR);
        Wire.write(0x6B); // Power management register
        Wire.write(0); // Wake up sensor
        Wire.endTransmission();
        delay(500);
        // Save current position as normal
        readAccelerometer(baseX, baseY, baseZ);
    } else {
        Serial.println("MPU6050 not found. Motion alert disabled.");
        accelFound = false;
    }
}
// -------------------- MAIN LOOP --------------------
void loop() {
    // Button click stops/mutes the current alert
    if (buttonPressed()) {
        clearAlerts();
        alertsMuted = true;
        Serial.println("Alerts muted by button.");
        delay(500); // Simple debounce
        return;
    }
    checkMotion(); // Check motion
    checkTemperature(); // Check real DS18B20 temperature
    // If nothing is wrong anymore, allow future alerts again
    if (!motionAlert && !tempAlert) {
        alertsMuted = false;
    }
    // If alert was muted, keep outputs off
    if (alertsMuted) {
        stopOutputs();
        delay(100);
        return;
    }
    // Choose alert pattern
    if (motionAlert && tempAlert) {
        playBothAlert(); // Continuous beep
    } else if (motionAlert) {
        playMotionAlert(); // Long beep pattern
    } else if (tempAlert) {
        playTempAlert(); // Double beep pattern
    } else {
        stopOutputs(); // No alert
    }
    delay(100);
}
// -------------------- MOTION CHECK --------------------
void checkMotion() {
    if (!accelFound) return; // Skip if accelerometer missing
    int16_t x, y, z; // Current X/Y/Z values
    readAccelerometer(x, y, z); // Read accelerometer
    // Compare current position to starting position
    int rawMovement = abs(x - baseX) + abs(y - baseY) + abs(z - baseZ);
    // Convert raw MPU6050 movement to approximate g units
    float movementG = rawMovement / 16384.0;
    Serial.print("Movement G: ");
    Serial.println(movementG);
    // Trigger alert if movement is above threshold
    if (movementG > motionThresholdG) {
        motionAlert = true;
    }
}
// -------------------- TEMPERATURE CHECK --------------------
void checkTemperature() {
    tempSensor.requestTemperatures(); // Ask DS18B20 for temp
    float tempC = tempSensor.getTempCByIndex(0); // Read first sensor
    Serial.print("Temp C: ");
    Serial.println(tempC);
    // -127 means sensor is disconnected/not detected
    if (tempC == DEVICE_DISCONNECTED_C) {
        Serial.println("Temperature sensor not detected!");
        tempAlert = true;
        return;
    }
    // Trigger temp alert if outside range
    if (tempC < lowTempC || tempC > highTempC) {
        tempAlert = true;
    }
}
// -------------------- ALERT PATTERNS --------------------
void playMotionAlert() {
    // Pattern: beeeep ... beeeep ... beeeep
    digitalWrite(motorPin, HIGH);
    tone(buzzerPin, 1000);
    delay(600);
    digitalWrite(motorPin, LOW);
    noTone(buzzerPin);
    delay(400);
}
void playTempAlert() {
    // Pattern: beepbeep ... beepbeep ... beepbeep
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
    // Pattern: continuous beeeeeeeeeep
    digitalWrite(motorPin, HIGH);
    tone(buzzerPin, 2000);
}
// -------------------- HELPER FUNCTIONS --------------------
void clearAlerts() {
    motionAlert = false; // Clear motion alert
    tempAlert = false; // Clear temp alert
    stopOutputs(); // Turn off motor/buzzer
}
void stopOutputs() {
    digitalWrite(motorPin, LOW); // Motor off
    noTone(buzzerPin); // Buzzer off
}
bool buttonPressed() {
    // INPUT_PULLUP means pressed = LOW
    return digitalRead(buttonPin) == LOW;
}
void readAccelerometer(int16_t & x, int16_t & y, int16_t & z) {
    // Tell MPU6050 we want accelerometer data
    Wire.beginTransmission(MPU_ADDR);
    Wire.write(0x3B); // First accelerometer register
    Wire.endTransmission(false);
    // Request 6 bytes: X, Y, Z
    Wire.requestFrom(MPU_ADDR, 6, true);
    // Combine high and low bytes for each axis
    x = Wire.read() << 8 | Wire.read();
    y = Wire.read() << 8 | Wire.read();
    z = Wire.read() << 8 | Wire.read();
}

```

# Bill of Materials
Here's where you'll list the parts in your project. To add more rows, just copy and paste the example rows below.
Don't forget to place the link of where to buy each component inside the quotation marks in the corresponding row after href =. Follow the guide [here]([url](https://www.markdownguide.org/extended-syntax/)) to learn how to customize this to your project needs. 

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
One of the best parts about Github is that you can view how other people set up their own work. Here are some past BSE portfolios that are awesome examples. You can view how they set up their portfolio, and you can view their index.md files to understand how they implemented different portfolio components.
- [Example 1](https://trashytuber.github.io/YimingJiaBlueStamp/)
- [Example 2](https://sviatil0.github.io/Sviatoslav_BSE/)
- [Example 3](https://arneshkumar.github.io/arneshbluestamp/)

To watch the BSE tutorial on how to create a portfolio, click here.

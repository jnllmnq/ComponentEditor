# circuito.io// Pin definitions
const int rainSensorPin = A0;    // Analog pin connected to the rain sensor
const int motorPin1 = 9;         // Motor driver input 1
const int motorPin2 = 10;        // Motor driver input 2
const int buttonPin1 = 7;        // Push button 1 pin (for mode switch)
const int buttonPin2 = 8;        // Push button 2 pin (for forward motor control)
const int buttonPin3 = 11;       // Push button 3 pin (for reverse motor control)
const int greenLED = 3;          // Green LED pin for manual mode indicator
const int yellowLED = 4;         // Yellow LED pin for automatic mode indicator
const int lightSensorPin = A1;   // Analog pin connected to the photoresistor

const int dryThreshold = 800;    // Threshold for dry condition (adjust based on testing)
const int rainThreshold = 500;   // Threshold for rain condition (adjust based on testing)
bool rainDetected = false;       // Tracks whether rain was previously detected

bool buttonState1 = false;       // Current state for button 1 (mode switch)
bool lastButtonState1 = false;   // Previous state for button 1
unsigned long lastDebounceTime1 = 0; // Debounce timer for button 1
const unsigned long debounceDelay = 50; // Debounce delay in milliseconds

bool buttonState2 = false;       // Current state for button 2 (forward motor control)
bool lastButtonState2 = false;   // Previous state for button 2
unsigned long lastDebounceTime2 = 0; // Debounce timer for button 2

bool buttonState3 = false;       // Current state for button 3 (reverse motor control)
bool lastButtonState3 = false;   // Previous state for button 3
unsigned long lastDebounceTime3 = 0; // Debounce timer for button 3

bool isAutomaticMode = true;     // Flag for mode selection (true = automatic, false = manual)
int motorState = 0;              // Variable to track the motor state in manual mode (0 = stopped, 1 = forward, 2 = reverse)

bool lightTriggered = false;     // Flag to ensure motor runs only once for light change

void setup() {
  pinMode(motorPin1, OUTPUT);    // Set motor pins as output
  pinMode(motorPin2, OUTPUT);
  pinMode(buttonPin1, INPUT_PULLUP); // Set button 1 pin as input with internal pull-up resistor
  pinMode(buttonPin2, INPUT_PULLUP); // Set button 2 pin as input with internal pull-up resistor
  pinMode(buttonPin3, INPUT_PULLUP); // Set button 3 pin as input with internal pull-up resistor
  pinMode(greenLED, OUTPUT);     // Set green LED pin as output for manual mode
  pinMode(yellowLED, OUTPUT);    // Set yellow LED pin as output for automatic mode
  pinMode(lightSensorPin, INPUT); // Set photoresistor pin as input
  Serial.begin(9600);            // Initialize serial monitor for debugging
}

void loop() {
  // Handle button 1 (Mode toggle button) debounce
  int reading1 = digitalRead(buttonPin1);
  if (reading1 != lastButtonState1) {
    lastDebounceTime1 = millis(); // Update debounce timer for button 1
  }

  if ((millis() - lastDebounceTime1) > debounceDelay) {
    if (reading1 != buttonState1) {
      buttonState1 = reading1;

      // Toggle the mode on button 1 press (LOW state)
      if (buttonState1 == LOW) {
        if (isAutomaticMode) {
          // Switch to manual mode
          isAutomaticMode = false;
          motorState = 0;  // Stop the motor when switching to manual mode
          stopMotor();
          Serial.println("Switching to MANUAL MODE");
        } else {
          // Switch to automatic mode
          isAutomaticMode = true;
          Serial.println("Switching to AUTOMATIC MODE");
        }
      }
    }
  }

  // Update last button state for button 1
  lastButtonState1 = reading1;

  // Handle button 2 (Motor forward control) debounce
  int reading2 = digitalRead(buttonPin2);
  if (reading2 != lastButtonState2) {
    lastDebounceTime2 = millis(); // Update debounce timer for button 2
  }

  if ((millis() - lastDebounceTime2) > debounceDelay) {
    if (reading2 != buttonState2) {
      buttonState2 = reading2;

      // If in manual mode, motor moves forward on button 2 press (LOW state)
      if (buttonState2 == LOW && !isAutomaticMode) {
        motorState = 1;  // Set motor to forward
        runMotorForward();
        Serial.println("Motor FORWARD");
      }
    }
  }

  // Update last button state for button 2
  lastButtonState2 = reading2;

  // Handle button 3 (Motor reverse control) debounce
  int reading3 = digitalRead(buttonPin3);
  if (reading3 != lastButtonState3) {
    lastDebounceTime3 = millis(); // Update debounce timer for button 3
  }

  if ((millis() - lastDebounceTime3) > debounceDelay) {
    if (reading3 != buttonState3) {
      buttonState3 = reading3;

      // If in manual mode, motor moves in reverse on button 3 press (LOW state)
      if (buttonState3 == LOW && !isAutomaticMode) {
        motorState = 2;  // Set motor to reverse
        runMotorReverse();
        Serial.println("Motor REVERSE");
      }
    }
  }

  // Update last button state for button 3
  lastButtonState3 = reading3;

  // Update LEDs based on the mode
  if (isAutomaticMode) {
    digitalWrite(greenLED, LOW);  // Turn off green LED (manual mode)
    digitalWrite(yellowLED, HIGH); // Turn on yellow LED (automatic mode)
  } else {
    digitalWrite(greenLED, HIGH);  // Turn on green LED (manual mode)
    digitalWrite(yellowLED, LOW);  // Turn off yellow LED (automatic mode)
  }

  if (isAutomaticMode) {
    // Automatic mode: Use the rain sensor and photoresistor to control the motor
    int rainValue = analogRead(rainSensorPin);  // Read the rain sensor value
    int lightValue = analogRead(lightSensorPin); // Read the light sensor value
    Serial.print("Rain Sensor Value: ");
    Serial.println(rainValue);  // Print rain sensor value to monitor
    Serial.print("Light Sensor Value: ");
    Serial.println(lightValue);  // Print light sensor value to monitor

    // Check for rain condition
    if (rainValue < rainThreshold && !rainDetected) {
      // Rain detected
      Serial.println("Rain detected! Running motor forward...");
      rainDetected = true;
      runMotorForward();  // Run motor forward
    } 
    // Check for dry condition
    else if (rainValue > dryThreshold && rainDetected) {
      // Sensor is completely dry
      Serial.println("No rain detected! Running motor reverse...");
      rainDetected = false;
      runMotorReverse();  // Run motor reverse
    }

    // Check for light condition (photoresistor)
    if (lightValue < 100 && !lightTriggered) { // If light is dim
      Serial.println("Light is dim! Running motor forward...");
      runMotorForward();  // Run motor forward for 2 seconds
      lightTriggered = true;  // Set flag to prevent further triggering
    } 
    else if (lightValue >= 400 && lightTriggered) { // If light is bright
      Serial.println("Light is bright! Running motor reverse...");
      runMotorReverse();  // Run motor reverse for 2 seconds
      lightTriggered = false;  // Reset flag to allow forward motion again if light dims
    }
  }

  delay(500);  // Short delay for responsiveness
}

// Function to run motor forward
void runMotorForward() {
  digitalWrite(motorPin1, HIGH);  // Rotate motor forward
  digitalWrite(motorPin2, LOW);
  delay(1100);                   // Run motor for 1.1 seconds
  stopMotor();                   // Stop motor
}

// Function to run motor reverse
void runMotorReverse() {
  digitalWrite(motorPin1, LOW);  // Rotate motor in reverse
  digitalWrite(motorPin2, HIGH);
  delay(1100);                   // Run motor for 1.1 seconds
  stopMotor();                   // Stop motor
}

// Function to stop motor
void stopMotor() {
  digitalWrite(motorPin1, LOW);  // Stop motor
  digitalWrite(motorPin2, LOW);
} Component Editor

## Video Tutorial
[![Intro Video](https://res.cloudinary.com/circuito/image/upload/w_300,b_white/v1550053341/circuito_youtube_help_title.png)](https://www.youtube.com/watch?v=i3CpeFhRLI4)

## Using GitPod - Recommended
[Edit using GitPod](http://gitpod.io/#https://github.com/Circuito-io/ComponentEditor)

## Creating a Local Development Environment
1. Make sure you have [Node.js](https://nodejs.org/en/download/)
2. Clone this repository and cd into it
```bash
git clone https://github.com/Circuito-io/ComponentEditor.git
cd ComponentEditor
```
3. Init and update submodule
```bash
git submodule init
git submodule update
```
3. Run npm install
```bash
npm install --legacy-peer-deps
```
4. Run the dev web server
```bash
npm run dev
```
5. Connect to the web server - http://localhost:8080
6. Edit your files locally - components are in the ```components``` subfloder
7. When ready - click the 'Preview' button to sync local files with our server and open your **private** circuito.io preview window

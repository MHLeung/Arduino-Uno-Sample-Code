# HW
HW list
/* ============================================================
   PROJECT: sample code for notes 
   BEHAVIOUR:
   - IR sensor "tap" arms the system: cover IR briefly to arm.
   - After arming, the user has a time window to press the button.
   - LED ON while IR is covered; LED blinks on dispense.
   - On dispense: blink -> jiggle -> rotate 360 -> wait button release
   ============================================================ */

// ---------------- PIN ASSIGNMENTS ----------------
int mp1 = 11;
int mp2 = 10;
int mp3 = 6;
int mp4 = 5;

int led_red   = 7;
int IR_sensor = A1;
int buttonPin = 2;

// ---------------- SYSTEM SETTINGS ----------------
int IR_THRESHOLD = 500;

// Motor timing (smaller = faster)
int startDelay  = 45;
int runDelay    = 10;
int jiggleDelay = 6;
int jiggleCount = 12;

int settle_us = 0;

// Motion constants
const int CYCLES_PER_REV = 12;  // 360°

// Arming behaviour
const unsigned long ARM_HOLD_MS   = 800;   // IR must be covered ~1s to arm
const unsigned long ARM_WINDOW_MS = 5000;  // time allowed to press button after arming

// ---------------- STATE ----------------
bool lastIrCovered = false;
unsigned long irCoveredSince = 0;

bool armed = false;
unsigned long armedAt = 0;

bool lastButton = HIGH; // INPUT_PULLUP: HIGH = not pressed, LOW = pressed

// ---------------- MOTOR STEP FUNCTION ----------------
void motor_step(bool a1, bool a2, bool b1, bool b2, int dly) {
  digitalWrite(mp1, a1);
  digitalWrite(mp2, a2);
  digitalWrite(mp3, b1);
  digitalWrite(mp4, b2);
  delay(dly);
  if (settle_us > 0) delayMicroseconds(settle_us);
}

void motor_off() {
  digitalWrite(mp1, LOW);
  digitalWrite(mp2, LOW);
  digitalWrite(mp3, LOW);
  digitalWrite(mp4, LOW);
}

// ---------------- FULL-STEP SEQUENCE (2-COIL ON) ----------------
void one_cycle_CCW(int dly) {
  motor_step(HIGH, LOW,  HIGH, LOW, dly);
  motor_step(LOW,  HIGH, HIGH, LOW, dly);
  motor_step(LOW,  HIGH, LOW,  HIGH, dly);
  motor_step(HIGH, LOW,  LOW,  HIGH, dly);
}

// ---------------- ANTI-JAM VIBRATION ----------------
void vibrate_antiJam() {
  for (int i = 0; i < jiggleCount; i++) {
    one_cycle_CCW(jiggleDelay);

    motor_step(HIGH, LOW,  LOW,  HIGH, jiggleDelay);
    motor_step(LOW,  HIGH, LOW,  HIGH, jiggleDelay);
    motor_step(LOW,  HIGH, HIGH, LOW,  jiggleDelay);
    motor_step(HIGH, LOW,  HIGH, LOW,  jiggleDelay);
  }
  motor_off();
}

// ---------------- DISPENSE ROTATION (360°) ----------------
void rotate_360() {
  for (int i = 0; i < CYCLES_PER_REV; i++) {
    int dly = (i < 2) ? (startDelay - (i * 15)) : runDelay; // quick ramp
    one_cycle_CCW(dly);
  }
  motor_off();
}

// ---------------- LED FEEDBACK ----------------
void blinkLED() {
  for (int i = 0; i < 3; i++) {
    digitalWrite(led_red, HIGH); delay(150);
    digitalWrite(led_red, LOW);  delay(150);
  }
}

// ---------------- INITIALIZATION ----------------
void setup() {
  pinMode(mp1, OUTPUT);
  pinMode(mp2, OUTPUT);
  pinMode(mp3, OUTPUT);
  pinMode(mp4, OUTPUT);

  pinMode(led_red, OUTPUT);
  pinMode(buttonPin, INPUT_PULLUP);

  Serial.begin(9600);
}

// ---------------- MAIN CONTROL LOOP ----------------
void loop() {
  unsigned long now = millis();

  int irValue = analogRead(IR_sensor);
  bool irCovered = (irValue > IR_THRESHOLD);

  bool buttonNow = digitalRead(buttonPin); // HIGH=not pressed, LOW=pressed
  bool buttonPressedEdge = (lastButton == HIGH && buttonNow == LOW);

  // LED ON only while IR is physically covered
  digitalWrite(led_red, irCovered ? HIGH : LOW);

  // Track when IR became covered
  if (!lastIrCovered && irCovered) {
    irCoveredSince = now;
  }

  // If IR stays covered long enough, ARM the system
  if (irCovered && !armed && (now - irCoveredSince >= ARM_HOLD_MS)) {
    armed = true;
    armedAt = now;
  }

  // Auto-disarm if the arming window expires
  if (armed && (now - armedAt > ARM_WINDOW_MS)) {
    armed = false;
  }

  // Dispense if armed and the button is pressed (IR does NOT need to remain covered)
  if (armed && buttonPressedEdge) {
    blinkLED();
    vibrate_antiJam();
    rotate_360();

    while (digitalRead(buttonPin) == LOW) { delay(10); } // wait release
    armed = false; // require another IR "tap" to arm again
  }

  lastIrCovered = irCovered;
  lastButton = buttonNow;

  delay(5);
}

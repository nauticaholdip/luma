#include <Servo.h>
#include <SoftwareSerial.h>
#include <DFRobotDFPlayerMini.h>
#include <Adafruit_NeoPixel.h>
#include "ProtoPieHandler.h" // Local file for ProtoPie communication

// PINS
#define PIN_LDR A5 // Photoresistor
#define PIN_BTN_NEXT 2 // Button
#define PIN_NEOPIXEL 3 // Neopixel Ring
#define PIN_SERVO 5 // Servo Motor
#define PIN_TOUCH 7 // Touch Sensor
#define PIN_DFP_RX 10 // DFPlayer Mini RX
#define PIN_DFP_TX 11 // Player Mini TX
#define PIN_TRIG 12 // Ultrasonic Sensor Trigger
#define PIN_ECHO 13 // Ultrasonic Sensor Echo

// CONFIGURATION
#define NUM_PIXELS 8
#define SERVO_OPENED 0
#define SERVO_CLOSED 180
#define SERVO_SPEED_MS 40 // Time (ms) between each degree of movement
#define THRESHOLD_LDR 30
#define DISTANCE_MAX_CM 110 // Max distance to consider user "present" during stretching

// TIMING (ms)
#define TIME_STRETCH_INTRO 10000 
#define TIME_STRETCH_SETUP 5000 
#define TIME_STRETCH_EX 180000 
#define TIME_BREATHING_INTRO 60000 
#define TIME_BREATHING_EX 112000 
#define TIME_BREATHING_OUTRO 10000 

// PEARL COLOR CONSTANTS
#define PEARL_R 255
#define PEARL_G 150
#define PEARL_B 50

// MP3 TRACKS
#define TRACK_OPEN_CLOSE 1
#define TRACK_WAVES 2
#define TRACK_CLICK 3
#define TRACK_STRETCH_INTRO 4
#define TRACK_STRETCH_EX 5
#define TRACK_BREATH_INTRO 6
#define TRACK_BREATH_EX 7
#define TRACK_BREATH_OUTRO 8

// OBJECTS
Servo myServo;
SoftwareSerial dfpSerial(PIN_DFP_RX, PIN_DFP_TX);
DFRobotDFPlayerMini myDFPlayer;
Adafruit_NeoPixel ring(NUM_PIXELS, PIN_NEOPIXEL, NEO_GRB + NEO_KHZ800);

// STATE MACHINE to manage states of the ritual
enum LumaState {
  ST_CLOSED, // Idle state
  ST_RITUAL_START, // Opening movement
  ST_STRETCHING, // Ultrasonic-monitored exercise
  ST_BREATHING, // Light-guided breathing
  ST_GRATITUDE, // Final reflection
  ST_END // Closing movement
};
LumaState currentState = ST_CLOSED; // Tracks the current active ritual phase

// GLOBAL VARIABLES
unsigned long stateStartTime = 0; // When a new state begins
unsigned long lastBtnTime = 0; // Last button press time for debouncing
unsigned long lastServoTime = 0; // Last time servo moved 1 degree
bool isShellOpen = false; // TRUE = Open, FALSE = Closed
int currentServoPos = SERVO_CLOSED; // Tracks current angle of the servo

// STRETCHING VARIABLES
int stretchPhase = 0; // 0=Intro, 1=Setup, 2=Exercise
unsigned long stretchPhaseStartTime = 0; // Timer for stretching sub-phases
unsigned long totalStretchTime = 0; // Counts time when user is in range
unsigned long lastStretchUpdate = 0; // Calculate time difference between loop cycles
int outOfRangeCounter = 0; // Counts missed sensor readings to filter noise
bool isStretchingPaused = false; // TRUE if the user is too far
bool stretchDoneSent = false; // Prevent looping the "End" signal

// BREATHING VARIABLES
unsigned long breathingExStartTime = 0; // Used to sync lights to audio
unsigned long breathingOutroStartTime = 0; // Controls timing for the start of the gratitude phase
bool isBreathingExActive = false; // TRUE if photoresistor triggered
bool isBreathingOutroActive = false; // TRUE if TIME_BREATHING_EX ends
bool breathingDoneSent = false; // Prevent looping the "End" signal

void setup() {
  Serial.begin(9600);
  Serial.setTimeout(10); // Quick serial reading for ProtoPie
  dfpSerial.begin(9600);

  pinMode(PIN_TOUCH, INPUT_PULLUP);
  pinMode(PIN_BTN_NEXT, INPUT_PULLUP);
  pinMode(PIN_TRIG, OUTPUT);
  pinMode(PIN_ECHO, INPUT);

  // CONFIGURE NEOPIXEL
  ring.begin();
  ring.setBrightness(150);
  ring.show();

  // CONFIGURE SERVO to closed position
  myServo.attach(PIN_SERVO);
  myServo.write(SERVO_CLOSED);

  // CONFIGURE DFPLAYER MINI
  // Stop execution if there are problems with the hardware
  if (!myDFPlayer.begin(dfpSerial)) {
    Serial.println("DFPlayer Error!");
    while (true);
  }
  myDFPlayer.volume(0);

  Serial.println(">>> LUMA READY");
}

void loop() {
  unsigned long now = millis(); // Capture current time once per loop
  checkProtoPieMessages(); // Check for new messages from ProtoPie

  // Main state logic
  switch (currentState) {
    case ST_CLOSED:      
      stateClosed();
      break;
    case ST_RITUAL_START:  
      isShellOpen = true;
      break;
    case ST_STRETCHING:    
      stateStretching(now);
      break;
    case ST_BREATHING:      
      stateBreathing(now);
      break;
    case ST_GRATITUDE:      
      stateGratitude(now);
      break;
    case ST_END:          
      stateEnd();
      break;
    default:
      currentState = ST_CLOSED; // If undefined, reset to closed state
      break;
  }

  handleTransitions(now); // Check for states transitions
  updateServo(now); // Handle physical shell movement
}

// Handles logic to move from one state to the next
void handleTransitions(unsigned long now) {
  // >>> FROM PROTOPIE
  // Resets ritual and returns to home page
  if (receivedData.message.equals("GO_HOME")) {
    safeAudioStop();
    transitionTo(ST_RITUAL_START);
    return; 
  }
  // Jumps to specific ritual phases
  if (receivedData.message.equals("START_STRETCHING")) { transitionTo(ST_STRETCHING); return; }
  if (receivedData.message.equals("START_BREATHING"))  { transitionTo(ST_BREATHING);  return; }
  if (receivedData.message.equals("START_GRATITUDE"))  { transitionTo(ST_GRATITUDE);  return; }
  // GRATITUDE -> END
  if (receivedData.message.equals("CLOSE_LUMA")) {
    transitionTo(ST_END);
    return;
  }

  // >>> FROM ARDUINO
  // STRETCH -> BREATH (Button)
  // if exercise finished and button pressed
  if (currentState == ST_STRETCHING && stretchPhase == 2 && totalStretchTime >= TIME_STRETCH_EX) {
    if (digitalRead(PIN_BTN_NEXT) == LOW && (now - lastBtnTime > 500)) {
      safeAudioPlay(TRACK_CLICK);
      lastBtnTime = now; // Update debounce timer
      transitionTo(ST_BREATHING);
    }
  }
  // BREATH -> GRATITUDE (Button)
  // after breathing outro audio finishes
  if (currentState == ST_BREATHING && isBreathingOutroActive) {
     if ((now - breathingOutroStartTime) >= TIME_BREATHING_OUTRO) {
        if (digitalRead(PIN_BTN_NEXT) == LOW && (now - lastBtnTime > 500)) {
          safeAudioPlay(TRACK_CLICK);
          lastBtnTime = now;
          transitionTo(ST_GRATITUDE);
        }
     }
  }
}

// Initializes variables, timers, and states when switching phases
void transitionTo(LumaState newState) {
  currentState = newState; // Update global state
  stateStartTime = millis(); // Start time for the new phase
  receivedData.message = ""; // Clear last ProtoPie message to prevent re-triggering

  // Resets timers and variables when changing states
  isBreathingExActive = false;
  isBreathingOutroActive = false;
  isStretchingPaused = false;

  if (newState == ST_RITUAL_START) {
    isShellOpen = true; 
    safeAudioStop();
    Serial.println("START_RITUAL");
  }
  if (newState == ST_STRETCHING) {
    stretchPhase = 0; 
    stretchPhaseStartTime = millis(); 
    totalStretchTime = 0; 
    stretchDoneSent = false;
    isShellOpen = true;
    safeAudioPlay(TRACK_STRETCH_INTRO);
    Serial.println("STRETCH_START");
  }
  if (newState == ST_BREATHING) {
    breathingDoneSent = false; 
    isShellOpen = true;
    safeAudioPlay(TRACK_BREATH_INTRO);
    Serial.println("BREATHING_START");
  }
  if (newState == ST_GRATITUDE) {
    isShellOpen = true;
    safeAudioPlay(TRACK_WAVES);
    Serial.println("GRATITUDE_START");
  }
  if (newState == ST_END) {
    safeAudioPlay(TRACK_OPEN_CLOSE);
    Serial.println("CLOSING");
  }
}

void stateClosed() {
  isShellOpen = false; 
  if (digitalRead(PIN_TOUCH) == HIGH) {
    transitionTo(ST_RITUAL_START);
    safeAudioPlay(TRACK_OPEN_CLOSE);
  }
}

// Pauses if user moves too far (Ultrasonic Sensor)
void stateStretching(unsigned long now) {
  // Logic switches between Intro (0), Setup (1), and Work (2)
  switch (stretchPhase) {
    case 0: // INTRO
      if (now - stretchPhaseStartTime >= TIME_STRETCH_INTRO) {
        stretchPhase = 1; // Move to Setup
        stretchPhaseStartTime = now; // Reset timer for next phase
      }
      break;
    case 1: // SETUP
      if (now - stretchPhaseStartTime >= TIME_STRETCH_SETUP) {
        stretchPhase = 2; // Move to Exercise
        totalStretchTime = 0; 
        lastStretchUpdate = now;
        safeAudioPlay(TRACK_STRETCH_EX);
        Serial.println("STRETCHING_EXERCISE");
      }
      break;
    case 2: // WORK
      static int lastProgress = -1;
      long currentDist = getDistance();
      // User is in range: play audio, accumulate time
      if (currentDist > 0 && currentDist <= DISTANCE_MAX_CM) {
        outOfRangeCounter = 0;
        if (isStretchingPaused) {
          isStretchingPaused = false;
          Serial.println("STRETCHING_RESUME");
          myDFPlayer.start(); // Resume audio
        }
        isShellOpen = true;
      }
      // User is too far: pause progress and close shell
      else {
        outOfRangeCounter++;
        if (outOfRangeCounter > 10 && !isStretchingPaused) {
          isStretchingPaused = true;
          isShellOpen = false;
          Serial.println("STRETCHING_PAUSE");
          myDFPlayer.pause(); // Stop audio
        }
      }
      if (!isStretchingPaused) {
        totalStretchTime += (now - lastStretchUpdate);
        // Progress bar
        int progress = map(totalStretchTime, 0, TIME_STRETCH_EX, 0, 100);
        progress = constrain(progress, 0, 100);
        if (progress != lastProgress) {
          Serial.print("STRETCH_PROGRESS||"); 
          Serial.println(progress);
          lastProgress = progress;
        }
      }
      lastStretchUpdate = now;
      if (totalStretchTime >= TIME_STRETCH_EX && !stretchDoneSent) {
          Serial.println("STRETCHING_DONE");
          safeAudioStop();
          stretchDoneSent = true;
      }
      break;
  }
}

// Detects hands (Photoresistor) and synchronizes LED pulses with audio
void stateBreathing(unsigned long now) {
  int ldr = analogRead(PIN_LDR);
  static unsigned long ldrLowStartTime = 0;

  // DEBUG
  static unsigned long lastLdrPrint = 0;
  if (now - lastLdrPrint > 1000) {
    Serial.print("LDR: ");
    Serial.print(ldr);
    if (isBreathingExActive) {
      long rem = (TIME_BREATHING_EX - ((now - breathingExStartTime) - 500)) / 1000;
      Serial.print(" | Remaining time: ");
      Serial.print(rem);
      Serial.println("s");
    } else {
      Serial.println();
    }
    lastLdrPrint = now;
  }


  // 1. WAITING FOR USER, if exercise not started and outro not active
  if (!isBreathingExActive && !isBreathingOutroActive) {
    unsigned long elapsedIntro = now - stateStartTime;

    // Turns NeoPixel OFF to invite the user to put their hans on the pearl
    if (elapsedIntro < TIME_BREATHING_INTRO) {
      // Starts fading during the last 5 seconds of the intro
      unsigned long fadeStart = TIME_BREATHING_INTRO - 4000;
      if (elapsedIntro > fadeStart) {
        // Calculate brightness from 255 down to 0
        int fadeVal = map(elapsedIntro, fadeStart, TIME_BREATHING_INTRO, 255, 0);
        fadeVal = constrain(fadeVal, 0, 255);
        setRingColor((fadeVal * PEARL_R) / 255, (fadeVal * PEARL_G) / 255, (fadeVal * PEARL_B) / 255);
      } 
      else {
        // Full brightness before the fade starts
        setRingColor(PEARL_R, PEARL_G, PEARL_B);
      }
      return;
    }

    // 2. TRIGGER LOGIC, pearl now dark
    if (ldr > THRESHOLD_LDR) { 
      setRingColor(0, 0, 0); 
      ldrLowStartTime = 0; 
    } 
    else {
      if (ldrLowStartTime == 0) ldrLowStartTime = now;
      // If hand is detected for more than 300ms
      if (now - ldrLowStartTime > 300) {
        isBreathingExActive = true; 
        breathingExStartTime = now; 
        safeAudioPlay(TRACK_BREATH_EX);
        Serial.println("BREATHING_EXERCISE");
      }
    }
  }

  // 3. BREATHING EXERCISE
  if (isBreathingExActive) {
    long elapsed = (now - breathingExStartTime) - 500;
    // Transition to outro
    if (elapsed >= TIME_BREATHING_EX) {
      isBreathingExActive = false; // Stop exercise
      isBreathingOutroActive = true; // Start outro
      breathingOutroStartTime = now;
      safeAudioPlay(TRACK_BREATH_OUTRO);
    } 
    // BRIGHTNESS PULSES: Maps time to LED intensity to guide the breath 
    else {
      int brightness = 0;
      if (elapsed < 0) brightness = 0;
      // Pattern 1: 4s Inhale / 4s Exhale (0-32s)
      else if (elapsed < 32000) {
        int cycle = elapsed % 8000;
        if (cycle < 4000) {
          // Inhale phase: 0 to 4 seconds, from 0 (off) to 255 (full bright)
          brightness = map(cycle, 0, 4000, 0, 255);
        } 
        else {
          // Exhale phase: 4 to 8 seconds, from 255 (full bright) to 0 (off)
          brightness = map(cycle, 4000, 8000, 255, 0);
        }
      }
      // Pattern 2: Box Breathing (4s In, 4s Hold, 4s Out, 4s Hold) (32-64s)
      else if (elapsed < 64000) {
        int cycle = (elapsed - 32000) % 16000;
        if (cycle < 4000) brightness = map(cycle, 0, 4000, 0, 255);
        else if (cycle < 8000) brightness = 255;
        else if (cycle < 12000) brightness = map(cycle, 8000, 12000, 255, 0);
        else brightness = 0;
      }
      // Pattern 3: Long Exhalations (64-112s)
      else {
        int cycle = (elapsed - 64000) % 16000;
        if (cycle < 4000) brightness = map(cycle, 0, 4000, 0, 200);
        else if (cycle < 6000) brightness = 255;
        else if (cycle < 12000) brightness = map(cycle, 6000, 12000, 255, 0);
        else brightness = 0;
      }
      // Apply the calculated brightness to the NeoPixel
      setRingColor((brightness * PEARL_R) / 255, (brightness * PEARL_G) / 255, (brightness * PEARL_B) / 255);
    }
  }

  // 4. OUTRO
  if (isBreathingOutroActive) {
    setRingColor(PEARL_R, PEARL_G, PEARL_B); 
    if ((now - breathingOutroStartTime) >= TIME_BREATHING_OUTRO && !breathingDoneSent) {
      Serial.println("BREATHING_DONE");
      breathingDoneSent = true;
    }
  }
}

void stateGratitude(unsigned long now) {
  int r = PEARL_R, g = PEARL_G, b = PEARL_B;
  // Overwrite pearl white values if an emotion is received from protopie
  if (receivedData.message.equals("HAPPY")) { r = 255; g = 100; b = 10; }
  else if (receivedData.message.equals("SAD")) { r = 0; g = 50; b = 255; }
  else if (receivedData.message.equals("CALM")) { r = 0; g = 130; b = 25; }
  
  // Sine wave based on elapsed time
  float time_s = millis() / 1000.0;
  float pulse = 0.2 + ((sin(time_s * 0.5) + 1.0) / 2.0) * 0.8;
  setRingColor(r * pulse, g * pulse, b * pulse);
}

void stateEnd() {
  isShellOpen = false;
  if (currentServoPos >= SERVO_CLOSED) { 
    setRingColor(0, 0, 0);
    currentState = ST_CLOSED;
  }
}

// >>> UTILITY FUNCTIONS

// Moves servo slowly and fades LEDs as the shell opens
void updateServo(unsigned long now) {
  // Determine target position based on shell state
  int target;
  if (isShellOpen) target = SERVO_OPENED;
  else target = SERVO_CLOSED;

  // Detach servo when stopped for energy saving and jitter prevention
  if (currentServoPos == target) {
    if (myServo.attached()) myServo.detach();
    return;
  }

  if (!myServo.attached()) 
    myServo.attach(PIN_SERVO);

  // Move 1 degree every SERVO_SPEED_MS
  if (now - lastServoTime >= SERVO_SPEED_MS) {
    if (currentServoPos < target) currentServoPos++;
    else currentServoPos--;
    myServo.write(currentServoPos);
    lastServoTime = now;
    // Fading LED color based on shell angle
    // when servo is at SERVO_CLOSED, brightness is 0
    // when servo is at SERVO_OPENED, brightness is (255,150,50)
    int r = map(currentServoPos, SERVO_CLOSED, SERVO_OPENED, 0, PEARL_R);
    int g = map(currentServoPos, SERVO_CLOSED, SERVO_OPENED, 0, PEARL_G);
    int b = map(currentServoPos, SERVO_CLOSED, SERVO_OPENED, 0, PEARL_B);
    setRingColor(r, g, b);
  }
}

// DFPlayer Handshake
void safeAudioPlay(int track) {
  myDFPlayer.stop();
  delay(200); // Give the buffer time to clear
  myDFPlayer.play(track);
  delay(200); // Give the track time to start
}

void safeAudioStop() {
  myDFPlayer.stop();
  delay(100);
  myDFPlayer.stop(); // Redundant stop for reliability
}

// Measures distance in cm with Ultrasonic Sensor
long getDistance() {
  digitalWrite(PIN_TRIG, LOW); 
  delayMicroseconds(5);
  digitalWrite(PIN_TRIG, HIGH); 
  delayMicroseconds(10);
  digitalWrite(PIN_TRIG, LOW);

  // Measure bounce-back time
  // 20ms timeout prevents code from hanging
  long dur = pulseIn(PIN_ECHO, HIGH, 20000);
  if (dur == 0) 
    return 999; // Return a large distance if no object is detected
  // Speed of sound calculation (divided by 2 for the round trip)
  long distance = (dur * 0.0343) / 2;

  // DEBUG
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  
  return distance;
}

// Sets NeoPixel to a specific color
void setRingColor(int r, int g, int b) {
  uint32_t color = ring.Color(r, g, b);
  for (int i = 0; i < NUM_PIXELS; i++) ring.setPixelColor(i, color);
  ring.show();
}

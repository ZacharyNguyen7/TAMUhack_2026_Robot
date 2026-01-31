# TAMUhack_2026_Robot: Gesture Controlled Car
# SENDER CODE
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <SPI.h>
#include <RF24.h>

// MPU6050
Adafruit_MPU6050 mpu;

// nRF24L01
RF24 radio(17, 22); // CE, CSN (change if needed)
const byte address[6] = "00001";

void setup() {
  Serial.begin(115200);
  Wire.begin();

  if (!mpu.begin()) {
    Serial.println("❌ MPU6050 not found");
    while (1);
  }

  SPI.begin();

  if (!radio.begin()) {
    Serial.println("❌ nRF24L01 not responding");
    while (1);
  }

  radio.openWritingPipe(address);
  radio.setPALevel(RF24_PA_LOW);        // stable transmission
  radio.setDataRate(RF24_250KBPS);      // slow & reliable
  radio.stopListening();

  Serial.println("✅ Transmitter ready");
}

void loop() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  // Map accelerometer to direction
  // 1 = Forward, 2 = Backward, 3 = Left, 4 = Right, 0 = Stop
  int direction = 0;

  if (a.acceleration.x > 8.0) direction = 1;       // Forward
  else if (a.acceleration.x < -8.0) direction = 2; // Backward
  else if (a.acceleration.y > 8.0) direction = 3;  // Left
  else if (a.acceleration.y < -8.0) direction = 4;  // Right
  else direction = 0; // Stop

  // Send direction as a number
  char message[32];
  snprintf(message, sizeof(message), "%d", direction);

  bool sent = radio.write(&message, sizeof(message));
  Serial.print("Sending direction: ");
  Serial.println(direction);

  delay(100); // 10 Hz
}

# RECEIVER CODE
#include <SPI.h>
#include <RF24.h>

RF24 radio(9, 10);   // CE, CSN
const byte address[6] = "00001";

// Use pure digital pins for direction
const int AIN1 = 2;
const int AIN2 = 3;
const int BIN1 = 4;
const int BIN2 = 7;
const int PWMA = 5;  // PWM pin
const int PWMB = 6;  // PWM pin
const int STBY = 8;  // Standby pin for TB6612

int motorSpeed = 0;

void setup() {
  Serial.begin(115200);

  if (!radio.begin()) {
    Serial.println("❌ nRF24L01 not responding");
    while (1);
  }

  radio.openReadingPipe(0, address);
  radio.setPALevel(RF24_PA_LOW);
  radio.setDataRate(RF24_250KBPS);
  radio.startListening();

  Serial.println("✅ Receiver listening");

  pinMode(AIN1, OUTPUT);
  pinMode(AIN2, OUTPUT);
  pinMode(BIN1, OUTPUT);
  pinMode(BIN2, OUTPUT);
  pinMode(PWMA, OUTPUT);
  pinMode(PWMB, OUTPUT);
  pinMode(STBY, OUTPUT);

  digitalWrite(STBY, HIGH);  // ENABLE motor driver
  delay(10); // small delay for TB6612 to wake up
}

void loop() {
  if (radio.available()) {
    char text[32] = {0};
    radio.read(&text, sizeof(text));
    motorSpeed = atoi(text);  // Convert received string to int

    Serial.print("Received motor speed: ");
    Serial.println(motorSpeed);

    spinMotor(motorSpeed);
  }
}

void spinMotor(int speed) {
  int pwmSpeed = constrain(abs(speed), 0, 255);

  // Minimum PWM to overcome motor stall
  if (pwmSpeed > 0 && pwmSpeed < 50) pwmSpeed = 50;

  if (speed == 3) { // Forward
    digitalWrite(AIN1, HIGH);
    digitalWrite(AIN2, LOW);
    digitalWrite(BIN1, HIGH);
    digitalWrite(BIN2, LOW);
  } 
  else if (speed == 4) { // Backward
    digitalWrite(AIN1, LOW);
    digitalWrite(AIN2, HIGH);
    digitalWrite(BIN1, LOW);
    digitalWrite(BIN2, HIGH);
  } 
  else if (speed == 2) { // Backward
    digitalWrite(AIN1, HIGH);
    digitalWrite(AIN2, LOW);
    digitalWrite(BIN1, LOW);
    digitalWrite(BIN2, HIGH);
  }
  else if (speed == 1) { // Backward
    digitalWrite(AIN1, LOW);
    digitalWrite(AIN2, HIGH);
    digitalWrite(BIN1, HIGH);
    digitalWrite(BIN2, LOW);
  }
  else { // Stop
    digitalWrite(AIN1, LOW);
    digitalWrite(AIN2, LOW);
    digitalWrite(BIN1, LOW);
    digitalWrite(BIN2, LOW);
  }

  analogWrite(PWMA, 200);
  analogWrite(PWMB, 200);
}

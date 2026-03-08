#include <WiFiS3.h>
#include <ArduinoMqttClient.h>
#include <DHT.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Servo.h>

// --- WIFI ---
char ssid[] = "TEJESH";
char pass[] = "tejesh1234";

// --- MQTT ---
const char broker[] = "broker.hivemq.com";
int port = 1883;
const char topic[] = "study/monitor/data";

WiFiClient wifiClient;
MqttClient mqttClient(wifiClient);

// --- PIN DEFINITIONS ---
#define SERVO_PIN   1
#define DHTPIN      2
#define IR_PIN      3
#define RELAY_PIN   4
#define BUZZER_PIN  5
#define RGB_RED     6
#define LDR_PIN     7
#define TRIG_PIN    9
#define ECHO_PIN    10
#define RGB_GREEN   11
#define RGB_BLUE    13

// --- SETTINGS ---
#define DHTTYPE DHT11
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define POSTURE_THRESHOLD 30

// --- OBJECTS ---
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
DHT dht(DHTPIN, DHTTYPE);
Servo lightServo;

void setup() {

  Serial.begin(9600);

  // Pin Modes
  pinMode(LDR_PIN, INPUT_PULLUP);
  pinMode(IR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(RGB_RED, OUTPUT);
  pinMode(RGB_GREEN, OUTPUT);
  pinMode(RGB_BLUE, OUTPUT);

  // Initialize Hardware
  dht.begin();
  lightServo.attach(SERVO_PIN);
  lightServo.write(0);

  // OLED INIT
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("OLED failed");
    for(;;);
  }

  display.clearDisplay();
  display.setTextColor(WHITE);
  display.setTextSize(1);

  // WIFI CONNECT
  Serial.print("Connecting to WiFi");
  while (WiFi.begin(ssid, pass) != WL_CONNECTED) {
    Serial.print(".");
    delay(2000);
  }

  Serial.println(" Connected to WiFi");

  // MQTT CONNECT
  Serial.print("Connecting to MQTT broker...");
  if (!mqttClient.connect(broker, port)) {
    Serial.print("MQTT failed! Error: ");
    Serial.println(mqttClient.connectError());
    while (1);
  }

  Serial.println("Connected to MQTT broker");
}

void loop() {

  // --- POSTURE DISTANCE ---
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.034 / 2;

  // --- LIGHT SENSOR ---
  bool isDark = digitalRead(LDR_PIN);

  // --- TEMPERATURE & HUMIDITY ---
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();

  // --- DISTRACTION SENSOR ---
  bool distraction = digitalRead(IR_PIN);

  // --- DISPLAY ---
  display.clearDisplay();
  display.setCursor(0,0);

  // LIGHT CONTROL
  if (isDark) {
    digitalWrite(RELAY_PIN, HIGH);
    lightServo.write(90);
    digitalWrite(RGB_BLUE, HIGH);
    display.println("LIGHT: AUTO-ON");
  } 
  else {
    digitalWrite(RELAY_PIN, LOW);
    lightServo.write(0);
    digitalWrite(RGB_BLUE, LOW);
    display.println("LIGHT: OK");
  }

  // POSTURE CHECK
  if (distance > 0 && distance < POSTURE_THRESHOLD) {

    display.println("POSTURE: TOO CLOSE!");
    display.println("DON'T MOVE OR SLEEP");

    digitalWrite(RGB_RED, HIGH);
    digitalWrite(RGB_GREEN, LOW);

    tone(BUZZER_PIN, 1000, 100);
  }
  else {

    display.println("POSTURE: GOOD");

    digitalWrite(RGB_RED, LOW);
    digitalWrite(RGB_GREEN, HIGH);
  }

  // DISTRACTION
  if (distraction == LOW) {

    display.println("STATUS: DISTRACTED");
    display.println("SEE THIS SIDE!!");

  } else {

    display.println("STATUS: FOCUSED");
  }

  display.print("TEMP: ");
  display.print(temp,1);
  display.println("C");

  display.print("HUMID: ");
  display.print(hum,0);
  display.println("%");

  display.display();

  // --- MQTT PUBLISH ---
  mqttClient.beginMessage(topic);

  mqttClient.print("Distance:");
  mqttClient.print(distance);
  mqttClient.print(",Temp:");
  mqttClient.print(temp);
  mqttClient.print(",Humidity:");
  mqttClient.print(hum);
  mqttClient.print(",Dark:");
  mqttClient.print(isDark);
  mqttClient.print(",Distraction:");
  mqttClient.print(distraction);

  mqttClient.endMessage();

  delay(500);
}

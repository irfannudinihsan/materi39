#include "secrets.h"
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include <WiFi.h>
#include <ESP32Servo.h>

#define AWS_IOT_SUBSCRIBE_TOPIC "esp32/sub"

#define led_pin_on 4
#define led_pin_off 2
#define servo_pin 13
#define pir_pin 14 // pin sensor PIR HCSR501

WiFiClientSecure net = WiFiClientSecure();
PubSubClient client(net);
Servo myServo;

bool autoMode = false; // mode otomatis yang berawal dari non aktif

void connectAWS() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  Serial.print("Connecting to ");
  Serial.println(WIFI_SSID);

  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println(".");
  }

  // Konfigurasi koneksi aman ke AWS dengan Device Certificate, AWS Root CA1, dan Private Key
  net.setCACert(AWS_CERT_CA);
  net.setCertificate(AWS_CERT_CRT);
  net.setPrivateKey(AWS_CERT_PRIVATE);

  // Koneksi ke MQTT Broker pada AWS endpoint 
  client.setServer(AWS_IOT_ENDPOINT, 8883); // port 8883 adalah port untuk koneksi aman protokol MQTT

  client.setCallback(messageHandler);
  Serial.println("Connecting to AWS IoT");

  while (!client.connect(THING_NAME)) {
    Serial.print(".");
    delay(200);
  }

  if (!client.connected()){
    Serial.println("AWS IoT Timeout!");
    return;
  }

  myServo.attach(servo_pin);
  pinMode(led_pin_on, OUTPUT);
  pinMode(led_pin_off, OUTPUT);
  pinMode(pir_pin, INPUT);

  // subscribe ke topic
  client.subscribe(AWS_IOT_SUBSCRIBE_TOPIC);
  Serial.println("AWS IoT Connected");
}

void messageHandler(char* topic, byte* payload, unsigned int length)
{
  Serial.print("incoming: ");
  Serial.println(topic);
 
  // Mencetak pesan JSON yang diterima
  Serial.print("Received message: ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();
  
  // Menyalin payload ke String agar dapat ditambahkan null-terminator
  char payloadString[length + 1];
  memcpy(payloadString, payload, length);
  payloadString[length] = '\0';

  Serial.print("Parsed message");
  Serial.println(payloadString);

  char led = payloadString[0];
  Serial.print("Command: ");
  Serial.println(led);

  if (strcmp(topic, "esp32/sub") == 0) {
    if (strcmp(payloadString, "AUTO_ON") == 0){
      autoMode = true;
      digitalWrite(led_pin_on, HIGH);
      digitalWrite(led_pin_off, LOW);
      Serial.println("Auto mode diaktifkan");
    } else if (strcmp(payloadString, "AUTO_OFF") == 0){
      autoMode = false;
      digitalWrite(led_pin_on, LOW);
      digitalWrite(led_pin_off, HIGH);
      Serial.println("Auto mode dinonaktifkan");
    } else if (strcmp(payloadString, "servo_open") == 0 && !autoMode){
      myServo.write(90); // putar servo ke posisi 90 Derajat
      Serial.println("Pintu terbuka");
    }
  }
  
}

void setup() {
  Serial.begin(115200);
  connectAWS();
}

void loop() {
  client.loop();
  // Baca status sensor PIR
  int pirStatus = digitalRead(pir_pin);

  // Jika mode otomatis aktif dan sensor PIR mendeteksi gerakan
  if (autoMode && pirStatus == HIGH) {
    myServo.write(90); // putar servo ke posisi 90 Derajat
    delay(2000); // Tunggu beberapa saat sebelum menutup pintu lagi
    myServo.write(0); // putar servo kembali ke posisi 0 Derajat
  }
  delay(1000);
}

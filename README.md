#include <WiFi.h>
#include <PubSubClient.h>

// --------- DATE WIFI ----------
const char* ssid = "WIFI_NAME";
const char* password = "WIFI_PASS";

// --------- MQTT --------------
const char* mqtt_server = "broker.hivemq.com";

// --------- PINI -------------
int sensorPin = A0;          // senzor umiditate
int pumpControlPin = 23;     // poarta MOSFET (comanda pompa)

// --------- MQTT CLIENT ------
WiFiClient espClient;
PubSubClient client(espClient);

// --------- FUNCTIE CALLBACK (comenzi manuale) ----------
void callback(char* topic, byte* payload, unsigned int length) {
  String message = "";

  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }

  if (String(topic) == "pump/control") {
    if (message == "ON") {
      digitalWrite(pumpControlPin, LOW);   // pornire pompa
    } 
    else if (message == "OFF") {
      digitalWrite(pumpControlPin, HIGH);  // oprire pompa
    }
  }
}

// --------- WIFI SETUP ----------
void setup_wifi() {
  delay(10);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
  }
}

// --------- MQTT RECONNECT ----------
void reconnect() {
  while (!client.connected()) {
    if (client.connect("ArduinoWaterSystem")) {
      client.subscribe("pump/control");
    } else {
      delay(5000);
    }
  }
}

// --------- SETUP ----------
void setup() {
  pinMode(pumpControlPin, OUTPUT);
  digitalWrite(pumpControlPin, HIGH);  // pompa oprita

  Serial.begin(9600);

  setup_wifi();

  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

// --------- LOOP ----------
void loop() {

  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // Citire senzor
  int soilValue = analogRead(sensorPin);

  // Trimitere valoare senzor
  char msg[10];
  itoa(soilValue, msg, 10);
  client.publish("sensor/soil", msg);

  // Control automat pompa
  if (soilValue > 600) {           // sol uscat
    digitalWrite(pumpControlPin, LOW);
    client.publish("pump/state", "ON");
  } else {                         // sol umed
    digitalWrite(pumpControlPin, HIGH);
    client.publish("pump/state", "OFF");
  }

  delay(5000);  // citire la 5 secunde
}

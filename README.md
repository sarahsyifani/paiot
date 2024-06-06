1. Source Code
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <PubSubClient.h>

// WiFi and MQTT server settings
const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "broker.mqtt.cool";

// Antares API settings
const char* serverName = "https://platform.antares.id:8443/~/antares-cse/antares-id/AQWM/weather_airQuality_nodeCore_teknik/la";
const char* apiKey = "ee8d16c4466b58e1:3b8d814324c84c89";

// MQTT topic
const char* sensorDataTopic = "iot/uas/sensor";
const char* controlling1 = "iot/uas/dapur";
const char* controlling2 = "iot/uas/kamar";
const char* controlling3 = "iot/uas/ruangtamu";

// Pins for output
const int led1 = 16;
const int led2 = 5;
const int led3 = 19;


WiFiClient espClient;
PubSubClient client(espClient);

void setup() {
  Serial.begin(115200);

  pinMode(led1, OUTPUT);
  pinMode(led2, OUTPUT);
  pinMode(led3, OUTPUT);
  

  // Connect to WiFi
  connectToWiFi();

  // Set up MQTT client
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();
  displaySensorData();
  delay(2000); // delay to avoid too frequent data retrieval
}

void connectToWiFi() {
  Serial.print("Connecting to WiFi...");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("Connected to WiFi");
}

void displaySensorData() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(serverName);
    http.addHeader("X-M2M-Origin", apiKey);
    http.addHeader("Accept", "application/json");

    int httpResponseCode = http.GET();

    if (httpResponseCode > 0) {
      String payload = http.getString();
      Serial.println("Received payload: " + payload);

      DynamicJsonDocument doc(1024);
      DeserializationError error = deserializeJson(doc, payload);

      if (!error) {
        JsonObject cinObj = doc["m2m:cin"];
        String conData = cinObj["con"];

        StaticJsonDocument<600> conDoc;
        DeserializationError conError = deserializeJson(conDoc, conData);

        if (!conError) {
          float temperature = conDoc["Temp"].as<float>();
          float humidity = conDoc["Hum"].as<float>();

          Serial.print("Temperature: ");
          Serial.println(temperature, 2);
          Serial.print("Humidity: ");
          Serial.println(humidity, 2);
          

          // Publish sensor data to MQTT as a single JSON object
          publishSensorData(temperature, humidity);
        } else {
          Serial.print("Error parsing 'con' JSON: ");
          Serial.println(conError.c_str());
        }
      } else {
        Serial.print("deserializeJson() failed: ");
        Serial.println(error.c_str());
      }
    } else {
      Serial.print("Error on HTTP request: ");
      Serial.println(httpResponseCode);
    }

    http.end();
  } else {
    connectToWiFi();
  }
}

void publishSensorData(float temperature, float humidity) {
  if (!client.connected()) {
    reconnect();
  }

  // Create a JSON object
  StaticJsonDocument<600> jsonDoc; // Increased buffer size
  JsonObject JSONencoder = jsonDoc.to<JsonObject>();

  JSONencoder["temperature"] = temperature;
  JSONencoder["humidity"] = humidity;


  // Serialize JSON object to a string
  char buffer[600]; // Increased buffer size
  size_t n = serializeJson(JSONencoder, buffer, sizeof(buffer));
  Serial.println("Publishing message: " + String(buffer)); // Debugging

  // Publish the JSON string to the MQTT topic
  bool success = client.publish(sensorDataTopic, buffer, n);
  if (success) {
    Serial.println("Message published successfully");
  } else {
    Serial.println("Message publishing failed");
  }
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Attempt to connect
    if (client.connect("ESP32Client")) {
      Serial.println("connected");
      client.subscribe(controlling1);
      client.subscribe(controlling2);
      client.subscribe(controlling3);
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      delay(5000);
    }
  }
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  payload[length] = '\0'; // Ensure payload is null-terminated
  String message = String((char*)payload);

  if (strcmp(topic, controlling1) == 0) {
    if (message == "0") {
      digitalWrite(led1, LOW);
    } else {
      digitalWrite(led1, HIGH);
    }
  }

  if (strcmp(topic, controlling2) == 0) {
    if (message == "0") {
      digitalWrite(led2, LOW);
    } else {
      digitalWrite(led2, HIGH);
    }
  }

  if (strcmp(topic, controlling3) == 0) {
    if (message == "0") {
      digitalWrite(led3, LOW);
    } else {
      digitalWrite(led3, HIGH);
    }
  }
}

2. Wokwi
   ![wokwi](https://github.com/sarahsyifani/paiot/assets/113895533/7fbd2508-42b4-46c8-9f66-bfedf38b77e5)

3. MQTT
   ![mqtt](https://github.com/sarahsyifani/paiot/assets/113895533/2178eeb1-ad11-40ee-826f-b2a63760af60)
4. Diagram Blok Kodular
   ![diagram blok](https://github.com/sarahsyifani/paiot/assets/113895533/4356ab25-cb2a-4723-bc56-0ed4edab77b9)



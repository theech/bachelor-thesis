/*********
  Rui Santos
  Complete project details at https://randomnerdtutorials.com  
*********/

#include <WiFi.h>
#include <PubSubClient.h>
#include <Wire.h>
#include <DHT.h>

// Replace the next variables with your SSID/Password combination
const char *ssid = "Smart IoT";
const char *password = "iot@2020";

// Add your MQTT Broker IP address, example:
//const char* mqtt_server = "192.168.1.144";
const char *mqtt_server = "192.168.9.75";

WiFiClient espClient;
PubSubClient client(espClient);

#define LED_BUILTIN 2
#define SENSOR 27
int interval_pub = 5000;
long lastMsg = 0;

//weter flow
long currentMillis = 0;
long previousMillis = 0;
int interval = 1000;
boolean ledState = LOW;
float calibrationFactor = 4.5;
volatile byte pulseCount;
byte pulse1Sec = 0;
float flowRate;
unsigned int flowMilliLitres;
unsigned long totalMilliLitres;

// Uncomment whatever type you're using!
// #define DHTTYPE DHT11 // DHT 11
#define DHTTYPE DHT22 // DHT 22 (AM2302), AM2321
//#define DHTTYPE DHT21 // DHT 21 (AM2301)

// Initialize DHT sensor. change the line below whatever DHT type you're using DHT11, DHT21 (AM2301), DHT22 (AM2302, AM2321)
DHT dht[] = {{0, DHTTYPE}, {2, DHTTYPE}};
float humids[2];
float temps[2];

const int moistures[] = {36, 39, 34};
float moisture[3];

const int ldrs[] = {35, 32};
float ldr[3];

void setup_wifi()
{
  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char *topic, byte *message, unsigned int length)
{
  Serial.print("Message arrived on topic: ");
  Serial.print(topic);
  Serial.print(". Message: ");
  String messageTemp;

  for (int i = 0; i < length; i++)
  {
    Serial.print((char)message[i]);
    messageTemp += (char)message[i];
  }
  Serial.println();

  // Feel free to add more if statements to control more GPIOs with MQTT

  // If a message is received on the topic esp32/output, you check if the message is either "on" or "off".
  // Changes the output state according to the message
  if (String(topic) == "local/pump")
  {
    Serial.print("Changing output to ");
    if (messageTemp == "on")
    {
      Serial.println("on");
      digitalWrite(4, LOW);
    }
    else if (messageTemp == "off")
    {
      Serial.println("off");
      digitalWrite(4, HIGH);
    }
  }
}

void reconnect()
{
  // Loop until we're reconnected
  while (!client.connected())
  {
    Serial.print("Attempting MQTT connection...");
    // Attempt to connect
    if (client.connect("ESP8266Client"))
    {
      Serial.println("connected");
      // Subscribe
      client.subscribe("local/pump");
    }
    else
    {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void IRAM_ATTR pulseCounter()
{
  pulseCount++;
}

void setup()
{
  Serial.begin(115200);

  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);

  pinMode(LED_BUILTIN, OUTPUT);
  pinMode(SENSOR, INPUT_PULLUP);

  pulseCount = 0;
  flowRate = 0.0;
  flowMilliLitres = 0;
  totalMilliLitres = 0;
  previousMillis = 0;

  attachInterrupt(digitalPinToInterrupt(SENSOR), pulseCounter, FALLING);

  for (auto &sensor : dht)
  {
    sensor.begin();
  }

  for (int index = 0; index < 2; index++)
  {
    pinMode(ldrs[index], INPUT);
  }

  for (int index = 0; index < 3; index++)
  {
    pinMode(moistures[index], INPUT);
  }
}

void meter_evn()
{
  currentMillis = millis();
  if (currentMillis - previousMillis > interval)
  {

    pulse1Sec = pulseCount;
    pulseCount = 0;

    // Because this loop may not complete in exactly 1 second intervals we calculate
    // the number of milliseconds that have passed since the last execution and use
    // that to scale the output. We also apply the calibrationFactor to scale the output
    // based on the number of pulses per second per units of measure (litres/minute in
    // this case) coming from the sensor.
    flowRate = ((1000.0 / (millis() - previousMillis)) * pulse1Sec) / calibrationFactor;
    previousMillis = millis();

    // Divide the flow rate in litres/minute by 60 to determine how many litres have
    // passed through the sensor in this 1 second interval, then multiply by 1000 to
    // convert to millilitres.
    flowMilliLitres = (flowRate / 60) * 1000;

    // Add the millilitres passed in this second to the cumulative total
    totalMilliLitres += flowMilliLitres;

    // Print the flow rate for this second in litres / minute
    Serial.print("Flow rate: ");
    Serial.print(int(flowRate)); // Print the integer part of the variable
    Serial.print("L/min");
    Serial.print("\t"); // Print tab space

    // Print the cumulative total of litres flowed since starting
    Serial.print("Output Liquid Quantity: ");
    Serial.print(totalMilliLitres);
    Serial.print("mL / ");
    Serial.print(totalMilliLitres / 1000);
    Serial.println("L");
  }
}

void dht_env()
{
  char msgTemp_in[16];
  char msgTemp_out[16];
  char msgHumd_in[16];
  char msgHumd_out[16];

  char *msgTemp[] = {msgTemp_in, msgTemp_out};
  char *msgHmud[] = {msgHumd_in, msgHumd_out};

  for (int index = 0; index < 2; index++)
  {
    humids[index] = dht[index].readHumidity();
    dtostrf(humids[index], 2, 2, msgHmud[index]);
    temps[index] = dht[index].readTemperature();
    dtostrf(temps[index], 2, 2, msgTemp[index]);
  }

  long now = millis();
  if (now - lastMsg > interval_pub)
  {
    lastMsg = now;
    client.publish("local/humid1", msgHmud[0]);
    client.publish("local/humid2", msgHmud[1]);
    client.publish("local/temp1", msgTemp[0]);
    client.publish("local/temp2", msgTemp[1]);
  }

  Serial.print("humd_in: ");
  Serial.println(humids[0]);
  Serial.print("humd_out: ");
  Serial.println(msgHmud[1]);
  Serial.print("temp_in: ");
  Serial.println(msgTemp[0]);
  Serial.print("temp_out: ");
  Serial.println(msgTemp[1]);
  Serial.println("------------------------------------");
}

void light_env()
{
  char msgLight_in[16];
  char msgLight_out[16];

  char *msgLights[] = {msgLight_in, msgLight_out};

  for (int index = 0; index < 2; index++)
  {
    ldr[index] = analogRead(ldrs[index]);
    ldr[index] = map(ldrs[index], 0, 4095, 0, 100);
    dtostrf(ldr[index], 2, 2, msgLights[index]);
  }

  long now = millis();
  if (now - lastMsg > interval_pub)
  {
    lastMsg = now;
    client.publish("local/light1", msgLights[0]);
    client.publish("local/light2", msgLights[1]);
  }

  Serial.print("light_in: ");
  Serial.println(msgLights[0]);
  Serial.print("light_out: ");
  Serial.println(msgLights[1]);
  Serial.println("------------------------------------");
}

void moisture_env()
{
  char msgMoisture_1_1[16];
  char msgMoisture_1_2[16];
  char msgMoisture_1_3[16];

  char *msgMoistures[] = {msgMoisture_1_1, msgMoisture_1_2, msgMoisture_1_3};

  for (int index = 0; index < 3; index++)
  {
    moisture[index] = analogRead(moistures[index]);
    moisture[index] = map(moisture[index], 0, 4095, 0, 100);
    dtostrf(moisture[index], 2, 2, msgMoistures[index]);
  }

  long now = millis();
  if (now - lastMsg > interval_pub)
  {
    lastMsg = now;
    client.publish("local/moisture1", msgMoistures[0]);
    client.publish("local/moisture2", msgMoistures[1]);
    client.publish("local/moisture3", msgMoistures[0]);
  }

  Serial.print("mt_1_1: ");
  Serial.println(msgMoistures[0]);
  Serial.print("mt_1_2: ");
  Serial.println(msgMoistures[1]);
  Serial.print("mt_1_3: ");
  Serial.println(msgMoistures[2]);
  Serial.println("------------------------------------");
}

void loop()
{
  if (!client.connected())
  {
    reconnect();
  }
  client.loop();

  meter_evn();
  moisture_env();
  dht_env();
  light_env();

  long now = millis();
  if (now - lastMsg > interval_pub)
  {
    lastMsg = now;
    // convert to char
    char msgFlowRate[16];
    char msgTotalMilliLitres[16];

    dtostrf(flowRate, 2, 2, msgFlowRate);
    dtostrf(totalMilliLitres, 2, 2, msgTotalMilliLitres);

    client.publish("local/meter/rate", msgFlowRate);
    client.publish("local/meter/quantity", msgTotalMilliLitres);
  }
}
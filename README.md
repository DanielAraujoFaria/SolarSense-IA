# SolarSense API - README

## 📋 **Descrição do Projeto**

A **SolarSense API** é uma aplicação para gerenciamento de sistemas de energia solar, oferecendo funcionalidades como cadastro de usuários, painéis solares, sensores e leitura de dados para monitoramento e otimização do desempenho. A solução foi desenvolvida para integrar sensores e dispositivos, como o DHT22 e potenciômetro, com o objetivo de monitorar a temperatura, umidade, tensão solar, potência e energia gerada.

---

## 🔧 **Instruções para Execução**

### **Passos para Configurar o Projeto**
1. Clone o repositório:
   ```bash
   git clone https://github.com/DanielAraujoFaria/SolarSense-IA
   ```
2. Ou acesse [Link do Projeto no Wokwi](https://wokwi.com/projects/414490927138079745)

---

## 📂 **Código Fonte**

### **Código do Microcontrolador (ESP32)**

```cpp
#include <ThingerESP32.h>
#include <WiFi.h>
#include <ArduinoJson.h>
#include <DHT.h>
#include <PubSubClient.h>
#include <time.h>

// Definições do DHT
#define DHTPIN 25
#define DHTTYPE DHT22
DHT dht(DHTPIN, DHTTYPE);

// Definições dos pinos
#define LED_PIN 26
#define boardLED 2
#define PINO_TENSAO 34

// Configuração do NTP
#define NTP_SERVER "pool.ntp.org"
#define TZ -3                  // Fuso horário (UTC-3 para o Brasil)
#define DST_MN 0               // Não usar horário de verão
#define GMT_OFFSET_SEC 3600 * TZ
#define DAYLIGHT_OFFSET_SEC 60 * DST_MN
#define NTP_MIN_VALID_EPOCH 1533081600  

unsigned long myTime;
String hora, Data;
tm *NOW_TM;
time_t NOW;
char Time[10];
char Date[12];

// Variáveis e objetos globais
float tensaoSolar = 0.0, temperatura = 0.0, umidade = 0.0, potenciaSolar = 0.0, energiaGerada = 0.0;
unsigned long ultimoTempo = 0, deltaTempo = 0;
WiFiClient espClient;
PubSubClient MQTT(espClient);

const char* ID = "tutu"; 
const char* SensorID = "id_solar";

// Credenciais WiFi
const char* SSID = "Wokwi-GUEST";
const char* PASSWORD = "";

// Configuração do servidor MQTT
const char* BROKER_MQTT = "74.179.84.66";
int BROKER_PORT = 1883;
const char* mqttServer = "74.179.84.66";
const int mqttPort = 1880;
const char* mqttUser = "gs2024";
const char* mqttPassword = "q1w2e3r4";
#define TOPICO_SUBSCRIBE "GS_2TDSPK"
#define TOPICO_PUBLISH   "GS_2TDSPK"

void initWiFi() {
  Serial.println("Conectando ao WiFi...");
  WiFi.begin(SSID, PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi conectado!");
}

void initMQTT() {
  MQTT.setServer(BROKER_MQTT, BROKER_PORT);
}

void VerificaConexoesWiFIEMQTT() {
  if (WiFi.status() != WL_CONNECTED) {
    initWiFi();
  }
  if (!MQTT.connected()) {
    while (!MQTT.connected()) {
      Serial.println("Conectando ao broker MQTT...");
      if (MQTT.connect(ID, mqttUser, mqttPassword)) {
        Serial.println("Conectado ao broker MQTT!");
      } else {
        Serial.print("Falha ao conectar, código de erro: ");
        Serial.print(MQTT.state());
        delay(2000);
      }
    }
  }
}

void lerDHT22() {
  temperatura = dht.readTemperature();
  umidade = dht.readHumidity();
  if (isnan(temperatura) || isnan(umidade)) {
    Serial.println("Falha ao ler dados do DHT22");
    return;
  }
}

void lePotenciometro() {
  int leitura = analogRead(PINO_TENSAO);
  tensaoSolar = leitura * (3.3 / 4095.0);
}

void calculaPotenciaSolar() {
  potenciaSolar = tensaoSolar * 0.5;  // Exemplo de cálculo
}

void calculaEnergiaGerada() {
  unsigned long tempoAtual = millis();
  deltaTempo = tempoAtual - ultimoTempo;
  energiaGerada += (potenciaSolar * deltaTempo) / 3600000.0;
  ultimoTempo = tempoAtual;
}

void controlaLED() {
  if (temperatura > 30) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }
}

// Inicializa o NTP
void initNTP() {
  configTime(GMT_OFFSET_SEC, DAYLIGHT_OFFSET_SEC, NTP_SERVER);
  unsigned long t0 = millis();
  Serial.print("Sincronizando hora via NTP ");
  while ((NOW = time(nullptr)) < NTP_MIN_VALID_EPOCH) {
    Serial.print("...");
    delay(20);
  }
  Serial.println("\nHora sincronizada!");
  unsigned long t1 = millis() - t0;
  Serial.println("Sincronização NTP demorou " + String(t1) + "ms");
}

void setNOW() {
  NOW = time(nullptr);
}

// Atualiza as variáveis de data e hora
void setNOW_TM() {
  NOW_TM = localtime(&NOW);
  strftime(Time, 10, "%T", NOW_TM);
  strftime(Date, 12, "%Y-%m-%d", NOW_TM);
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  pinMode(boardLED, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  digitalWrite(boardLED, LOW);

  initWiFi();
  initMQTT();
  initNTP();
  dht.begin();
  ultimoTempo = millis();
}

void loop() {
  VerificaConexoesWiFIEMQTT();
  setNOW();
  setNOW_TM();

  lerDHT22();
  lePotenciometro();
  calculaPotenciaSolar();
  calculaEnergiaGerada();

  StaticJsonDocument<200> doc;
  doc["Temperatura"] = temperatura;
  doc["Umidade"] = umidade;
  doc["TensaoSolar"] = tensaoSolar;
  doc["PotenciaSolar"] = potenciaSolar;
  doc["EnergiaGerada"] = energiaGerada;
  doc["Hora"] = String(Time);
  doc["Data"] = String(Date);

  String payload;
  serializeJson(doc, payload);
  MQTT.publish(TOPICO_PUBLISH, payload.c_str());
  Serial.println("Dados enviados via MQTT!");

  controlaLED();
  delay(10000);
}
```
### **Diagrama**

```json
{
  "version": 1,
  "author": "Arthur Yamamoto",
  "editor": "Wokwi",
  "parts": [
    { "type": "board-esp32-devkit-c-v4", "id": "esp", "top": 76.8, "left": -71.96, "attrs": {} },
    { "type": "wokwi-potentiometer", "id": "pot1", "top": -39.7, "left": -0.2, "attrs": {} },
    { "type": "led", "id": "ledOnboard", "top": 8, "left": -4.76, "attrs": { "color": "green" } },
    { "type": "led", "id": "ledExternal", "top": 60, "left": 50, "attrs": { "color": "red" } },
    { "type": "dht22", "id": "dht", "top": -60, "left": 80, "attrs": {} },
    { "type": "wokwi-dht22", "id": "dht1", "top": -66.9, "left": -130.2, "attrs": {} }
  ]
}
```

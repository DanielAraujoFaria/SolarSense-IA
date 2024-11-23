# SolarSense API - README

## 📋 **Descrição do Projeto**

O **SolarSense** é uma solução que visa oferecer monitoramento em tempo real do desempenho de painéis solares, permitindo que usuários maximizem a eficiência energética e reduzam custos. A plataforma utiliza tecnologia para identificar problemas de funcionamento, otimizar a geração de energia e proporcionar dados precisos sobre o funcionamento dos painéis solares cadastrados pelos usuários.
Com aplicações em residências, empresas, agricultura e grandes instalações solares, o SolarSense promove economia, sustentabilidade e transparência, atendendo às crescentes demandas por energia renovável no mercado global.

### SOLUÇÃO PROPOSTA E BENEFÍCIOS PARA O NEGÓCIO
O SolarSense combina tecnologia e praticidade para atender às necessidades de monitoramento e otimização de sistemas de painéis solares. A seguir, estão destacados os principais recursos e benefícios que tornam a solução indispensável para consumidores e empresas de grande porte:

1.	Monitoramento em Tempo Real: Dados atualizados sobre desempenho e eficiência dos painéis solares.

2.	Identificação Proativa de Problemas: Ajuda na manutenção preventiva, evitando prejuízos por falhas.

3.	Otimização de Recursos: Permite ajustes no consumo com base na geração de energia, reduzindo custos operacionais.

4.	Relatórios e Transparência: Facilita a prestação de contas para stakeholders e promove práticas de responsabilidade ambiental.
Benefícios que buscamos alcançar para o Negócio:
•	Redução de custos operacionais para empresas que gerenciam grandes instalações solares
•	Aumento da competitividade ao agregar valor a empresas do setor solar.
•	Redução de custos e maior retorno sobre investimento para consumidores.
•	Contribuição direta para iniciativas de sustentabilidade e práticas ESG.


---

## 🔧 **Instruções para Execução**

### **Passos para Configurar o Projeto**
1. Clone o repositório:
   ```bash
   git clone https://github.com/DanielAraujoFaria/SolarSense-IA
   ```
   Depois, na pasta src faça build do main.cpp e então abre o arquivo diagram.json e rode o projeto
   
2. Ou acesse [Link do Projeto no Wokwi](https://wokwi.com/projects/415311717366535169)

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
    { "type": "wokwi-dht22", "id": "dht1", "top": -66.9, "left": -130.2, "attrs": {} },
    {
      "type": "wokwi-led",
      "id": "led1",
      "top": 73.2,
      "left": -188.2,
      "attrs": { "color": "red", "flip": "" }
    }
  ],
  "connections": [
    [ "esp:TX", "$serialMonitor:RX", "", [] ],
    [ "esp:RX", "$serialMonitor:TX", "", [] ],
    [ "esp:D34", "pot1:W", "analog", [ "PINO_TENSAO" ] ],
    [ "esp:D2", "ledOnboard:anode", "digital", [ "boardLED" ] ],
    [ "ledOnboard:cathode", "esp:GND", "ground", [] ],
    [ "pot1:GND", "esp:GND", "ground", [] ],
    [ "pot1:VCC", "esp:3V3", "power", [] ],
    [ "esp:D25", "dht:SIG", "digital", [ "DHTPIN" ] ],
    [ "dht:GND", "esp:GND", "ground", [] ],
    [ "dht:VCC", "esp:3V3", "power", [] ],
    [ "esp:D26", "ledExternal:anode", "digital", [ "LED_PIN" ] ],
    [ "ledExternal:cathode", "esp:GND", "ground", [] ],
    [ "pot1:GND", "esp:GND.2", "black", [ "v57.6", "h57.6" ] ],
    [ "pot1:SIG", "esp:34", "white", [ "v28.8", "h-106.15" ] ],
    [ "dht1:GND", "esp:GND.1", "black", [ "v0" ] ],
    [ "dht1:VCC", "esp:3V3", "red", [ "v0" ] ],
    [ "led1:A", "esp:26", "green", [ "v0" ] ],
    [ "led1:C", "esp:GND.3", "cyan", [ "v211.2", "h259.6", "v-163.2" ] ]
  ],
  "dependencies": {}
}
```

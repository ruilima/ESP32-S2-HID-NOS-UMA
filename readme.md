# 🎮 UMA NOS CONTROLLER
# Controlo da Box UMA via Home Assistant

=======================================
📦 CONTEÚDO
=======================================

Este ficheiro contém TUDO o que precisas para instalar o projeto:

1. ✅ Documentação completa
2. ✅ Código Arduino pronto a usar
3. ✅ Configurações Home Assistant
4. ✅ Dashboard exemplo
5. ✅ Scripts e automações

=======================================
🚀 INÍCIO RÁPIDO
=======================================

HARDWARE NECESSÁRIO:
- ESP32-S2
- Cabo USB - Caso o ESP32-S2 não tenha Porta USB
- Box UMA da NOS

PASSOS:
1. Comprar ESP32-S2
2. Instalar Arduino IDE
3. Copiar código abaixo
4. Configurar WiFi/MQTT
5. Upload para ESP32
6. Ligar à UMA
7. Pronto! 🎉

=======================================
📝 CÓDIGO ARDUINO (uma_controller.ino)
=======================================

INSTRUÇÕES:
1. Abrir Arduino IDE
2. Criar novo sketch
3. Copiar TODO o código abaixo
4. Editar WiFi_SSID, WIFI_PASSWORD, MQTT_SERVER
5. Tools → Board → ESP32S2 Dev Module
6. Tools → USB CDC On Boot → Enabled
7. Upload!

--- INÍCIO DO CÓDIGO ---

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include "USB.h"
#include "USBHIDKeyboard.h"
#include "USBHIDConsumerControl.h"
#include <ArduinoOTA.h>

// ========== EDITA AQUI ==========
const char* WIFI_SSID = "TUA_REDE_WIFI";
const char* WIFI_PASSWORD = "TUA_PASSWORD";
const char* MQTT_SERVER = "192.168.1.100";  // IP do Home Assistant
const int MQTT_PORT = 1883;
const char* MQTT_USER = "";  // Opcional
const char* MQTT_PASSWORD = "";  // Opcional
const char* DEVICE_NAME = "uma_controller";
const char* DISCOVERY_PREFIX = "homeassistant";
// ================================

WiFiClient espClient;
PubSubClient mqttClient(espClient);
USBHIDKeyboard Keyboard;
USBHIDConsumerControl ConsumerControl;
unsigned long lastReconnectAttempt = 0;

void sendKey(uint8_t key) {
  Keyboard.press(key);
  delay(50);
  Keyboard.releaseAll();
  delay(50);
}

void sendChar(char c) {
  Keyboard.write(c);
  delay(50);
}

void setupWiFi() {
  Serial.println("A ligar ao WiFi...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi ligado! IP: " + WiFi.localIP().toString());
}

void setupOTA() {
  ArduinoOTA.setHostname(DEVICE_NAME);
  ArduinoOTA.setPassword("uma123");
  ArduinoOTA.begin();
  Serial.println("OTA Ready!");
}

void mqttCallback(char* topic, byte* payload, unsigned int length) {
  String message;
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  
  if (String(topic) == String(DISCOVERY_PREFIX) + "/uma/command") {
    if (message == "page_up") sendKey(KEY_PAGE_UP);
    else if (message == "page_down") sendKey(KEY_PAGE_DOWN);
    else if (message == "f1") sendKey(KEY_F1);
    else if (message == "back") sendKey(KEY_BACKSPACE);
    else if (message == "enter") sendKey(KEY_RETURN);
    else if (message == "rewind") sendChar('r');
    else if (message == "forward") sendChar('f');
    else if (message == "play_pause") sendChar('p');
    else if (message == "power") sendKey(KEY_F12);
    else if (message == "menu") sendKey(KEY_HOME);
    else if (message == "up") sendKey(KEY_UP_ARROW);
    else if (message == "down") sendKey(KEY_DOWN_ARROW);
    else if (message == "left") sendKey(KEY_LEFT_ARROW);
    else if (message == "right") sendKey(KEY_RIGHT_ARROW);
    else if (message == "volume_up") {
      ConsumerControl.press(CONSUMER_CONTROL_VOLUME_INCREMENT);
      delay(50);
      ConsumerControl.release();
    }
    else if (message == "volume_down") {
      ConsumerControl.press(CONSUMER_CONTROL_VOLUME_DECREMENT);
      delay(50);
      ConsumerControl.release();
    }
    else if (message == "mute") {
      ConsumerControl.press(CONSUMER_CONTROL_MUTE);
      delay(50);
      ConsumerControl.release();
    }
    else if (message == "num_0") sendChar('0');
    else if (message == "num_1") sendChar('1');
    else if (message == "num_2") sendChar('2');
    else if (message == "num_3") sendChar('3');
    else if (message == "num_4") sendChar('4');
    else if (message == "num_5") sendChar('5');
    else if (message == "num_6") sendChar('6');
    else if (message == "num_7") sendChar('7');
    else if (message == "num_8") sendChar('8');
    else if (message == "num_9") sendChar('9');
    
    mqttClient.publish((String(DISCOVERY_PREFIX) + "/uma/status").c_str(), message.c_str());
  }
}

void publishDiscoveryConfigs() {
  String buttons[] = {"up", "down", "left", "right", "enter", "back", "menu", 
                     "page_up", "page_down", "f1", "play_pause", "forward", 
                     "rewind", "power", "volume_up", "volume_down", "mute",
                     "num_0", "num_1", "num_2", "num_3", "num_4", 
                     "num_5", "num_6", "num_7", "num_8", "num_9"};
  
  String names[] = {"Seta Cima", "Seta Baixo", "Seta Esquerda", "Seta Direita",
                   "Enter/OK", "Voltar", "Menu/Home", "Canal Acima", "Canal Abaixo",
                   "Info (i)", "Play/Pause", "Avançar", "Recuar", "Power",
                   "Volume +", "Volume -", "Mute",
                   "Número 0", "Número 1", "Número 2", "Número 3", "Número 4",
                   "Número 5", "Número 6", "Número 7", "Número 8", "Número 9"};
  
  for (int i = 0; i < 27; i++) {
    StaticJsonDocument<512> doc;
    doc["name"] = names[i];
    doc["unique_id"] = String(DEVICE_NAME) + "_" + buttons[i];
    doc["command_topic"] = String(DISCOVERY_PREFIX) + "/uma/command";
    doc["payload_press"] = buttons[i];
    doc["qos"] = 0;
    
    JsonObject device = doc.createNestedObject("device");
    device["identifiers"][0] = DEVICE_NAME;
    device["name"] = "UMA NOS Controller";
    device["model"] = "ESP32-S2";
    device["manufacturer"] = "DIY";
    
    String configTopic = String(DISCOVERY_PREFIX) + "/button/" + 
                        DEVICE_NAME + "_" + buttons[i] + "/config";
    
    String output;
    serializeJson(doc, output);
    mqttClient.publish(configTopic.c_str(), output.c_str(), true);
    delay(50);
  }
}

boolean mqttReconnect() {
  if (mqttClient.connect(DEVICE_NAME, MQTT_USER, MQTT_PASSWORD)) {
    mqttClient.subscribe((String(DISCOVERY_PREFIX) + "/uma/command").c_str());
    publishDiscoveryConfigs();
    mqttClient.publish((String(DISCOVERY_PREFIX) + "/uma/availability").c_str(), "online", true);
    return true;
  }
  return false;
}

void setup() {
  Serial.begin(115200);
  Serial.println("\nUMA NOS Controller - ESP32-S2");
  
  Keyboard.begin();
  ConsumerControl.begin();
  USB.begin();
  
  setupWiFi();
  setupOTA();
  
  mqttClient.setServer(MQTT_SERVER, MQTT_PORT);
  mqttClient.setCallback(mqttCallback);
  mqttClient.setBufferSize(1024);
  
  Serial.println("Sistema pronto!");
}

void loop() {
  ArduinoOTA.handle();
  
  if (WiFi.status() != WL_CONNECTED) {
    setupWiFi();
  }
  
  if (!mqttClient.connected()) {
    unsigned long now = millis();
    if (now - lastReconnectAttempt > 5000) {
      lastReconnectAttempt = now;
      if (mqttReconnect()) {
        lastReconnectAttempt = 0;
      }
    }
  } else {
    mqttClient.loop();
  }
}
```

--- FIM DO CÓDIGO ---

=======================================
🏠 HOME ASSISTANT - DASHBOARD
=======================================

Copia este YAML para o teu dashboard:

Settings → Dashboards → Editar → Adicionar Card → Manual

```yaml
type: vertical-stack
cards:
  - type: horizontal-stack
    cards:
      - type: button
        entity: button.uma_nos_controller_power
        name: Power
        icon: mdi:power
      - type: button
        entity: button.uma_nos_controller_menu_home
        name: Menu
        icon: mdi:menu
      - type: button
        entity: button.uma_nos_controller_info_i
        name: Info
        icon: mdi:information
  
  - type: grid
    columns: 3
    cards:
      - show_name: false
        show_icon: false
        type: button
      - type: button
        entity: button.uma_nos_controller_seta_cima
        icon: mdi:chevron-up
        show_name: false
      - show_name: false
        show_icon: false
        type: button
      - type: button
        entity: button.uma_nos_controller_seta_esquerda
        icon: mdi:chevron-left
        show_name: false
      - type: button
        entity: button.uma_nos_controller_enter_ok
        name: OK
      - type: button
        entity: button.uma_nos_controller_seta_direita
        icon: mdi:chevron-right
        show_name: false
      - show_name: false
        show_icon: false
        type: button
      - type: button
        entity: button.uma_nos_controller_seta_baixo
        icon: mdi:chevron-down
        show_name: false
      - show_name: false
        show_icon: false
        type: button
```

=======================================
📚 BIBLIOTECAS ARDUINO NECESSÁRIAS
=======================================

Instalar via Tools → Manage Libraries:

1. PubSubClient (por Nick O'Leary)
2. ArduinoJson (por Benoit Blanchon)

=======================================
⚙️ CONFIGURAÇÃO ESP32
=======================================

Tools → Board → ESP32S2 Dev Module

Tools → USB CDC On Boot → Enabled

Tools → Upload Speed → 921600

=======================================
✅ CHECKLIST DE INSTALAÇÃO
=======================================

[ ] Comprar ESP32-S2

[ ] Instalar Arduino IDE

[ ] Adicionar board ESP32 (URL: https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json)

[ ] Instalar bibliotecas (PubSubClient, ArduinoJson)

[ ] Copiar código Arduino

[ ] Editar WiFi_SSID e MQTT_SERVER

[ ] Upload para ESP32

[ ] Testar Serial Monitor (115200 baud)

[ ] Desligar do PC

[ ] Ligar à porta USB da UMA

[ ] Verificar botões no Home Assistant

[ ] Adicionar dashboard

[ ] Testar controlo!

=======================================
🎯 FUNCIONALIDADES
=======================================

27 BOTÕES DISPONÍVEIS:
- Navegação (setas, enter, voltar, menu)
- Canais (page up/down, números 0-9)
- Volume (up, down, mute)
- Reprodução (play/pause, forward, rewind, info)
- Sistema (power)

OTA UPDATES:
- Após primeira instalação, updates via WiFi
- Tools → Port → uma_controller at 192.168.X.X


=======================================
📞 SUPORTE
=======================================

Problemas? Verifica:

1. ESP32 está conectado ao WiFi?
   - Serial Monitor mostra "WiFi ligado"?

2. MQTT está OK?
   - Settings → Devices → MQTT
   - ESP32 aparece online?

3. Botões não aparecem?
   - Reinicia ESP32 (desliga/liga)
   - Verifica MQTT Discovery ativado no HA

4. Botões não funcionam na UMA?
   - ESP32 está ligado à PORTA USB da UMA?
   - NÃO ao computador!

=======================================
📄 LICENÇA
=======================================

MIT License - Uso livre

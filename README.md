# ESP32-Fardriver-translator

EN: This code enables you to retrieve data from the Fardriver controller and transmit it to the web interface of your device. Default WI-FI name: Fardriver_Dash, pass: password.

RU: Код который берет данные из контроллера Fardriver и передает из по WI-FI на веб интерфейс вашего устройства.
По умолчанию точка доступа называется Fardriver_Dash, пароль: password.

релизы новых версий буду распологатся тут в README.MD а старые версии будут в ветке с их версией.txt

Releases of new versions will be located here in README.MD, and older versions will be in the branch with their version.txt.


Code/Код V2: 
```C++
#include <WiFi.h>
#include <WebServer.h>
#include <HardwareSerial.h>

// === YOUR CONFIG ===
const float WHEEL_CIRCUMFERENCE_M = 0.877; // Окружность 11-дюймового колеса M5 Pro (метры)
const float BATTERY_MAX_V = 54.6;          // 100% заряда (13S)
const float BATTERY_MIN_V = 39.0;          // 0% заряда (13S)
// ==========================

const char* ssid = "Fardriver_Dash";
const char* password = "password"; 

WebServer server(80);

struct FardriverData {
  float voltage = 0;
  float lineCurrent = 0;
  float power = 0;
  float rpm = 0;
  int16_t rawRpm = 0;
  int gear = 0;
  float speed = 0;
  int controllerTemp = 0;
  int motorTemp = 0;
  int soc = 0;
  bool isRegenFromCurrent = false;
  unsigned long lastUpdate = 0;
};

FardriverData controllerData;

const uint8_t flash_read_addr[55] = {
  0xE2, 0xE8, 0xEE, 0xE4, 0x06, 0x0C, 0x12, 0xE2, 0xE8, 0xEE, 0x18, 0x1E, 0x24, 0x2A,
  0xE2, 0xE8, 0xEE, 0x30, 0x5D, 0x63, 0x69, 0xE2, 0xE8, 0xEE, 0x7C, 0x82, 0x88, 0x8E,
  0xE2, 0xE8, 0xEE, 0x94, 0x9A, 0xA0, 0xA6, 0xE2, 0xE8, 0xEE, 0xAC, 0xB2, 0xB8, 0xBE,
  0xE2, 0xE8, 0xEE, 0xC4, 0xCA, 0xD0, 0xE2, 0xE8, 0xEE, 0xD6, 0xDC, 0xF4, 0xFA
};

String passwords[] = {
  "+VERSION:1.0\r\n", "OK\r\n", "+VERSION=1.0\r\n", "OK",
  "+VERSION:3.0\r\n", "JDY-08\r\n", "JDY-10\r\n", "JDY-24\r\n",
  "DX-BT24\r\n", "HC-08\r\n", "AT+VERSION\r\n", "VERSION\r\n"
};
int num_passwords = 12;
int current_pwd = 0;
uint8_t requestPacket[] = {0x02, 0x01, 0x04, 0x40, 0x84, 0x03};

bool verifyCRC(uint8_t* data, uint16_t length) {
  if (length != 16) return false;
  uint16_t crc = 0x7F3C;
  for (int i = 0; i < 14; i++) {
    crc ^= data[i];
    for (int j = 0; j < 8; j++) {
      crc = (crc & 1) ? (crc >> 1) ^ 0xA001 : crc >> 1;
    }
  }
  return crc == ((data[15] << 8) | data[14]);
}

void processPacket(uint8_t* packet, size_t length) {
  if (!verifyCRC(packet, length)) return;
  uint8_t id = packet[1] & 0x3F;
  if (id >= sizeof(flash_read_addr)) return;
  uint8_t address = flash_read_addr[id];
  uint8_t* pData = &packet[2];
  
  switch (address) {
    case 0xE2:
      controllerData.gear = ((pData[0] >> 2) & 0b11) + 1;
      controllerData.rawRpm = (int16_t)((pData[7] << 8) | pData[6]);
      break;
    case 0xE8: {
      float new_voltage = ((pData[1] << 8) | pData[0]) / 10.0f;
      if (new_voltage >= 0 && new_voltage <= 150.0f) { controllerData.voltage = new_voltage; }
      float new_lineCurrent = (int16_t)((pData[5] << 8) | pData[4]) / 4.0f;
      if (new_lineCurrent >= -100.0f && new_lineCurrent <= 300.0f) {  
        controllerData.lineCurrent = new_lineCurrent;  
      }
      controllerData.isRegenFromCurrent = (controllerData.lineCurrent < 0);
      break;
    }
    case 0xD6: {
      int new_controllerTemp = (int16_t)((pData[11] << 8) | pData[10]);
      if (new_controllerTemp > -20 && new_controllerTemp < 150) { controllerData.controllerTemp = new_controllerTemp; }
      break;
    }
    case 0xF4: {
      int new_motorTemp = (int16_t)((pData[1] << 8) | pData[0]);
      if (new_motorTemp > -20 && new_motorTemp < 200) { controllerData.motorTemp = new_motorTemp; }
      // SOC от Фарика игнорируем, считаем сами ниже!
      break;
    }
  }
  
  // 1. Исправленная скорость (Фарик шлет реальные RPM, делить не надо)
  controllerData.rpm = (controllerData.rawRpm < 0) ? 0 : controllerData.rawRpm;
  controllerData.speed = (controllerData.rpm * WHEEL_CIRCUMFERENCE_M * 60.0) / 1000.0;
  
  // 2. Исправленный SOC (свой собственный расчет по вольтажу)
  if (controllerData.voltage >= BATTERY_MAX_V) {
    controllerData.soc = 100;
  } else if (controllerData.voltage <= BATTERY_MIN_V) {
    controllerData.soc = 0;
  } else {
    controllerData.soc = (int)(((controllerData.voltage - BATTERY_MIN_V) / (BATTERY_MAX_V - BATTERY_MIN_V)) * 100.0);
  }

  // 3. Мощность и таймер
  controllerData.power = controllerData.voltage * controllerData.lineCurrent;
  controllerData.lastUpdate = millis();
}

// Версия 2.0.1 HTML
const char* htmlPage = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Fardriver Ultimate Dash v2.0</title>
  <style>
    body { background-color: #080a0f; color: #c9d1d9; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; text-align: center; padding: 10px; margin: 0; }
    .header { display: flex; justify-content: space-between; align-items: center; background: #161b22; padding: 12px 20px; border-radius: 14px; border: 1px solid #30363d; max-width: 460px; margin: 0 auto 15px auto; }
    .header h1 { font-size: 18px; color: #f0f6fc; margin: 0; display: flex; align-items: center; gap: 10px; }
    
    .status-dot { width: 10px; height: 10px; background-color: #f85149; border-radius: 50%; box-shadow: 0 0 8px #f85149; transition: 0.3s; }
    .status-dot.active { background-color: #3fb950; box-shadow: 0 0 10px #3fb950; }
    .btn-settings { background: none; border: none; font-size: 20px; cursor: pointer; color: #8b949e; transition: 0.2s; }
    .btn-settings:hover { color: #f0f6fc; }
    .top-right { display: flex; align-items: center; gap: 15px; }

    .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; max-width: 500px; margin: 0 auto; }
    .card { background: #161b22; padding: 15px; border-radius: 14px; border: 1px solid #30363d; text-align: left; transition: display 0.3s; }
    .card label { display: block; color: #8b949e; font-size: 11px; text-transform: uppercase; margin-bottom: 5px; font-weight: 600; letter-spacing: 0.5px; }
    .card span { font-size: 24px; font-weight: 700; color: #58a6ff; }
    .card.accent span { color: #3fb950; }
    .card.warning span { color: #d29922; }
    .big-card { grid-column: span 2; background: linear-gradient(135deg, #161b22 0%, #1f242c 100%); padding: 20px; text-align: center; }
    .big-card span { font-size: 48px; color: #58a6ff; }
    .unit { font-size: 16px; color: #8b949e; font-weight: normal; }
    
    .modal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.8); z-index: 100; justify-content: center; align-items: center; }
    .modal-content { background: #161b22; padding: 20px; border-radius: 14px; border: 1px solid #30363d; width: 85%; max-width: 350px; text-align: left; }
    .modal-content h2 { margin-top: 0; color: #f0f6fc; font-size: 18px; margin-bottom: 15px; }
    .toggle-row { display: flex; justify-content: space-between; align-items: center; padding: 8px 0; border-bottom: 1px solid #30363d; }
    .toggle-row:last-child { border-bottom: none; }
    .close-btn { width: 100%; padding: 10px; margin-top: 15px; background: #238636; color: white; border: none; border-radius: 8px; font-weight: bold; cursor: pointer; }
    
    .switch { position: relative; display: inline-block; width: 40px; height: 20px; }
    .switch input { opacity: 0; width: 0; height: 0; }
    .slider { position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background-color: #30363d; transition: .4s; border-radius: 20px; }
    .slider:before { position: absolute; content: ""; height: 14px; width: 14px; left: 3px; bottom: 3px; background-color: #8b949e; transition: .4s; border-radius: 50%; }
    input:checked + .slider { background-color: #238636; }
    input:checked + .slider:before { transform: translateX(20px); background-color: #fff; }

    .footer { margin-top: 15px; color: #484f58; font-size: 12px; }
  </style>
</head>
<body>
  <div class="header">
    <h1>🛴 Dash v2.0</h1>
    <div class="top-right">
      <span id="statusDot" class="status-dot"></span>
      <button class="btn-settings" onclick="openSettings()">⚙️</button>
    </div>
  </div>

  <div class="grid" id="dashGrid">
    <div class="card big-card" id="card_speed">
      <label>Скорость</label>
      <span id="speed">0.0</span><span class="unit" style="font-size:24px"> km/h</span>
    </div>
    <div class="card accent" id="card_volt">
      <label>Напряжение</label>
      <span id="volt">-- V</span>
    </div>
    <div class="card warning" id="card_current">
      <label>Ток</label>
      <span id="current">-- A</span>
    </div>
    <div class="card" id="card_power">
      <label>Мощность</label>
      <span id="power">-- W</span>
    </div>
    <div class="card" id="card_soc">
      <label>Заряд (SOC)</label>
      <span id="soc">-- %</span>
    </div>
    <div class="card" id="card_ctemp">
      <label>Т. Контроллера</label>
      <span id="c_temp">-- °C</span>
    </div>
    <div class="card" id="card_mtemp">
      <label>Т. Мотора</label>
      <span id="m_temp">-- °C</span>
    </div>
  </div>

  <div class="footer">ESP32 Direct UART Bridge</div>

  <div class="modal" id="settingsModal">
    <div class="modal-content">
      <h2>Настройки дашборда</h2>
      <div id="togglesContainer"></div>
      <button class="close-btn" onclick="closeSettings()">Готово</button>
    </div>
  </div>

  <script>
    const widgets = [
      { id: 'card_speed', name: 'Скорость (Main)' },
      { id: 'card_volt', name: 'Напряжение' },
      { id: 'card_current', name: 'Ток' },
      { id: 'card_power', name: 'Мощность' },
      { id: 'card_soc', name: 'Заряд (SOC)' },
      { id: 'card_ctemp', name: 'Т. Контроллера' },
      { id: 'card_mtemp', name: 'Т. Мотора' }
    ];

    function initSettings() {
      let container = document.getElementById('togglesContainer');
      widgets.forEach(w => {
        let isVisible = localStorage.getItem(w.id) !== 'hidden';
        if (!isVisible) document.getElementById(w.id).style.display = 'none';

        container.innerHTML += `
          <div class="toggle-row">
            <span style="color:#c9d1d9; font-size:14px;">${w.name}</span>
            <label class="switch">
              <input type="checkbox" id="toggle_${w.id}" ${isVisible ? 'checked' : ''} onchange="toggleCard('${w.id}')">
              <span class="slider"></span>
            </label>
          </div>
        `;
      });
    }

    function toggleCard(id) {
      let cb = document.getElementById('toggle_' + id);
      if (cb.checked) {
        document.getElementById(id).style.display = 'block';
        localStorage.setItem(id, 'visible');
      } else {
        document.getElementById(id).style.display = 'none';
        localStorage.setItem(id, 'hidden');
      }
    }

    function openSettings() { document.getElementById('settingsModal').style.display = 'flex'; }
    function closeSettings() { document.getElementById('settingsModal').style.display = 'none'; }

    initSettings();

    setInterval(function() {
      fetch('/json').then(response => response.json()).then(data => {
        document.getElementById("volt").innerHTML = data.volt.toFixed(1) + '<span class="unit"> V</span>';
        document.getElementById("current").innerHTML = data.current.toFixed(1) + '<span class="unit"> A</span>';
        document.getElementById("speed").innerHTML = data.speed.toFixed(1);
        document.getElementById("power").innerHTML = Math.round(data.power) + '<span class="unit"> W</span>';
        document.getElementById("c_temp").innerHTML = data.cTemp + '<span class="unit"> °C</span>';
        document.getElementById("m_temp").innerHTML = data.mTemp + '<span class="unit"> °C</span>';
        document.getElementById("soc").innerHTML = data.soc + '<span class="unit"> %</span>';
        
        let dot = document.getElementById("statusDot");
        if (data.online) dot.classList.add("active");
        else dot.classList.remove("active");
      }).catch(e => {
        document.getElementById("statusDot").classList.remove("active");
      });
    }, 250); 
  </script>
</body>
</html>
)rawliteral";

void setup() {
  Serial.begin(115200);
  Serial2.begin(19200, SERIAL_8N1, 16, 17);

  WiFi.softAP(ssid, password);
  
  server.on("/", []() { server.send(200, "text/html", htmlPage); });
  
  server.on("/json", []() {
    bool isOnline = (millis() - controllerData.lastUpdate < 2000); 
    String json = "{";
    json += "\"volt\":" + String(controllerData.voltage) + ",";
    json += "\"current\":" + String(controllerData.lineCurrent) + ",";
    json += "\"speed\":" + String(controllerData.speed) + ",";
    json += "\"power\":" + String(controllerData.power) + ",";
    json += "\"cTemp\":" + String(controllerData.controllerTemp) + ",";
    json += "\"mTemp\":" + String(controllerData.motorTemp) + ",";
    json += "\"soc\":" + String(controllerData.soc) + ",";
    json += "\"online\":" + String(isOnline ? "true" : "false");
    json += "}";
    server.send(200, "application/json", json);
  });

  server.begin();
}

void loop() {
  server.handleClient();

  if (Serial2.available()) {
    delay(20);
    uint8_t buf[128];
    int i = 0;
    bool hasAT = false;

    while (Serial2.available() && i < 128) {
      buf[i] = Serial2.read();
      if (i >= 1 && buf[i-1] == 0x41 && buf[i] == 0x54) hasAT = true;
      i++;
    }

    if (hasAT) {
      String pwd = passwords[current_pwd % num_passwords];
      Serial2.print(pwd);
      delay(50);
      Serial2.write(requestPacket, sizeof(requestPacket));
      current_pwd++;
      delay(150);
    } 
    else {
      for (int k = 0; k <= i - 16; k++) {
        if (buf[k] == 0xAA) {
          processPacket(&buf[k], 16);
          k += 15; 
        }
      }
    }
  }
}

```

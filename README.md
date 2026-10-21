# ESP32-Fardriver-translator
EN: This code enables you to retrieve data from the Fardriver controller and transmit it to the web interface of your device.
RU: Код который берет данные из контроллера Fardriver и передает из по WI-FI на веб интерфейс вашего устройства.
По умолчанию точка доступа называется Fardriver_Dash, пароль: password.
Code/Код: 
#include <WiFi.h>
#include <WebServer.h>
#include <HardwareSerial.h>

const char* ssid = "Fardriver_Dash";
const char* password = "password"; 

WebServer server(80);

// Тут кстати вся структура
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
      controllerData.soc = pData[3];
      break;
    }
  }
  
  const float wheel_circumference_m = 1.416;
  const int motor_pole_pairs = 20;

  int16_t displayRawRpm = (controllerData.rawRpm < 0) ? 0 : controllerData.rawRpm;
  controllerData.rpm = displayRawRpm * 4.0 / motor_pole_pairs;
  controllerData.speed = (controllerData.rpm * wheel_circumference_m * 60.0) / 1000.0;
  
  controllerData.power = controllerData.voltage * controllerData.lineCurrent;
  controllerData.lastUpdate = millis();
}

const char* htmlPage = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Fardriver Ultimate Dash</title>
  <style>
    body { background-color: #080a0f; color: #c9d1d9; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; text-align: center; padding: 10px; margin: 0; }
    .header { display: flex; justify-content: space-between; align-items: center; background: #161b22; padding: 12px 20px; border-radius: 14px; border: 1px solid #30363d; max-width: 460px; margin: 0 auto 15px auto; }
    .header h1 { font-size: 18px; color: #f0f6fc; margin: 0; }
    .status-dot { width: 10px; height: 10px; background-color: #f85149; border-radius: 50%; display: inline-block; box-shadow: 0 0 8px #f85149; transition: 0.3s; }
    .status-dot.active { background-color: #3fb950; box-shadow: 0 0 10px #3fb950; }
    
    .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; max-width: 500px; margin: 0 auto; }
    .card { background: #161b22; padding: 15px; border-radius: 14px; border: 1px solid #30363d; text-align: left; position: relative; overflow: hidden; }
    .card label { display: block; color: #8b949e; font-size: 11px; text-transform: uppercase; margin-bottom: 5px; font-weight: 600; letter-spacing: 0.5px; }
    .card span { font-size: 24px; font-weight: 700; color: #58a6ff; }
    .card.accent span { color: #3fb950; }
    .card.warning span { color: #d29922; }
    
    .big-card { grid-column: span 2; background: linear-gradient(135deg, #161b22 0%, #1f242c 100%); padding: 20px; text-align: center; }
    .big-card span { font-size: 48px; color: #58a6ff; }
    .unit { font-size: 16px; color: #8b949e; font-weight: normal; }
    
    .footer { margin-top: 15px; color: #484f58; font-size: 12px; }
  </style>
  <script>
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
        if (data.online) {
          dot.classList.add("active");
        } else {
          dot.classList.remove("active");
        }
      });
    }, 250); 
  </script>
</head>
<body>
  <div class="header">
    <h1>Fardriver Dash</h1>
    <div><span id="statusDot" class="status-dot"></span></div>
  </div>

  <div class="grid">
    <div class="card big-card">
      <label>Speed</label>
      <span id="speed">0.0</span><span class="unit" style="font-size:24px"> km/h</span>
    </div>

    <div class="card accent">
      <label>Voltage</label>
      <span id="volt">-- V</span>
    </div>
    
    <div class="card warning">
      <label>Current</label>
      <span id="current">-- A</span>
    </div>

    <div class="card">
      <label>Power</label>
      <span id="power">-- W</span>
    </div>

    <div class="card">
      <label>Charge (SOC)</label>
      <span id="soc">-- %</span>
    </div>

    <div class="card">
      <label>T° Contoller</label>
      <span id="c_temp">-- °C</span>
    </div>

    <div class="card">
      <label>T° Motors</label>
      <span id="m_temp">-- °C</span>
    </div>
  </div>

  <div class="footer">ESP32 Direct UART Bridge</div>
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

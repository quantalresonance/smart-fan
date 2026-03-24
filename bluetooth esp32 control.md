#include <WiFi.h>
#include <WebServer.h>

WebServer server(80);

const char* ssid = "ESP32-Control";
const char* password = "12345678";

const int LED_PIN = 2;
bool blinkMode = false;      // For FAN / LED blinking
bool lightOn = false;        // For Toggle Light button

// ------------------ BEAUTIFUL HTML PAGE ------------------
String webpage = R"=====(
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Smart Home Remote - WiFi Version</title>
  <style>
    body { font-family: Arial, sans-serif; margin:0; padding:0; display:flex; justify-content:center; align-items:center; min-height:100vh; background:#f5f5f5; }
    .remote { width:350px; padding:20px; background:white; border-radius:20px; box-shadow:0 4px 10px rgba(0,0,0,0.2); text-align:center; }
    .temperature-section { margin-bottom:20px; }
    .temperature-section h2 { font-size:18px; margin-bottom:10px; }
    .number-display { font-size:32px; font-weight:bold; margin-bottom:15px; }
    #temperature-slider { display:block; margin:0 auto 20px; width:80%; }
    .rgb-wheel { position:relative; margin:20px 0; display:flex; justify-content:center; align-items:center; }
    .rgb-circle { width:150px; height:150px; border-radius:50%; background:conic-gradient(red,yellow,green,cyan,blue,magenta,red); display:flex; justify-content:center; align-items:center; }
    .rgb-circle span { width:80px; height:80px; background:white; border-radius:50%; display:flex; justify-content:center; align-items:center; font-size:18px; font-weight:bold; }
    .circle-scroll { position:absolute; width:30px; height:30px; border-radius:50%; background:white; border:3px solid #007bff; cursor:pointer; }
    .current-color { margin-top:20px; padding:10px; background:#007bff; color:white; border-radius:5px; font-size:14px; }
    .modes { display:flex; flex-wrap:wrap; justify-content:space-around; margin-top:20px; }
    .modes button { padding:10px 15px; font-size:14px; border:none; border-radius:10px; cursor:pointer; background:#007bff; color:white; margin:5px; }
  </style>
</head>

<body>
<div class="remote">

  <div class="temperature-section">
    <h2>Temperature to be Maintained</h2>
    <div class="number-display" id="temperature-display">25°C</div>
    <input type="range" id="temperature-slider" min="20" max="40" value="25"
      oninput="updateTemperature(this.value)">
  </div>

  <div class="rgb-wheel">
    <div class="rgb-circle">
      <span>RGB</span>
    </div>
    <div class="circle-scroll" id="circle-scroll" style="top:10px; left:65px;" onmousedown="startDrag(event)"></div>
  </div>

  <div class="current-color" id="current-color">Current Color: #FF0000</div>

  <div class="modes">
    <button onclick="sendMode('RGB')">RGB Cyclic Mode</button>
    <button onclick="sendMode('TIME')">Display Time</button>
    <button onclick="sendMode('FAN')">Fan ON</button>
    <button onclick="sendMode('TEMP')">Display Temperature</button>
    <button onclick="toggleLight()">Toggle Light</button>
  </div>

</div>

<script>

let isDragging = false;

async function sendColorToESP32(color) {
  const rgb = color.match(/\d+/g).map(Number);

  fetch(`/color`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ r: rgb[0], g: rgb[1], b: rgb[2] })
  });
}

function sendMode(modeName) {
  fetch(`/mode`, {
    method: "POST",
    headers: {"Content-Type": "application/json"},
    body: JSON.stringify({ mode: modeName })
  });
}

function updateTemperature(value) {
  document.getElementById("temperature-display").innerText = value + "°C";

  fetch(`/temperature`, {
    method: "POST",
    headers: {"Content-Type": "application/json"},
    body: JSON.stringify({ temperature: value })
  });
}

// ------------------- Toggle Light -------------------
function toggleLight() {
  fetch(`/toggleLight`, {
    method: "POST",
    headers: {"Content-Type": "application/json"}
  });
}

// ------------------- RGB Color Wheel -----------------
let circleScroll = document.getElementById("circle-scroll");
let rgbWheel = document.querySelector(".rgb-wheel");
let currentColorBox = document.getElementById("current-color");

function startDrag(e) {
  isDragging = true;
  document.addEventListener("mousemove", drag);
  document.addEventListener("mouseup", stopDrag);
}

function drag(e) {
  if (!isDragging) return;

  let rect = rgbWheel.getBoundingClientRect();
  let centerX = rect.left + rect.width / 2;
  let centerY = rect.top + rect.height / 2;

  let angle = Math.atan2(e.clientY - centerY, e.clientX - centerX);
  let radius = 65;

  let x = centerX + radius * Math.cos(angle) - circleScroll.offsetWidth / 2;
  let y = centerY + radius * Math.sin(angle) - circleScroll.offsetHeight / 2;

  circleScroll.style.left = `${x - rect.left}px`;
  circleScroll.style.top = `${y - rect.top}px`;

  let color = getColorFromAngle(angle);
  currentColorBox.style.backgroundColor = color;
  currentColorBox.innerText = "Current Color: " + color;

  sendColorToESP32(color);
}

function stopDrag() {
  isDragging = false;
  document.removeEventListener("mousemove", drag);
  document.removeEventListener("mouseup", stopDrag);
}

function getColorFromAngle(angle) {
  let hue = (angle + Math.PI) / (2 * Math.PI);
  let rgb = hsvToRgb(hue, 1, 1);
  return `rgb(${rgb[0]}, ${rgb[1]}, ${rgb[2]})`;
}

function hsvToRgb(h, s, v) {
  let r, g, b;
  let i = Math.floor(h * 6);
  let f = h * 6 - i;
  let p = v * (1 - s);
  let q = v * (1 - f * s);
  let t = v * (1 - (1 - f) * s);

  switch (i % 6) {
    case 0: r=v,g=t,b=p; break;
    case 1: r=q,g=v,b=p; break;
    case 2: r=p,g=v,b=t; break;
    case 3: r=p,g=q,b=v; break;
    case 4: r=t,g=p,b=v; break;
    case 5: r=v,g=p,b=q; break;
  }

  return [Math.round(r*255), Math.round(g*255), Math.round(b*255)];
}

</script>
</body>
</html>
)=====";

// ------------------ SERVER HANDLERS ------------------

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);

  WiFi.softAP(ssid, password);

  Serial.println("WiFi Created: ESP32-Control");
  Serial.println("Password: 12345678");
  Serial.println("Open browser: http://192.168.4.1");

  server.on("/", []() {
    server.send(200, "text/html", webpage);
  });

  server.on("/mode", HTTP_POST, []() {
    String body = server.arg("plain");
    Serial.println(body);

  if (body.indexOf("FAN") != -1) {
      blinkMode = true;
    } else {
      blinkMode = false;
    }

  server.send(200, "text/plain", "OK");
  });

  server.on("/color", HTTP_POST, []() {
    Serial.println(server.arg("plain"));
    server.send(200, "text/plain", "OK");
  });

  server.on("/temperature", HTTP_POST, []() {
    Serial.println(server.arg("plain"));
    server.send(200, "text/plain", "OK");
  });

  // ------------------ Toggle Light ------------------
  server.on("/toggleLight", HTTP_POST, []() {
    lightOn = !lightOn;
    blinkMode = lightOn;   // LED ON/OFF toggle
    if(!lightOn) digitalWrite(LED_PIN, LOW);
    server.send(200, "text/plain", "OK");
  });

  server.begin();
}

void loop() {
  server.handleClient();

  if (blinkMode) {
    digitalWrite(LED_PIN, HIGH);
    delay(200);
    digitalWrite(LED_PIN, LOW);
    delay(200);
  }
}

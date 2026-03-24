# smart-fan bluetooth remote html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Smart Home Remote</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      margin: 0;
      padding: 0;
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
      background-color: #f5f5f5;
    }

    .remote {
      width: 350px;
      padding: 20px;
      background: white;
      border-radius: 20px;
      box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
      text-align: center;
    }

    .temperature-section {
      margin-bottom: 20px;
    }

    .temperature-section h2 {
      font-size: 18px;
      margin-bottom: 10px;
    }

    .number-display {
      font-size: 32px;
      font-weight: bold;
      margin-bottom: 15px;
    }

    #temperature-slider {
      display: block;
      margin: 0 auto 20px;
    }

    .bluetooth-button {
      margin-top: 20px;
      padding: 10px 15px;
      font-size: 14px;
      border: none;
      border-radius: 10px;
      cursor: pointer;
      background: #007bff;
      color: white;
      transition: background-color 0.3s;
    }

    .bluetooth-button:hover {
      background-color: #0056b3;
    }

    .rgb-wheel {
      position: relative;
      margin: 20px 0;
      display: flex;
      justify-content: center;
      align-items: center;
    }

    .rgb-circle {
      width: 150px;
      height: 150px;
      border-radius: 50%;
      background: conic-gradient(
        red, yellow, green, cyan, blue, magenta, red
      );
      display: flex;
      justify-content: center;
      align-items: center;
    }

    .rgb-circle span {
      width: 80px;
      height: 80px;
      background-color: white;
      border-radius: 50%;
      display: flex;
      justify-content: center;
      align-items: center;
      font-size: 18px;
      font-weight: bold;
    }

    .circle-scroll {
      position: absolute;
      width: 30px;
      height: 30px;
      border-radius: 50%;
      background-color: white;
      border: 3px solid #007bff;
      cursor: pointer;
      transition: background-color 0.3s ease;
    }

    .current-color {
      margin-top: 20px;
      padding: 10px;
      background-color: #007bff;
      color: white;
      border-radius: 5px;
      font-size: 14px;
    }

    .modes {
      display: flex;
      justify-content: space-around;
      margin-top: 20px;
    }

    .modes button {
      padding: 10px 15px;
      font-size: 14px;
      border: none;
      border-radius: 10px;
      cursor: pointer;
      background: #007bff;
      color: white;
      transition: background-color 0.3s;
    }

    .modes button:hover {
      background-color: #0056b3;
    }

    .features {
      margin-top: 20px;
      text-align: left;
    }

    .features span {
      display: block;
      margin-bottom: 5px;
      font-size: 14px;
    }

    .display-temp-button {
      margin-top: 20px;
      padding: 10px 15px;
      font-size: 14px;
      border: none;
      border-radius: 10px;
      cursor: pointer;
      background: #28a745;
      color: white;
      transition: background-color 0.3s;
    }

    .display-temp-button:hover {
      background-color: #218838;
    }
  </style>
</head>
<body>
  <div class="remote">
    <!-- Temperature Section -->
    <div class="temperature-section">
      <h2>Temperature to be Maintained</h2>
      <div class="number-display" id="temperature-display">25°C</div>
      <input
        type="range"
        id="temperature-slider"
        min="20"
        max="40"
        value="25"
        oninput="updateTemperature(this.value)"
      />
      <!-- Bluetooth Button -->
      <button class="bluetooth-button" onclick="connectBluetooth()">Connect Bluetooth</button>
    </div>

    <!-- RGB Color Wheel -->
    <div class="rgb-wheel">
      <div class="rgb-circle">
        <span>RGB</span>
      </div>
      <div class="circle-scroll" id="circle-scroll" style="top: 10px; left: 65px;" onmousedown="startDrag(event)"></div>
    </div>

    <!-- Current Color Display -->
    <div class="current-color" id="current-color">Current Color: #FF0000</div>

    <!-- Modes Section -->
    <div class="modes">
      <button onclick="toggleMode('RGB Cyclic Mode')">RGB Cyclic Mode</button>
      <button onclick="toggleMode('Display Time')"> Display Time </button>
      <button onclick="toggleMode('Auto Fan')"> Auto Fan</button>
      <button onclick="toggleMode('Display Temperature')">Display Temperature</button>
    </div>

    <!-- Features Section -->
    <div class="features">
      <h4>Features:</h4>
      <span>Display Time Mode</span>
      <span>Auto Fan ON/OFF Mode</span>
      <span>Display Temperature Mode</span>
    </div>
  </div>

  <script>
    let circleScroll = document.getElementById("circle-scroll");
    let rgbWheel = document.querySelector(".rgb-wheel");
    let currentColorBox = document.getElementById("current-color");
    let isDragging = false;

    let bleDevice = null;
    let bleServer = null;
    let bleService = null;
    let bleCharacteristic = null;

    // UUIDs for the BLE service and characteristic (replace with your own)
    const SERVICE_UUID = "00001234-0000-1000-8000-00805f9b34fb"; // Example UUID
    const CHARACTERISTIC_UUID = "00001234-0000-1000-8000-00805f9b34fb"; // Example UUID

    // Function to connect to the ESP32
    async function connectBluetooth() {
      try {
        bleDevice = await navigator.bluetooth.requestDevice({
          acceptAllDevices: true,
          optionalServices: [SERVICE_UUID]
        });
        bleServer = await bleDevice.gatt.connect();
        bleService = await bleServer.getPrimaryService(SERVICE_UUID);
        bleCharacteristic = await bleService.getCharacteristic(CHARACTERISTIC_UUID);
        alert("Bluetooth connected successfully");
      } catch (error) {
        alert("Failed to connect to Bluetooth: " + error.message);
      }
    }

    // Function to send color data via BLE
    async function sendColorToBLE(color) {
      if (bleCharacteristic) {
        const rgb = color.match(/\d+/g).map(Number); // Extract RGB values
        const data = new Uint8Array(rgb); // Create a byte array
        try {
          await bleCharacteristic.writeValue(data);
        } catch (error) {
          console.error("Error sending data via BLE: ", error);
        }
      }
    }

    // Function to update temperature
    function updateTemperature(value) {
      document.getElementById("temperature-display").innerText = value + "°C";
    }

    // Function to toggle modes
    function toggleMode(mode) {
      alert("Toggled " + mode);
    }

    // Function to start dragging the circle
    function startDrag(e) {
      isDragging = true;
      document.addEventListener("mousemove", drag);
      document.addEventListener("mouseup", stopDrag);
    }

    // Function to handle dragging the circle around the wheel
    function drag(e) {
      if (!isDragging) return;

      let rect = rgbWheel.getBoundingClientRect();
      let centerX = rect.left + rect.width / 2;
      let centerY = rect.top + rect.height / 2;
      let mouseX = e.clientX;
      let mouseY = e.clientY;

      // Calculate angle
      let angle = Math.atan2(mouseY - centerY, mouseX - centerX);
      let radius = 65; // distance from the center of the wheel

      let x = centerX + radius * Math.cos(angle) - circleScroll.offsetWidth / 2;
      let y = centerY + radius * Math.sin(angle) - circleScroll.offsetHeight / 2;

      circleScroll.style.left = `${x - rect.left}px`;
      circleScroll.style.top = `${y - rect.top}px`;

      // Update the current color based on angle
      let color = getColorFromAngle(angle);
      currentColorBox.style.backgroundColor = color;
      currentColorBox.textContent = `Current Color: ${color}`;

      // Send the color to BLE
      sendColorToBLE(color);
    }

    // Function to stop dragging the circle
    function stopDrag() {
      isDragging = false;
      document.removeEventListener("mousemove", drag);
      document.removeEventListener("mouseup", stopDrag);
    }

    // Function to map angle to RGB color (simplified)
    function getColorFromAngle(angle) {
      let hue = (angle + Math.PI) / (2 * Math.PI); // Normalize angle to 0-1 range
      let rgb = hsvToRgb(hue, 1, 1);
      return `rgb(${rgb[0]}, ${rgb[1]}, ${rgb[2]})`;
    }

    // Function to set the initial position of the circle-scroll
    function setInitialCursorPosition() {
      let angle = 0; // Starting angle in radians (e.g., 0 for red)
      let rect = rgbWheel.getBoundingClientRect();
      let centerX = rect.width / 2;
      let centerY = rect.height / 2;
      let radius = 65; // Distance from the center of the wheel

      let x = centerX + radius * Math.cos(angle) - circleScroll.offsetWidth / 2;
      let y = centerY + radius * Math.sin(angle) - circleScroll.offsetHeight / 2;

      circleScroll.style.left = `${x}px`;
      circleScroll.style.top = `${y}px`;

      // Set initial color
      let color = getColorFromAngle(angle);
      currentColorBox.style.backgroundColor = color;
      currentColorBox.textContent = `Current Color: ${color}`;
    }

    // Call the function on page load
    window.onload = setInitialCursorPosition;

    // Function to convert HSV to RGB
    function hsvToRgb(h, s, v) {
      let r, g, b;
      let i = Math.floor(h * 6);
      let f = h * 6 - i;
      let p = v * (1 - s);
      let q = v * (1 - f * s);
      let t = v * (1 - (1 - f) * s);

      switch (i % 6) {
        case 0: r = v, g = t, b = p; break;
        case 1: r = q, g = v, b = p; break;
        case 2: r = p, g = v, b = t; break;
        case 3: r = p, g = q, b = v; break;
        case 4: r = t, g = p, b = v; break;
        case 5: r = v, g = p, b = q; break;
      }

      return [Math.round(r * 255), Math.round(g * 255), Math.round(b * 255)];
    }
  </script>
</body>
</html>


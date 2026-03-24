// -------- DISPLAY VARIABLES --------
bool showingTempAndClimate = true;
unsigned long lastDisplayChange = 0;
const unsigned long displayChangeInterval = 5000; // 5 seconds

bool dhtErrorDisplayed = false;
bool dhtSensorFailed = false;


// -------- DISPLAY FUNCTION --------
void updateDisplay(float temperature) {
  unsigned long currentMillis = millis();

  // Toggle display every 5 seconds
  if (currentMillis - lastDisplayChange >= displayChangeInterval) {
    lastDisplayChange = currentMillis;
    showingTempAndClimate = !showingTempAndClimate;
    lcd.clear();
  }

  if (showingTempAndClimate) {
    // ---- Temperature & Climate ----
    lcd.setCursor(0, 0);
    lcd.print("Temp: ");

    if (dhtSensorFailed) {
      lcd.print("N/A");
    } else {
      lcd.print(temperature);
    }
    lcd.print("C");

    lcd.setCursor(0, 1);
    if (!dhtSensorFailed) {
      if (temperature < 36) {
        lcd.print("Winter");
      } else {
        lcd.print("Summer");
      }
    } else {
      lcd.print("Sensor Error");
    }

  } else {
    // ---- Time from Bluetooth ----
    lcd.setCursor(0, 0);
    lcd.print("Time: ");

    if (bluetooth.available()) {
      String currentTime = bluetooth.readStringUntil('\n');
      lcd.print(currentTime);
    } else {
      lcd.print("00:00");
    }
  }
}

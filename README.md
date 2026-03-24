// -------- DISPLAY VARIABLES --------
bool showingHumidityAndTime = true;
unsigned long lastDisplayChange = 0;
const unsigned long displayChangeInterval = 5000; // 5 seconds


// -------- DISPLAY FUNCTION --------
void updateDisplay(float humidity) {
  unsigned long currentMillis = millis();

  // Toggle display every 5 seconds
  if (currentMillis - lastDisplayChange >= displayChangeInterval) {
    lastDisplayChange = currentMillis;
    showingHumidityAndTime = !showingHumidityAndTime;
    lcd.clear();
  }

  if (showingHumidityAndTime) {
    // ---- HUMIDITY DISPLAY ----
    lcd.setCursor(0, 0);
    lcd.print("Humidity: ");
    lcd.setCursor(0, 1);

    if (isnan(humidity)) {
      lcd.print("Sensor Error");
    } else {
      lcd.print(humidity);
      lcd.print("%");
    }

  } else {
    // ---- BLUETOOTH TIME ----
    lcd.setCursor(0, 0);
    lcd.print("Time: ");

    if (bluetooth.available()) {
      String currentTime = bluetooth.readStringUntil('\n');
      lcd.setCursor(0, 1);
      lcd.print(currentTime);
    } else {
      lcd.setCursor(0, 1);
      lcd.print("00:00");
    }
  }
}

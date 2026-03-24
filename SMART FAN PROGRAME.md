#include <LiquidCrystal_I2C.h>
#include <SoftwareSerial.h>
#include <TimeLib.h>
#include <DHT.h>
#include <Servo.h>

#define DHT_PIN 2            // Pin connected to the DHT11 sensor
#define DHT_TYPE DHT11       // Type of the DHT sensor (DHT11)
#define ESC_PIN 9            // Pin connected to the ESC signal wire
#define RED_PIN 6            // RGB RED LED Pin
#define GREEN_PIN 7          // RGB GREEN LED Pin
#define BLUE_PIN 8           // RGB BLUE LED Pin

LiquidCrystal_I2C lcd(0x27, 16, 2); // Initialize the LCD
SoftwareSerial bluetooth(0, 1);     // Initialize Bluetooth communication
DHT dht(DHT_PIN, DHT_TYPE);         // Initialize DHT sensor

Servo esc;                         // Create Servo object for ESC control

bool showingTempAndClimate = true; // Display mode toggle
unsigned long lastDisplayChange = 0;
const unsigned long displayChangeInterval = 5000; // 5 seconds

bool dhtErrorDisplayed = false;    // Track if the error has already been displayed
bool dhtSensorFailed = false;      // Flag to track if the sensor is permanently unavailable

float readTemperature() {
  float temperature = dht.readTemperature();
  if (isnan(temperature)) {
    return -1; // Return -1 for a failed reading
  }
  return temperature;
}

void setup() {
  Serial.begin(9600);  // Serial communication
  bluetooth.begin(9600);

  // LCD setup
  lcd.init();
  lcd.backlight();

  // Show initial "SMART FAN" message
  lcd.setCursor(0, 0);
  lcd.print("SMART FAN");
  delay(2000);
  lcd.clear();

  // Initialize DHT sensor
  dht.begin();

  // ESC setup
  esc.attach(ESC_PIN);
  esc.writeMicroseconds(1000); // Arm ESC
  delay(2000);
}

void loop() {
  if (!dhtSensorFailed) {
    float temperature = readTemperature();

    if (temperature == -1) {
      if (!dhtErrorDisplayed) {
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Failed to read");
        lcd.setCursor(0, 1);
        lcd.print("DHT sensor!");
        delay(2000); // Display the error message for 2 seconds
        lcd.clear();
        dhtErrorDisplayed = true; // Set the flag to avoid repeated messages
        dhtSensorFailed = true;  // Mark the sensor as permanently unavailable
      }
      return; // Skip further processing in this loop iteration
    } else {
      dhtErrorDisplayed = false; // Reset the flag when a successful reading occurs
    }
  }

  unsigned long currentMillis = millis();

  if (currentMillis - lastDisplayChange >= displayChangeInterval) {
    lastDisplayChange = currentMillis;
    showingTempAndClimate = !showingTempAndClimate;
    lcd.clear();
  }

  if (showingTempAndClimate) {
    // Display temperature and climate
    lcd.setCursor(0, 0);
    lcd.print("Temp: ");
    if (dhtSensorFailed) {
        lcd.print("N/A");
    } else {
        lcd.print(readTemperature());
    }
    lcd.print("C");

    lcd.setCursor(0, 1);
    if (!dhtSensorFailed) {
        float temperature = readTemperature();
        if (temperature < 36) {
            lcd.print("Winter");
        } else {
            lcd.print("Summer");
        }
    } else {
        lcd.print("Sensor Error");
    }
  } else { // This was previously an issue
    // Display real-time clock (from Bluetooth)
    lcd.setCursor(0, 0);
    lcd.print("Time: ");
    if (bluetooth.available()) {
      String currentTime = bluetooth.readStringUntil('\n');
      lcd.print(currentTime);
    } else {
      lcd.print("00:00");
    }
  }

  if (!dhtSensorFailed) {
    // Motor control logic
    float temperature = readTemperature();
    int escSignal = 1100;
    if (temperature >= 25 && temperature <= 50) {
      escSignal = map(temperature, 25, 50, 1100, 2000);
    } else if (temperature > 50) {
      escSignal = 2000;
    }
    esc.writeMicroseconds(escSignal);

    // Debug output
    Serial.print("Temperature: ");
    Serial.print(temperature);
    Serial.print("\u00B0C, ESC Signal: ");
    Serial.println(escSignal);
    Serial.print(currentTime);
  }
}

# GSM
GSM module (SIM900A) using AT commands
#include <HardwareSerial.h>

HardwareSerial sim900(2);

const int buttonPin = 4;
bool sent = false;

const long baudRates[] = {9600, 19200, 38400, 57600, 115200};
const int numBaudRates = sizeof(baudRates) / sizeof(baudRates[0]);

const char* phoneNumber = "+91XXXXXXXXXX";

void setup() {

  Serial.begin(115200);
  pinMode(buttonPin, INPUT_PULLUP);

  delay(2000);

  Serial.println("ESP32 GSM Monitor Starting...");

  long detectedBaud = detectSIM900Baud();

  if (detectedBaud > 0) {

    Serial.print("Detected SIM900 Baud: ");
    Serial.println(detectedBaud);

    sim900.begin(detectedBaud, SERIAL_8N1, 16, 17);
    delay(1000);

    checkGSMStatus();
  }

  else {

    Serial.println("SIM900 baud not detected!");
  }
}

void loop() {

  while (sim900.available()) {
    Serial.write(sim900.read());
  }

  if (digitalRead(buttonPin) == LOW && !sent) {

    sendSMS();
    sent = true;
  }

  if (digitalRead(buttonPin) == HIGH) {
    sent = false;
  }
}

long detectSIM900Baud() {

  for (int i = 0; i < numBaudRates; i++) {

    sim900.begin(baudRates[i], SERIAL_8N1, 16, 17);
    delay(500);

    sim900.flush();
    sim900.println("AT");

    delay(500);

    String response = "";

    unsigned long start = millis();

    while (millis() - start < 1000) {

      while (sim900.available()) {

        char c = sim900.read();
        response += c;
      }
    }

    if (response.indexOf("OK") != -1) {

      return baudRates[i];
    }
  }

  return 0;
}

void checkGSMStatus() {

  sim900.println("AT+CPIN?");
  delay(500);

  sim900.println("AT+CREG?");
  delay(500);

  sim900.println("AT+CSQ");
  delay(500);
}

void sendSMS() {

  Serial.println("Sending SMS...");

  sim900.println("AT+CMGF=1");
  delay(500);

  sim900.print("AT+CMGS=\"");
  sim900.print(phoneNumber);
  sim900.println("\"");

  unsigned long timeout = millis() + 5000;
  bool promptReceived = false;
  String response = "";

  while (millis() < timeout) {

    while (sim900.available()) {

      char c = sim900.read();

      response += c;

      Serial.write(c);

      if (c == '>') {
        promptReceived = true;
        break;
      }
    }

    if (promptReceived) break;
  }

  if (promptReceived) {

    sim900.print("Button Pressed Alert!");
    sim900.write(26);

    Serial.println("\nSMS command sent.");
  }

  else {

    Serial.println("\nSMS failed.");
    Serial.println(response);
  }
}
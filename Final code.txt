#include <MFRC522.h>
#include <LiquidCrystal_I2C.h>
#include <Keypad.h>
#include <SoftwareSerial.h>
#include <Servo.h>
#include <SPI.h>

SoftwareSerial SIM900(3, 4); // SoftwareSerial SIM900(Rx, Tx)
MFRC522 mfrc522(10, 9); // MFRC522 mfrc522(SS_PIN, RST_PIN)
LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo sg90;

constexpr uint8_t greenLed = 7;
constexpr uint8_t redLed = 6;
constexpr uint8_t servoPin = 8;
constexpr uint8_t buzzerPin = 5;

String tagUID = "93 D4 70 BD";  // String to store UID of tag. Change it with your tag's UID
char password[4];   // Variable to store users password
char generatedOTP[5]; // Variable to store generated OTP
boolean RFIDMode = true; // boolean to change modes
boolean NormalMode = true; // boolean to change modes
char key_pressed = 0; // Variable to store incoming keys
uint8_t i = 0;  // Variable used for counter

const byte rows = 4;
const byte columns = 4;
char hexaKeys[rows][columns] = {
  {'1', '2', '3', 'A'},
  {'4', '5', '6', 'B'},
  {'7', '8', '9', 'C'},
  {'*', '0', '#', 'D'}
};
byte row_pins[rows] = {A0, A1, A2, A3};
byte column_pins[columns] = {2, 1, 0};
Keypad keypad_key = Keypad(makeKeymap(hexaKeys), row_pins, column_pins, rows, columns);

void setup() {
  pinMode(buzzerPin, OUTPUT);
  pinMode(redLed, OUTPUT);
  pinMode(greenLed, OUTPUT);
  sg90.attach(servoPin);  //Declare pin 8 for servo
  sg90.write(0); // Set initial position at 0 degrees
  lcd.init();   // LCD screen
  lcd.backlight();
  SPI.begin();      // Init SPI bus
  mfrc522.PCD_Init();   // Init MFRC522
  SIM900.begin(19200);
  SIM900.print("AT+CMGF=1\r");
  delay(100);
  SIM900.print("AT+CNMI=2,2,0,0,0\r");
  delay(100);
  lcd.clear(); // Clear LCD screen
}

void loop() {
  if (NormalMode == false) {
    receive_message();
  }
  else if (NormalMode == true) {
    if (RFIDMode == true) {
      receive_message();
      lcd.setCursor(0, 0);
      lcd.print(" Door Lock ");
      lcd.setCursor(0, 1);
      lcd.print(" Scan Your Tag ");
      if (!mfrc522.PICC_IsNewCardPresent()) {
        return;
      }
      if (!mfrc522.PICC_ReadCardSerial()) {
        return;
      }
      String tag = "";
      for (byte j = 0; j < mfrc522.uid.size; j++) {
        tag.concat(String(mfrc522.uid.uidByte[j] < 0x10 ? " 0" : " "));
        tag.concat(String(mfrc522.uid.uidByte[j], HEX));
      }
      tag.toUpperCase();
      if (tag.substring(1) == tagUID) 
      {
        lcd.clear();
        lcd.print(" Valid Tag ");
        digitalWrite(greenLed, HIGH);
        delay(3000);
        digitalWrite(greenLed, LOW);
        lcd.clear();
        lcd.print(" Enter OTP: ");
        lcd.setCursor(0, 1);
        generateOTP(); // Generate OTP
        send_message(" Your OTP is: " + String(generatedOTP)); // Send OTP via SMS
        RFIDMode = false; // Make RFID mode false
      } 

      else 
      {
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("Invalid Tag");
        lcd.setCursor(0, 1);
        lcd.print("Access Denied");
        digitalWrite(buzzerPin, HIGH);
        digitalWrite(redLed, HIGH);
        send_message(" Someone is accessing the door with an invalid tag ");
        delay(3000);
        digitalWrite(buzzerPin, LOW);
        digitalWrite(redLed, LOW);
        lcd.clear();
      }
    }


    if (RFIDMode == false) {
      key_pressed = keypad_key.getKey(); // Storing keys
      if (key_pressed) {
        password[i++] = key_pressed; // Storing in password variable
        lcd.print("*");
      }
      if (i == 4) { // If 4 keys are completed
        delay(2000);
        password[4] = '\0'; // Null-terminate the password string
        if (strcmp(password, generatedOTP) == 0) { // If OTP is matched
          lcd.clear();
          lcd.print(" OTP Accepted ");
          sg90.write(90); // Door Opened
          digitalWrite(greenLed, HIGH);
          send_message(" Door Opened ");
          delay(10000);
          digitalWrite(greenLed, LOW);
          sg90.write(0); // Door Closed
          lcd.clear();
          i = 0;
          RFIDMode = true; // Make RFID mode true
        } 
        else 
        { // If OTP is not matched
          lcd.clear();
          lcd.print("Wrong OTP");
          digitalWrite(buzzerPin, HIGH);
          digitalWrite(redLed, HIGH);
          send_message(" Someone is accessing the door with an invalid OTP ");
          delay(3000);
          digitalWrite(buzzerPin, LOW);
          digitalWrite(redLed, LOW);
          lcd.clear();
          i = 0;
          RFIDMode = true;  // Make RFID mode true
        }
      }
    }
  }
}

void generateOTP() {
  for (uint8_t i = 0; i < 4; i++) {
    generatedOTP[i] = random(0, 10) + '0'; // Generate a random digit
  }
  generatedOTP[4] = '\0'; // Null-terminate the OTP string
}

void receive_message() {
  char incoming_char = 0;
  String incomingData;
  if (SIM900.available() > 0) {
    incomingData = SIM900.readString(); // Get the incoming data.
    delay(10);
  }
  if (incomingData.indexOf("open") >= 0) {
    sg90.write(90);
    NormalMode = true;
    send_message("Opened");
    delay(10000);
    sg90.write(0);
  }
  if (incomingData.indexOf("close") >= 0) {
    NormalMode = false;
    send_message("Closed");
  }
  incomingData = "";
}

void send_message(String message) {
  SIM900.println("AT+CMGF=1");    //Set the GSM Module in Text Mode
  delay(100);
  SIM900.println("AT+CMGS=\"+918971292001\""); // Replace it with your mobile number
  delay(100);
  SIM900.println(message);   // The SMS text you want to send
  delay(100);
  SIM900.println((char)26);  // ASCII code of CTRL+Z
  delay(100);
  SIM900.println();
  delay(1000);
}

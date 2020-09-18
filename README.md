# Absesni-NodeMcu
absesni node mcu dengan rfid
#include <ESP8266WiFi.h>
#include <WiFiClient.h> 
#include <ESP8266HTTPClient.h>
#include <ArduinoJson.h>        // version 6.13
#include <SPI.h>
#include <RFID.h>
#include <Wire.h> 
#include <LiquidCrystal_I2C.h>
#include <RTClib.h>

 
RTC_DS3231 rtc;
 
char t[32];
LiquidCrystal_I2C lcd(0x27, 20, 4);       // sesuaikan alamat i2c (0x27) dengan alamat i2c kalian

#define SS_PIN D4
#define RST_PIN D3
#define Buzzer D8
#define lampu D0

//const char* wifiName = "KOSAN KEMBANG TURI";
//const char* wifiPass = "KOSAN12345";
const char* wifiName = "Cheng meongk";
const char* wifiPass = "1sampai9";

const String iddev = "98";

String hostMode = "http://absensi.xyz/api/getmodejson?key=mrRkmgf82kjrJNkVYjsfnxz&iddev=" + iddev;
String hostSCAN = "http://iotperizinan.herokuapp.com/api/pushrfid";
String hostADD = "http://iotperizinan.herokuapp.com/api/pushrfid"; //

//String hostMode = "http://192.168.43.106/absensi/api/getmodejson?key=asdkjWEQEDasd12ksnd&iddev=" + iddev;
//String hostSCAN = "http://eac54b2d0da5.ngrok.io/p/keluar?";
//String hostADD = "http://eac54b2d0da5.ngrok.io/p/keluar?";

String ModeAlat = "";

RFID rfid(SS_PIN, RST_PIN);

void setup() {
  Serial.begin(115200);
  Wire.begin(5, 4);   //Setting wire (5 untuk SDA dan 4 untuk SCL)
 
  rtc.begin();
  rtc.adjust(DateTime(F(__DATE__),F(__TIME__)));  //Setting Time
  
  // Kalian dapat menambahkan bagian dibawah ini untuk set manual jam
  // rtc.adjust(DateTime(2014, 1, 21, 3, 0, 0));
  SPI.begin();
  rfid.init();
  delay(10);

  pinMode(Buzzer, OUTPUT);
  pinMode(lampu, OUTPUT);
  
//  Wire.begin(D2,D1);
  lcd.begin();
  lcd.home ();
  lcd.print("RFID Reader Absensi");  
  delay (1000);
  Serial.println();
  
  Serial.print("Connecting to ");
  Serial.println(wifiName);

  lcd.setCursor(0,1);
  lcd.print("Connecting to");
  lcd.setCursor(0,2);
  lcd.print(wifiName);  

  WiFi.begin(wifiName, wifiPass);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());   //You can get IP address assigned to ESP

  ModeDevice();
}

void ModeDevice(){
  HTTPClient http;

  Serial.print("Request Link:");
  Serial.println(hostMode);
  
  http.begin(hostMode);
  
  int httpCode = http.GET();            //Send the request
  String payload = http.getString();    //Get the response payload from server

  Serial.print("Response Code:"); //200 is OK
  Serial.println(httpCode);       //Print HTTP return code

  Serial.print("Returned data from Server:");
  Serial.println(payload);    //Print request response payload
  
  if(httpCode == 200)
  {
    DynamicJsonDocument doc(1024);
  
   // Parse JSON object
    auto error = deserializeJson(doc, payload);
    if (error) {
      Serial.print(F("deserializeJson() failed with code "));
      Serial.println(error.c_str());
      return;
    }
  
    // Decode JSON/Extract values
    String responStatus = doc["status"].as<String>();
    String responMode = doc["mode"].as<String>();
    String responKet = doc["ket"].as<String>();

    Serial.println();
    Serial.print("status : ");
    Serial.println(responStatus);
    
    Serial.print("mode : ");
    Serial.println(responMode);
    
    Serial.print("ket : ");
    Serial.println(responKet);
    Serial.println("-------------------");
    Serial.println();

    lcd.clear();
    lcd.print("System Absensi RFID");
    if (responMode == "SCAN"){
      ModeAlat = "SCAN";
      lcd.setCursor(0,1);
      lcd.print(" SCAN Your RFID Card");
    }else if (responMode == "ADD"){
      ModeAlat = "ADD";
      lcd.setCursor(0,1);
      lcd.print(" ADD Your RFID Card");
    }else{
      ModeAlat = "";
      lcd.setCursor(0,2);
      lcd.print(responKet);
    }
  }
  else
  {
    Serial.println("Error in response");
  }

  http.end();

  delay(100);
}

void loop() {
  if (ModeAlat == "SCAN"){
    Serial.println("SCAN RFID CARD");
    if (rfid.isCard()) {
      if (rfid.readCardSerial()) {
          //Serial.println(rfid.serNum.length());
          BEEP(2, 200);

          Serial.println("");
          Serial.println("Card found");
          String RFID = String(rfid.serNum[0],HEX) +"-"+ String(rfid.serNum[1],HEX) +"-"+ String(rfid.serNum[2],HEX) +"-"+ String(rfid.serNum[3],HEX) +"-"+ String(rfid.serNum[4],HEX);

          
          lcd.setCursor(0,2);
          lcd.print("UID:");
          lcd.print(RFID);
          lcd.print(" ");
          Serial.println(RFID);
          Serial.println("");

          String host = hostSCAN;
          host += "rfid=";
          host += RFID;

          HTTPClient http;

          Serial.print("Request Link:");
          Serial.println(host);
          
          http.begin(host);
          
          int httpCode = http.GET();            //Send the GET request
          String payload = http.getString();    //Get the response payload from server
        
          Serial.print("Response Code:"); //200 is OK
          Serial.println(httpCode);       //Print HTTP return code
        
          Serial.print("Returned data from Server:");
          Serial.println(payload);    //Print request response payload
          
          if(httpCode == 200)
          {
            DynamicJsonDocument doc(1024);
          
           // Parse JSON object
            auto error = deserializeJson(doc, payload);
            if (error) {
              Serial.print(F("deserializeJson() failed with code "));
              Serial.println(error.c_str());
              return;
            }
          
            // Decode JSON/Extract values
            String responStatus = doc["status"].as<String>();
            String responKet = doc["ket"].as<String>();

            lcd.setCursor(0,3);
            lcd.print(responKet);
        
            Serial.println();
            Serial.print("status : ");
            Serial.println(responStatus);
            
            Serial.print("ket : ");
            Serial.println(responKet);
            Serial.println("-------------------");
            Serial.println();

            delay(1000);
            lcd.setCursor(0,3);
            lcd.print("                    ");
        }
      }
    }else{
      Serial.println("WAITING RFID CARD");
      lcd.setCursor(0,2);
      lcd.print("Menunggu Kartu RFID ");
    }
    rfid.halt();
    delay(1000);
  }else if (ModeAlat == "ADD"){
    Serial.println("ADD RFID CARD");
    digitalWrite(lampu, HIGH);
    if (rfid.isCard()) {
      if (rfid.readCardSerial()) {
          //Serial.println(rfid.serNum.length());
          digitalWrite(lampu, LOW);
          BEEP(2, 500);
        //  digitalWrite(lampu, HIGH);
          Serial.println("");
          Serial.println("Card found");
          String RFID = String(rfid.serNum[0],HEX) +"-"+ String(rfid.serNum[1],HEX) +"-"+ String(rfid.serNum[2],HEX) +"-"+ String(rfid.serNum[3],HEX) +"-"+ String(rfid.serNum[4],HEX);
          DateTime now = rtc.now();       //Menampilkan RTC pada variable now

          Serial.print("Tanggal : ");
          Serial.print(now.day());        //Menampilkan Tanggal
          Serial.print("/");
          Serial.print(now.month());      //Menampilkan Bulan
          Serial.print("/");
          Serial.print(now.year());       //Menampilkan Tahun
          Serial.print(" ");
          
          Serial.print("Jam : ");
          Serial.print(now.hour());       //Menampilkan Jam
          Serial.print(":");
          Serial.print(now.minute());     //Menampilkan Menit
          Serial.print(":");
          Serial.print(now.second());     //Menampilkan Detik
          Serial.println();

          lcd.setCursor(0,2);
          lcd.print("UID:");
          lcd.print(RFID);
          lcd.print(" ");
          Serial.println(RFID);
          Serial.println("");

          String host = hostADD;
          host += "?rfid=";
          host += RFID;

          HTTPClient http;

          Serial.print("Request Link:");
          Serial.println(host);
          
          http.begin(host);
          
          int httpCode = http.GET();            //Send the GET request
          String payload = http.getString();    //Get the response payload from server
        
          Serial.print("Response Code:"); //200 is OK
          Serial.println(httpCode);       //Print HTTP return code
        
          Serial.print("Returned data from Server:");
          Serial.println(payload);    //Print request response payload
          
          if(httpCode == 200)
          {
            DynamicJsonDocument doc(1024);
          
           // Parse JSON object
            auto error = deserializeJson(doc, payload);
            if (error) {
              Serial.print(F("deserializeJson() failed with code "));
              Serial.println(error.c_str());
              return;
            }
          
            // Decode JSON/Extract values
            String responStatus = doc["status"].as<String>();
            String responKet = doc["ket"].as<String>();

            lcd.setCursor(0,3);
            lcd.print(responKet);
        
            Serial.println();
            Serial.print("status : ");
            Serial.println(responStatus);
            
            Serial.print("ket : ");
            Serial.println(responKet);
            Serial.println("-------------------");
            Serial.println();
            
            delay(1000);
            lcd.setCursor(0,3);
            lcd.print("                    ");
        }
      }
    }else{
      Serial.println("WAITING RFID CARD");
      lcd.setCursor(0,2);
      lcd.print("Menunggu Kartu RFID");
    }
    rfid.halt();
    delay(1000);
  }else{
    Serial.println("Tidak Mendapatkan MODE ALAT dari server");
    Serial.println("Cek IP Server dan URL");
    Serial.println("Restart NodeMCU");
    lcd.setCursor(0,1);
    lcd.print("MODE ALAT ERROR");
    lcd.setCursor(0,2); 
    Serial.println("Cek IP Server dan URL");
    lcd.setCursor(0,3);
    Serial.println("Restart NodeMCU");
    delay(10000);         
    ModeDevice();
  }
}

// function sound beep = BEEP(how many, delay on in ms, delay off in ms);
void BEEP(byte c, int wait1){
  for(byte b=0; b<c; b++){
    digitalWrite(Buzzer,HIGH);
    delay(wait1);
    digitalWrite(Buzzer,LOW);
//    delay(wait2);
  }
}
  

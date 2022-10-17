# esp32_firebase
#include <RTClib.h>
#include <SoftwareSerial.h>
SoftwareSerial gsm(4, 2); // RX, TX

#define TINY_GSM_MODEM_SIM800

//Increase RX buffer
#define TINY_GSM_RX_BUFFER 256


#include <TinyGPS++.h> 
#include <TinyGsmClient.h> 
#include <ArduinoHttpClient.h>

#include "RTClib.h"
RTC_Millis rtc;
String receivedDate;
String msg;
String tarih = " ";
String saat= " ";
String dakika= " ";

/*rtc.begin(DateTime(F(__DATE__), F(__TIME__)));
  DateTime now = rtc.now();
      double endTime =now.hour()+float(0.01*now.minute());*/

const char FIREBASE_HOST[]  = "esp32-af323-default-rtdb.firebaseio.com";
const String FIREBASE_AUTH  = "74nQ7rOF7De1fyuHYJkbqsVDD7PD8dfC52BMSOrJ";
const String FIREBASE_PATH  = "/";
const int SSL_PORT          = 443;

char apn[]  = "internet";
char user[] = "";
char pass[] = "";

//GSM Module RX pin to ESP32 2
//GSM Module TX pin to ESP32 4 ne yazıyorrsa o şekilde bağlantı yap
#define rxPin 4
#define txPin 2
HardwareSerial sim800(1);
TinyGsm modem(sim800);

//GPS Module RX pin to ESP32 17
//GPS Module TX pin to ESP32 16
#define RXD2 16
#define TXD2 17
HardwareSerial neogps(2);
TinyGPSPlus gps;

TinyGsmClientSecure gsm_client_secure_modem(modem, 0);
HttpClient http_client = HttpClient(gsm_client_secure_modem, FIREBASE_HOST, SSL_PORT);


unsigned long previousMillis = 0;
long interval = 10000;
int counter;

void setup() {
  Serial.begin(115200);
  gsm.begin(9600);
  delay(3000);
  gsm.println("AT+CLTS=1");
  delay(1000);
  gsm.println("AT&W");
  delay(2000);


#ifndef ESP8266
    while (!Serial); // wait for serial port to connect. Needed for native USB
#endif

rtc.begin(DateTime(F(__DATE__), F(__TIME__)));

/*DateTime now = rtc.now();
      double endTime =now.hour()+float(0.01*now.minute());*/
      
  Serial.begin(115200);
  Serial.println("esp32 serial initialize");
  
  sim800.begin(9600, SERIAL_8N1, rxPin, txPin);
  Serial.println("SIM800L serial initialize");

  neogps.begin(9600, SERIAL_8N1, RXD2, TXD2);
  Serial.println("neogps serial initialize");
  
  
  //NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
  //Restart takes quite some time
  //To skip it, call init() instead of restart()
  Serial.println("Initializing modem...");
  modem.restart();
  String modemInfo = modem.getModemInfo();
  Serial.print("Modem: ");
  Serial.println(modemInfo);
  //NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
  
  // Unlock your SIM card with a PIN
  //modem.simUnlock("1234");
  
  http_client.setHttpResponseTimeout(90 * 1000); //^0 secs timeout
}

void loop() {
  DateTime now = rtc.now();
      double endTime =now.hour()+float(0.01*now.minute());
   gsm.println("AT+CCLK?");
  delay(1000);

  if(gsm.find("+CCLK: \"")){
    tarih = gsm.readStringUntil(',');
    saat = gsm.readStringUntil('+');
    //dakika = gsm.readStringUntil(':');
    gsm.readStringUntil('\r');
    
  }
  /*DateTime now = rtc.now();
      double endTime =now.hour()+float(0.01*now.minute());*/
  /*if (mySerial.available()) {
    Serial.write(mySerial.read());
  }
  if (Serial.available()) {
    mySerial.write(Serial.read());
  }
Serial.print("AT+CCLK?");
  if (sim800.available()>0){
    simdi=sim800.read();
    if (simdi==58 && onceki==75){
      sayiyorum=1;
      veri=0;
    }
    else if (sayiyorum==1) {
      data[veri]=simdi;
      veri++;
      if (veri==34){
        sayiyorum=0;
        zaman=(10*(data[10]-48))+(data[11]-48)+(float(0,1*data[13]-48))+(float(0,01*data[14]-48));
      //int zaman=(data[1]-48)+(data[2]-48)+(data[3]-48)+(data[4]-48)+(data[5]-48)+(data[6]-48)+(data[7]-48)+(data[8]-48)+(data[9]-48)+(data[10]-48)+(data[11]-48)+(data[12]-48)+(data[2]-48)+(data[13]-48)+(data[14]-48)+(data[15]-48)+(data[16]-48)+(data[17]-48)(data[18]-48)+(data[19]-48)+(data[20]-48);
      }
    }onceki=simdi;
  }*/

  Serial.print(F("Connecting to "));
  Serial.print(apn);
  if (!modem.gprsConnect(apn, user, pass)) {
    Serial.println(" fail");
  
    return;
  }
  Serial.println(" OK");
  //NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
  
  http_client.connect(FIREBASE_HOST, SSL_PORT);
  
  //MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM
  while (true) {
    if (!http_client.connected()) {
      Serial.println();
      http_client.stop();// Shutdown
      Serial.println("HTTP  not connect");
      break;
    }
    else{
     /* DateTime now = rtc.now();
      double endTime =now.hour()+float(0.01*now.minute());*/
      gps_loop();
    }
  }
  //MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM
}

void PostToFirebase(const char* method, const String & path , const String & data, HttpClient* http) {
  String response;
  int statusCode = 0;
  http->connectionKeepAlive(); // Currently, this is needed for HTTPS
  
  String url;
  if (path[0] != '/') {
    url = "/";
  }
  url += path + ".json";
  url += "?auth=" + FIREBASE_AUTH;
  Serial.print("POST:");
  Serial.println(url);
  Serial.print("Data:");
  Serial.println(data);
  
  String contentType = "application/json";
  http->put(url, contentType, data);
  
  //NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
  // read the status code and body of the response
  //statusCode-200 (OK) | statusCode -3 (TimeOut)
  statusCode = http->responseStatusCode();
  Serial.print("Status code: ");
  Serial.println(statusCode);
  response = http->responseBody();
  Serial.print("Response: ");
  Serial.println(response);

  if (!http->connected()) {
    Serial.println();
    http->stop();// Shutdown
    Serial.println("HTTP POST disconnected");
  }
  //NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
}

void gps_loop()
{
  //NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN
  //Can take up to 60 seconds
 DateTime now = rtc.now(); 

 double startTime =now.hour()+float(0.01*now.minute());
 //String endtime = String(now.hour(), DEC) + ':' + String(now.minute(), DEC) + ':' + String(now.second(), DEC);
 //SerialTransmit("$RTC: " + str);
 //String endtime =  now.hour()+String(":")+now.minute()+String(":")+now.minute();
// String  endtime = rtc.now();   
  /*  Serial.print(now.hour(), DEC);
    Serial.print(':');
    Serial.print(now.minute(), DEC);
    Serial.print(':');
    Serial.println(now.second(), DEC);
    delay(60000);*/
    
  boolean newData = false;
  for (unsigned long start = millis(); millis() - start < 2000;){
    while (neogps.available()){
      if (gps.encode(neogps.read())){
        newData = true;
        break;
      }
    }
  }
 
 int counter=0;
  for (int counter = 0 ; counter <= 100000; counter = counter +1)

   
 // counter = counter +1;
 // counter++;
  
 
  //If newData is true
  if(true){
  newData = false;
  
  String latitude, longitude;
  //float altitude;
  //unsigned long date, time, speed, satellites;
  
  //latitude = String(gps.location.lat(), 6); // Latitude in degrees (double)
  latitude = 6;
  longitude = String(gps.location.lng(), 6); // Longitude in degrees (double)
/* int houro= (now.hour(), DEC);
 int minuteo=(now.minute(), DEC);
 int secondo=(now.second(), DEC);*/
  
  //altitude = gps.altitude.meters(); // Altitude in meters (double)
  //date = gps.date.value(); // Raw date in DDMMYY format (u32)
  //time = gps.time.value(); // Raw time in HHMMSSCC format (u32)
  //speed = gps.speed.kmph();
  
  Serial.print("Latitude= "); 
  Serial.print(latitude);
  Serial.print(" Longitude= "); 
  Serial.println(longitude);
  Serial.print(" Counter= "); 
  Serial.println(counter);
  Serial.print("EndTime= ");
//  Serial.print(endtime); 

  
 /* Serial.print(houro);
  Serial.print(":"); 
  Serial.print(minuteo);
  Serial.print(":"); 
  Serial.print(secondo);*/
  
// String endtime =  now.hour()+String(".")+now.minute()+String(".")+now.minute();
 // double endTime =now.hour()+float(0.01*now.minute());
 // double endtime =now.t()+float(0.01*now.minute());
//  String endtime =now.TimeSpan(1);
  String gpsData = "{";
  gpsData += "\"counter\":";
  gpsData += counter;
  gpsData += ",";
  /*gpsData += "\"saat\":";
 /* gpsData += now.hour();
  gpsData += ",";
  gpsData += "\"dakika\":";
  gpsData += now.minute();
  gpsData += ",";
  gpsData += "\"saniye\":";
  gpsData += now.second();
  gpsData += ",";*/
  gpsData += "\"startTime\":";
  gpsData += startTime;
  gpsData += ",";
 /* gpsData += "\"endTime\":" ;
  gpsData += endTime;
  gpsData += ",";*/
  gpsData += "\"saat\":";
  gpsData += 'saat';
  gpsData += ",";
 /*  gpsData += "\"dakika\":";
  gpsData += dakika;
  gpsData += ",";*/
  /*gpsData += "\"EndTime\":";
  gpsData += now.hour();
  gpsData += ":";
  gpsData +=now.minute();
  gpsData += ":";
  gpsData +=now.second() ;
  gpsData += ",";*/
  gpsData += "\"lat\":" + latitude + ",";
  gpsData += "\"lng\":" + longitude + "";
  gpsData += "}";
//delay(1000);
  //PUT   Write or replace data to a defined path, like messages/users/user1/<data>
  //PATCH   Update some of the keys for a defined path without replacing all of the data.
  //POST  Add to a list of data in our Firebase database. Every time we send a POST request, the Firebase client generates a unique key, like messages/users/<unique-id>/<data>
  //https://firebase.google.com/docs/database/rest/save-data
  
  PostToFirebase("PATCH", FIREBASE_PATH, gpsData, &http_client);
  

  }
}

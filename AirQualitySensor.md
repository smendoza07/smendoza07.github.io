---
title: "Air Quality Sensor"
layout: page
categories: media
---
## Project Description

Using a NodeMCU ESP8266 Microcontroller, a DHT22 Temperature and Humidity Sensor, a MQ-135 Gas Sensor, and a PMS5003 Particulate Matter Sensor, I created an air quality sensor that displays the temperature, humidity, air quality index, and particulate matter concentration on a Blynk app. The app also displays a graph of the temperature, humidity, and air quality index over time. The sensor also has a built in alarm that will sound if the air quality index is too high.
  
## Demo Video
{% include embed.html url="https://youtube.com/embed/YKd9GHc0tCI" %}

## Parts Used

- NodeMCU ESP8266 Microcontroller
- DHT22 Temperature and Humidity Sensor
- MQ-135 Gas Sensor
- PMS5003 Particulate Matter Sensor
- 5V Power Supply

## Code

<h3>AirQualitySensor.ino<h3>
{% highlight c++ %}
#define BLYNK_PRINT Serial

#define BLYNK_TEMPLATE_ID "TMPLH8GQcxqU"
#define BLYNK_DEVICE_NAME "Air Quality Sensor"
//#define BLYNK_AUTH_TOKEN "FOqG-iLxb-g4lD7wvfbK2_xMRxjByq2C"

#include <SPI.h>
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>
#include <DHT.h>
#include <Arduino.h>
#define LENG 31   //0x42 + 31 bytes equal to 32 bytes
unsigned char buf[LENG];
// You should get Auth Token in the Blynk App.
// Go to the Project Settings (nut icon).
char auth[] = "FOqG-iLxb-g4lD7wvfbK2_xMRxjByq2C";


// Your WiFi credentials.
// Set password to "" for open networks.
char ssid[] = "Arduino_Hotspot"; //Enter your WIFI Name
char pass[] = "Arduino122A"; //Enter your WIFI Password
//char ssid[] = "SJH-2.4";  
//char pass[] = "5625068858";  
//char ssid[] = "RSFish2.4";  
//char pass[] = "3108046815";  


#define DHTPIN 4          // D4 pin


// Uncomment whatever type you're using!

//#define DHTTYPE DHT11     // DHT 11
#define DHTTYPE DHT22   // DHT 22, AM2302, AM2321
//#define DHTTYPE DHT21   // DHT 21, AM2301

int mq135 = A0; // smoke sensor is connected with the analog pin A0 
int data = 0;

int PM01Value=0;          //define PM1.0 value of the air detector module
int PM2_5Value=0;         //define PM2.5 value of the air detector module
int PM10Value=0;         //define PM10 value of the air detector module

DHT dht(DHTPIN, DHTTYPE);
BlynkTimer timer;


// This function sends Arduino's up time every second to Virtual Pin (5).
// In the app, Widget's reading frequency should be set to PUSH. This means
// that you define how often to send data to Blynk App.

void sendSensor()
{
  data = analogRead(mq135); 
  
  float h = dht.readHumidity();
  float t = dht.readTemperature(true); // or dht.readTemperature(true) for Fahrenheit
  
  Serial.print("Humidity: ");
  Serial.print(h);
  Serial.print(" %, Temp: ");
  Serial.print(t);
  Serial.print(" %, Smoke: ");
  Serial.println(data);
  
  if (isnan(h) || isnan(t)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }
  
  // You can send any value at any time.
  // Please don't send more that 10 values per second.
  Blynk.virtualWrite(V5, h); //send humidity data to blynk
  Blynk.virtualWrite(V6, t); //send temp data to blynk
  Blynk.virtualWrite(V2, data); //send CO data to blynk
  Blynk.virtualWrite(V7, PM01Value);
  Blynk.virtualWrite(V8, PM2_5Value);
  Blynk.virtualWrite(V9, PM10Value);
}

void setup()
{
  // Debug console
  Serial.begin(9600);

  Blynk.begin(auth, ssid, pass);

  dht.begin();

  // Setup a function to be called every second
  timer.setInterval(1000L, sendSensor);
}

void loop()
{
  Blynk.run();
  timer.run();

  if(Serial.find(0x42)){    //start to read when detect 0x42
    Serial.readBytes(buf,LENG);

    if(buf[0] == 0x4d){
      if(checkValue(buf,LENG)){
        PM01Value=transmitPM01(buf); //count PM1.0 value of the air detector module
        PM2_5Value=transmitPM2_5(buf);//count PM2.5 value of the air detector module
        PM10Value=transmitPM10(buf); //count PM10 value of the air detector module
      }
    }
  }

  static unsigned long OledTimer=millis();
    if (millis() - OledTimer >=1000)
    {
      OledTimer=millis();

      Serial.print("PM1.0: ");
      Serial.print(PM01Value);
      Serial.println("  ug/m3");

      Serial.print("PM2.5: ");
      Serial.print(PM2_5Value);
      Serial.println("  ug/m3");

      Serial.print("PM10: ");
      Serial.print(PM10Value);
      Serial.println("  ug/m3");
      Serial.println();
    }
}

char checkValue(unsigned char *thebuf, char leng)
{
  char receiveflag=0;
  int receiveSum=0;

  for(int i=0; i<(leng-2); i++){
  receiveSum=receiveSum+thebuf[i];
  }
  receiveSum=receiveSum + 0x42;

  if(receiveSum == ((thebuf[leng-2]<<8)+thebuf[leng-1]))  //check the serial data
  {
    receiveSum = 0;
    receiveflag = 1;
  }
  return receiveflag;
}

int transmitPM01(unsigned char *thebuf)
{
  int PM01Val;
  PM01Val=((thebuf[3]<<8) + thebuf[4]); //count PM1.0 value of the air detector module
  return PM01Val;
}

//transmit PM Value to PC
int transmitPM2_5(unsigned char *thebuf)
{
  int PM2_5Val;
  PM2_5Val=((thebuf[5]<<8) + thebuf[6]);//count PM2.5 value of the air detector module
  return PM2_5Val;
  }

//transmit PM Value to PC
int transmitPM10(unsigned char *thebuf)
{
  int PM10Val;
  PM10Val=((thebuf[7]<<8) + thebuf[8]); //count PM10 value of the air detector module
  return PM10Val;
}
{% endhighlight %}
  


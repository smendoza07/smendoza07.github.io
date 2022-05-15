---
title: Proximity Alert Sensor
layout: page
---

  
<h2>Code<h2>

<h3>SonicSensorSM.ino<h3>
{% highlight c++ %}

#include <LiquidCrystal.h>
#include <NewPing.h>
#include <TimerOne.h>
#include "notes_Library.h"

#define TRIGGER_PIN  40  // Trig pin, sends out pulse
#define ECHO_PIN     50  // Echo pin, recieves the signal back in
#define MAX_DISTANCE 400 // Sensor distance is rated at 400-500cm. Set to max distance in cm.

LiquidCrystal lcd(44, 45, 30, 31, 32, 33);  // (RS, E, D4, D5, D6, D7)
NewPing sonar(TRIGGER_PIN, ECHO_PIN, MAX_DISTANCE); // NewPing setup of pins and maximum distance.

enum Sonar_States { Sonar_Ping } Sonar_State;
enum LCD_Dist_States { LCD_Distance } LCD_Dist_State;
enum LCD_Message_States { LCD_MsgInit, LCD_Msg1, LCD_Msg2 } LCD_Msg_State;
enum Song_States { Play_Song0, Play_Song1, Play_Song2 } Song_State;

volatile unsigned char TimerFlag = 0;
volatile unsigned long distance_cm = 0;
unsigned long timerPeriod = 50000;
unsigned long Sonar_elapsedTime = 50000;
unsigned long Song_elapsedTime = 100000;

void TimerISR(){
  TimerFlag = 1;
}

void Sonar_SM () {
  
  switch ( Sonar_State ) {
    case Sonar_Ping:
      Sonar_State = Sonar_Ping;
      break;
    default:
      break;
  }
  
  switch ( Sonar_State ){
    case Sonar_Ping:
      distance_cm = sonar.ping_cm();
      break;
    default:
      break;      
  }
  
}

void Tick_LCD_Dist () {
  
  switch ( LCD_Dist_State ) {
    case LCD_Distance:
      LCD_Dist_State = LCD_Distance;
      break;
    default:
      LCD_Dist_State = LCD_Distance;
      break;
  }

  switch ( LCD_Dist_State ) {
    case LCD_Distance:
      lcd.setCursor(0, 0);              // places cursor at top left of lcd
      lcd.print("Distance: ");
      lcd.print(distance_cm * .01);   // Converts cm to m and prints out
      lcd.print(" m");
      break;
    default:
      break;
  }
}


void Tick_LCD_Message() {   //Will display "Speed Up!" if distance is too small and "Safe" if distance is larger
  
  switch ( LCD_Msg_State ) {
    case LCD_MsgInit:
      if(distance_cm >= 100){
        LCD_Msg_State = LCD_Msg1;
      }
      else if(distance_cm < 100){
        LCD_Msg_State = LCD_Msg2;
      }
      break;
    case LCD_Msg1:
      if(distance_cm >= 100){
        LCD_Msg_State = LCD_Msg1;
      }
      else if(distance_cm < 100){
        LCD_Msg_State = LCD_MsgInit;
      }
      break;
    case LCD_Msg2:
      if(distance_cm >= 100){
        LCD_Msg_State = LCD_MsgInit;
      }
      else if(distance_cm < 100){
        LCD_Msg_State = LCD_Msg2;
      }
      break;
    default:
      LCD_Msg_State = LCD_MsgInit;
      break;
  }
  
  switch ( LCD_Msg_State ) {
    case LCD_MsgInit:
      lcd.setCursor(0, 1);
      lcd.print("                ");    //used to clear display
      break;
    case LCD_Msg1:
      lcd.setCursor(0, 1);
      lcd.print("SAFE");     
      break;
    case LCD_Msg2:
      lcd.setCursor(0, 1);
      lcd.print("SPEED UP!       ");
      if ( distance_cm == 0 ) {
        lcd.setCursor(0,1);
        lcd.print("Out of Range    ");
      }
      else{
        lcd.setCursor(0, 1);
        lcd.print("SPEED UP!       ");        
      }
      break;
    default:
      break;
  }
}

void Song_SM () {

  switch (Song_State) {
    case Play_Song0:
      if(distance_cm >= 100){
        Song_State = Play_Song1;
      }
      else if(distance_cm < 100){
        Song_State = Play_Song2;
      }
      break;
    case Play_Song1:
      if(distance_cm >= 100){
        Song_State = Play_Song1;
      }
      else if(distance_cm < 100){
        Song_State = Play_Song2;
      }
      break;
    case Play_Song2:
      if(distance_cm >= 100){
        Song_State = Play_Song1;
      }
      else if(distance_cm < 100){
        Song_State = Play_Song2;
      }
      break;
    default:
      Song_State = Play_Song0;
      break;
  }
  
    switch (Song_State) {
    case Play_Song0:
      break;
    case Play_Song1:
      buzz(melodyPin, 0, 100);
      break;
    case Play_Song2:
      if (distance_cm == 0){
        buzz(melodyPin, 0, 100);
      }
      else{
        buzz(melodyPin, 1800, 100);
      }
      break;
    default:
      break;
  }
}

void setup() {
  lcd.begin(16, 2);
  pinMode(melodyPin, OUTPUT);
  Timer1.initialize(timerPeriod);
  Timer1.attachInterrupt(TimerISR);
  LCD_Dist_State = LCD_Distance;
  LCD_Msg_State = LCD_MsgInit;
  Sonar_State = Sonar_Ping;
}

void loop() {
  
  if( Sonar_elapsedTime >= timerPeriod ) {
    Sonar_SM();
    Sonar_elapsedTime = 0;
  }

  if (Song_elapsedTime >= (timerPeriod * 2) ) {
    Song_SM();
    Song_elapsedTime = 0;
  }
  
  Tick_LCD_Dist ();  
  Tick_LCD_Message();

  while( !TimerFlag ) {}
  TimerFlag = 0;
  Sonar_elapsedTime += timerPeriod;
  Song_elapsedTime += timerPeriod;
  
}
{% endhighlight %}
  
<h3>notes_Library.h<h3>
{% highlight c++ %}
  
#ifndef NOTES_LIBRARY_H
#define NOTES_LIBRARY_H

#include <Arduino.h>

#define melodyPin 49


void buzz(int targetPin, long frequency, long length) {
  digitalWrite(13, HIGH);
  long delayValue = 1000000 / frequency / 2; // calculate the delay value between transitions
  //// 1 second's worth of microseconds, divided by the frequency, then split in half since
  //// there are two phases to each cycle
  long numCycles = frequency * length / 1000; // calculate the number of cycles for proper timing
  //// multiply frequency, which is really cycles per second, by the number of seconds to
  //// get the total number of cycles to produce
  for (long i = 0; i < numCycles; i++) { // for the calculated length of time...
    digitalWrite(targetPin, HIGH); // write the buzzer pin high to push out the diaphram
    delayMicroseconds(delayValue); // wait for the calculated delay value
    digitalWrite(targetPin, LOW); // write the buzzer pin low to pull back the diaphram
    delayMicroseconds(delayValue); // wait again or the calculated delay value
  }
  digitalWrite(9, LOW);

}

#endif

{% endhighlight %}

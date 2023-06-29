---
title: "Digital Door Lock"
layout: page
categories: media
---
## Project Description

Using a digital keypad, unlock a "door" by inputting the correct sequence of numbers. When the correct sequence "12345" is inputted the door will unlock and an LED will light up to show the door is unlocked. If the wrong key is entered, the door will remain locked and the LED will remain off. The door will re-lock upon "closing" the door, which is simulated by pressing a button.
  
## Demo Video
{% include embed.html url="https://www.youtube.com/watch?v=kftMkPg5F00" %}

## Parts Used

- ATmega1284 Microcontroller
- Digital Keypad
- LEDs
- Buttons

## Code

<h3>SonicSensorSM.ino<h3>
{% highlight c++ %}
#include <avr/io.h>
#include "timer.h"
#include "scheduler.h"
#include "keypad.h"
#include "bit.h"
#ifdef _SIMULATE_
#include "simAVRHeader.h"
#endif

//--------Shared Variables----------------------------------------------------
unsigned char keypad_output = 0x00;
unsigned char PB0_state = 0x00;
//--------End Shared Variables------------------------------------------------

enum KeypadSM_States { K_Init };

//Get keypad press and assign to variable
int KeypadSM(int state) {
	state = GetKeypadKey();
	
	switch (state) {
		case '\0': keypad_output = 0x1F; break; // All 5 LEDs on
		case '1': keypad_output = 0x01; break; // hex equivalent
		case '2': keypad_output = 0x02; break;
		case '3': keypad_output = 0x03; break;
		case '4': keypad_output = 0x04; break;
		case '5': keypad_output = 0x05; break;
		case '6': keypad_output = 0x06; break;
		case '7': keypad_output = 0x07; break;
		case '8': keypad_output = 0x08; break;
		case '9': keypad_output = 0x09; break;
		case 'A': keypad_output = 0x0A; break;
		case 'B': keypad_output = 0x0B; break;
		case 'C': keypad_output = 0x0C; break;
		case 'D': keypad_output = 0x0D; break;
		case '*': keypad_output = 0x0E; break;
		case '0': keypad_output = 0x00; break;
		case '#': keypad_output = 0x0F; break;
	}
	return state;
}

enum Unlock_States { Wait, Start, One, Two, Three, Four, Unlocked };

int Unlock_Sequence_SM (int state) {
	switch(state) {
		case Wait:
			if(keypad_output == 0x1F)
				state = Wait;
			else if(keypad_output == 0x0F)
				state = Start;
			else
				state = Wait;
			break;
		case Start:
			if(keypad_output == ( 0x1F))
				state = Start;
			else if(keypad_output == 0x0F)
				state = Start;
			else if(keypad_output == 0x01)
				state = One;
			else
				state = Wait;
			break;
		case One:
			if(keypad_output == 0x1F)
				state = One;
			else if(keypad_output == 0x01)
				state = One;
			else if(keypad_output == 0x0F)
				state = Start;
			else if(keypad_output == 0x02)
				state = Two;
			else
				state = Wait;
			break;
		case Two:
			if(keypad_output == 0x1F)
				state = Two;
			else if(keypad_output == 0x02)
				state = Two;
			else if(keypad_output == 0x0F)
				state = Start;
			else if(keypad_output == 0x03)
				state = Three;
			else
				state = Wait;
			break;
		case Three:
			if(keypad_output == 0x1F)
				state = Three;
			else if(keypad_output == 0x03)
				state = Three;
			else if(keypad_output == 0x0F)
				state = Start;
			else if(keypad_output == 0x04)
				state = Four;
			else 
				state = Wait;
			break;
		case Four:
			if(keypad_output == 0x1F)
				state = Four;
			else if(keypad_output == 0x04)
				state = Four;
			else if(keypad_output == 0x0F)
				state = Start;
			else if(keypad_output == 0x05)
				state = Unlocked;
			break;
		case Unlocked:
			state = Wait;
			break;
	}

	switch(state) {
		case Wait:			
			break;
		case Start:
			break;
		case One:
			break;
		case Two:
			break;
		case Three:
			break;
		case Four:
			break;
		case Unlocked:
			PB0_state = 0x01;
			break;
	}
	return state;
}

enum Lock_States { Lock_Wait, Lock };

int Lock_SM (int state) {
	switch(state) {
		case Lock_Wait:
			if((~PINB & 0x80) == 0x80)
				state = Lock;
			else
				state = Lock_Wait;
			break;
		case Lock:
			state = Lock_Wait;
			break;
	}

	switch(state) {
		case Lock_Wait:
			break;
		case Lock:
			PB0_state = 0x00;
			break;
	}
	return state;
}


//Enumeration of states.
enum display_States { display_display };

// Combine outputs from SM's, and output on PORTB
int displaySMTick(int state) {
	// Local Variables
	unsigned char output;
	
	switch (state) { //State machine transitions
		case display_display: state = display_display; break;
		default: state = display_display; break;
	}
	switch(state) { //State machine actions
		case display_display:	
		output = PB0_state; // write shared outputs
				// to local variables
		break;
	}
	PORTB = output;	// Write combined, shared output variables to PORTB
	return state;
}

	
int main() {
       	DDRB = 0x7F; PORTB = 0x80; // PORTB set to output, outputs init 0s
	DDRC = 0xF0; PORTC = 0x0F; // PC7..4 outputs init 0s, PC3..0 inputs init 1s

	//Declare an array of tasks 
	static task task1, task2, task3, task4;
	task *tasks[] = { &task1, &task2, &task3, &task4 };
	const unsigned short numTasks = sizeof(tasks)/sizeof(task*);

	const char start = 0;
	// Task 1 (KeypadSM)
	task1.state = start;//Task initial state.
	task1.period = 50;//Task Period.
	task1.elapsedTime = task1.period;//Task current elapsed time.
	task1.TickFct = &KeypadSM;//Function pointer for the tick.
	//Task 2 (UnlockSM)
	task2.state = start; //Task initial state.
	task2.period = 100; //Task period.
	task2.elapsedTime = task2.period; //Task current elapsed time.
	task2.TickFct = &Unlock_Sequence_SM; //Function pointer for the tick.
	// Task 3 (LockSM)
	task3.state = start; //Task initial state.
	task3.period = 50; //Task period.
	task3.elapsedTime = task3.period; //Task current elapsed time.
	task3.TickFct = &Lock_SM; //Function pointer for the tick.
	// Task 4 (displaySM)
	task4.state = start;//Task initial state.
	task4.period = 10;//Task Period.
	task4.elapsedTime = task4.period; // Task current elasped time.
	task4.TickFct = &displaySMTick; // Function pointer for the tick.
	
	unsigned short j;
	unsigned long GCD = tasks[0]->period;
	for ( j = 1; j < numTasks; j++ ) {
		GCD = findGCD(GCD,tasks[j]->period);
	}	


	// Set the timer and turn it on
	TimerSet(GCD);
	TimerOn();

	unsigned short i; // Scheduler for-loop iterator
	while(1) {	
		for ( i = 0; i < numTasks; i++ ) { // Scheduler code
			if ( tasks[i]->elapsedTime == tasks[i]->period ) { // Task is ready to tick
				tasks[i]->state = tasks[i]->TickFct(tasks[i]->state); // Set next state 
				tasks[i]->elapsedTime = 0; // Reset the elapsed time for next tick.
			}
			tasks[i]->elapsedTime += GCD;
		}
		while(!TimerFlag);
			TimerFlag = 0;
	}
	return 0; // Error: Program should not exit!
}
{% endhighlight %}
  


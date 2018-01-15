---
date: "2018-01-14T14:58:20Z"
title: "Automatic audio switch with Arduino"
authors: []
tags:
  - electronics
  - c
  - arduino
draft: false
toc: true
typora-copy-images-to: ../../static/imgs
typora-root-url: ../../static
---

## Back-story

I was one of the people who invested quite a lot of money into the [Sonos](https://sonos.com) ecosystem when they where the "smart" speaker to buy, but things have moved on quite a bit now and I now have a [Google Home Mini](https://store.google.com/product/google_home_mini) which has far superior features. My problem now is that when I ask the Google Home to play music it can only play from itself, and the speaker on it is not great. Ideally I'd want it to come out of the Sonos.

My first attempt at this was to get the Sonos to in someway emulate a Chromecast but alas the system is too locked down (thats a first for an IoT device). So my only option was to buy a Chromecast Audio and get the signal into the Sonos. I was lucky enough to have a Sonos:5 which has a line-in, but I'm already using it for a record player. What am I to do? I googled for an automatic audio switch but nothing seems to exist. The answer is to build a solution. 

## Design

I decided on an Arduino based solution because thats what I know and they work quite well from my experience. I didn't want a full ATMega chip so I eventually chose the ATTiny 84. The ATTiny 85 was not an option due to the lack of I/O. Next was the actual switching. I went for the simple option; a relay. It ensured complete isolation between the inputs and a break-before-make function. The next challenge was detecting the signal on the Arduino. After a hour or so of Googling I discovered this circuit.

![Audio input](imgs/arduinoaudiofig1.png#center)

The main reason this is needed is that an audio signal has both an positive and a negative part. The pins on the ATTiny cannot be driven below -0.5V so we need to add a DC offset to the signal so that a 0V signal on the input is at 2.5V on the pin (half Vcc). The 10uf capacitor ensures this offset does not bleed back to the original signal.

## Programming

Unlike the standard Arduino the ATTiny does not have a USB port so I need to use ICSP (In circuit serial programming) to get anything onto it. I used my other Arduino Yun as an ICSP.  This is easy enough, simply load up the ArduinoISP example sketch and program away! The one thing to watch is if your are using something other than an Arduino Uno you need to set it to use `OLD_STYLE_WIRING`. Find this line and uncomment it (It's near the top).

```c
// #define USE_OLD_STYLE_WIRING
```

becomes:

```c
#define USE_OLD_STYLE_WIRING
```



![Screenshot_2018-01-14_15-30-18](/imgs/Screenshot_2018-01-14_15-30-18.png#center)

The next problem is that the Arduino IDE does not have support for ATTiny chips. This is solved by using avr-gcc directly. Here is my build/install script:

```bash
#!/bin/bash

cp audio_switch.ino audio_switch.cpp &&
avr-g++ -c -Os -w -s -mmcu=attiny84 -DF_CPU=8000000L *.cpp *.c &&
avr-gcc -w -Os -g -mmcu=attiny84  -o audio_switch.elf *.o -lm &&
avr-objcopy -O ihex -R .eeprom  audio_switch.elf audio_switch.hex &&

avrdude -pattiny84 -cstk500v1 -P/dev/ttyACM0 -b19200 -Uflash:w:audio_switch.hex:i
```

You will need change the files names to whatever you called your project and the serial port `/dev/ttyACM0` to whatever your computer assigned the other Arduino.

Next you need to connect the Arduino to the ATTiny following the handy table:

| ATTiny Pin    | Arduino Pin |
| ------------- | ----------- |
| 5V            | Vcc         |
| Gnd           | Gnd         |
| RESET - Pin 4 | Pin 10      |
| Pin 7         | Pin 11      |
| Pin 8         | Pin 12      |
| Pin 9         | Pin 13      |

Then you can run the script and hopefully it should program. This is the expected output:

```bash
>> ./build.sh
avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.01s

avrdude: Device signature = 0x1e930c (probably t84)
avrdude: NOTE: "flash" memory has been specified, an erase cycle will be performed
         To disable this feature, specify the -D option.
avrdude: erasing chip
avrdude: reading input file "audio_switch.hex"
avrdude: writing flash (668 bytes):

Writing | ################################################## | 100% 1.08s

avrdude: 668 bytes of flash written
avrdude: verifying flash memory against audio_switch.hex:
avrdude: load data flash data from input file audio_switch.hex:
avrdude: input file audio_switch.hex contains 668 bytes
avrdude: reading on-chip flash data:

Reading | ################################################## | 100% 0.74s

avrdude: verifying ...
avrdude: 668 bytes of flash verified

avrdude: safemode: Fuses OK (E:FF, H:DF, L:E2)

avrdude done.  Thank you.
>>
```



## Writing code for the chip

### Blinky test code

As this is not a normal Arduino we cannot just call `digitalWrite` we need to directly access the port registers. In my case the control pin for the relay is on pin 8 which is `PA7`. So to output on that pin we need to set bit 7 of `DDRA` to 1 and then bit 7 of `PORTA` to our desired output value.

```c
DDRA |= _BV(DDA7);
PORTA |= _BV(PORTA7);
```

We `OR` here so as not to affect any other values. The full program to turn this on and off every second is:

```c
#define F_CPU 8000000L
#include <avr/io.h>
#include <util/delay.h>
 
#define BLINK_DELAY_MS 1000

int main(void) {
  DDRA |= _BV(DDA7);
  
  while (1) {
    PORTA |= _BV(PORTA7);
    _delay_ms(BLINK_DELAY_MS);

    PORTA &= ~_BV(PORTA7);
    _delay_ms(BLINK_DELAY_MS);
  }
}
```

This should turn the relay on and off every second. Note: the clock fuses and clock frequency in the code must be the same for `_delay_ms` to be correct. I have my fuses set to internal 8 MHz clock with this `avrdude` command.

```bash
avrdude -P /dev/ttyACM1 -b 19200 -c avrisp -p t84 -U lfuse:w:0xe2:m -U hfuse:w:0xdf:m -U efuse:w:0xff:m
# /dev/ttyACM1 is the serial port of my other Ardunio
```

### Reading in the signal

As this is an analog signal we need to use the built in ADC. This is connected to each pin on `PORTA`. My two audio sources come in on pin 10 and 11 or `PA3` and `PA2` respectively. First we need to set our voltage reference. This is done by setting the bits 7 and 6 of `ADMUX`. We want our reference at VCC so those should both be `0`.

```c
ADMUX &= ~_BV(REFS1);
ADMUX &= ~_BV(REFS0);
```

We need to set the prescaler correctly by setting bits 2 to 0 of `ADCSRA`. First we need to clear bits 7 to 3 then set the necessary ones.

```c
ADCSRA &= B11111000;
ADCSRA |= B00000111;
```

We need enable the ADC by setting bit 7 of `ADCSRA`.  After enabling or changing the voltage reference the ADC requires at least 1ms to settle. I've delayed for 5 here to be certain.

```c
ADCSRA |= _BV(ADEN);
_delay_ms(5);
```

Next we set what pin we want to read in. This is done by setting bits 5 to 0 of `ADMUX`. For `PA2` we need to set these to `000010`. First we need to clear bits 5 to 0 then set the necessary ones.

```c
ADMUX &= B11000000;
ADMUX |= B00000010;
```

Finally we start the conversion by setting bit 6 of `ADCSRA` to 1 and wait until it is set by the processor back to a 0. We need to and the value from `ADCSRA` to get the correct bit out then shift it by 6 to get a 1 or a 0;

```c
ADCSRA |= _BV(ADSC);
bool conv_state = 1;
while (conv_state) {
  conv_state = (bool)((ADCSRA & _BV(ADSC)) >> 6);
}
```

The value is then stored in `ADCL` and `ADCH`. This is a 10 bit value and thus can be read into an `int`. We read in the first 8 bits from `ADCL` then the last 2 bits from `ADCH`.

```c
unsigned int sample = ADCL;
sample |= ADCH << 8;
```

And tada! We have out value! That wasn't hard or complicated now was it.

### Custom millis function

Before we can read audio levels theres one thing left to do. Due to not having the Arduino libraries we need to write our own function that gives us how many milliseconds since startup. This is done by using the first 8-bit counter and set it up to overflow every millisecond. The ISR (Interupt Service Routine) then increments a millis variable.

```c
uint64_t _millis = 0;

ISR(TIM0_COMPA_vect) {
   _millis++;
}
```

We now need to add some setup code to make the timer overflow every 1ms. The first thing we do is calculate how many ticks per second the timer gets. This is equal to the clock speed divided by the prescaler value. My prescaler is set to divide by 64 so the TPS is `F_CPU/64`. We then need to figure out how many ticks it will take to get to 1ms, this is the TPS divided by 1000 because there are 1000 milliseconds in a second. This value is then written to the `OCR0A` register. We then set the mode to `CTC` which counts up until the timer value reaches the value in `OCR0A`, this then triggers a reset of the timer and calls the ISR.

```c
#define TIMER_TPS F_CPU/64
// set mode
TCCR0A |= _BV(WGM01);
TCCR0A &= ~_BV(WGM00);
// set prescaler
TCCR0B &= ~_BV(CS02);
TCCR0B |= _BV(CS01);
TCCR0B |= _BV(CS00);
// set overflow value
OCR0A = TIMER_TPS/1000;
// enable overflow interupt
TIMSK0 |= _BV(OCIE0A);
sei();
```

Finally we can write a function that allows safe access to the millis variable by disabling interrupts and then re-enabling them after reading the value.

```c
long unsigned int millis() {
  uint64_t m;
  cli();
  m = _millis;
  sei();
  return m;
}
```

### Reading audio level

As audio changes over time and can be silent for a short time we have to sample the maximum values over a window. The lowest audible frequency is around 20Hz so if we sample for 0.05 seconds or more we should get everything.

We start by defining a couple variables for the read. A start millis to calculate from, a maximum smallest value, a minimum largest value and a peak-to-peak level to be calculated later.

```c
unsigned long start_millis = millis();
unsigned int peak_to_peak = 0;

unsigned int signal_max = 0;
unsigned int signal_min = 1024;
```

We then enter an loop constantly reading values and checking if they are larger than our previous maximum or smaller than our previous minimum, until we have been reading for 50 milliseconds. We then subtract the two to get the peak-to-peak level.

```c
while (millis() - start_millis < 50) {
  ADMUX &= B11000000;
  ADMUX |= B00000010;
  ADCSRA |= _BV(ADSC);
  bool conv_state = 1;
  while (conv_state) {
    conv_state = (bool)((ADCSRA & _BV(ADSC)) >> 6);
  }
  unsigned int sample = ADCL;
  sample |= ADCH << 8;
  if (sample < 1024) {
    if (sample > signal_max) {
      signal_max = sample;
    } else if (sample < signal_min) {
      signal_min = sample;
    }
  }
}
peak_to_peak = signal_max - signal_min;
```

Finally we can modify that code to read in both variables at the same time. Our second input pin is on `PA3` which required bits 5 to 0 of `ADMUX` to be `000011`.

```c
unsigned long start_millis= millis();
unsigned int peak_to_peak_a = 0;
unsigned int peak_to_peak_b = 0;

unsigned int signal_max_a = 0;
unsigned int signal_min_a = 1024;
unsigned int signal_max_b = 0;
unsigned int signal_min_b = 1024;

while (millis() - start_millis < 50) {
  ADMUX &= B11000000;
  ADMUX |= B00000010;
  ADCSRA |= _BV(ADSC);
  bool conv_state = 1;
  while (conv_state) {
    conv_state = (bool)((ADCSRA & _BV(ADSC)) >> 6);
  }
  unsigned int sample_a = ADCL;
  sample_b |= ADCH << 8;
  ADMUX &= B11000000;
  ADMUX |= B00000011;
  ADCSRA |= _BV(ADSC);
  conv_state = 1;
  while (conv_state) {
    conv_state = (bool)((ADCSRA & _BV(ADSC)) >> 6);
  }
  unsigned int sample_b = ADCL;
  sample_b |= ADCH << 8;
  if (sample_a > signal_max_a) {
    signal_max_a = sample_a;
  } else if (sample_a < signal_min_a) {
    signal_min_a = sample_a;
  }
  if (sample_b > signal_max_b) {
    signal_max_b = sample_b;
  } else if (sample_b < signal_min_b) {
    signal_min_b = sample_b;
  }
}
peak_to_peak_a = signal_max_a - signal_min_a;
peak_to_peak_b = signal_max_b - signal_min_b;
```

After much debugging I discovered that the audio levels where so low that without a signal I was getting a reading of 2-3 and with a signal a reading of 3-5, this is on a scale of 0-1024! It also jumped around a bit so I couldn't just threshold it at 3 or something. My solution was a moving average with a circular buffer.

First we setup a few global variables to store the state of both moving averages:

```c
#define AVERAGE_LEN 10
unsigned int a_average_buf[AVERAGE_LEN];
int a_average_ptr = 0; // This is the bit that makes it circular. We can move this around.
float a_average = 0;
unsigned int b_average_buf[AVERAGE_LEN];
int b_average_ptr = 0;
float b_average = 0;
```

Next for each peak to peak reading we update the average. The `X_average_ptr` stores where the oldest value is, so we first take the average and remove the oldest value from it. We then add our new reading and insert into the buffer. Lastly we move the circular pointer forwards one in the array jumping back to the start if we reach the end with a modulo operation.

```c
a_average = a_average - ((float)a_average_buf[a_average_ptr] / (float)AVERAGE_LEN) + ((float)peak_to_peak_a / (float)AVERAGE_LEN);
a_average_buf[a_average_ptr] = peak_to_peak_a;
a_average_ptr = (a_average_ptr+1) % AVERAGE_LEN;

b_average = b_average - ((float)b_average_buf[b_average_ptr] / (float)AVERAGE_LEN) + ((float)peak_to_peak_b / (float)AVERAGE_LEN);
b_average_buf[b_average_ptr] = peak_to_peak_b;
b_average_ptr = (b_average_ptr+1) % AVERAGE_LEN;
```

The average reads a consistent 1-2 for no signal and above 2 for a signal. This is good enough to work with.

### Switching logic

This bit looks simple at first but actually requires some thought. If you just switched to whichever had a higher signal level, then two signals at the same time would result in constant switching.

My solution is to store which input is already on and the previous state of each input. The state is simply a boolean that stores weather the signal is above a certain threshold. My switching logic is then as follows:

| Input A cur | Input B cur | Input A prev | Input B prev | Cur output | Output |
| :---------: | :---------: | :----------: | :----------: | :--------: | :----: |
|     OFF     |     OFF     |     ANY      |     ANY      |     A      |   A    |
|     OFF     |     OFF     |     ANY      |     ANY      |     B      |   B    |
|     ON      |     OFF     |     ANY      |     ANY      |    ANY     |   A    |
|     OFF     |     ON      |     ANY      |     ANY      |    ANY     |   B    |
|     ON      |     ON      |     OFF      |     OFF      |     A      |   A    |
|     ON      |     ON      |     OFF      |     OFF      |     B      |   B    |
|     ON      |     ON      |      ON      |     OFF      |    ANY     |   B    |
|     ON      |     ON      |     OFF      |      ON      |    ANY     |   A    |
|     ON      |     ON      |      ON      |      ON      |     A      |   A    |
|     ON      |     ON      |      ON      |      ON      |     B      |   B    |

First we setup variables to store all those states. I've used an `enum` here to make the rest of the code easier to read.

```c
bool input_a_cur, input_b_cur, input_a_prev, input_b_prev;
typedef enum Output {
  A, B
};
Output output;
```
Now let's calculate which inputs are on or off.

```c
input_a_prev = input_a_cur;
input_b_prev = input_b_cur;
input_a_cur = ((int)a_average > 2);
input_b_cur = ((int)b_average > 2);
```

Let's start with the first two rows of the table. If both inputs are off then nothing should happen so we just start by checking that one of the inputs is on.

```c
if (input_a_cur || input_b_cur) {
  
}
```

Next we have when one input only is on.

```c
if (input_a_cur || input_b_cur) {
  if (input_a_cur && !input_b_cur) {
    output = A;
  } else if (!input_a_cur && input_b_cur) {
    output = B;
  }
}
```

Now we get to the complicated bit. The next two rows are if both are on but both previous states are off nothing changes.

```c
if (input_a_cur || input_b_cur) {
  if (input_a_cur && !input_b_cur) {
    output = A;
  } else if (!input_a_cur && input_b_cur) {
    output = B;
  } else if (input_a_cur && input_b_cur) {
    if (input_a_prev || input_b_prev) {
      
    }
  }
}
```

Then if one has changed from off to on we switch to that.

```c
if (input_a_cur || input_b_cur) {
  if (input_a_cur && !input_b_cur) {
    output = A;
  } else if (!input_a_cur && input_b_cur) {
    output = B;
  } else if (input_a_cur && input_b_cur) {
    if (input_a_prev || input_b_prev) {
      if (input_a_prev && !input_b_prev) {
        output = A;
      } else if (!input_a_prev && input_b_prev) {
        output = B;
      }
    }
  }
}
```

And the last two rows are if both are on a both previous states are on there is no change so no code.

### Put it all together

We finally have enough code to put this all together into a functioning program. Yay!

```c
// Includes
#define F_CPU 8000000UL
#define TIMER_TPS F_CPU/64
#include <avr/io.h>
#include <avr/interrupt.h>
#include <util/delay.h>

// Global state
bool input_a_cur, input_b_cur, input_a_prev, input_b_prev;
typedef enum Output {
  A, B
};
Output output = A;

#define AVERAGE_LEN 10
unsigned int a_average_buf[AVERAGE_LEN];
int a_average_ptr = 0;
float a_average = 0;
unsigned int b_average_buf[AVERAGE_LEN];
int b_average_ptr = 0;
float b_average = 0;

uint64_t _millis = 0;

ISR(TIM0_COMPA_vect) {
   _millis++;
};

long unsigned int millis() {
  uint64_t m;
  cli();
  m = _millis;
  sei();
  return m;
};

// Entrypoint
int main(void) {
  cli();
  
  // Timer setup
  TCCR0A |= _BV(WGM01);
  TCCR0A &= ~_BV(WGM00);
  TCCR0B &= ~_BV(CS02);
  TCCR0B |= _BV(CS01);
  TCCR0B |= _BV(CS00);
  OCR0A = TIMER_TPS/1000;
  TIMSK0 |= _BV(OCIE0A);

  // Pin config and ADC setup
  DDRA |= _BV(DDA7);
  ADMUX &= ~_BV(REFS1);
  ADMUX &= ~_BV(REFS0);
  ADCSRA &= 0b11111000;
  ADCSRA |= 0b00000111;
  ADCSRA |= _BV(ADEN);
  _delay_ms(5);

  sei();

  while (1) {
    // Variables for this reading
    unsigned long start_millis = millis();
    unsigned int peak_to_peak_a = 0;
    unsigned int peak_to_peak_b = 0;
    
    unsigned int signal_max_a = 0;
    unsigned int signal_min_a = 1024;
    unsigned int signal_max_b = 0;
    unsigned int signal_min_b = 1024;
    
    // Do the reading
    while (millis() - start_millis < 50) {
      // Input A
      ADMUX &= B11000000;
      ADMUX |= B00000010;
      ADCSRA |= _BV(ADSC);
      bool conv_state = 1;
      while (conv_state) {
        conv_state = (bool)((ADCSRA & _BV(ADSC)) >> 6);
      }
      unsigned int sample_a = ADCL;
      sample_a |= ADCH << 8;
      // Input B
      ADMUX &= B11000000;
      ADMUX |= B00000011;
      ADCSRA |= _BV(ADSC);
      conv_state = 1;
      while (conv_state) {
        conv_state = (bool)((ADCSRA & _BV(ADSC)) >> 6);
      }
      unsigned int sample_b = ADCL;
      sample_b |= ADCH << 8;
      if (sample_a > signal_max_a) {
        signal_max_a = sample_a;
      } else if (sample_a < signal_min_a) {
        signal_min_a = sample_a;
      }
      if (sample_b > signal_max_b) {
        signal_max_b = sample_b;
      } else if (sample_b < signal_min_b) {
        signal_min_b = sample_b;
      }
    }
    peak_to_peak_a = signal_max_a - signal_min_a;
    peak_to_peak_b = signal_max_b - signal_min_b;

    // Moving average
    a_average = a_average - ((float)a_average_buf[a_average_ptr] / (float)AVERAGE_LEN) + ((float)peak_to_peak_a / (float)AVERAGE_LEN);
    a_average_buf[a_average_ptr] = peak_to_peak_a;
    a_average_ptr = (a_average_ptr+1) % AVERAGE_LEN;

    b_average = b_average - ((float)b_average_buf[b_average_ptr] / (float)AVERAGE_LEN) + ((float)peak_to_peak_b / (float)AVERAGE_LEN);
    b_average_buf[b_average_ptr] = peak_to_peak_b;
    b_average_ptr = (b_average_ptr+1) % AVERAGE_LEN;
    
    // Threshold inputs
    input_a_prev = input_a_cur;
    input_b_prev = input_b_cur;
    input_a_cur = ((int)a_average > 2);
    input_b_cur = ((int)b_average > 2);

    // Switching logic
    if (input_a_cur || input_b_cur) {
      if (input_a_cur && !input_b_cur) {
        output = A;
      } else if (!input_a_cur && input_b_cur) {
        output = B;
      } else if (input_a_cur && input_b_cur) {
        if (input_a_prev || input_b_prev) {
          if (input_a_prev && !input_b_prev) {
            output = A;
          } else if (!input_a_prev && input_b_prev) {
            output = B;
          }
        }
      }
    }

    // Do the siwtch
    if (output == A) {
      PORTA |= _BV(PORTA7);
    } else if (output == B) {
      PORTA &= ~_BV(PORTA7);
    }
  }
}
```

## Extra features

These are not required but I decided to include them anyway.

### Indicator LEDs

First we need to set the two LED pins as outputs. Mine are pins 8 & 9 or `PA4` & `PA5`.

```c
DDRA |= _BV(DDA4);
DDRA |= _BV(DDA7);
```

Then we setting the output we also set the LEDs.

```c
if (output == A) {
  PORTA |= _BV(PORTA7);
  PORTA |= _BV(PORTA4);
  PORTA &= ~_BV(PORTA5);
} else if (output == B) {
  PORTA &= ~_BV(PORTA7);
  PORTA &= ~_BV(PORTA4);
  PORTA |= _BV(PORTA5);
}
```

### Manual overrides

I decided to add buttons to my switch so that I can manually override it in case it doesn't detect a signal etc.

The first step is to set both button pins as inputs. This requires setting bits to 0 in the `DDRA` register. My buttons are on pins 12 & 13 or `PA1` and `PA0`. This can be added to the startup code.

```c
DDRA &= ~_BV(DDA0);
DDRA &= ~_BV(DDA1);
```

We do not want internal pull-ups since I have added my own ones. This is accomplished by setting the correct bits in `PORTA` to 0. This code also goes in startup.

```c
PORTA &= ~_BV(PORTA0);
PORTA &= ~_BV(PORTA1);
```

In each cycle we can then read the input from the `PINA` register. We do not need to shift the first read as it is already at the end of the byte. We invert both inputs as they are low when active.

```c
bool button_a, button_b;
button_a = !(PINA & _BV(PINA0));
button_b = !((PINA & _BV(PINA1)) >> 1);
```

Then if one but not both of the buttons are pressed set the output correspondingly.

```c
if (button_a && !button_b) {
  output = A;
} else if (!button_a && button_b) {
  output = B;
}
```

With those two extras the file looks like this:

```c
// Includes
#define F_CPU 8000000UL
#define TIMER_TPS F_CPU/64
#include <avr/io.h>
#include <avr/interrupt.h>
#include <util/delay.h>

// Global state
bool input_a_cur, input_b_cur, input_a_prev, input_b_prev;
typedef enum Output {
  A, B
};
Output output = A;

#define AVERAGE_LEN 10
unsigned int a_average_buf[AVERAGE_LEN];
int a_average_ptr = 0;
float a_average = 0;
unsigned int b_average_buf[AVERAGE_LEN];
int b_average_ptr = 0;
float b_average = 0;

uint64_t _millis = 0;

ISR(TIM0_COMPA_vect) {
   _millis++;
};

long unsigned int millis() {
  uint64_t m;
  cli();
  m = _millis;
  sei();
  return m;
};

// Entrypoint
int main(void) {
  cli();
  
  // Timer setup
  TCCR0A |= _BV(WGM01);
  TCCR0A &= ~_BV(WGM00);
  TCCR0B &= ~_BV(CS02);
  TCCR0B |= _BV(CS01);
  TCCR0B |= _BV(CS00);
  OCR0A = TIMER_TPS/1000;
  TIMSK0 |= _BV(OCIE0A);

  // Button input setup
  DDRA &= ~_BV(DDA0);
  DDRA &= ~_BV(DDA1);
  PORTA &= ~_BV(PORTA0);
  PORTA &= ~_BV(PORTA1);
  // LED output setup
  DDRA |= _BV(DDA4);
  DDRA |= _BV(DDA5);
  // Pin config and ADC setup
  DDRA |= _BV(DDA7);
  ADMUX &= ~_BV(REFS1);
  ADMUX &= ~_BV(REFS0);
  ADCSRA &= 0b11111000;
  ADCSRA |= 0b00000111;
  ADCSRA |= _BV(ADEN);
  _delay_ms(5);

  sei();

  while (1) {
    // Variables for this reading
    unsigned long start_millis = millis();
    unsigned int peak_to_peak_a = 0;
    unsigned int peak_to_peak_b = 0;
    
    unsigned int signal_max_a = 0;
    unsigned int signal_min_a = 1024;
    unsigned int signal_max_b = 0;
    unsigned int signal_min_b = 1024;
    
    // Do the reading
    while (millis() - start_millis < 50) {
      // Input A
      ADMUX &= B11000000;
      ADMUX |= B00000010;
      ADCSRA |= _BV(ADSC);
      bool conv_state = 1;
      while (conv_state) {
        conv_state = (bool)((ADCSRA & _BV(ADSC)) >> 6);
      }
      unsigned int sample_a = ADCL;
      sample_a |= ADCH << 8;
      // Input B
      ADMUX &= B11000000;
      ADMUX |= B00000011;
      ADCSRA |= _BV(ADSC);
      conv_state = 1;
      while (conv_state) {
        conv_state = (bool)((ADCSRA & _BV(ADSC)) >> 6);
      }
      unsigned int sample_b = ADCL;
      sample_b |= ADCH << 8;
      if (sample_a > signal_max_a) {
        signal_max_a = sample_a;
      } else if (sample_a < signal_min_a) {
        signal_min_a = sample_a;
      }
      if (sample_b > signal_max_b) {
        signal_max_b = sample_b;
      } else if (sample_b < signal_min_b) {
        signal_min_b = sample_b;
      }
    }
    peak_to_peak_a = signal_max_a - signal_min_a;
    peak_to_peak_b = signal_max_b - signal_min_b;
    
    // Read buttons
    bool button_a, button_b;
    button_a = !(PINA & _BV(PINA0));
    button_b = !((PINA & _BV(PINA1)) >> 1);

    // Moving average
    a_average = a_average - ((float)a_average_buf[a_average_ptr] / (float)AVERAGE_LEN) + ((float)peak_to_peak_a / (float)AVERAGE_LEN);
    a_average_buf[a_average_ptr] = peak_to_peak_a;
    a_average_ptr = (a_average_ptr+1) % AVERAGE_LEN;

    b_average = b_average - ((float)b_average_buf[b_average_ptr] / (float)AVERAGE_LEN) + ((float)peak_to_peak_b / (float)AVERAGE_LEN);
    b_average_buf[b_average_ptr] = peak_to_peak_b;
    b_average_ptr = (b_average_ptr+1) % AVERAGE_LEN;
    
    // Threshold inputs
    input_a_prev = input_a_cur;
    input_b_prev = input_b_cur;
    input_a_cur = ((int)a_average > 2);
    input_b_cur = ((int)b_average > 2);

    // Switching logic
    if (input_a_cur || input_b_cur) {
      if (input_a_cur && !input_b_cur) {
        output = A;
      } else if (!input_a_cur && input_b_cur) {
        output = B;
      } else if (input_a_cur && input_b_cur) {
        if (input_a_prev || input_b_prev) {
          if (input_a_prev && !input_b_prev) {
            output = A;
          } else if (!input_a_prev && input_b_prev) {
            output = B;
          }
        }
      }
    }

    // Button logic
    if (button_a && !button_b) {
      output = A;
    } else if (!button_a && button_b) {
      output = B;
    }

    // Do the siwtch
    if (output == A) {
      PORTA |= _BV(PORTA7);
      // Set LEDs
      PORTA |= _BV(PORTA4);
      PORTA &= ~_BV(PORTA5);
    } else if (output == B) {
      PORTA &= ~_BV(PORTA7);
      // Set LEDs
      PORTA &= ~_BV(PORTA4);
      PORTA |= _BV(PORTA5);
    }
  }
}
```



## Conclusion

In conclusion; was this way too complicated and over engineered? Yes. Could I have bought something with the same functionality? Yes, but it probably would have been far more expensive. Was it worth it for the learning? Definitely yes!

Thanks for reading, I apologize if this was way to low-level for you but I hope you learn't something regardless.
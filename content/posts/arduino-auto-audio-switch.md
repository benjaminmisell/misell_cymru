---
date: "2018-01-14T14:58:20Z"
title: "Automatic audio switch with Arduino"
authors: []
tags:
  - electrical
  - arduino
draft: true
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

Unlike the standard Arduino the ATTiny does not have a USB port so I need to use ICSP (In circuit serial programming) to get anything onto it. I used my other Arduino Yun as an ICSP.  This is easy enough, simply load up the ArduinoISP example sketch and program away!

![Screenshot_2018-01-14_15-30-18](/imgs/Screenshot_2018-01-14_15-30-18.png#center)

The next problem is that the Arduino IDE does not have support for ATTiny chips. This is solved by adding this URL: https://raw.githubusercontent.com/damellis/attiny/ide-1.6.x-boards-manager/package_damellis_attiny_index.json into the additional boards manager URLs in preferences.

![img](https://hackster.imgix.net/uploads/image/file/50822/2015-06-15%2022_27_58-Edit%20project%20-%20Hackster.io.png?auto=compress%2Cformat&w=680&h=510&fit=max#center)

Then going to Tools -> Board -> Board Manager

![img](https://hackster.imgix.net/uploads/image/file/50918/2015-06-15%2022_30_32-ArduinoISP%20_%20Arduino%201.6.4.png?auto=compress%2Cformat&w=680&h=510&fit=max#center)

Then go to the bottom and look for **attiny by Davis A. Mellis** and click **Install**

![img](https://hackster.imgix.net/uploads/image/file/50919/2015-06-15%2022_31_02-Boards%20Manager.png?auto=compress%2Cformat&w=680&h=510&fit=max#center)

You should now see ATTiny as a board option.

![img](https://hackster.imgix.net/uploads/image/file/50920/2015-06-15%2022_31_41-ArduinoISP%20_%20Arduino%201.6.4.png?auto=compress%2Cformat&w=680&h=510&fit=max#center)

Next you need to connect the Arduino to the ATTiny.

| ATTiny Pin    | Arduino Pin |
| ------------- | ----------- |
| 5V            | Vcc         |
| Gnd           | Gnd         |
| RESET - Pin 4 | Pin 10      |
| Pin 7         | Pin 11      |
| Pin 8         | Pin 12      |
| Pin 9         | Pin 13      |

## Writing code for the chip

### Blinky test code

As this is not a normal Arduino we cannot just call `digitalWrite` we need to directly access the port registers. In my case the control pin for the relay is on pin 8 which is `PA7`. So to output on that pin we need to set bit 7 of `DDRA` to 1 and then bit 7 of `PORTA` to our desired output value.

```c
DDRA |= _BV(DDA7);
PORTA |= _BV(PORTA7);
```

We `OR` here so as not to affect any other values. The full program to turn this on and off every second is:

```c
#include <avr/io.h>
#include <util/delay.h>
 
#define BLINK_DELAY_MS 1000

void setup() {
  DDRA |= _BV(DDA7);
}

void loop() {
  PORTA |= _BV(PORTA7);
  _delay_ms(BLINK_DELAY_MS);
 
  PORTA &= ~_BV(PORTA7);
  _delay_ms(BLINK_DELAY_MS);
}
```

This should turn the relay on and off every second. Note: the clock fuses and clock frequency in the IDE must be the same for `_delay_ms` to be correct. I have my fuses set to internal 8 MHz clock with this avrdude command.

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
int conv_val = ADCL;
conv_val |= ADCH << 8;
```

And tada! We have out value! That wasn't hard or complicated now was it.

### Reading audio level

As audio changes over time and can be silent for a short time we have to sample the maximum values over a window. The lowest audible frequency is around 20Hz so if we sample for 0.05 seconds or more we should get everything.
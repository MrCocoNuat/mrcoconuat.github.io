# Tiny Dungeon 
The classic ATtiny series are wonderful microcontrollers for all sorts of microcontollery tasks like [measuring instruments](https://github.com/MrCocoNuat/tiny-oscilloscope), edge computing/automation, and of course fun electronic toys. With up to `8KiB` of flash storage, `512B` of SRAM (that's right, no `Ki` or `Mi` here), and a bonus of `512B` EEPROM, simple single tasks like those are perfectly suited for a computer chip that is just capable enough, and that is why they are so beloved here on LowTierTech.

But, I dream of more! Why can't I game on one of these? And thus, [Tiny Dungeon](https://github.com/mrcoconuat/tiny-dungeon) was born.

## Hardware

Really, there's nothing special here.
![schematic](https://github.com/MrCocoNuat/tiny-dungeon/blob/main/schematic/tiny-dungeon-schematic.png)


### The Smarts, and In and Out

Already committed to the classic ATtiny, our choice is between '85 and '84, and of course we pick the '84 for its additional I/Os. This gives us 12, or realistically 11 excluding reset. Take another 2 off for a cheap `128x64` [I2C OLED display](https://www.amazon.com/Hosyond-Display-Self-Luminous-Compatible-Raspberry/dp/B09T6SJBV5/ref=sr_1_3) and 1 off for a piezospeaker, and that leaves 8, just enough for 2 directional pad inputs. I used these handy [miniature directional switches](https://www.amazon.com/10x10x9mm-Momentary-Square-Tactile-Switch/dp/B00E6QM2F0) that bundle 4 buttons into 1 - they also contain a 5th switch actuated by pushing the stick in, but we have run out of inputs!

If you wanted to extend the input/output capability of the microcontroller, you could of course use port extenders or clever analog tricks to extract more information from the outside world per pin. But for the purposes of Tiny Dungeon, this is enough.

A special note - be considerate of others, and include a mute switch for the speaker!

### Power

There's no need for specialty equipment here, a `1S` lithium cell will work perfectly - remember to use protection and a power switch!

## Support Software

### Input Sensing - Where are You Touching Me?

Inputs for real-time systems should ideally all generate interrupts. However, this piece of software is a game, and so the game's main thread should always take priority. Therefore, a simple polling approach is better - stuff the 8 directional stick buttons into a byte.

This somewhat arcane code takes advantage of the `PIN*` registers to grab many inputs all at once, then place them into the output in the right place.
```c++
// cram the 8 bits (high = direction pushed) into 1 return byte
// with order: msb- LD LR LU LL RD RR RU RL -lsb
uint8_t getJoystickState() {
  // remember that joystick pins are active low so invert too to get active directions
  //   and that A4, A6 are SPI pins, and A5 is the buzzer! So mask those out of pinA!!
  uint8_t pinA = PINA & ~(1 << PORTA6 | 1 << PORTA5 | 1 << PORTA4) , pinB = PINB;
  return ~(pinA | ((pinB & (1 << PORTB0)) << 6) | ((pinB & (1 << PORTB1)) << 4) | ((pinB & (1 << PORTB2)) << 2));
}
```

### Video Output

Our canvas is `128` by `64`, monochrome of course. And we have help here - fellow blogger Technoblogy has an excellent [library](http://www.technoblogy.com/show?23OS) for interfacing with this display's controller. I heavily modified it to add many new features exposed by the display controller and also to change the character plotting mode so that instead of plotting the pixels of text on top of contents already on the screen ("OR plotting") new content simply replaces what is already there.

Unfortunately, over the bit-banged I2C speeds that our microcontroller can reach, even this tiny display is so many pixels that writing solid blocks manually to the entire screen takes around 2 seconds, and the pixel-addressing here doesn't help that. Therefore animations are generally right out, and our designs will have to be sparing on the lit pixels - no matter, black is the new black!

Additionally, for saving program space there are several more optimizations to be made. The character map included in the library can be trimmed down quite a bit (who needs all those letters) and in fact customized freely to support arbitrary 8-pixel high glyphs that can represent dungeon elements - see [tiny-dungeon's spritesheet](https://github.com/MrCocoNuat/tiny-dungeon/blob/main/spritesheet/spritesheet.png)
```c++
// Character set for text - stored in program memory
// 6 columns of 8
const uint8_t CharMap[96][6] PROGMEM = {
  { 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 }, // space
  { 0x00, 0x00, 0x5F, 0x00, 0x00, 0x00 }, // !
  { 0x00, 0x07, 0x00, 0x07, 0x00, 0x00 }, // "
  ...
```

### Music to My Ears

No room for fancy audio controllers here, we have got 1 piezospeaker and a variable frequency square wave. I used Timer1's output compare B, located on pin A5. By adjusting the value of `OCR1A` and `OCR1B` and the prescaler (if that wide of a range is needed), different frequencies and duty cycles can be output. However, the prescaler adjustment just isn't necessary, since a `16` bit timer gives such a large frequency range that it would easily cover the entire human audible range.
```c++
  // compare match output on OC1B
  TCCR1A = (1 << COM1B0);
  // CTC (clear on OCR1A), /8 prescaler
  TCCR1B = (1 << WGM12);
```

For some simple math, with a prescaler of `/8` the timer ticks at `1MHz` and with `2^16` ticks in total the timer overflows at `64Hz`, which is absolutely enough as a frequency minimum. Then, we find the nearest value that gives a 12TET chord tone - C0 at `65Hz`. Dividing by `2^(1/12)` 12 times gives timer values for the rest of the octave.
```c++
// These values are in geometric progression.
const uint16_t timerTable[] = {
  15288, // C for 65Hz: 1MHz timer ticks / 65Hz desired overflow frequency = 15288 ticks to overflow
  14430, // C#
  13620, // D
  12856, // Db
  12134, // E
  11453, // F
  10810, // F#
  10204, // G
  9631, // Ab
  9090, // A
  8580, // Bb
  8098 // B
};
```
We use a fun trick to simplify the code greatly, bit shifting to divide these values by `2` each time raises by whole octaves. Additionally, alternating the prescaler between `/8` for sound output and `/0` for stop output is predicated on the sign bit of the input argument. Both `OCR1A` (for CTC so that the timer overflows upon hitting that value) and `OCR1B` (actually controlling the pin output) must be set.
```c++
void playTone(int8_t tone){
  // tone < 0 means stop output, so check that sign bit
  // in TCCR1B[CS12:CS10], 010 is prescaler /8 and 000 is stop timer
  TCCR1B = (TCCR1B & 0b11111000) | (!(tone & (1 << 7)) << CS11);
  // just set this anyway, no point adding a branch
  OCR1A = OCR1B = timerTable[tone % 12] >> (tone / 12);
}
```

## The Game

Here is of course where the real substance of the program is, and where much of the innovation needs to be manifest.

For starters, `8KiB` of flash is very very little - less than even the plain text of a typical writing assignment. Heck, even NES games start at several times larger.

What is worse is that the support functions listed above already steal about `2.2KiB` away, so we are left with even less!

### Overall Design - What's a Dungeon Crawler?

This game is somewhat inspired by the wildly successful [Pixel Dungeon](https://pixeldungeon.watabou.ru/) and its many derivatives. Its procedurally generated design and focus on replayability through random environmental factors lends itself very well to a high playability vs. program code size ratio.
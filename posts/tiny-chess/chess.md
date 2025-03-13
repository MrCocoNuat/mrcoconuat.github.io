The classic ATtiny series are wonderful microcontrollers for all sorts of microcontollery tasks like [measuring instruments](https://github.com/MrCocoNuat/tiny-oscilloscope), edge computing/automation, and of course fun electronic toys. With up to `8KiB` of flash storage, `512B` of SRAM (that's right, no `Ki` or `Mi` here), and a bonus of `512B` EEPROM, simple single tasks like those are perfectly suited for a computer chip that is just capable enough, and that is why they are so beloved here on LowTierTech.

But, I dream of more! Why can't I game on one of these? And thus, [Tiny Chess960](https://github.com/mrcoconuat/tiny-chess-960) was born.

## Hardware

Really, there's nothing special here.
![schematic](https://github.com/MrCocoNuat/tiny-chess-960/raw/4454a35d5a49e746bfeda2a8038fe422a04f1cc3/schematic/tiny-dungeon-schematic.png)


### The Smarts, and In and Out

Already committed to the classic ATtiny, our choice is between '85 and '84, and of course we pick the '84 for its additional I/Os. This gives us 12, or realistically 11 excluding reset. Take another 2 off for a cheap `128x64` [I2C OLED display](https://www.amazon.com/Hosyond-Display-Self-Luminous-Compatible-Raspberry/dp/B09T6SJBV5/ref=sr_1_3) and 1 off for a piezospeaker, and that leaves 8, just enough for 2 directional pad inputs. I used these handy [miniature directional switches](https://www.amazon.com/10x10x9mm-Momentary-Square-Tactile-Switch/dp/B00E6QM2F0) that bundle 4 buttons into 1 - they also contain a 5th switch actuated by pushing the stick in, but we have run out of inputs!

If you wanted to extend the input/output capability of the microcontroller, you could of course use port extenders or clever analog tricks to extract more information from the outside world per pin. But for the purposes of Tiny Chess960, this is enough.

A special note - be considerate of others, and include a mute switch for the speaker!

### Power

There's no need for specialty equipment here, a `1S` lithium cell will work perfectly - remember to use protection and a power switch!

---

## Support Software

The hardware is only one part of this creation - interfacing with it is not complicated, but not trivial either.

### Input Sensing

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

Our canvas is `128` by `64`, monochrome of course. And we have help here - fellow blogger Technoblogy has an excellent [library](http://www.technoblogy.com/show?23OS) for interfacing with this display's controller. I [heavily modified it](https://github.com/MrCocoNuat/tiny-dungeon/blob/main/tiny-dungeon/sh1106.cpp) to add many new features (smooth scrolling, inversion control, contrast control) exposed by the display controller and also to change the character plotting mode so that instead of plotting the pixels of text on top of contents already on the screen ("OR plotting") new content simply replaces what is already there.

Unfortunately, over the bit-banged I2C speeds that our microcontroller can reach, even this tiny display is so many pixels that writing solid blocks manually to the entire screen takes around 0.5 seconds, and the pixel-addressing here doesn't help that. Therefore animations are generally right out, and our designs will have to be sparing on the lit pixels - no matter, black is the new black!

Additionally, for saving program space there are several more optimizations to be made. The character map included in the library can be trimmed down quite a bit (who needs all those letters) and in fact customized freely to support arbitrary 8-pixel high glyphs that can represent chess elements - see [the really small spritesheet](https://github.com/MrCocoNuat/tiny-chess-960/blob/main/spritesheet/pieces.png)

```
const uint8_t monochrome_sprites[64][8] = {
    {0b10101010, 0b11101010, 0b10101110, 0b10111011, 0b10101010, 0b11101110, 0b10101011, 0b10111010},
    {0b10101010, 0b10101110, 0b11101011, 0b10111010, 0b10101110, 0b10101010, 0b10111011, 0b11101010},
    {0b10101010, 0b10111111, 0b01001010, 0b01000000, 0b01000000, 0b01000100, 0b10111111, 0b10101010},
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
  9631,  // Ab
  9090,  // A
  8580,  // Bb
  8098   // B
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

---

## The Game

Here is of course where the real substance of the program is, and where much of the innovation needs to be manifest.

[Chess960](https://en.wikipedia.org/wiki/Chess960) was introduced by former world chess champion Bobby Fischer in 1996 to reduce the emphasis on opening preparation and to encourage creativity in play (I simply paraphrase the introduction on Wikipedia).

### Working with Memory and Program Storage Constraints

For starters, `8KiB` of flash is very very little - less than even the plain text of a typical writing assignment. Heck, even NES games start at several times larger.

What is worse is that the support functions listed above already steal about `2KiB` away, so we are left with even less!

Perhaps even more stringent than the `8KiB` of program storage is the `512B` of SRAM. Almost all computation has to be done on this - even storing the current state of the game in EEPROM to get a bonus `512B` would:
  - be way way too slow, the EEPROM access times are in the `100us` range, and
  - very quickly kill the EEPROM's `100000` write cycle endurance.

In fact, I can provide, right now, a full example map of all of the memory used in this program. Of course, a dynamic heap cannot be tolerated with this little memory to go around.

```hexdump
      00 01 02 03 04 05 06 07 08 09 0A 0B 0C 0D 0E 0F
0000  3A 5F A1 1B 89 34 65 76 98 12 AB BC 32 64 72 81  #│
0010  F3 3C 2D 1A 55 73 84 D0 1E 62 A7 6A 91 8C 3B F5  #│
0020  9C 0D 6E 4F B1 8B 71 21 6D C5 D9 23 88 E7 7C 99  #│
0030  54 B2 98 E1 03 9E D2 6B 37 47 10 69 40 18 5A 83  #│
0040  6F 9A 54 38 2F B4 50 1C 7A 3E 60 6C A5 04 4E E1  #│
0050  82 2B 91 B0 36 0A D9 57 D1 A3 F2 7D 08 E5 33 79  #│
0060  6E 2D 4F 8F F0 5C 3B 51 A8 D0 62 47 C3 9B 12 C7  #│
0070  0C 71 89 B7 30 26 9D 47 01 44 C8 A6 93 B8 54 10  #│
0080  7E 64 79 53 41 D8 21 6B A7 2A 91 1D 58 09 6C 73  #│
0090  65 50 29 88 63 0A B9 F3 7A 15 52 C0 39 56 6F 4C  #│
00A0  2E 01 C4 E7 92 65 3F 5C 4A 8E 18 2C B0 3A F1 59  #│
00B0  77 A2 F6 B4 10 87 A1 4F 0B 95 03 29 47 81 D2 35  #|
00C0  41 9A 63 75 D0 50 A4 8B 9E F4 8A B3 30 0C 22 C8  #│
00D0  2B A4 79 5D F5 C6 0A 43 90 3E 56 F2 70 06 B3 28  #│
00E0  76 62 27 8A 33 4C 1F 99 5E D1 02 48 A1 76 58 0B  #│
00F0  23 9B 38 C3 B8 76 47 9A 21 8B 0D 51 D0 6D 4A 60  #└ STACK (already too small!)
0100  73 8C 92 F1 5F 61 B4 40 56 73 09 4D A0 23 89 30
0110  F4 68 0A 24 9C B2 6E 17 9F A7 59 0E 1C 7B 60 B5
0120  A4 21 81 D6 8B 4A 6D 73 91 2C 38 65 19 73 84 92
0130  68 75 6E 2D 14 36 51 9D 4C 7C 35 88 4F D0 52 97
0140  7B A6 A3 93 D0 9B 58 A1 35 72 B9 69 F5 A2 5E 81
0150  5C 17 98 4E 72 D1 53 31 19 08 8D 49 7C 66 02 10
0160  0A 96 9C 8F 32 2C 45 64 C1 A4 E9 54 B8 99 39 F3
0170  5F D8 C7 71 0C 21 37 84 A5 93 B8 74 67 45 9E 56  #│
0180  39 0E 1B 74 8B C9 10 A3 23 54 5D 77 81 9F 56 42  #│
0190  D8 A7 66 32 B4 F0 58 0C A5 1F 9A 4F B1 12 4E 27  #│
01A0  88 1B 29 A3 5B C9 F4 66 93 B5 80 4C F2 69 71 0D  #└ Chessboard
01B0  A7 68 5F 0B 29 D3 86 44 F7 C1 09 38 8D 2E 8E 71  #│
01C0  A9 0E 74 58 13 C1 5D 74 56 79 23 88 4A 3C F5 C8  #│
01D0  71 0A 58 B0 62 44 1D C4 7A F2 10 6A 8D 32 A8 F1  #└ Assorted global state (turns, positions of kings, etc.)
01E0  52 7F 3C 9E C1 28 A2 D1 A5 56 72 94 60 2F F7 10  #│
01F0  87 04 93 32 A6 1A 39 71 94 0E A8 B0 13 D1 F6 4A  #└ RANDOM LIBRARY CRAP
```

Judicious usage of storage is needed here!

#### The State Machine

Both the Main Menu and the Game are internally state machines:
```c++
enum SuperState {
  NEW_TURN,
  CONTEMPLATING,
  PIECE_IN_HAND,
  PROMOTING,
  VIEWING_HISTORY,
  GAME_OVER
};
SuperState superState = CONTEMPLATING;
```
This conceptual layout is ideal for microcomputer operations, since they can interrupt directly on inputs and immediately do their job, then change states. For example, `PIECE_IN_HAND` transitions to `NEW_TURN` when the "piece down" command is received and the cursor is over a legal move square.

Some simple globals handle shared state for the entire chess game. Here, globals make sense because... come on, there is nothing outside of this game to worry about anyway.
```c++
uint8_t cursorFile; // displayed on the screen as a selection
uint8_t cursorRank;
uint8_t handFile; // for a "piece in hand"
uint8_t handRank;
```

#### Pieces and Pawns
Each piece and pawn gets their own unique id:
```c++
enum PieceType {
    EMPTY = 0,
    // walkers
    PAWN = 0b001,
    KING = 0b010,
    KNIGHT = 0b011,
    // sliders
    BISHOP = 0b100,
    ROOK = 0b101,
    QUEEN = 0b110
};
```
These assignments are no accident, in particular the sliders are separated from the walkers by a flag bit, and everything from the knight to the queen is consecutive so that possible pawn promotions are a consecutive block of ids.

Additionally, the spritesheet is indexed by a piece's id to save space in conversion!
```c++
const uint8_t monochrome_sprites[16][8] PROGMEM = {
    {0b00000000, 0b00000000, 0b00000000, 0b00000000, 0b00000000, 0b00000000, 0b00000000, 0b00000000}, // black tile
    {0b00000000, 0b00000000, 0b00000010, 0b00101110, 0b00101110, 0b00000010, 0b00000000, 0b00000000}, // white pawn
    {0b00000000, 0b00001100, 0b00010010, 0b01001110, 0b01001110, 0b00010010, 0b00001100, 0b00000000}, // white king
    {0b00000000, 0b00000000, 0b00010010, 0b00110110, 0b01111110, 0b00111010, 0b00000000, 0b00000000}, // white knight
    {0b00000000, 0b00000000, 0b00011010, 0b00111110, 0b01001110, 0b00011010, 0b00000000, 0b00000000}, // white bishop
    {0b00000000, 0b00000000, 0b01100010, 0b00111110, 0b01111110, 0b01100010, 0b00000000, 0b00000000}, // white rook
    {0b00000000, 0b00110000, 0b00001010, 0b01101110, 0b01101110, 0b00001010, 0b00110000, 0b00000000}, // white queen
...
```

#### The Chessboard
I store the entire state of the chessboard as a `uint8_t[64]`, one byte for each square. More than just the 3 low bits that make up the identity of a piece, numerous flags for each piece have to be kept track of, the most essential of which being which player a piece belongs to.

``` c++
// bits 6 and 7 reserved for now
const uint8_t MASK_JUST_DOUBLE_MOVED = 1 << 5;  // for en-passant
const uint8_t MASK_NEVER_MOVED = 1 << 4;        // for castling
const uint8_t BIT_BLACK_ALLEGIANCE = 3;
const uint8_t MASK_BLACK_ALLEGIANCE = 1 << 3;
const uint8_t MASK_PIECE_EXISTS = 0x7;
// when just passing a turn, these are preserved;
const uint8_t MASK_LONG_LIVED = MASK_PIECE_EXISTS | MASK_BLACK_ALLEGIANCE | MASK_NEVER_MOVED;
// when moving a piece, these are always preserved
const uint8_t MASK_PERMANENT = MASK_PIECE_EXISTS | MASK_BLACK_ALLEGIANCE;
```

Generation of the initial board state is done according to the rules of Chess960, in `chess.hh`
```c++
// assumes input is in [0,960)
void initializeBoard(uint8_t theBoard[8][8], uint16_t seed);
```

Several "landmarks" on the board are always saved for quick reference:
```c++
// for castling:
// although the names don't really match, the king and queen can be anywhere relative to each other
const uint8_t CASTLE_KINGSIDE_KING_FILE = 6;
const uint8_t CASTLE_KINGSIDE_ROOK_FILE = 5;
const uint8_t CASTLE_QUEENSIDE_KING_FILE = 2;
const uint8_t CASTLE_QUEENSIDE_ROOK_FILE = 3;

// store for check calculations
uint8_t kingFiles[2];
uint8_t kingRanks[2];
```
#### Moves and Legality

Once a piece is picked up, each potential landing square has to be checked for legality with a simple algorithm that acts on information about what kind of move this is (a normal move, a capture, en passant capture, castling, promotion)
```c++
/ pre-supposes that from actually has a piece on it
uint8_t kindOfLegalMove(uint8_t turn, uint8_t theBoard[8][8], uint8_t fromFile, uint8_t fromRank, uint8_t toFile, uint8_t toRank) {
  // needs to be a move or capture in the first place
  // kindOfLegalMove is called a lot and moves/attacks calls are real expensive, short circuit when possible
  bool isMove = !(theBoard[toFile][toRank] & MASK_PIECE_EXISTS)
                && moves(theBoard, fromFile, fromRank, toFile, toRank);
  bool isAttack = theBoard[toFile][toRank] & MASK_PIECE_EXISTS
                  && ((theBoard[fromFile][fromRank] & MASK_BLACK_ALLEGIANCE) != (theBoard[toFile][toRank] & MASK_BLACK_ALLEGIANCE))
                  && attacks(theBoard, fromFile, fromRank, toFile, toRank);
  bool isEnPassant = (theBoard[fromFile][fromRank] & MASK_PIECE_EXISTS) == PAWN
                     && theBoard[toFile][fromRank] & MASK_JUST_DOUBLE_MOVED
                     && ((theBoard[fromFile][fromRank] & MASK_BLACK_ALLEGIANCE) != (theBoard[toFile][fromRank] & MASK_BLACK_ALLEGIANCE))
                     && attacks(theBoard, fromFile, fromRank, toFile, toRank);
  bool isPromotion = (theBoard[fromFile][fromRank] & MASK_PIECE_EXISTS) == PAWN
                     && ((toRank == 7 && !(theBoard[fromFile][fromRank] & MASK_BLACK_ALLEGIANCE))
                         || (toRank == 0 && theBoard[fromFile][fromRank] & MASK_BLACK_ALLEGIANCE));
  bool isPotentialCastle = (fromRank == toRank)
                  && (theBoard[fromFile][fromRank] & MASK_PIECE_EXISTS) == KING
                  && (theBoard[toFile][toRank] & MASK_PIECE_EXISTS) == ROOK
                  && theBoard[fromFile][fromRank] & MASK_NEVER_MOVED
                  && theBoard[toFile][toRank] & MASK_NEVER_MOVED
                  && ((theBoard[fromFile][fromRank] & MASK_BLACK_ALLEGIANCE) == (theBoard[toFile][fromRank] & MASK_BLACK_ALLEGIANCE));
...
```
Once that is done, the 2 parallel checks of "could my piece move there at all?" and "if I moved there, would my king be under attack?" can authoritatively determine whether the move is legal - the oft called methods are defined as:
```c++
// simplest check for "can piece on XX attack square YY", disregarding the allegiance of pieces, any king threats that would open, etc.
bool attacks(uint8_t theBoard[8][8], uint8_t fromFile, uint8_t fromRank, uint8_t toFile, uint8_t toRank) {

  uint8_t pieceType = theBoard[fromFile][fromRank] & MASK_PIECE_EXISTS;
  if (pieceType == PAWN) {
    // for the pawn to go "forward", how does its rank change? 1 for white, -1 for black
    int8_t forwardOne = 1 - 2 * ((theBoard[fromFile][fromRank] & MASK_BLACK_ALLEGIANCE) >> BIT_BLACK_ALLEGIANCE);
    return toRank - fromRank == forwardOne && (fromFile - toFile == 1 || toFile - fromFile == 1);
  }
  if (pieceType == KNIGHT) {
    return (norm(toRank - fromRank) == 2 && norm(toFile - fromFile) == 1) || (norm(toRank - fromRank) == 1 && norm(toFile - fromFile) == 2);
  }
...

bool moves(uint8_t theBoard[8][8], uint8_t fromFile, uint8_t fromRank, uint8_t toFile, uint8_t toRank) {

  uint8_t pieceType = theBoard[fromFile][fromRank] & MASK_PIECE_EXISTS;
  if (pieceType == PAWN) {
    if (fromFile != toFile)
      return false;
    // for the pawn to go "forward", how does its rank change? 1 for white, -1 for black
    int8_t forwardOne = 1 - 2 * ((theBoard[fromFile][fromRank] & MASK_BLACK_ALLEGIANCE) >> BIT_BLACK_ALLEGIANCE);
    uint8_t startingRank = (theBoard[fromFile][fromRank] & MASK_BLACK_ALLEGIANCE) ? 6 : 1;
    if (fromRank == startingRank) {
      // double move? check the intervening square
      if (!(theBoard[fromFile][fromRank + forwardOne] & MASK_PIECE_EXISTS) && toRank - fromRank == forwardOne * 2)
        return true;
    }
    return toRank - fromRank == forwardOne;
  }
  // all other pieces move like they attack
  return attacks(theBoard, fromFile, fromRank, toFile, toRank);
}
```

Castling and En Passant are handled separately, but similar.

#### Persistence

With just `512B` of EEPROM to save moves in, some smart layout will be needed to get as many moves as possible. Most chess games end in less than 100 turns, so hitting that is fine.

The `512B` are split into:
- `1B` reserved
- `1B` for how many valid turns are saved
- `2B` for the saved Chess960 seed
- `127*(2*2)B` for 127 turns of moves!

Each move saved in EEPROM looks like this:
```
  Each move is recorded in 2 bytes
  and is not stored reversibly (that is, it is impossible to "go back" just 1 turn, to do so requires replaying the whole history)
  because of the need to keep track of flags like NEVER_MOVED, JUST_DOUBLE_MOVED, etc. for captured pieces

  This is not so bad since applying known-legal moves to a board state is EXTREMELY fast compared to checking for legal moves

  2B in bitfield:
  LSB 0 1 2 3 4 5 6 7 8 9 A B C D E F MSB
  0x0 fromF fromR -toF- -toR- pr/cs

  fromFile: 0x00-0x02 (needs only store 1 out of 8 integers)
  fromRank: 0x03-0x05
  toFile: 0x06-0x08
  toRank: 0x09-0x0B
  promotion/castle: 0x0C-0x0E (needs to store 1 out of 8 values, 4 promotions, 2 castlings, and 1 more for none-of-the-above)
  reserved: 0x0F
```

You can do the conversion of these in your head into algebraic notation - it really does suffice for all legal moves!
# Tiny Dungeon 

be patient!

Most of the microcontroller faculties are described [here](/posts/tiny-chess/chess.md), the other 8KiB game article.

Memory extern 320B
- STACK >=128B (no recursion!)
- random library crap 128B
Memory (max 256B) 243B:
- SEED u32
- TURN TIMER u16
- current floor counter u4
- adventurer 18B:
  -  health (u8)
  -  location (u16)
  -  satiety (u8)
  -  level (4b), strength(4b)
  -  xp (u8)
  -  player effect bytes (u8[2])
    - type (3b)
      - none
      - haste
      - cripple
      - invis
      - arcanearmor
      - recharging
      - stun
      - bleed
     - duration (5b)
  - inventory (u8[10])
    - id (5b)
    - upgrade/count (3b)
- current floor:
  - displaybuffer (u8[12][8]) 96B
    - speeds display refreshing by not generating everyfuckingtile every refresh, only that which entities moved off, and that which the viewport scrolls onto
  - current floor details, see EEPROM section but just the 1 floor (63B[1])
  - generated terrain 36B
    - room corners (u16[2][6])
    - corridor corners (u16[3][6])
    - chasms
    - grass
EEPROM 512B:
- current floor counter saved (u4)
- persisted ordinary floors (EEPROM struct 63B[8]):
    - floor type (u8)
      - ordinary
      - overgrown (spawns grass everywhere it can)
      - chasmy (corridors are bridges)
      - large (48\*48 and 1.5\* the rooms)
    - seed (u32) that generated all permanent terrain
    - trampled permanent grass (overgrown floors... forget it) (1b[16])
    monsters: entity map (u40[8])
      -  location (10b)
      -  id (u6)
      -  health (u8)
      -  effect bytes (u8[2])
        - type (3b)
          - none
          - haste
          - cripple
          - corrode
          - weaken
          -
          - fear
          - amok
        - duration (5b)
    - items (u16[8])
      - location (10b)
      - id (5b)
      - upgrade/count (3b)

### `SEED`: Procedural Generation



Therefore, a highly efficient generation and storage algorithm is required. With this we pretty much have to make use of procedural generation.

I derive all of the randomness in the game from a single starting psuedorandom value `uint32_t SEED`. This is fed into a linear feedback shift register, which is a cheap way to generate a psuedorandom bitstream from a starting seed. The math behind these is a little more complicated than I want to include in what is mostly a software article (although if you are a fan of Galois, please read more about these, you will be very happy!). Put simply, choosing the right series of taps to derive `nextBit` will give the maximum possible period out of an `n`-bit LFSR, `2^n - 1`. For convenience, a single `nextByte` function is here, and any caller can pick out however many bits they need.
```
// simple LSFR, 2^32 - 1 period is good enough
static uint32_t randomState = 0b10000000000000000000000000000001;
uint8_t nextByte(){
  uint8_t nextByte = 0;
  for (uint8_t i = 8; i > 0; i--){
    // Feedback bit (tap positions are defined by the polynomial x^32 + x^22 + x^2 + x + 1)
    uint32_t feedback = ((randomState >> 31) ^ (randomState >> 21) ^ (randomState >> 1) ^ randomState) & 0x1;
    // Shift the register left by 1 and add the feedback at the rightmost bit
    randomState = (randomState << 1) | feedback;
    nextByte = (nextByte << 1) | feedback;
  }
  return nextByte;
}
```

As for addressing the "psuedo" in "psuedorandom", the typical microcontroller approach to seeding the initial value `randomState` is to either sample a noisy input or human user input timings - or simply keeping it persistent across restarts by writing it into EEPROM every once in a while - the right timing opportunities can be figured out later.

### Level Generation and Storage

Each normal floor (not a boss arena) should be generated with:
- 3-5 rooms (depending on region) of size 6\*8 or so in a loop, or branching off it, all be contained in a 32\*32 region for convenience
- single width corridors connecting them to make the loop
- 6-12 monsters (depending on region) spawned initially
- Stairs leading to the previous floor and the next floor

and there are supposed to be 9 of these! Obviously, storing even a single floor as a array of dungeon tiles is impossible (32\*32 = 1024). So not only does the floor need to be generated procedurally, its data needs to be stored and sent to the display buffer procedurally too!

I use this specific struct:
```

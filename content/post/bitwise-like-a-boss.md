---
title: "Bitwise Like a Boss"
date: 2019-03-06T17:47:40-05:00
draft: true
---

## The problem

Lately, I've been back at it with rust. Particurally, I'm all in on building a [chip-8](https://en.wikipedia.org/wiki/CHIP-8) emulator. Things have been going really well, but one thing I relized is dealing with binary and hex really isn't my forte.  That said since the struggle is real, I figured jotting down some of what I learned might possibly help someone out in the future.

I defined a `Cpu` struct to house the memory among other things. As you can see below we're storing 4096 bytes of memory available.  When loading the rom, We'll read from the rom file and load the game into memory starting at address `0x200`.  However, when fetching opcodes from memory to determine what heppens next we actually need 2 bytes and not one, so basically we need memory address `n` and `n + 1`.

```rust
pub struct Cpu {
    ...
    pub memory: [u8; 4096],
    pub pc: u16,
    ...
}
```

### How it works

The program in initiated by passing in a rom to load, during initilization the program counter or `pc` as defined in the `Cpu` struct is set to address `0x200`.  The passed in rom file is then loaded into the `Cpu`s memory array beginning at the address of the program counter `0x200`.

At this point typically the emulation cycle would begin and we'd start a loop in which the first step would be to fetch a opcode from memory to execute.

### Fetching the opcode

We defined a method on `Cpu` in which we grab the program counter and then `byte_one` and `byte_two`.  Since opcodes are 2 bytes long we need to grab two successive opcodes.  This is all well and good until you need to merge the two together.

```rust
// Opcodes are 2 bytes long but our memory is 1 byte so we fetch 2 succesive opcodes
fn fetch_opcode(&self) -> usize {
    let pc = self.pc;
    let byte_one: usize = self.memory[pc as usize] as usize;
    let byte_two: usize = self.memory[(pc + 1) as usize] as usize;

    // Merging the opcodes, take `byte_one` and shift it left 8 bits (add 8 zeros on right side)
    // Then we do a bitwise or with byte two which isn't shifted so will be compared against the 0's
    // add 0 | 1 == 1
    byte_one << 8 | byte_two
}
```

For example lets fetch a couple of codes from memorry based on the code above. we might have something like this:

```rust
// Hex
byte_one: 0x6a
byte_two: 0x2

// Binary
byte_one: 0b 0110 1010
byte_two: 0b 0000 0010
```

So, `byte_one` represented in 8-bit binary is `01101010`, as for `byte_two` we get `00000010`.  So how do we combine these for a single 2 byte output. So we have `byte_one` which is on byte, we're going to shift it left 8 bits with the bitshift operator `<<` to account for the other byte we're giong to combine it with.

```rust
byte_one: 01101010 << 8
// above bitshift results in: 0110|1010|0000|0000 (pipes denoting nibbles)
```

So we added those 8 bits to the end of `byte_one` which is exactly the size of `byte_two`. Now, using the bitwise `|` (OR) operator we can determine the result:

```rust
byte_one: 0110|1010|0000|0000 // notice the 8 zeros on the right hand side from shifting left 8 bits.
byte_two:           0000|0010
          -------------------
          0110|1010|0000|0010
```

So shown above is `byte_one | byte_two` (after `byte_one` bitshift). We get a 2 byte result of `0110101000000010` which translates to `0x6a02` in hex.

So that's how we obtain our opcode, pretty simple.
  1. Take the first byte and shift it left by 8 bits (byte).
  2. Use the bitwise `| OR` operator to compare.  The `OR` will look at the two bits and if either is `1` that's what it will return.

---
title: "Exploring Embedded in Rust"
date: "March 30 2024"
description: "Creating a strobe light."

---

## The Background

For no reason at all, I decided to see what it was like to write embedded firmware with Rust. After watching some videos, I picked up a microbit:v2 microcontroller from SparkFun. With a board this popular, and especially because this board is typically used for education, the libraries available abstract away all of the scary logic with manipulating registers. The `microbit-v2` crate exposes a nice API, that essentially makes it no different from writing standard Rust code.

Another project lead me to write a ChaCha20 stream-cipher implementation. All that you need to know is this structure can scramble up bytes to make them look psuedorandom. I used this to make the 25 LED pins flash randomly. The result is a random strobe light.

## The Code

```rust
#![no_std]
#![no_main]
use cortex_m::asm::nop;
use chacha20::ChaCha20;
use cortex_m_rt::entry;
use panic_halt as _;
use microbit::{
    display::blocking::Display, hal::{prelude::*, uarte::{self, Baudrate, Parity}, Timer}, Board
};
use rtt_target::{rprintln, rtt_init_print};
const DANCE_PROB: u8 = 224;

#[entry]
fn main() -> ! {
    rtt_init_print!();
    let board = Board::take().unwrap();
    let mut timer = Timer::new(board.TIMER0);
    let mut display = Display::new(board.display_pins);
    let nonce = b"Hello, world";
    let key = [0; 32];
    let mut chacha = ChaCha20::new(key, *nonce, 0);
    let mut to = *b"Lite dance";
    rprintln!("!");
    loop {
        let mut light_none = [
            [0, 0, 0, 0, 0],
            [0, 0, 0, 0, 0],
            [0, 0, 0, 0, 0],
            [0, 0, 0, 0, 0],
            [0, 0, 0, 0, 0],
        ];
        for row in light_none.iter_mut() {
            chacha.apply_keystream(&mut to);
            for (i, t) in to.iter().enumerate() {
                if *t > DANCE_PROB {
                    row[i % 5] = 1;
                }
            }
        }
        display.show(&mut timer, light_none, 100);
        display.clear();
        timer.delay_ms(100_u32);
    }
}
```

## Closing Thought

I might make a keyboard next...
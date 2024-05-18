# Getting started using RUST on RP2040 (Raspberry Pi PICO)!

<!--toc:start-->
- [Getting started using RUST on RP2040 (Raspberry Pi PICO)!](#getting-started-using-rust-on-rp2040-raspberry-pi-pico)
  - [Introduction](#introduction)
    - [What is actually an RP2040 ?](#what-is-actually-an-rp2040)
    - [Understanding target](#understanding-target)
  - [Configuration](#configuration)
    - [Installing RUST](#installing-rust)
    - [First project](#first-project)
    - [Using a Hardware Abstraction Layer (HAL)](#using-a-hardware-abstraction-layer-hal)
    - [Using a custom board](#using-a-custom-board)
    - [Memory layout and Linking](#memory-layout-and-linking)
    - [Send your program to the microcontroller](#send-your-program-to-the-microcontroller)
  - [Final Thoughts](#final-thoughts)
  - [Going further](#going-further)
<!--toc:end-->

If you are here, you probably want to use RUST on some RP2040 development board like the Raspberry Pi PICO or the Waveshare RP2040-Zero. Well, it turns out you are in the right place. Let's just being by understanding what we need to set up in order to have a working project.

## Introduction

### What is actually an RP2040 ?

Let's start with the beginning, the RP2040 is a System-On-Chip, this means that it includes a more that just a CPU. In fact, inside this chip you will find a [Dual-core Arm Cortex M0+ processor](https://developer.arm.com/Processors/Cortex-M0-Plus), but also the SRAM block, a block to handle Direct Memory Access, Inputs/Outputs (like GPIO or bus like SPI or I2C), etc...

This means that the code that we will be writing will be instructions for the Arm Cortex M0+ processor, to then either, do software stuff like mathematical operations or memory handling, or hardware like control I/O pins to blink a LED.

### Understanding target

> Wait so, can't I just build my code and execute it on the board ?!

Unfortunately (or maybe fortunately), all the CPU are not the same. What I mean by that is that different kind of architecture exist. For example, AMD and INTEL CPU that you can find in most of modern computer are different processor, but they use the same architecture: x86. This is why you can download software without thinking about compatibility between the two. But, new macOS users probably remember the switch from INTEL processor to the Apple's M1 chip, indeed, software needed to be especially compiled for it in order to run. This is because M1 chip actually use an ARM architecture and not the x86.

But, if you have been programming on only small local project, you probably never heard of this target thing. This is because you most likely have only built code from the same device that you will run the code on (which is not our case today). By default, the compiler will use the current device as target, if not, you need to do cross-compilation. Cross-compilation is when you use a compiler compiled for a target to compile code to another target. For example, if you use a compiler that runs on x86 to compile code for an ARM target.

> Then, why is there a difference between Linux & Windows software on the same hardware ?

This is because there in fact, 3 parameters that compose a target known as the [Target Triplet](https://wiki.osdev.org/Target_Triplet). First, you have the architecture (what we discuss earlier), then you can have a vendor name (like "apple" or "pc") but it can be ignored if null, finally you have the operating system, and you can have additional information about target environment.

And the operating system is important because it provides the std features.

Now, that we understand these basics principles it will be easier to understand the project configuration.

## Configuration

### Installing RUST

First things first, to create a RUST project, handle dependencies and compile your code you will need a bunch of tools such as package manager (cargo), rust copiler and so on. Luckily, you can use the [rustup](https://www.rust-lang.org/tools/install) tool that will help use install everything easily and handle targets and toolchain.

### First project

First things first, we are going to need a RUST project, which can be created using these commands:

```sh
cargo new my_project
cd ./my_project
```

Now, you should be inside your RUST project. You can try to run it using `cargo run` to behold the timeless but always beautiful "Hello, World!" message appears on the screen.

Now, as explain earlier we need to find the target that match our processor. Target list for RUST can be found [here](https://doc.rust-lang.org/rustc/platform-support.html), we will need: `thumbv6m-none-eabi`. If we look at the description it is the target for `Bare ARMv6-M` which is what we want. As we can see on the target name the operating system is `none`, this is perfectly normal as we want to run our code on the bare metal. Therefore, we will not have access to the std crate.

> Bare metal means that there is no abstraction layer between your code and the hardware.

Now that we have located which target we want, we will ensure that the latest version of RUST with the target using:
```sh
rustup self update
rustup update stable
rustup target add thumbv6m-none-eabi
```

If we try to compile the previous code using our target we should have the following error:

```console
~/my_project$ cargo build --target=thumbv6m-none-eabi
   Compiling my_project v0.1.0 (~/my_project)
error[E0463]: can't find crate for `std`
```

> Of course, this time we cannot run it because our computer does not support a thumbv6m architecture. This is why we switch to the build command for now.

This is perfectly normal because our target does not have any underlying layer to handle the std library. As suggested by the error, we should write our code in a `#[no_std]` environment. Let's try this [minimal code](https://docs.rust-embedded.org/embedonomicon/smallest-no-std.html):

```rust
#![no_main]
#![no_std]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_panic: &PanicInfo<'_>) -> ! {
    loop {}
}
```

This code will do nothing, but will not give errors on compilation either. It provides the minimal information for the compiler: not using std, not expecting a main function and what to do if a panic happened. It can be compiled as previously using `cargo build --target=thumbv6m-none-eabi`.

### Using a Hardware Abstraction Layer (HAL)

Now, that we have a working sample code for our target, it is now time to try doing stuff with our microcontroller because it is actually the goal here. For that, you have two choices: write your how function by directly addressing memory map (registers) by referring to the RP2040 Datasheet (hard way), or use a library that do this hard work for us. The second option is what we called a HAL, it allows for an abstraction between your code and the hardware.

For the RP2040 there is [this](https://github.com/rp-rs/rp-hal) HAL. If you take a look at the README, you will notice that the HAL also provides various Board support packages. These packages are build on top of the HAL to specify a board behavior, for example, to provide the built-in LED pin or aliases for pins to specify their capabilities.

Secondly, a lot of HAL are built atop of the [embedded_hal](https://docs.rs/embedded-hal/latest/embedded_hal/) crate and other abstraction crate. This allows us to use generic driver for interacting with sensors or actuators, that are not specific for a type of board.

For the rest of this article I will be using the [Waveshare RP2040 Zero](https://www.waveshare.com/wiki/RP2040-Zero) with the [corresponding board hal](https://github.com/rp-rs/rp-hal-boards/tree/main/boards/waveshare-rp2040-zero).

The setup is now quite simple thanks to the cargo package manager. We need to tell cargo about our dependencies by adding this inside our `Cargo.toml`:

```toml
[dependencies]
cortex-m-rt = "0.7.4"
embedded-hal = "1.0.0"
panic-halt = "0.2.0"
smart-leds = "0.3.0"
waveshare-rp2040-zero = "0.8.0"
ws2812-pio = "0.8.0"
```

Now, we can add the following code to our `main.rs`:

```rust
//! Rainbow effect color wheel using the onboard NeoPixel on an Waveshare RP2040 Zero board
//!
//! This flows smoothly through various colors on the onboard NeoPixel.
//! Uses the `ws2812_pio` driver to control the NeoPixel, which in turns uses the
//! RP2040's PIO block.
#![no_std]
#![no_main]

use core::iter::once;
use embedded_hal::delay::DelayNs;
use panic_halt as _;
use smart_leds::{brightness, SmartLedsWrite, RGB8};
use waveshare_rp2040_zero::entry;
use waveshare_rp2040_zero::{
    hal::{
        clocks::{init_clocks_and_plls, Clock},
        pac,
        pio::PIOExt,
        timer::Timer,
        watchdog::Watchdog,
        Sio,
    },
    Pins, XOSC_CRYSTAL_FREQ,
};
use ws2812_pio::Ws2812;

#[entry]
fn main() -> ! {
    let mut pac = pac::Peripherals::take().unwrap();

    let mut watchdog = Watchdog::new(pac.WATCHDOG);

    let clocks = init_clocks_and_plls(
        XOSC_CRYSTAL_FREQ,
        pac.XOSC,
        pac.CLOCKS,
        pac.PLL_SYS,
        pac.PLL_USB,
        &mut pac.RESETS,
        &mut watchdog,
    )
    .ok()
    .unwrap();

    let sio = Sio::new(pac.SIO);
    let pins = Pins::new(
        pac.IO_BANK0,
        pac.PADS_BANK0,
        sio.gpio_bank0,
        &mut pac.RESETS,
    );

    let timer = Timer::new(pac.TIMER, &mut pac.RESETS, &clocks);

    // Configure the addressable LED
    let (mut pio, sm0, _, _, _) = pac.PIO0.split(&mut pac.RESETS);
    let mut ws = Ws2812::new(
        // The onboard NeoPixel is attached to GPIO pin #16 on the Feather RP2040.
        pins.neopixel.into_function(),
        &mut pio,
        sm0,
        clocks.peripheral_clock.freq(),
        timer.count_down(),
    );

    // Infinite colour wheel loop
    let mut n: u8 = 128;
    let mut timer = timer; // rebind to force a copy of the timer
    loop {
        ws.write(brightness(once(wheel(n)), 32)).unwrap();
        n = n.wrapping_add(1);

        timer.delay_ms(25);
    }
}

/// Convert a number from `0..=255` to an RGB color triplet.
///
/// The colours are a transition from red, to green, to blue and back to red.
fn wheel(mut wheel_pos: u8) -> RGB8 {
    wheel_pos = 255 - wheel_pos;
    if wheel_pos < 85 {
        // No green in this sector - red and blue only
        (255 - (wheel_pos * 3), 0, wheel_pos * 3).into()
    } else if wheel_pos < 170 {
        // No red in this sector - green and blue only
        wheel_pos -= 85;
        (0, wheel_pos * 3, 255 - (wheel_pos * 3)).into()
    } else {
        // No blue in this sector - red and green only
        wheel_pos -= 170;
        (wheel_pos * 3, 255 - (wheel_pos * 3), 0).into()
    }
}
```

This is the code from the example directory of the board package. Let's break down the code slowly to understand it.

The first thing you will notice is that our `panic_handler` function is no longer needed. This is thanks to the `panic_halt` package that provide an implementation of the panic function in a `no_std` context by halting the microcontroller if a panic happens. 

> This is not super convenient to understand what is happening. If your program is more complex you may prefer to write your own panic handler to store or show the error message somewhere.

Then, you have the `#[entry]` flag that represents the entry function of your program (a main function if you prefer).

Finally, we are using a crate to communicate with the built-in Neopixel LED on the board.

> Note that if you want to use another board it is the same process, you just need to use a different board package (and using whatever interface you need: LED, RGB LED, OLED Screen, PWM, ...). 

### Using a custom board

You may also want to use an RP2040 on a board that does not provide a package. In this case, no problem, simply import directly these dependencies and features:

```toml
[dependencies]
cortex-m-rt = { version = "0.7.3", optional = true }
rp2040-boot2 = { version = "0.3.0", optional = true }
embedded-hal = "0.2.7"

[features]
# This is the set of features we enable by default
default = ["boot2", "rt", "critical-section-impl", "rom-func-cache"]

# critical section that is safe for multicore use
critical-section-impl = ["rp2040-hal/critical-section-impl"]

# 2nd stage bootloaders for rp2040
boot2 = ["rp2040-boot2"]

# Minimal startup / runtime for Cortex-M microcontrollers
rt = ["cortex-m-rt","rp2040-hal/rt"]

# This enables a fix for USB errata 5: USB device fails to exit RESET state on busy USB bus.
# Only required for RP2040 B0 and RP2040 B1, but it doesn't hurt to enable it
rp2040-e5 = ["rp2040-hal/rp2040-e5"]

# Memoize(cache) ROM function pointers on first use to improve performance
rom-func-cache = ["rp2040-hal/rom-func-cache"]

# Disable automatic mapping of language features (like floating point math) to ROM functions
disable-intrinsics = ["rp2040-hal/disable-intrinsics"]

# This enables ROM functions for f64 math that were not present in the earliest RP2040s
rom-v2-intrinsics = ["rp2040-hal/rom-v2-intrinsics"]
```

Then, you can provide (it is optional) the pin description using the `hal::bsp_pins!` macro. And you will need to add these lines to set up bootloader and define the external clock used:

```rust
/// The linker will place this boot block at the start of our program image. We
/// need this to help the ROM bootloader get our code up and running.
/// Note: This boot block is not necessary when using a rp-hal based BSP
/// as the BSPs already perform this step.
#[link_section = ".boot2"]
#[used]
pub static BOOT2: [u8; 256] = rp2040_boot2::BOOT_LOADER_GENERIC_03H;

/// External high-speed crystal on the Raspberry Pi Pico board is 12 MHz. Adjust
/// if your board has a different frequency
const XTAL_FREQ_HZ: u32 = 12_000_000u32;
```

### Memory layout and Linking

Now, we are able to successfully build our code (without errors or warnings hopefully). But, there is one last thing that we didn't talk about: Memory Layout. As you might or might not know, a CPU expect memory to be arranged in a certain order. For example, for our RP2040: ROM starts at address 0x00000000 and the SRAM at address 0x20000000.

> Note that the layout is more complex than that, but it is simplified to better understand the problem.

So, we need a way to tell our compiler how it is supposed to use the memory layout to put data in the right place. For that, we are going to use a link script. Simply write that on a `memory.x` file.

```
MEMORY {
    BOOT2 : ORIGIN = 0x10000000, LENGTH = 0x100
    FLASH : ORIGIN = 0x10000100, LENGTH = 2048K - 0x100
    RAM   : ORIGIN = 0x20000000, LENGTH = 256K
}

EXTERN(BOOT2_FIRMWARE)

SECTIONS {
    /* ### Boot loader */
    .boot2 ORIGIN(BOOT2) :
    {
        KEEP(*(.boot2));
    } > BOOT2
} INSERT BEFORE .text;
```

What this does is simply defining how the memory should be used, and we are defining a custom section so that we can write the bootloader at the very beginning of the ROM before the program (before the .text section).

> You can also change the FLASH size to match the exact size of the flash that is used in your board.

Now, to tell the RUST compiler to use our script we will need to add a cargo config file under: `.cargo/config.toml` with:

```toml
# set default build target for rp2040
[build]
target = "thumbv6m-none-eabi"
# upload binary to rp2040 instead of running on host
[target.thumbv6m-none-eabi]
runner = "elf2uf2-rs -d"
# use appropriate memory layout
rustflags = ["-C", "link-arg=-Tlink.x"]
```

With these configs we tell cargo to use our target as default, so we don't have to use the `--target` argument each time. Then, we use the `elf2uf2-rs` runner to translate the ELF output file to an uf2 file that can be dragged into the RP2040 in boot mode using the USB interface.

### Send your program to the microcontroller

Finally!! We have everything we need to now, compile and send our code to our RP2040 board. For that, press the BOOT button on your board and plug in using the USB port on you machine. The board should act as a mass storage for your computer. Then, simply run `cargo run --release` command to compile and load the uf2 file into the board.

And that's it, you can now simply program your RUST project with any crate that you want (if they are no_std compliant) and plug your board and run your program.

## Final Thoughts

To conclude this, let me tell you what do I think about this. First, you should have notice but set up the project is not trivial even if with appropriate knowledge it is not difficult either. And, once the first set-up is done, it is really easy to build and run the project. Which means that you could use a template to start a new project quickly. 

This idea of quick setup through template join my second point which is that by using RUST, we can take advantages of the Cargo package manager to easily handle multiple dependencies for our project.

Finally, as RUST provide the possibility to create high level abstraction through crate and trait. Using it for embedded development is fascinating. For example, you can use a driver for a sensor that rely on a well-known I2C trait that is also implemented by your HAL, allowing for full compatibility between different crate more easily.

So, I would say that at the end, even if sometimes unsafe code is required by the low level nature of the code in bare metal development, I really enjoy using RUST with the RP2040 chip.

## Going further

If you want see more code example you can check out my GitHub where I have [this repo](https://github.com/Captainfl4me/rust-2-rp/tree/master) for testing purpose. You can also use the [RP2040-HAL project template](https://github.com/rp-rs/rp2040-project-template).
If you want to make any comment on this article or have any questions, feel free to raise a GitHub issue [here](https://github.com/Captainfl4me/wab/issues).

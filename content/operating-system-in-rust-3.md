+++
title = "Building an Operating System In Rust Part 3"
slug = "building-an-operating-system-in-rust-part-3"
date = 2026-06-20
description = "In high level land, writing to the screen is a trivial task: you want text then just call println, you want shapes then just use a drawing library like bevy or wgpu. In kernel land, these luxuries have been stripped away. You have to earn every pixel yourself."

[extra]
header_img = "/images/header-rust.jpg"
header_alt = "Rust"
section = "Programming"
author = "Benjamin Chibuzor-Orie"
under_construction = true
tags = ["Rust", "Operating System", "Low Level"]

[taxonomies]
series = ["Building an Operating System in Rust"]
+++

Before you start, note that this chapter is still under construction. The complete source code already lives on [Github](https://github.com/iambenkay/kluster/tree/a2b2d150e4a5013b133b77816c124033358692be) but preparing a deep dive takes time. If you are up to it, you can clone the repository and actually get ahead while I work on the blog.

In [Part 2 we built an entrypoint for our operating system](operating-system-in-rust-2.md) and ran it successfully on QEMU and a physical Raspberry Pi 5 device. We also learnt about the boot image and how to link our kernel source code into one kernel binary. Next we are going to write to the screen!

If you have been binging this series, now is the time to take a short break before going into the next part because it's the most complex part yet. In high level land, writing to the screen is a trivial task: you want text then just call `println`, you want shapes then just use a drawing library like `bevy` or `wgpu`. In kernel land, these luxuries have been stripped away. You have to earn every pixel yourself. We will learn our first brutal lesson when it comes to writing operating systems: You have to be willing to dig through obscure documentation to find what you are looking for. In many cases the documentation will be non-existent and you need to find other ways to achieve your goal.

As I stated in previous parts, I built this operating system to run on Raspberry Pi 5, a poorly documented interface compared to the previous generations (Pi 3 & 4) and my only hope for implementing some features, for example, drawing to the screen was by digging into the disk image of the raspberry pi OS image. I will explain all the pain I went through and how I found the resources hidden where no one would think to look but be prepared to go through worse if you venture out into building more complicated features outside the scope of this tutorial. Operating System Development has never been an easy interest to pursue and the industry players don't seem to be keen about reducing the barrier to entry so you need a heart of stone to push through. Mind you I went through a lot of turmoil to support just one motherboard (raspberry pi). Imagine trying to support all the computers that exist out there. Just sit on that...

Okay, saddle up! let us dive into the topic for today.

# Writing to the screen

Different motherboards have different peripheral systems and in order to successfully write to the screen, we have to handle it the way each motherboard expects. For Raspberry Pi, there are onboard HDMI ports for video output. The goal of this exercise is to learn how to draw to its video output so that when it is connected to a monitor we can see those shapes! In order to understand how the shapes are rendered we have to talk about the CPU architecture of this board.

## Raspberry Pi CPU Architecture

The Pi is actually a two processor system consisting of the VideoCore GPU and the ARM Cores. The VideoCore GPU is the primary processor; it starts up first, runs proprietary firmware, sets up system clocks, initializes RAM, sets up the HDMI display and starts up the ARM cores. It is the most important processor on the board. It owns and controls access to the frame buffer, power rails, board info, temperature, audio, etc. and other system peripherals. Our kernel however is running on the ARM Cores so the question becomes how do we communicate to VideoCore when we need to use any of those resources? We do this using the **mailbox**.

## Mailbox

The mailbox is how we send commands to and from VideoCore in order to control any resource it manages. It's a simple and straightforward interface actually; we send a mailbox request, we wait for a response from the VideoCore and then we use the response. Full disclosure, using the mailbox for writing to the screen is a legacy approach in raspberry pi 5; there is a newer API available specifically for this task but I read that it is more complicated than the mailbox (I have not tried it yet but will do so eventually) so I decided to use mailbox for the sake of this tutorial. Let us configure our mailbox request-response flow!

Create a new file `src/motherboards/raspberrypi/mailbox.rs` and add the following content:

```rust
use core::ptr::{addr_of, addr_of_mut, read_volatile, write_volatile};

const MBOX_RESPONSE: u32 = 0x8000_0000;
const MBOX_FULL: u32 = 0x8000_0000;
const MBOX_EMPTY: u32 = 0x4000_0000;

struct MboxAddress;

impl MboxAddress {
    #[inline(always)]
    fn root() -> usize {
        #[cfg(feature = "rpi5")]
        return 0x10_7C01_3880;

        #[cfg(feature = "rpi4")]
        0xFE00_B880
    }
    #[inline(always)]
    fn read() -> *const u32 {
        (Self::root() + 0x00) as *const u32
    }
    #[inline(always)]
    fn status() -> *const u32 {
        (Self::root() + 0x18) as *const u32
    }
    #[inline(always)]
    fn write() -> *mut u32 {
        (Self::root() + 0x20) as *mut u32
    }
}

#[repr(C, align(16))]
struct MailboxBuffer([u32; 36]);

impl MailboxBuffer {
    const fn new() -> MailboxBuffer {
        MailboxBuffer([0; 36])
    }
}

static mut MBOX_BUFFER: MailboxBuffer = MailboxBuffer::new();

/// set value at mailbox index
pub fn mbox_set(i: usize, v: u32) {
    unsafe {
        let base = addr_of_mut!(MBOX_BUFFER.0) as *mut u32;
        write_volatile(base.add(i), v);
    }
}

/// get value at mailbox index
pub fn mbox_get(i: usize) -> u32 {
    unsafe {
        let base = addr_of!(MBOX_BUFFER.0) as *const u32;
        read_volatile(base.add(i))
    }
}

/// Send mailbox request on channel `channel` and wait for response
pub fn mbox_call(channel: u32) -> bool {
    let addr = addr_of!(MBOX_BUFFER) as usize as u32;
    let channel = (addr & !0xF) | (channel & 0xF);

    while is_mbox_full() {
        core::hint::spin_loop();
    }

    write_mbox_request(channel);

    loop {
        while is_mbox_empty() {
            core::hint::spin_loop();
        }
        if does_mbox_response_match(channel) {
            return mbox_get(1) == MBOX_RESPONSE;
        }
    }
}

fn is_mbox_full() -> bool {
    unsafe { (read_volatile(MboxAddress::status()) & MBOX_FULL) != 0 }
}

fn is_mbox_empty() -> bool {
    unsafe { (read_volatile(MboxAddress::status()) & MBOX_EMPTY) != 0 }
}

fn write_mbox_request(request: u32) {
    unsafe { write_volatile(MboxAddress::write(), request) }
}

fn does_mbox_response_match(request: u32) -> bool {
    unsafe { read_volatile(MboxAddress::read()) == request }
}
```

Then add the module to `src/motherboards/raspberrypi/mod.rs`:

```rust
pub mod mailbox;
```

As usual, let's zoom in on each line in the earlier snippet and see what is going on.

```rust
use core::ptr::{addr_of, addr_of_mut, read_volatile, write_volatile};
```

Let me describe what each of these imports do for us:

1. `addr_of` is a macro that returns the memory location of a variable.
2. `addr_of_mut` is a macro similar to `addr_of` but it returns a mutable pointer to the memory location.
3. `read_volatile` is a function that reads data from a memory location. The naive way to do this would have been to dereference a pointer but its not reliable to do it that way as it may be entirely elided or reordered by the Rust compiler so the safe bet is this function.
4. `write_volatile` is a function that writes to a memory location in a persistent way that will not be elided or reordered by the Rust compiler.

Next we define masks for interpreting the data returned by the mailbox at any given point in time:

```rust
const MBOX_RESPONSE: u32 = 0x8000_0000;
const MBOX_FULL: u32 = 0x8000_0000;
const MBOX_EMPTY: u32 = 0x4000_0000;
```

Where:

1. `MBOX_RESPONSE` is used to check if the mailbox has received a response
2. `MBOX_FULL` is used to check if the mailbox is ready to process a new request
3. `MBOX_EMPTY` is used to check if the mailbox has received a response for a pending request

Next, we define the various memory addresses required for interacting with the VideoCore processor's mailbox:

```rust
struct MboxAddress;

impl MboxAddress {
    #[inline(always)]
    fn root() -> usize {
        #[cfg(feature = "rpi5")]
        return 0x10_7C01_3880;

        #[cfg(feature = "rpi4")]
        0xFE00_B880
    }
    #[inline(always)]
    fn read() -> *const u32 {
        (Self::root() + 0x00) as *const u32
    }
    #[inline(always)]
    fn status() -> *const u32 {
        (Self::root() + 0x18) as *const u32
    }
    #[inline(always)]
    fn write() -> *mut u32 {
        (Self::root() + 0x20) as *mut u32
    }
}
```

In `fn root()`, we use two of our previously configured feature flags from `Cargo.toml`. We configure a different address for either Pi 5 or Pi 4. If we wanted to support Pi 3 then we would define that here too. Mind you, I didn't pick these addresses. They are predefined addresses and they have to be exactly correct for any code you are writing to work. Later, I will show you how I got these addresses and how you can apply the approach when implementing other features for your raspberry pi. This is the base address for the mailbox and all the other addresses are calculated as offsets from it.

When the mailbox responds to a specific request it stores information that can identify that request in the memory address returned by `fn read()` above which is just the same as the base address.

The memory address returned by `fn status()` tracks the busy/idle status of the mailbox. It is used in conjunction with the masks to ascertain the exact status of the mailbox at any given time.

The memory address returned by `fn write()` is used to submit a mailbox request to the VideoCore processor.

We use `#[inline(always)]` to instruct the compiler to always inline the instructions generated for this function at the call site. Helps reduce function call overhead but it is not to be abused since it can actually harm performance if misused. It's good to use it here since we are just computing some values that don't deserve the overhead of a function call.

Next, we define the Mailbox buffer:

```rust
#[repr(C, align(16))]
struct MailboxBuffer([u32; 36]);

impl MailboxBuffer {
    const fn new() -> MailboxBuffer {
        MailboxBuffer([0; 36])
    }
}

static mut MBOX_BUFFER: MailboxBuffer = MailboxBuffer::new();
```

Before we send any request to the VideoCore processor, we have to store the details of our request in this buffer, retrieve it's address and then send it so the VideoCore knows where to retrieve the request information from.

```rust
/// set value at mailbox index
pub fn mbox_set(i: usize, v: u32) {
    unsafe {
        let base = addr_of_mut!(MBOX_BUFFER.0) as *mut u32;
        write_volatile(base.add(i), v);
    }
}

/// get value at mailbox index
pub fn mbox_get(i: usize) -> u32 {
    unsafe {
        let base = addr_of!(MBOX_BUFFER.0) as *const u32;
        read_volatile(base.add(i))
    }
}
```

The two functions above are helpers used to read/write to any index of the Mailbox buffer.

Next we define `fn mbox_call()` which actually coordinates firing the request and waiting for a response for each request:

```rust
// Send mailbox request on channel `channel` and wait for response
pub fn mbox_call(channel: u32) -> bool {
    let addr = addr_of!(MBOX_BUFFER) as usize as u32;
    let channel = (addr & !0xF) | (channel & 0xF);

    while is_mbox_full() {
        core::hint::spin_loop();
    }

    write_mbox_request(channel);

    loop {
        while is_mbox_empty() {
            core::hint::spin_loop();
        }
        if does_mbox_response_match(channel) {
            return mbox_get(1) == MBOX_RESPONSE;
        }
    }
}
```

The flow is simple; if the mailbox is full, wait for it to be ready. then write the mailbox buffer's location to the mailbox write address, then wait for the mailbox to not be empty (return a response) then check if the response saved to the buffer matches the request that was sent. If it does then our response has been successfully saved to the mailbox buffer.

Finally, the helpers referenced in the previous snippet:

```rust
fn is_mbox_full() -> bool {
    unsafe { (read_volatile(MboxAddress::status()) & MBOX_FULL) != 0 }
}

fn is_mbox_empty() -> bool {
    unsafe { (read_volatile(MboxAddress::status()) & MBOX_EMPTY) != 0 }
}

fn write_mbox_request(request: u32) {
    unsafe { write_volatile(MboxAddress::write(), request) }
}

fn does_mbox_response_match(request: u32) -> bool {
    unsafe { read_volatile(MboxAddress::read()) == request }
}
```

The names are quite descriptive but let's talk about each one briefly:

1. `fn is_mbox_full()` uses the `MBOX_FULL` to check the status address of the mailbox. Returns `true` when the mailbox is ready to process another request.
2. `fn is_mbox_empty()` uses the `MBOX_EMPTY` and returns true if we are still waiting for a response from the mailbox.
3. `fn write_mbox_request()` is used to tell the VideoCore the memory location where it can find our mailbox buffer.
4. `fn does_mbox_response_match()` compares the request with the response header to make sure we are reading the right data for each request.

### Why so much unsafety?

If you are uncomfortable with the word `unsafe` showing up in your code you are going to have to get past that because if you want to do any real low level work, you need to be willing to get your hands dirty. Worry not, we isolate them properly so that we can easily debug any seg faults arising from our use of unsafe.

We just implemented an interface for interacting with the mailbox now we need to actually send requests using this interface.

## Can I get a frame buffer?

In Raspberry Pi, the screen is driven by memory mapped IO which means each pixel on the display is directly mapped to a memory location in the VideoCore processor. In order to instruct the VideoCore to allocate memory for the screen aka the frame buffer, we need to make a call to the mailbox specifying the parameters of our display which include but is not limited to: virtual screen size, physical screen size, pitch, depth, bits per pixel, pixel order, etc. After we provide all this in our request and send it, then the frame buffer will be returned in the mailbox response and we can now write to it each time we want to draw to the screen.

Before we start implementing our code for requesting the frame buffer let's set up the foundation with some helper utilities!

Create a new file `src/color.rs`:

```rust
pub struct Color {
    r: u8,
    g: u8,
    b: u8,
    a: u8,
}

impl Color {
    pub fn from_rgb(r: u8, g: u8, b: u8) -> Self {
        Self::from_rgba(r, g, b, u8::MAX)
    }

    pub fn from_rgba(r: u8, g: u8, b: u8, a: u8) -> Self {
        Self { r, g, b, a }
    }

    pub fn red() -> Self {
        Self::from_rgb(0xFF, 0, 0)
    }

    pub fn green() -> Self {
        Self::from_rgb(0, 0xFF, 0)
    }

    pub fn blue() -> Self {
        Self::from_rgb(0, 0, 0xFF)
    }

    pub fn black() -> Color {
        Self::from_rgb(0, 0, 0)
    }

    pub fn white() -> Self {
        Self::from_rgb(0xFF, 0xFF, 0xFF)
    }

    fn to_rgb565(&self) -> u16 {
        let r = (self.r as u16 >> 3) & 0x1F;
        let g = (self.g as u16 >> 2) & 0x3F;
        let b = (self.b as u16 >> 3) & 0x1F;
        (r << 11) | (g << 5) | b
    }

    fn to_rgb(&self) -> u32 {
        #[cfg(feature = "ltr-rgb")]
        return u32::from_le_bytes([self.r, self.g, self.b, self.a]);

        #[cfg(not(feature = "ltr-rgb"))]
        return u32::from_le_bytes([self.b, self.g, self.r, self.a]);
    }
}

impl From<&Color> for u32 {
    fn from(value: &Color) -> Self {
        value.to_rgb()
    }
}

impl From<&Color> for u16 {
    fn from(value: &Color) -> Self {
        value.to_rgb565()
    }
}
```

This code is self explanatory. Just a basic color utility for when we want to print to screen. We have options for regular `rgb` and `rgb565` which is for 16 bit color mode.

Add the module to `src/main.rs`:

```rust
mod color;
```

Create a new file `src/geometry.rs`:

```rust
pub struct Point(i32, i32);

impl Point {
    pub fn new(x: i32, y: i32) -> Self {
        Point(x, y)
    }

    pub fn xy(xy: i32) -> Self {
        Self::new(xy, xy)
    }

    pub fn x(&self) -> i32 {
        self.0
    }

    pub fn y(&self) -> i32 {
        self.1
    }
}
```

Simple code also, stores geometry information of a point. Add the module to `src/main.rs`:

```rust
mod geometry;
```

We are still setting up the foundation. Go ahead and create a new file `src/motherboards/display.rs`:

```rust
use crate::color;
use crate::geometry;
use crate::types::KernelResult;

pub fn display_buffer() -> KernelResult<impl DisplayBuffer> {
    unimplemented!()
}

pub trait DisplayBuffer {
    fn clear(&self, color: &color::Color);
    fn draw_rect(&self, p1: geometry::Point, p2: geometry::Point, color: &color::Color);
    fn draw_triangle(
        &self,
        p0: geometry::Point,
        p1: geometry::Point,
        p2: geometry::Point,
        color: &color::Color,
    );
    fn debug(&self);
}
```

Add the module to `src/motherboards/mod.rs`:

```rust
pub mod display;
```

Create another file `src/types.rs`:

```rust
pub enum KernelError {
    BufferInitError,
}

pub type KernelResult<T> = Result<T, KernelError>;
```

Add the module to `src/main.rs`:

```rust
mod types;
```

Create another file `src/motherboards/raspberrypi/hdmi.rs`:

```rust
use super::mailbox::{mbox_call, mbox_get, mbox_set};

use crate::color::Color;

#[cfg(feature = "device")]
const PHYSICAL_WIDTH: u32 = 3440;
#[cfg(feature = "device")]
const PHYSICAL_HEIGHT: u32 = 1440;

#[cfg(feature = "emulator")]
const PHYSICAL_WIDTH: u32 = 1500;
#[cfg(feature = "emulator")]
const PHYSICAL_HEIGHT: u32 = 900;

const MBOX_REQUEST: u32 = 0;
const MBOX_CH_PROP: u32 = 8;
const MBOX_TAG_LAST: u32 = 0;

struct FrameConfig {
    buffer: usize,
    size: usize,
    width: u32,
    height: u32,
    pitch: u32,
    depth: u32,
    bits_per_pixel: u32,
    is_rgb: u32,
}

impl FrameConfig {
    const fn new() -> Self {
        FrameConfig {
            buffer: 0,
            size: 0,
            width: 0,
            height: 0,
            pitch: 0,
            depth: 0,
            bits_per_pixel: 4,
            is_rgb: 0,
        }
    }
}

static mut FRAME_CONFIG: FrameConfig = FrameConfig::new();

pub struct ScreenDimensions {
    pub height: u32,
    pub width: u32,
}

pub fn get_screen_dimensions() -> ScreenDimensions {
    unsafe {
        ScreenDimensions {
            width: FRAME_CONFIG.width,
            height: FRAME_CONFIG.height,
        }
    }
}

const PHYSICAL_SIZE_TAG: u32 = 0x0004_8003;
const VIRTUAL_SIZE_TAG: u32 = 0x0004_8004;
const VIRTUAL_OFFSET_TAG: u32 = 0x0004_8009;
const DEPTH_TAG: u32 = 0x0004_8005;
const PIXEL_ORDER_TAG: u32 = 0x0004_8006;
const FRAME_BUFFER_TAG: u32 = 0x0004_0001;
const PITCH_TAG: u32 = 0x0004_0008;

fn init_buffer() -> bool {
    const BUFFER_SIZE: u32 = 35 * 4;

    mbox_set(0, BUFFER_SIZE);
    mbox_set(1, MBOX_REQUEST);

    mbox_set(2, PHYSICAL_SIZE_TAG);
    mbox_set(3, 8);
    mbox_set(4, 8);
    mbox_set(5, PHYSICAL_WIDTH);
    mbox_set(6, PHYSICAL_HEIGHT);

    mbox_set(7, VIRTUAL_SIZE_TAG);
    mbox_set(8, 8);
    mbox_set(9, 8);
    mbox_set(10, PHYSICAL_WIDTH);
    mbox_set(11, PHYSICAL_HEIGHT);

    mbox_set(12, VIRTUAL_OFFSET_TAG);
    mbox_set(13, 8);
    mbox_set(14, 8);
    mbox_set(15, 0);
    mbox_set(16, 0);

    mbox_set(17, DEPTH_TAG);
    mbox_set(18, 4);
    mbox_set(19, 4);
    mbox_set(20, 32);

    mbox_set(21, PIXEL_ORDER_TAG);
    mbox_set(22, 4);
    mbox_set(23, 4);
    mbox_set(24, 1); // 1 = RGB

    mbox_set(25, FRAME_BUFFER_TAG);
    mbox_set(26, 8);
    mbox_set(27, 8);
    mbox_set(28, 4096); // request 4096-byte alignment; returns frame buffer ptr here
    mbox_set(29, 0); //    returns size here

    mbox_set(30, PITCH_TAG);
    mbox_set(31, 4);
    mbox_set(32, 4);
    mbox_set(33, 0); // returns pitch here

    mbox_set(34, MBOX_TAG_LAST);

    if mbox_call(MBOX_CH_PROP) && mbox_get(20) == 32 && mbox_get(28) != 0 {
        unsafe {
            // Convert the GPU bus address to an ARM physical address and store the buffer
            FRAME_CONFIG.buffer = (mbox_get(28) & 0x3FFF_FFFF) as usize;
            FRAME_CONFIG.size = mbox_get(29) as usize;
            FRAME_CONFIG.width = mbox_get(5);
            FRAME_CONFIG.height = mbox_get(6);
            FRAME_CONFIG.pitch = mbox_get(33);
            FRAME_CONFIG.depth = mbox_get(20);
            FRAME_CONFIG.bits_per_pixel = if FRAME_CONFIG.width != 0 {
                FRAME_CONFIG.pitch / FRAME_CONFIG.width
            } else {
                4
            };
            FRAME_CONFIG.is_rgb = mbox_get(24);
        }
        true
    } else {
        false
    }
}

fn clear(color: &Color) {
    let (base, size) = unsafe { (FRAME_CONFIG.buffer, FRAME_CONFIG.size) };
    if base == 0 || size == 0 {
        return;
    }

    for i in 0..size / 4 {
        write_color(base + i * 4, color);
    }
}

/// Write a color to a single pixel, honoring the framebuffer's bits-per-pixel.
/// Coordinates outside the screen (including negative) are silently dropped.
fn draw_pixel(x: i32, y: i32, color: &Color) {
    unsafe {
        if FRAME_CONFIG.buffer == 0
            || x < 0
            || y < 0
            || (x as u32) >= FRAME_CONFIG.width
            || (y as u32) >= FRAME_CONFIG.height
        {
            return;
        }
        let x = x as u32;
        let y = y as u32;
        let addr = FRAME_CONFIG.buffer
            + (y * FRAME_CONFIG.pitch + x * FRAME_CONFIG.bits_per_pixel) as usize;
        write_color(addr, color);
    }
}

/// Fill a solid rectangle with the given color. The rect is clipped to the
/// screen, so off-screen (including negative) corners are fine.
fn draw_rect(x1: i32, y1: i32, x2: i32, y2: i32, color: &Color) {
    let (w, h) = unsafe { (FRAME_CONFIG.width as i32, FRAME_CONFIG.height as i32) };
    let xa = x1.min(x2).max(0);
    let xb = x1.max(x2).min(w - 1);
    let ya = y1.min(y2).max(0);
    let yb = y1.max(y2).min(h - 1);
    if xa > xb || ya > yb {
        return; // fully off-screen
    }
    for y in ya..=yb {
        for x in xa..=xb {
            draw_pixel(x, y, color);
        }
    }
}

// Signed area of the triangle (a, b, p); sign tells which side of edge a->b p is on.
fn edge(ax: i32, ay: i32, bx: i32, by: i32, px: i32, py: i32) -> i32 {
    (px - ax) * (by - ay) - (py - ay) * (bx - ax)
}

/// Fill a triangle given by three vertices, using half-space rasterization.
fn draw_triangle(x0: i32, y0: i32, x1: i32, y1: i32, x2: i32, y2: i32, color: &Color) {
    let (w, h) = unsafe { (FRAME_CONFIG.width as i32, FRAME_CONFIG.height as i32) };

    // Bounding box of the triangle, clamped to the screen.
    let min_x = x0.min(x1).min(x2).max(0);
    let min_y = y0.min(y1).min(y2).max(0);
    let max_x = x0.max(x1).max(x2).min(w - 1);
    let max_y = y0.max(y1).max(y2).min(h - 1);

    // Orientation of the whole triangle; a point is inside when all three edge
    // functions share this sign (covers both clockwise and counter-clockwise).
    let area = edge(x0, y0, x1, y1, x2, y2);
    if area == 0 {
        return;
    }

    for y in min_y..=max_y {
        for x in min_x..=max_x {
            let w0 = edge(x1, y1, x2, y2, x, y);
            let w1 = edge(x2, y2, x0, y0, x, y);
            let w2 = edge(x0, y0, x1, y1, x, y);
            let inside = if area > 0 {
                w0 >= 0 && w1 >= 0 && w2 >= 0
            } else {
                w0 <= 0 && w1 <= 0 && w2 <= 0
            };
            if inside {
                draw_pixel(x, y, color);
            }
        }
    }
}

fn glyph(c: u8) -> [u8; 8] {
    match c {
        b'0' => [0x70, 0x88, 0x98, 0xA8, 0xC8, 0x88, 0x70, 0x00],
        b'1' => [0x20, 0x60, 0x20, 0x20, 0x20, 0x20, 0x70, 0x00],
        b'2' => [0x70, 0x88, 0x08, 0x10, 0x20, 0x40, 0xF8, 0x00],
        b'3' => [0x70, 0x88, 0x08, 0x30, 0x08, 0x88, 0x70, 0x00],
        b'4' => [0x10, 0x30, 0x50, 0x90, 0xF8, 0x10, 0x10, 0x00],
        b'5' => [0xF8, 0x80, 0xF0, 0x08, 0x08, 0x88, 0x70, 0x00],
        b'6' => [0x30, 0x40, 0x80, 0xF0, 0x88, 0x88, 0x70, 0x00],
        b'7' => [0xF8, 0x08, 0x10, 0x20, 0x40, 0x40, 0x40, 0x00],
        b'8' => [0x70, 0x88, 0x88, 0x70, 0x88, 0x88, 0x70, 0x00],
        b'9' => [0x70, 0x88, 0x88, 0x78, 0x08, 0x10, 0x60, 0x00],
        b'A' => [0x70, 0x88, 0x88, 0xF8, 0x88, 0x88, 0x88, 0x00],
        b'B' => [0xF0, 0x88, 0x88, 0xF0, 0x88, 0x88, 0xF0, 0x00],
        b'C' => [0x70, 0x88, 0x80, 0x80, 0x80, 0x88, 0x70, 0x00],
        b'D' => [0xF0, 0x88, 0x88, 0x88, 0x88, 0x88, 0xF0, 0x00],
        b'E' => [0xF8, 0x80, 0x80, 0xF0, 0x80, 0x80, 0xF8, 0x00],
        b'F' => [0xF8, 0x80, 0x80, 0xF0, 0x80, 0x80, 0x80, 0x00],
        b'G' => [0x70, 0x88, 0x80, 0xB8, 0x88, 0x88, 0x70, 0x00],
        b'H' => [0x88, 0x88, 0x88, 0xF8, 0x88, 0x88, 0x88, 0x00],
        b'I' => [0x70, 0x20, 0x20, 0x20, 0x20, 0x20, 0x70, 0x00],
        b'J' => [0x38, 0x10, 0x10, 0x10, 0x90, 0x90, 0x60, 0x00],
        b'K' => [0x88, 0x90, 0xA0, 0xC0, 0xA0, 0x90, 0x88, 0x00],
        b'L' => [0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0xF8, 0x00],
        b'M' => [0x88, 0xD8, 0xA8, 0xA8, 0x88, 0x88, 0x88, 0x00],
        b'N' => [0x88, 0xC8, 0xA8, 0x98, 0x88, 0x88, 0x88, 0x00],
        b'O' => [0x70, 0x88, 0x88, 0x88, 0x88, 0x88, 0x70, 0x00],
        b'P' => [0xF0, 0x88, 0x88, 0xF0, 0x80, 0x80, 0x80, 0x00],
        b'Q' => [0x70, 0x88, 0x88, 0x88, 0xA8, 0x90, 0x68, 0x00],
        b'R' => [0xF0, 0x88, 0x88, 0xF0, 0xA0, 0x90, 0x88, 0x00],
        b'S' => [0x70, 0x88, 0x80, 0x70, 0x08, 0x88, 0x70, 0x00],
        b'T' => [0xF8, 0x20, 0x20, 0x20, 0x20, 0x20, 0x20, 0x00],
        b'U' => [0x88, 0x88, 0x88, 0x88, 0x88, 0x88, 0x70, 0x00],
        b'V' => [0x88, 0x88, 0x88, 0x88, 0x88, 0x50, 0x20, 0x00],
        b'W' => [0x88, 0x88, 0x88, 0xA8, 0xA8, 0xD8, 0x88, 0x00],
        b'X' => [0x88, 0x88, 0x50, 0x20, 0x50, 0x88, 0x88, 0x00],
        b'Y' => [0x88, 0x88, 0x50, 0x20, 0x20, 0x20, 0x20, 0x00],
        b'Z' => [0xF8, 0x08, 0x10, 0x20, 0x40, 0x80, 0xF8, 0x00],
        b':' => [0x00, 0x20, 0x20, 0x00, 0x00, 0x20, 0x20, 0x00],
        _ => [0x00; 8],
    }
}

/// Draw a text glyph to the screen based on a bitmap
pub fn draw_char(c: u8, px: u32, py: u32, scale: u32) {
    let g = glyph(c);
    for row in 0..8u32 {
        let bits = g[row as usize];
        for col in 0..8u32 {
            let pixel_color = if (bits >> (7 - col)) & 1 == 1 {
                Color::white()
            } else {
                Color::black()
            };
            for sy in 0..scale {
                for sx in 0..scale {
                    draw_pixel(
                        (px + col * scale + sx) as i32,
                        (py + row * scale + sy) as i32,
                        &pixel_color,
                    );
                }
            }
        }
    }
}

fn write_color(addr: usize, color: &Color) {
    unsafe {
        if FRAME_CONFIG.bits_per_pixel == 2 {
            core::ptr::write_volatile(addr as *mut u16, color.into());
        } else {
            core::ptr::write_volatile(addr as *mut u32, color.into());
        }
    }
}
```

And add the module to `src/motherboards/raspberrypi/mod.rs`:

```rust
pub mod hdmi;
```

Now for an indepth explanation of what is going on:

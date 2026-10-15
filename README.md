# u8g2 for sysl

Drawing and text on small monochrome displays (the SSD1306 and SH1106 OLED modules of a
microcontroller kit, and about a hundred other controllers), bound to
[u8g2](https://github.com/olikraus/u8g2) 2.37.1, vendored unchanged.

*An independent sysl package for u8g2. It is not affiliated with or endorsed by u8g2's authors or trademark holders.*

- **Lines, boxes, circles, discs, ellipses, arcs, triangles and bitmaps**, drawn into a frame buffer in
  RAM and sent to the display with one call.
- **Text in hundreds of fonts**, from UTF-8, in four directions.
- **You pay only for what you use.** u8g2 carries hundreds of fonts and a hundred display drivers; a
  program keeps the one font and the one driver it names (measured below).
- **No heap.** The display is a value; its frame buffer is u8g2's own static array.

```hocon
dependencies {
  u8g2 { git = "github.com/sysl-lang/u8g2", version = "<the first tag>" }
}
```

The module is `sh.sysl.u8g2`; a few fonts are in `sh.sysl.u8g2.fonts`.

## Using it

A board implements `Bus`: start a transfer to an address, write bytes, stop, and wait some
milliseconds. That is all an I2C display needs. Here is an SSD1306 on I2C:

```sysl
import sh.sysl.u8g2.{Bus, ssd1306_i2c_128x64}
import sh.sysl.u8g2.fonts.font_6x10

// The board's I2C peripheral, whatever its library calls it.
struct OledI2c
    port: I2cPort

impl Bus for OledI2c
    start(*self, address: u8) = self.port.begin_write(address)
    write(*self, bytes: []const u8) = self.port.write(bytes)
    stop(*self) = self.port.end_write()
    delay_ms(*self, ms: u32) = sleep_ms(ms)

var bus = OledI2c(i2c0())
var oled = ssd1306_i2c_128x64(&bus)
oled.begin()

oled.set_font(&font_6x10)
oled.text(0, 10, "Hello!")
oled.frame(0, 16, 128, 48)
oled.disc(64, 40, 12)
oled.send()
```

`begin` resets the display, sends its start-up commands and switches it on. Drawing changes the frame
buffer only; `send` puts the whole picture on the display. `clear` empties the buffer.

On SPI a bus also gets `data_mode(data)` for the D/C line, and `reset(high)` where a reset pin is
wired. `start` and `stop` select and release the chip. `init`, `data_mode`, `reset` and `delay_us`
have bodies that do nothing, so an I2C bus writes the four above and no more.

**A display that has begun must stay where it is**, because u8g2 keeps pointers into its own state and
the binding hands u8g2 the bus's address. A copy taken after `begin` is a display u8g2 knows nothing
about. (sysl's coming `@pinned` will refuse such a copy.)

**On a bare board today, keep the bus and the display in locals of the entry function**, not in module
storage: sysl cannot yet lay a value holding an address into a freestanding image (see "Bare metal"
below).

| | |
|---|---|
| displays | `ssd1306_i2c_128x64`, `ssd1306_spi_128x64`, `ssd1306_i2c_128x32`, `sh1106_i2c_128x64`, `sh1106_spi_128x64`, `ssd1306_i2c_128x64_paged`, `null_display`, and `display(bus, setup)` for any other |
| the display | `begin`, `configure`, `sleep`, `set_contrast`, `set_rotation` (`R0`, `R90`, `R180`, `R270`, `Mirror`, `MirrorVertical`), `set_i2c_address`, `i2c_address` |
| the buffer | `clear`, `send`, `clear_display`, `first_page`/`next_page`, `buffer`, `width`, `height`, `tile_width`, `tile_height`, `buffer_tile_height` |
| drawing | `set_color` (`On`, `Off`, `Invert`), `pixel`, `hline`, `vline`, `line`, `box`, `frame`, `rounded_box`, `rounded_frame`, `circle`, `disc`, `ellipse`, `filled_ellipse` (with `UPPER_RIGHT`, `UPPER_LEFT`, `LOWER_LEFT`, `LOWER_RIGHT`, `ALL`), `arc`, `triangle`, `bitmap` (XBM), `transparent_bitmaps`, `clip`, `unclip` |
| text | `set_font`, `text`, `text_width`, `glyph`, `has_glyph`, `set_direction`, `set_anchor`, `transparent_text`, `ascent`, `descent`, `max_char_width`, `max_char_height` |

Coordinates are `int`. Anything off the display is clipped, including negative positions, so text can
scroll in from the left. `text` takes UTF-8 and answers how far it advanced; `text_width` is u8g2's
measure, which ends at the last column the last letter inks (one less than the advance for most
fonts). A string longer than `MAX_TEXT` (255) bytes is drawn up to the last whole character that fits;
a NUL byte ends it, as it does in C.

### Any other display

Every one of u8g2's setup functions is reached with one `extern` in the program:

```sysl
import sh.sysl.u8g2.c.{Msg, Rotation, U8g2}

extern "u8g2_Setup_ssd1309_i2c_128x64_noname0_f" setup_ssd1309(u8g2: *U8g2, rotation: *Rotation, byte_cb: Msg, gpio_cb: Msg)

var oled = display(&bus, &setup_ssd1309)
```

The names are in [u8g2's setup list](https://github.com/olikraus/u8g2/wiki/u8g2setupc): a `_f` setup
keeps the whole picture in RAM; `_1` and `_2` keep one or two pages, and the picture is drawn once per
page between `first_page` and `next_page`.

## Fonts

A font is an array in u8g2's C. A program names the one it wants with an `extern` of the type `Font`
and hands `set_font` its address:

```sysl
import sh.sysl.u8g2.Font

extern "u8g2_font_courB18_tf" courier: Font

oled.set_font(&courier)
```

Any of the fonts in [u8g2's font list](https://github.com/olikraus/u8g2/wiki/fntlistall) works this
way. `sh.sysl.u8g2.fonts` declares six so a first program needs no `extern`: `font_5x7`, `font_6x10`,
`font_7x13_bold`, `font_helvetica_bold_10`, `font_digits_24` and `font_symbols` (Unifont's arrows,
suits and weather symbols).

**Why this way.** Declaring every font in the package would be hundreds of lines nobody reads, and
choosing a curated set alone would shut out the rest. An `extern` per font costs one line, names the
font exactly as u8g2's documentation does, and is typed: `set_font` takes a `*Font`, not any pointer.
Naming a font costs nothing until a program takes its address; the linker drops the rest, so the six
in `fonts` are free. The one thing the form cannot say is that the array is `const`: an `extern`
variable is writable in sysl, and writing one would fault on flash.

## What a program pays for

u8g2's fonts are in one 40 MB C file and every display's setup in one 460 KB file, so this was
measured. sysl compiles a package's C with `-ffunction-sections -fdata-sections` and links with
`-dead_strip` (macOS) or `--gc-sections`, so each font, driver and setup is its own section and the
linker keeps only what is reached.

A program that starts an SSD1306 on I2C, sets `font_6x10`, draws a string and a box and sends the
buffer:

| build | text | read-only data | bss |
|---|---:|---:|---:|
| host, `sysl build` (arm64 macOS) | 11,136 | 2,784 | 66,584 (64 KB of it the runtime's signal stack; 1,024 the frame buffer) |
| Cortex-M33, the same C with `arm-none-eabi-gcc -Os`, `--gc-sections` | 6,589 (code and data) | | 1,624 |
| Cortex-M33, the same, without `--gc-sections` | 14,676,192 | | 275,916 |

The image keeps `u8g2_font_6x10_tf` and no other font, `u8g2_Setup_ssd1306_i2c_128x64_noname_f` and
no other setup, and the one SSD1306 driver. Without section garbage collection every font and driver
would land in flash: 14 MB, against the Pico 2's 4 MB.

The Cortex-M rows were built from the vendored C directly, because `sysl build-c` for
`thumb-freestanding` does not yet get that far (see "Bare metal").

## What it is compiled with, and what is left out

The package compiles u8g2 with one option, `U8X8_WITH_USER_PTR`, which gives `u8x8_t` the pointer the
binding's callbacks use to find their bus. It changes the layout of `u8g2_t`, so every carried file
gets it, and the offsets sysl reads are asked of the header under the same option.

Bound: the full-buffer and page-buffer drawing API above, fonts, the display controls, and setups for
the kit's displays. Deliberately not bound: u8g2's menu and message-box helpers (`u8g2_UserInterface*`,
`mui`), `u8log`, the `u8x8` character-only API, `DrawStr` (8-bit encodings; `text` takes UTF-8),
kerning tables, button frames, polygon building, `SetFontRefHeight*`, and u8g2's own software I2C and
SPI byte procedures (a sysl `Bus` does the transport). Each is an addition when something needs it.

**Two displays with the same setup share one frame buffer**, because a `_f` setup's buffer is a static
array inside u8g2 (`u8g2_m_16_8_f` and the like), one per buffer size. Two different setups, such as a
128x64 and a 128x32, each have their own.

## Bare metal

The package needs no operating system and no allocator, but today it does not build for a freestanding
target:

- `sysl build-lib . --target thumb-freestanding` stops at `sysl: error: cannot read
  …/sh/sysl/u8g2/c/u8g2_fonts.c`: the compiler reads carried C as UTF-8 text, and u8g2's two font files
  each have one Latin-1 byte in a comment. This is a compiler bug, reported.
- Past that, `sysl build-c` of a program stops at `mui.h:79:10: fatal error: 'string.h' file not
  found`: a package's C cannot yet be given `sysl-lang/cstd`'s headers. Accepted for alpha.14.
- A display in module storage cannot be laid into a freestanding image, because its initializer holds
  the address of its bus and of a setup function. Reported; meanwhile keep both in locals of the entry
  function.

A target with a C library (the Pico SDK's newlib, Zephyr, FreeRTOS) has the headers already.

## Tests

`sysl test .` runs 23 tests.

- **Against u8g2 driven from C.** A C program written straight against `u8g2.h`, built with clang from
  the same vendored source with the same option, sets up an SSD1306 128x64 on I2C, starts it, and draws
  one scene: text in two fonts, UTF-8 (`Grüße 25°C`), a frame, a box, a line, a circle, discs (one off
  the left edge, one in two quarters), a rounded frame, an ellipse, an arc, a triangle off two edges,
  an XBM bitmap, an inverted box over the edge, a glyph, text running off the right edge and text
  drawn downwards. The binding draws the same scene. **The two frame buffers must be the same 1,024
  bytes**, and **everything the two callbacks were asked to do must be the same, in order**: for
  `begin` and for one `send`, every transfer with its address and bytes, every reset level and every
  delay.
- Pixels land where u8g2's layout puts them; boxes, frames, lines, circles and discs set exactly their
  pixels; drawing is clipped at all four edges and at negative coordinates, and by a clip window;
  `Off` and `Invert`; text widths in two fonts; a UTF-8 character draws the glyph it encodes;
  `has_glyph`; text clipped at the edges; long text cut at a character boundary and a NUL ending it;
  anchors; direction; all six rotations; bitmaps, and one too short refused; `begin`'s reset, waits,
  address and commands; `send` with a changed address; two displays on two buses through the user
  pointer; a paged display's eight pages; the null display.

Mutations, each run and each red: the last byte of every transfer dropped (5 tests red); `R0` drawing
with the `R180` table (11); `font_6x10` naming `u8g2_font_6x12_tf` (2); every coordinate off by one
(13); the user pointer aimed at the display instead of the bus (6); the I2C address handed over
unshifted (3); the reset line inverted (2, after the `begin` test was made to check the reset levels;
before, only the oracle saw it); `On` and `Off` swapped (2).

**AddressSanitizer**: `SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .` passes 23/23. The test
binary links 10 `__asan_report` hooks, and u8g2's own C is instrumented too (`u8g2_font_decode_glyph`
reads the shadow memory).

## Vendored

`sh/sysl/u8g2/c/` holds every file of `csrc/` from
[olikraus/u8g2](https://github.com/olikraus/u8g2) release 2.37.1, unchanged, from the tarball
`https://github.com/olikraus/u8g2/archive/refs/tags/2.37.1.tar.gz` whose SHA-256 is
`16e63a2e131550a8e22a26bfbb4730e0015de5c9397532d8852a1659441c97ec`. Vendoring means upstream fixes do
not arrive on their own: watch u8g2's releases and re-vendor (and add any new `.c` to the `defines`
key in `package.hocon`).

Licensing: u8g2's code is BSD-2-Clause, and each font carries the license of the font it was converted
from ([u8g2's font groups](https://github.com/olikraus/u8g2/wiki/fntgrp)); u8g2's LICENSE file is
carried verbatim in `LICENSE`. The rest of this package is ISC.

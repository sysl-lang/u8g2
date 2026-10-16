# What vendoring u8g2 found in sysl

Found against sysl 0.1.0-alpha.12; every finding is fixed as of sysl 0.1.1. Nothing was worked around in the package.

## 1. `build-lib` refuses carried C that is not UTF-8 (bug)

`u8g2_fonts.c` and `u8x8_fonts.c` each carry one Latin-1 byte in a copyright comment. `sysl test .`
works; `sysl build-lib .`, and with it every freestanding build, fails:

```
sysl: error: cannot read …/sh/sysl/u8g2/c/u8g2_fonts.c
```

The shortest input is a package whose `seven.c` is `/* Copyright: oshimo<0xFF>o */ int seven(void) {
return 7; }`. The compiler reads every carried `.c` as text (`std_artifact.sysl`, and the same pattern in
`run_cache.sysl`); C source is bytes, so it should be read and hashed as bytes. The workaround not
taken: transcoding upstream's files.

**Status: Fixed in sysl 0.1.x; workaround removed.** (`build-lib` of this package now gets past the font files; what remains is the next section.)

## 2. Module storage whose initializer holds an address cannot go into a freestanding image (gap)

```sysl
var bus: Quiet = Quiet(0)
var oled: Display[Quiet] = ssd1306_i2c_128x64(&bus)
```

```
'oled' cannot be computed while compiling: 'TAddrOf' is not evaluated while compiling yet
```

Making the constructor `const` is refused, because it takes a `*extern` setup function, although it
only stores it. C writes `static T x = { &bus, setup };` and gets both addresses as link-time
relocations. Possibly the same root as skiff's FINDINGS 6 (accepted for alpha.13). Meanwhile a
bare-metal program keeps its bus and display as locals of its entry function, as the README says. The
workaround not taken: an API of `begin(&bus)` plus a setup enum.

**Status: Fixed in sysl 0.1.x; workaround removed.** The display constructors are `const`, and a display declared as module storage is laid into a thumbv7em-freestanding image as data (`sysl emit-llvm --target thumbv7em-freestanding` shows `@oled = { ..., ptr @bus, ptr @u8g2_Setup_..., ... }`); `display_in_module_storage` in the tests keeps the shape.

## 3. `unit` has no `Eq` (gap)

`oled.bitmap(…) == Err(Error.BitmapTooShort)` is refused: `'==' between sysl.Result[unit,
sh.sysl.u8g2.Error] needs 'sysl.Eq' … unit, which does not implement it`. The tests compare `.err()`.
The same as `sysl-lang/fat`'s FINDINGS 5.

**Status: Fixed in sysl 0.1.x; workaround removed.** The tests compare the `Result` directly.

## Also blocking bare metal

Past finding 1, `build-c` for `thumb-freestanding` stopped at `mui.h:79:10: fatal error: 'string.h' file
not found`: a package's C could not use `sysl-lang/cstd`'s headers (littlefs's FINDINGS 1).

**Status: Fixed in sysl 0.1.2.** `cstd { ..., if = "!libc" }` gives the C cstd's headers on a board; the
suite builds the package for `thumbv7em-freestanding` and `riscv32-freestanding`.

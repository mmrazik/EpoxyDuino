# EpoxyDuino

[![AUnit Tests](https://github.com/bxparks/EpoxyDuino/actions/workflows/aunit_tests.yml/badge.svg)](https://github.com/bxparks/EpoxyDuino/actions/workflows/aunit_tests.yml)

This project contains a small (but often effective) implementation of the
Arduino programming framework for Linux, MacOS, FreeBSD (experimental) and
potentially other POSIX-like systems. Originally, it was created to allow
[AUnit](https://github.com/bxparks/AUnit) unit tests to be compiled and run on a
desktop class machine, instead of running on the embedded microcontroller. As
more Arduino functionality was added, I found it useful for doing certain types
of application development on my Linux laptop, especially the parts that were
more algorithmic instead of hardware dependent. EpoxyDuino can be effectively
used in Continuous Integration (CI) pipeline (like
[GitHub Actions](https://github.com/features/actions)) for automatically
validating that a library or application compiles without errors.

The build process uses [GNU Make](https://www.gnu.org/software/make/manual/).
A simple `Makefile` needs to be created inside the sketch folder. For example,
if the sketch is `SampleTest/SampleTest.ino`, then the makefile should be
`SampleTest/Makefile`. The sketch is compiled with just a `make` command. It
produces an executable with a `.out` extension, for example, `SampleTest.out`.

Most hardware dependent functions are stubbed out (defined but don't do
anything) to allow the Arduino programs to compile. This may be sufficient for a
CI pipeline. For actual application development, I have started to build
a set of libraries within EpoxyDuino which emulate the versions that run the
actual hardware:

* EpoxyFS: emulation of the ESP8266 LittleFS or ESP32 LITTLEFS
* EpoxyPromAvr: emulation of AVR-flavored `EEPROM`
* EpoxyPromEsp: emulation of ESP-flavored `EEPROM`

If your program has limited hardware dependencies so that it is conceptually
portable to a vanilla Unix environment, EpoxyDuino may work well for you.

Running an Arduino program natively on a desktop-class machine has some
advantages:

* The development cycle can be lot faster because the compilers on the the
  desktop machines are a lot faster, and we also avoid the upload and flash
  process to the microcontroller.
* The desktop machine can run unit tests which require too much flash or too
  much memory to fit inside an embedded microcontroller.
* It may help you write platform-independent code, because if it runs under
  EpoxyDuino, it has a good chance of running on most Arduino-compatible
  platforms.

The disadvantages are:

* Only a subset of Arduino functions are supported (see below).
* Many 3rd party libraries will not compile under EpoxyDuino.
* There may be compiler differences between the desktop and the embedded
  environments (e.g. 16-bit `int` versus 32-bit `int`, or 32-bit `long` versus
  64-bit `long`).

**Version**: 0.6.2 (2021-03-15)

**Changelog**: See [CHANGELOG.md](CHANGELOG.md)

**Breaking Change**: Prior to v0.5, this project was known as "UnixHostDuino".
The old `UNIX_HOST_DUINO` macro and `UnixHostDuino.mk` include file still exist
for backwards compatibility. See
[Issue #15](https://github.com/bxparks/EpoxyDuino/issues/15)
for more details.

## Table of Contents

* [Installation](#Installation)
* [Usage](#Usage)
    * [Makefile](#Makefile)
    * [Additional Arduino Libraries](#AdditionalLibraries)
    * [Additional Arduino Library Locations](#AdditionalLibraryLocations)
    * [Alternate C++ Compiler](#AlternateCompiler)
    * [Difference from Arduino IDE](#DifferenceFromArduinoIDE)
    * [Conditional Code](#ConditionalCode)
    * [Continuous Integration](#ContinuousIntegration)
* [Supported Arduino Features](#SupportedArduinoFeatures)
    * [Arduino Functions](#ArduinoFunctions)
    * [Built-in Libraries](#BuiltInLibraries)
    * [Serial Port Emulation](#SerialPortEmulation)
* [System Requirements](#SystemRequirements)
* [License](#License)
* [Feedback](#Feedback)
* [Authors](#Authors)

<a name="Installation"></a>
## Installation

You need to grab the sources directly from GitHub. This project is *not* an
Arduino library so it is not available through the [Arduino Library
Manager](https://www.arduino.cc/en/guide/libraries) in the Arduino IDE.

The location of the EpoxyDuino directory can be arbitrary, but a convenient
location might be the same `./libraries/` directory used by the Arduino IDE to
store other Arduino libraries:

```
$ cd {sketchbook_directory}/libraries
$ git clone https://github.com/bxparks/EpoxyDuino.git
```

This will create a directory called
`{sketchbook_directory}/libraries/EpoxyDuino`.

### Dependencies

The core of EpoxyDuino depends on:

* a C++ compiler (`g++` or `clang++`)
* GNU Make (usually `make` but sometimes `gmake`)

These are normally installed on the host OS by default.

The example and test code under `./tests/`, `./examples/`,
`./libraries/*/tests/`, and `./libraries/*/examples/` depend on:

* AUnit (https://github.com/bxparks/AUnit)
* AceCRC (https://github.com/bxparks/AceCRC)
* AceCommon (https://github.com/bxparks/AceCommon)
* AceRoutine (https://github.com/bxparks/AceRoutine)
* AceUtils (https://github.com/bxparks/AceUtils)

<a name="Usage"></a>
## Usage

<a name="Makefile"></a>
### Makefile

The minimal `Makefile` has 3 lines:
```
APP_NAME := {name of project}
ARDUINO_LIBS := {list of dependent Arduino libraries}
include {path/to/EpoxyDuino.mk}
```

For example, the [examples/BlinkSOS](examples/BlinkSOS) project contains this
Makefile:
```
APP_NAME := BlinkSOS
ARDUINO_LIBS :=
include ../../../EpoxyDuino/EpoxyDuino.mk
```

To build the program, just run `make`:
```
$ cd examples/BlinkSOS
$ make clean
$ make
```

The executable will be created with a `.out` extension. To run it, just type:
```
$ ./BlinkSOS.out
```

The output that would normally be printed on the `Serial` on an Arduino
board will be sent to the `STDOUT` of the Linux, MacOS, or FreeBSD terminal. The
output should be identical to what would be shown on the serial port of the
Arduino controller.

<a name="AdditionalLibraries"></a>
### Additional Arduino Libraries

If the Arduino program depends on additional Arduino libraries, they must be
specified in the `Makefile` using the `ARDUINO_LIBS` parameter. For example,
this includes the [AUnit](https://github.com/bxparks/AUnit) library if it is at
the same level as EpoxyDuino:

```
APP_NAME := SampleTest
ARDUINO_LIBS := AUnit AceButton AceTime
include ../../EpoxyDuino/EpoxyDuino.mk
```

The libraries are referred to by their base directory name (e.g. `AceButton`,
or `AceTime`) not the full path. By default, the `EpoxyDuino.mk` file will look
for these additional libraries at the following locations:

* `EPOXY_DUINO_DIR/..` - in other words, siblings to the `EpoxyDuino` install
  directory (this assumes that EpoxyDuino was installed in the Arduino
  `libraries` directory as recommended above)
* `EPOXY_DUINO_DIR/libraries/` - additional libraries provided by the EpoxyDuino
  project itself
* under each of the additional directories listed in `ARDUINO_LIB_DIRS` (see
  below)

<a name="AdditionalLibraryLocations"></a>
### Additional Arduino Library Locations

As explained above, EpoxyDuino normally assumes that the additional libraries
are siblings to the`EpoxyDuino/` directory or under the `EpoxyDuino/libraries/`
directory. If you need to import additional Arduino libraries, you need to tell
`EpoxyDuino` where they are because Arduino libraries tend to be scattered among
many different locations. These additional locations can be specified using the
`ARDUINO_LIB_DIRS` variable. For example,

```
APP_NAME := SampleTest
arduino_ide_dir := ../../arduino-1.8.9
ARDUINO_LIBS := AUnit AceButton AceTime
ARDUINO_LIB_DIRS := \
	$(arduino_ide_dir)/portable/packages/arduino/hardware/avr/1.8.2/libraries \
	$(arduino_ide_dir)/libraries \
	$(arduino_ide_dir)/hardware/arduino/avr/libraries
include ../../EpoxyDuino/EpoxyDuino.mk
```

Each of the `AUnit`, `AceButton` and `AceTime` libraries will be searched in
each of the 3 directories given in the `ARDUINO_LIB_DIRS`. (The
`arduino_ide_dir` is a convenience temporary variable. It has no significance to
`EpoxyDuino.mk`)

<a name="AlternateCompiler"></a>
### Alternate C++ Compiler

Normally the C++ compiler on Linux is `g++`. If you have `clang++` installed
you can use that instead by specifying the `CXX` environment variable:
```
$ CXX=clang++ make
```
(This sets the `CXX` shell environment variable temporarily, for the duration of
the `make` command, which causes `make` to set its internal `CXX` variable,
which causes `EpoxyDuino.mk` to use `clang++` over the default `g++`.)

<a name="DifferenceFromArduinoIDE"></a>
### Difference from Arduino IDE

There are a number of differences compared to the programming environment
provided by the Arduino IDE:

* The `*.ino` file is treated like a normal `*.cpp` file. So it must have
  an `#include <Arduino.h>` include line at the top of the file. This is
  compatible with the Arduino IDE which automatically includes `<Arduino.h>`.
* The Arduino IDE supports multiple `ino` files in the same directory. (I
  believe it simply concontenates them all into a single file.) EpoxyDuino
  supports only one `ino` file in a given directory.
* The Arduino IDE automatically generates forward declarations for functions
  that appear *after* the global `setup()` and `loop()` methods. In a normal
  C++ file, these forward declarations must be created by hand. The other
  alternative is to move `loop()` and `setup()` functions to the end of the
  `ino` file.

Fortunately, the changes required to make an `ino` file compatible with
EpoxyDuino are backwards compatible with the Arduino IDE. In other words, a
program that compiles with EpoxyDuino will also compile under Ardunio IDE.

There are other substantial differences. The Arduino IDE supports multiple
microcontroller board types, each using its own set of compiler tools and
library locations. There is a complicated set of files and rules that determine
how to find and use those tools and libraries. The EpoxyDuino tool does *not*
use any of the configuration files used by the Arduino IDE. Sometimes, you can
use the `ARDUINO_LIB_DIRS` to get around this limitations. However, when you
start using `ARDUINO_LIB_DIRS`, you will often run into third party libraries
using features which are not supported by the EpoxyDuino framework emulation
layer.

<a name="ConditionalCode"></a>
### Conditional Code

If you want to add code that takes effect only on EpoxyDuino, you can use
the following macro:
```C++
#if defined(EPOXY_DUINO)
  ...
#endif
```

If you need to target a particular desktop OS, you can use the following:

**Linux**:
```C++
#if defined(__linux__)
  ...
#endif
```

**MacOS**:
```C++
#if defined(__APPLE__)
  ...
#endif
```

**FreeBSD**:
```C++
#if defined(__FreeBSD__)
  ...
#endif
```

<a name="ContinuousIntegration"></a>
### Continuous Integration

You can use EpoxyDuino to run continuous integration tests or
validations on the [GitHub Actions](https://github.com/features/actions)
infrastructure. The basic `ubuntu-18.04` docker image already contains the C++
compiler and `make` binary. You don't need to install the Arduino IDE or the
Arduino CLI. You need:

* EpoxyDuino,
* your project that you want to test,
* any additional Arduino libraries that you use.

Take a look at some of my GitHub Actions YAML config files:

* https://github.com/bxparks/AceButton/tree/develop/.github/workflows
* https://github.com/bxparks/AceCRC/tree/develop/.github/workflows
* https://github.com/bxparks/AceCommon/tree/develop/.github/workflows
* https://github.com/bxparks/AceRoutine/tree/develop/.github/workflows
* https://github.com/bxparks/AceTime/tree/develop/.github/workflows

<a name="SupportedArduinoFeatures"></a>
## Supported Arduino Features

<a name="ArduinoFunctions"></a>
### Arduino Functions

The following functions and features of the Arduino framework are implemented:

* `Arduino.h`
    * `setup()`, `loop()`
    * `delay()`, `yield()`, `delayMicroSeconds()`
    * `millis()`, `micros()`
    * `digitalWrite()`, `digitalRead()`, `pinMode()` (empty stubs)
    * `analogRead()`, `analogWrite()` (empty stubs)
    * `pulseIn()`, `pulseInLong()`, `shiftIn()`, `shiftOut()` (empty stubs)
    * `HIGH`, `LOW`, `INPUT`, `OUTPUT`, `INPUT_PULLUP`
    * I2C and SPI pins: SS, MOSI, MISO, SCK, SDA, SCL
    * typedefs: `boolean`, `byte`, `word`
* `StdioSerial.h`
    * `Serial.print()`, `Serial.println()`, `Serial.write()`
    * `Serial.read()`, `Serial.peek()`, `Serial.available()`
    * `SERIAL_PORT_MONITOR`
* `WString.h`
    * `class String`
    * `class __FlashStringHelper`, `F()`, `FPSTR()`
* `Print.h`
    * `class Print`, `class Printable`
    * `Print.printf()` - extended function supported by some Arduino compatible
      microcontrollers
* `pgmspace.h`
    * `pgm_read_byte()`, `pgm_read_word()`, `pgm_read_dword()`,
      `pgm_read_float()`, `pgm_read_ptr()`
    * `strlen_P()`, `strcat_P()`, `strcpy_P()`, `strncpy_P()`, `strcmp_P()`,
      `strncmp_P()`, `strcasecmp_P()`, `strchr_P()`, `strrchr_P()`
    * `PROGMEM`, `PGM_P`, `PGM_VOID_P`, `PSTR()`
* `Wire.h` (stub implementation)
* `SPI.h` (stub implementation)

See [Arduino.h](https://github.com/bxparks/EpoxyDuino/blob/develop/Arduino.h)
for the latest list. Most of the header files included by this `Arduino.h`
file were copied and modified from the [arduino:avr
core](https://github.com/arduino/ArduinoCore-avr/tree/master/cores/arduino),
versions 1.8.2 (if I recall) or 1.8.3. A number of tweaks have been made to
support slight variations in the API of other platforms, particularly the
ESP8266 and ESP32 cores.

The `Print.printf()` function is an extension to the `Print` class that is
provided by many Arduino-compatible microcontrollers (but not the AVR
controllers). It is implemented here for convenience. The size of the internal
buffer is `250` characters, which can be changed by changing the
`PRINTF_BUFFER_SIZE` parameter if needed.

<a name="BuiltInLibraries"></a>
### Built-in Libraries

The following libraries are built-in to the EpoxyDuino project as substitutes
for the equivalent libraries on various Cores:

* [libraries/EpoxyFS](libraries/EpoxyFS)
    * An implementation of a file system compatible with
      [ESP8266 LittleFS](https://arduino-esp8266.readthedocs.io/en/latest/filesystem.html)
      and [ESP32 LITTLEFS](https://github.com/lorol/LITTLEFS).
* Two EEPROM implementations:
    * [libraries/EpoxyPromAvr](libraries/EpoxyPromAvr)
        * API compatible with
          [EEPROM on AVR](https://github.com/arduino/ArduinoCore-avr/tree/master/libraries/EEPROM)
    * [libraries/EpoxyPromEsp](libraries/EpoxyPromEsp)
        * API compatible with
          [EEPROM on ESP8266](https://github.com/esp8266/Arduino/tree/master/libraries/EEPROM)
          and
          [EEPROM on ESP32](https://github.com/espressif/arduino-esp32/tree/master/libraries/EEPROM)

I hope to create additional emulations of various network libraries (HTTP
client, HTTP Server, MQTT client, etc) so that even more of the Arduino
development can be done on the Linux/MacOS host.

<a name="SerialPortEmulation"></a>
### Serial Port Emulation

The `Serial` object is an instance of the `StdioSerial` class which emulates the
Serial port using the `STDIN` and `STDOUT` of the Unix system. `Serial.print()`
sends the output to the `STDOUT` and `Serial.read()` reads from the `STDIN`.

The interaction with the Unix `tty` device is complicated, and I am not entirely
sure that I have implemented things properly. See [Entering raw
mode](https://viewsourcecode.org/snaptoken/kilo/02.enteringRawMode.html) for
in-depth details. The following is a quick summary of how this is implemented
under `EpoxyDuino`.

The `STDOUT` remains mostly in normal mode. In particular, `ONLCR` mode is
enabled, which translates `\n` (NL) to `\r\n` (CR-NL). This allows the program
to print a line of string terminating in just `\n` (e.g. in a `printf()`
function) and the Unix `tty` device will automatically add the `\r` (CR) to
start the next line at the left. (Interestingly, the `Print.println()` method
prints `\r\n`, which gets translated into `\r\r\n` by the terminal, which still
does the correct thing. The extra `\r` does not do any harm.)

The `STDIN` is put into "raw" mode to avoid blocking the `loop()` function while
waiting for input from the keyboard. It also allows `ICRNL` and `INLCR` which
flips the mapping of `\r` and `\n` from the keyboard. That's because normally,
the "Enter" or "Return" key transmits a `\r`, but internally, most string
processing code wants to see a line terminated by `\n` instead. This is
convenient because when the `\n` is printed back to the screen, it becomes
translated into `\r\n`, which is what most people expect is the correct
behavior.

The `ISIG` option on the `tty` device is *enabled*. This allows the usual Unix
signals to be active, such as Ctrl-C to quit the program, or Ctrl-Z to suspend
the program. But this convenience means that the Arduino program running under
`EpoxyDuino` will never receive a control character through the
`Serial.read()` function. The advantages of having normal Unix signals seemed
worth the trade-off.

<a name="SystemRequirements"></a>
## System Requirements

This library has been tested on:

* Ubuntu 18.04
    * g++ (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0
    * clang++ 8.0.0-3~ubuntu18.04.2
    * clang++ 6.0.0-1ubuntu2
    * GNU Make 4.1
* Ubuntu 20.04
    * g++ (Ubuntu 9.3.0-10ubuntu2) 9.3.0
    * clang++ version 10.0.0-4ubuntu1
    * GNU Make 4.2.1
* Raspbian GNU/Linux 10 (buster)
    * On Raspberry Pi Model 3B
    * g++ (Raspbian 8.3.0-6+rpi1) 8.3.0
    * GNU Make 4.2.1
* MacOS 10.14.5 (Mojave)
    * clang++ Apple LLVM version 10.0.1
    * GNU Make 3.81
* MacOS 10.14.6 (Mojave)
    * Apple clang version 11.0.0 (clang-1100.0.33.17)
    * GNU Make 3.81
* FreeBSD 12.2 (Experimental)
    * c++: FreeBSD clang version 10.0.1
    * gmake: GNU Make 4.3
        * Install using `$ pkg install gmake`
        * You can type `gmake` instead of `make`, or
        * Create a shell alias, or
        * Create a symlink in `~/bin`.

<a name="License"></a>
## License

[MIT License](https://opensource.org/licenses/MIT)

<a name="Bugs"></a>
## Bugs and Limitations

If the executable (e.g. `SampleTest.out`) is piped to the `less(1)` or `more(1)`
command, sometimes (not all the time) the executable hangs and displays nothing
on the pager program. I don't know why, it probably has to do with the way that
the `less` or `more` programs manipulate the `stdin`. The solution is to
explicitly redirect the `stdin`:

```
$ ./SampleTest.out | grep failed # works

$ ./SampleTest.out | less # hangs

$ ./SampleTest.out < /dev/null | less # works
```

<a name="Feedback"></a>
## Feedback and Support

If you find this library useful, consider starring this project on GitHub. The
stars will let me prioritize the more popular libraries over the less popular
ones.

If you have any questions, comments, bug reports, or feature requests, please
file a GitHub ticket instead of emailing me unless the content is sensitive.
(The problem with email is that I cannot reference the email conversation when
other people ask similar questions later.) I'd love to hear about how this
software and its documentation can be improved. I can't promise that I will
incorporate everything, but I will give your ideas serious consideration.

<a name="Authors"></a>
## Authors

* Created by Brian T. Park (brian@xparks.net).
* Support for using as library, by making `main()` a weak reference, added
  by Max Prokhorov (prokhorov.max@outlook.com).
* Add `delayMicroSeconds()`, `WCharacter.h`, and stub implementations of
  `IPAddress.h`, `SPI.h`, by Erik Tideman (@ramboerik), see
  [PR #18](https://github.com/bxparks/EpoxyDuino/pull/18).

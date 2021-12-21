# TBoot

**A self-contained bootloader for MC68HCS08 (a.k.a 9S08) MCUs**

No external application is required.  Firmware is sent as plain S19 (or S28 for MMU based MCUs)
ASCII file using any standard serial terminal emulator (such as PuTTY).

This project is compatible with my own assembler only: [ASM8](http://aspisys.com/asm8.htm)

I do not recommend porting to other assemblers as there are too many dependencies on
specific ASM8 syntax features and/or idioms which may be very hard to express in most
other assemblers.

(That does not restrict your application to be in whatever language and tools you prefer.)

# Overview

TBoot's purpose is to permanently reside inside your product for the possibility you ever
need to upgrade the firmware.  It is specifically designed to not require any external
support by a custom application running on a PC.

Requiring a custom PC application is a common long-term failure pattern.
A device that needs to be upgraded years after it was put into service is often found
unable to perform the upgrade due to the original upgrade companion application failing
to run in later PC/OS and/or hardware combinations than for which it was designed.

With TBoot, the end-user may have any PC/OS. or other type of device that can send ASCII
text over serial communications, and still be able to perform the firmware update.

TBoot is safe in that only the S19 firmware ASCII file is required.
Any terminal emulation program can be used to connect to the device, enter TBoot, and
transfer the firmware with a simple copy-paste operation.  Example: User clicks on
the firmware website link, the firmware file opens up as text page, the user presses
CTRL-A to mark the whole text, then CTRL-C to copy it to the clipboard, swithes to the
terminal emulator window (with TBoot ready to accept the firmware), and presses CTRL-V
to paste the file to the device.

Due to the nature of S19/S28 files, one could even carefully type in the firmware (e.g.
from a magazine page).  Tedious, but not impossible.

TBoot is written 100% in optimized assembly language offering both very low reset-to-app
latency, and tiny memory footprint.

For most MCU configurations, it takes less than two full Flash pages (1KB in most 9S08
MCUs) of your precious application Flash memory.

TBoot will redirect all vectors to your application using either hardware vector
redirection (where available) or low-overhead software vector redirection.

TBoot has been shipped with a wide variety of commercial products for over ten years
with excellent performance.

# Assembly-time conditionals

A variety of assembly-time conditions can alter TBoot features and/or behavior.

* HZ................: MCU effective clock as Hz
* KHZ...............: MCU effective clock as KHz
* MHZ...............: MCU effective clock as MHz
* BDIV..............: Bus divisor (where available)
* FLASH_DATA_SIZE...: Flash size for user data
* ALLOW_EEPROM......: Allow EEPROM address range
* NVOPT_VALUE.......: Use a specific NVOPT value
* HARD_FLOW_CONTROL.: For RTS/CTS control
* RXINV.............: SCI RX line inverted
* TXINV.............: SCI TX line inverted
* BPS...............: BPS = 3/12/24/48/96/192/384/576(00)
* SCI...............: SCI = (SCI)1 or (SCI)2 or SoftSCI (-1)
* ENABLE_RUN........: Enable [R]un command
* NO_IRQ............: Disable IRQ pin test
* DISABLE_SURE......: Disable 'Sure?' message (not recommended)
* DEBUG.............: For debugging only

`HZ`, `KHZ`, and `MHZ` define the effective MCU clock in the respective units.
Together with `BDIV` (where available), one can define the exact MCU bus clock required,
if other than the carefully chosen default.

`FLASH_DATA_SIZE` defines the Flash size to be used for saving user configuration
for your app.  From experience, most apps need no more than a single Flash page, and
that's the default.

`ALLOW_EEPROM` will enable EEPROM address range overwrites. If you don't care to preserve
your end-user's configuration (if any), you can use this option.

`NVOPT_VALUE` defines a specific NVOPT value.  Since this value is stored in the protected
memory region together with TBoot, you should be careful what you use.

`HARD_FLOW_CONTROL` enables RTS/CTS control and requires the corresponding definition of
`RTS_LINE` and `CTS_LINE` pins.  Unless your device is connected to equipment that
requires hard flow control at all times, and cannot possibly function without it, you
should disable this setting.  This way, TBoot will also allow updates when either flow
control lines are damaged or disconnected.  Whether your application uses flow control or
not is irrelevant as you can always update that to the other possibility at a later time.

`RXINV` will invert the SCI RX line.  Some circuits require inversion to work correctly.

`TXINV` will invert the SCI TX line.  Some circuits require inversion to work correctly.

`BPS` overrides the default bits-per-second selection.  You can set it to one of the
following standard bps rates: 300, 1200, 2400, 4800, 9600, 19200, 38400, 57600.  If you
plan ahead, it would be preferrable to choose the same rate as that used by the application.
This will make it easier for the end user to switch between TBoot and your application
without having to re-adjust the terminal emulator's bps rate.

`SCI` overrides the default SCI to use.  The default is the primary SCI (SCI1) or, for
MCUs without hardware SCI, the software SCI (-1)

For the few MCU variants that lack a hardware SCI, a bit-banged software SCI driver is
used.  You should use relatively low bps speeds with those to avoid dropping characters.

For software SCI to work you will have to define `SCI_TX_PIN` and `SCI_RX_PIN`
to whichever port pin can support regular I/O.

`ENABLE_RUN` enables an extra `[R]un` command which lets you run the loaded application
without resetting the MCU.  This is a deprecated feature meant mostly for some
debugging scenarios.  It's use is NOT recommended.

`NO_IRQ` disables the startup IRQ pin test for forced entry to TBoot.  A low IRQ during
startup is the primary and ***recommended*** method of entry to TBoot.  It has the advantage
of always succeeding even if your application is faulty enough to not allow software
entry to TBoot.

This conditional is provided for those few cases where the IRQ pin is used by your
application for free running signals that appear *before* any MCU initialization.
Such condition could cause an inadverted entry into TBoot if the timing is unfortunate.

On the other hand, applications that use the IRQ pin after initialization of peripherals
do not need to remove this functionality from TBoot.  TBoot is smart enough to release
the IRQ pin back to your application immediately after checking it at startup.  At no
other time is the IRQ pin checked by TBoot as TBoot does not run in the background, having
absolutely no control of the MCU.

`DISABLE_SURE` option should normally not be used.  However, in some circumstances
where the binary overflows into one more Flash page by just a few bytes (thus, taking
a whole new Flash page), this option can be used to remove some extra code and
squeeze the overall size down a bit.

In general, however, you would want a strong confirmation by the end-user to proceed
with the Erase action.

`DEBUG` is meant to enable debugging definition and/or code.
It is currently not used at all in the public version of TBoot so you can ignore it.

Note: All assembly time options are fixed for the life of your product.  You cannot
      make changes afterwards.  So, choose wisely to accommodate for possible future
      updates. For example, if the original version does not need to keep user
      configuration, you shouldn't just disable that without thinking ahead.
      What if you will need that in the future?
      TBoot should be burned inside the end user's MCU to allow for that possibility.

# Assembly

Assuming all required files can be found by the assembler's -I option (default is usually
adequate), the command:

`asm8 tboot -dXXX` will assemble TBoot for MCU XXX which is one of the supported MCU variants.

To see a list of supported MCUs and options, give `?` for XXX in the above command.

It's recommended to produce an EXP file to be used by your application.  To do this, you'll
have to add the `-exp+` option.

For MMU variants, you may turn on MMU support, or leave MMU support off.  The difference
is the amount of Flash that TBoot will be able to access for erasure or programming.

For example (assuming default ASM8 configuration):

`asm8 tboot -dQE128 -mmu -exp`

will assemble TBoot for 9S08QE128 with MMU support and produce an export file to be
used (`#Uses` or `#Include`) by the application.

# End-user experience

For the end-user, firmware updates should be easier than sending the product back to
the factory with delays and costs for both parties involved.  They could also be remotely
performed by tech support if an end-user is unable to complete the process on their own.

Once a connection is made to the device (often this is already present for the sake of
the application itself), a simple Erase-Load-Escape (from TBoot) cycle is used to update
the firmware within a few seconds.

Having the upgrade process interrupted mid-way (e.g. power/battery failure or user error)
is 100% safe.  Simply try the process again.

The user sees a series of dots ending with an exclamation point and back to the prompt.
If nothing but dots appear, s/he knows the update went well.

The following section shows all possible feedback codes.

# Error codes during loading

The following symbols indicate success, warning, or failure of each loaded S19/S29 record:

* `.` = Successful programming of S19 record (informational)
* `!` = End of S19/S29 file (informational)
* `C` = S19 record CRC failure (warning)
* `F` = Flash programming failure (error)
* `R` = S19 record address range violation (warning)

Warnings may not necessarily result in bad code loaded.

As an example, a failed CRC may be the result of user error (e.g., editing the S19 file
accidentally), or momentary SCI error (especially with the lower accuracy of the
software SCI) due to noise.

A range violation may happen if, for example, you try to load the user configuration
area with defaults, but TBoot was assembled not to allow that.

Of course, it could happen when attempting to load a completely irrelevant firmware file.

# Community

If you would like to contibute support for additional MCU variants, please contact me with
pull requests.  Make all your work in a separate, uniquely named branch, such as your
[nick]name for easier management by both parties.

# Steppenprobe
Open Source Hardware JTAG/SWD/UART/SWO interface board

https://github.com/diegoherranz/steppenprobe

## Basic specs:

- Open Source Hardware and designed in [KiCad](https://kicad-pcb.org).
- [FT2232H](https://ftdichip.com/products/ft2232hl)-based
- Simultaneous operation of JTAG/SWD and UART/SWO.
- STDC14 connector: standard Arm Cortex debug connector (1.27 mm pitch) with extra pins for UART. JTAG/SWD and UART on a single connector while being compatible with the standard Arm Cortex debug connector (middle 10 pins).
- USB powered: no power supply needed.
- USB type-C receptacle: more convenient (reversible) and future-proof (it should be the most common connector during the next few years).
- OpenOCD compatible, supported from version 0.11.0 (older versions can be used by adapting this [config file](https://sourceforge.net/p/openocd/code/ci/master/tree/tcl/interface/ftdi/steppenprobe.cfg) to the old sintax).
- JTAG or SWD use without any jumper or switch operation required (configured from OpenOCD).
- Switch-selectable UART RX or SWO operation.
- Signals also available on 2.54 mm pin header to work with any custom pinout.
- 4 GPIO.
- System Reset tactile switch.
- I/O buffers for protection and to be able to use a wide voltage range (1.65 V to 5.5 V).
- Descriptive LED indicators.

## Schematic PDF, Gerbers and other generated files:
See [releases](https://github.com/diegoherranz/steppenprobe/releases).

## FTDI channels

The FT2232H has 2 independent channels. In this board:

- Channel A: Used for JTAG or SWD (MPSSE mode) + GPIOs
- Channel B: Used as UART. TX+RX or TX+SWO depending on the position of the physical switch.

After enumeration, both channels will default to UART mode, so on a Linux system they will show up as `/dev/ttyUSB0` and `/dev/ttyUSB1`. When channel A gets reconfigured to MPSSE mode (e.g. when using OpenOCD with it), then `/dev/ttyUSB0` will disappear and channel B will remain as `/dev/ttyUSB1`.

## 3D renders

![PCB top view](images/PCB_render_top.png) ![PCB bottom view](images/PCB_render_bottom.png) ![PCB isometric view](images/PCB_render_iso.png)

## Options for J2 connector (1.27 mm pitch header)
You can solder a 10-way or 14-way header depending on what suits you best:
 - 10-way header: [FTSH-105-01-F-DV-K](https://www.samtec.com/products/ftsh-105-01-f-dv-k) or equivalent. Use the central 10 pads in this case. JTAG, SWD, SWO and RESET (Arm Cortex debug connector standard).
 - 14-way header: [FTSH-107-01-F-DV-K](https://www.samtec.com/products/ftsh-107-01-f-dv-k) or equivalent. Compared to the 10 pin option, it adds UART TX and RX (STDC14 standard). This can still connect to targets with a 10-way header by using a 14-way to 10-way cable.


## OpenOCD example commands

### Flashing
Flashing [NuttX](https://nuttx.apache.org) to a [Blue Pill](https://stm32-base.org/boards/STM32F103C8T6-Blue-Pill.html) using JTAG:

```
openocd -f interface/ftdi/steppenprobe.cfg -f target/stm32f1x.cfg -c "program nuttx.bin 0x08000000 verify reset exit"
```

Same but using SWD:

```
openocd -f interface/ftdi/steppenprobe.cfg -c "transport select swd" -f target/stm32f1x.cfg -c "program nuttx.bin 0x08000000 verify reset exit"
```
All these examples use the default reset signals configuration. You may need to adjust it for your target board to work or even if it works, you may want to define it explicitely. See the [Reset Configuration chapter](http://openocd.org/doc-release/html/Reset-Configuration.html) of the OpenOCD documentation for more details.

You can do so directly on the OpenOCD command:

```
openocd -f interface/ftdi/steppenprobe.cfg -c "reset_config srst_only" -f target/stm32f1x.cfg -c "program nuttx.bin 0x08000000 verify reset exit"
```

or defining your board config file (your_board.cfg) which may look like this:

```
reset_config srst_only
source [find target/stm32f4x.cfg]
```

And then you do:

```
openocd -f interface/ftdi/steppenprobe.cfg -f your_board.cfg -c "program nuttx.bin 0x08000000 verify reset exit"
```


### Debugging
Debugging example for an STM32F4 target.

Start OpenOCD:
- JTAG: `openocd -f interface/ftdi/steppenprobe.cfg -f target/stm32f4x.cfg`
- SWD: `openocd -f interface/ftdi/steppenprobe.cfg -c "transport select swd" -f target/stm32f4x.cfg
`

On a different terminal:
```
$ gdb-multiarch your_code.elf # or gcc-arm-none-eabi depending on your system.

# Attach to OpenOCD
(gdb) target extended-remote localhost:3333

# Reset the boad
(gdb) monitor reset halt

# Load the code to flash
(gdb) load

# Start running
continue
```
Then use GDB as normal: see https://www.gnu.org/software/gdb/documentation/


### Debugging + SWO

Start OpenOCD in SWD mode and use the `tpiu config` command. Also, the UART switch should be in the "TX+SWO" position.

```
openocd -f interface/ftdi/steppenprobe.cfg -c "transport select swd" -f target/stm32f4x.cfg -c "tpiu config external uart off 168000000 115200"
```

The `tpiu config` command configures the ARM ITM module and the SWO output. In the command above, `16800000` represents the TRACECLKIN frequency (usually the same as HCLK), and `115200` is the frequency/baurate of the SWO line. Adjust both as desired. See the [OpenOCD documentation](http://openocd.org/doc-release/html/Architecture-and-Core-Commands.html#ARMv7_002dM-specific-commands
) for more info.

If the selected SWO frequency can't be achieved, OpenOCD will output a message like:

> Info : Can not obtain 115200 trace port frequency from 168000000 TRACECLKIN frequency, using 115147 instead

In this case, the frequency achieved is close enough, but pay attention in case it wasn't.

Then, on another terminal:
```
# configure the serial port speed
stty -F /dev/ttyUSB1 115200

# start the ITM decoder
./itmdump -f /dev/ttyUSB1 -d1 
```

Note: ITM decoder available [here](https://sourceforge.net/p/openocd/code/ci/master/tree/contrib/itmdump.c), just compile by doing `gcc itmdump.c -o itmdump`.

One of the advantages of using the SWO signal is that it can be faster than a standard UART, specially for slow core clock frequencies (see [this nice article](https://blog.japaric.io/itm/) for more info). To benefit from that, we need to use faster SWO frequencies.

The FT2232H is capable or running the UART up to 12 Mbps, but the configuration is more cumbersome for the fastest speeds.

On a recent Linux system, you can go up to 4 Mbps, configuring everything the same way as explained above for 115200 bps. On different OSes, you may not be able to go above 115200 bps or even 38400 bps.

For 4 Mbps:

```
openocd -f interface/ftdi/steppenprobe.cfg -c "transport select swd" -f target/stm32f4x.cfg -c "tpiu config external uart off 168000000 4000000"
```

On another terminal:
```
stty -F /dev/ttyUSB1 4000000
./itmdump -f /dev/ttyUSB1 -d1
```

To use even higher frequencies, up to the maximum of 12 MHz if your setup and cabling allows it, it's necessary to use setserial to alias 38400 bps to a custom divider. The base frequency of the FT2232H is 60 MHz, so for instance, for 12 MHz, the divisor must be set to 5.

```
openocd -f interface/ftdi/steppenprobe.cfg -c "transport select swd" -f target/stm32f4x.cfg -c "tpiu config external uart off 168000000 12000000"
```

On a different terminal:
```
setserial /dev/ttyUSB1 spd_cust divisor 5
stty -F /dev/ttyUSB1 38400
./itmdump -f /dev/ttyUSB1 -d1
```

### GPIO usage
There are 4 GPIOs available: GPIO_A to GPIO_D. By default they are High-Impedance (Z).

They can be set through OpenOCD by using `ftdi_set_signal name 0|1|z`. For example:
```
ftdi_set_signal GPIO_B 1
```

And similarly, they can be read using `ftdi_get_signal name`.





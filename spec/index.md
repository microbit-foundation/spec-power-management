---
layout: page
title: Power Management Specification
heading: Power Management Specification
description: Specification for the micro:bit Power Management
permalink: /software/spec-power-management/
ref: spec-power-management
lang: en
---

# Power Management Specification

This is version 1.0.3 of the specification.

## Table of Contents

- [Introduction](#introduction)
- [Terminology](#terminology)
- [micro:bit Power Modes](#microbit-power-modes)
- [Interface MCU Power Modes](#interface-mcu-power-modes)
- [LED behaviour](#led-behaviour)
- [Waking Up The Interface MCU](#waking-up-the-interface-mcu)
- [Target MCU Power Modes](#target-mcu-power-modes)
- [Waking Up The Target MCU](#waking-up-the-target-mcu)
- [Power Mode Transitions](#power-mode-transitions)
- [Changelog](#changelog)


## Introduction

The micro:bit contains two microcontrollers, the Interface MCU (KL27 in micro:bit V2.00 or nRF52 in micro:bit V2.2x) which provides the USB functionality, and the Target MCU (nRF52 in all micro:bit V2 versions) where the user code runs.
More information can be found in the [Tech Site DAPLink page](https://tech.microbit.org/software/daplink-interface/).

In micro:bit V1 the Interface MCU (KL26) is not powered via batteries or the Edge Connector, so the sleep functionality is only implemented in the Target MCU (nRF51).
The micro:bit V2 powers both MCUs with all power sources, so to set the board into a sleep mode some co-operation via the [I2C protocol](https://github.com/microbit-foundation/spec-i2c-protocol) is needed.

The micro:bit V2 has four power modes described in this document:

- [On Mode](#on-mode)
- [Deep Sleep Mode](#deep-sleep-mode)
- [Off Mode](#off-mode)
- [Stand-by Mode](#stand-by-mode)


## Terminology

| Term                | Definition |
|---------------------|------------|
| MCU                 | Microcontroller |
| Interface MCU       | The microcontroller providing USB (DAPLink) functionality |
| Target MCU          | The microcontroller where the user code runs |
| Sleep               | A sleep mode for the individual MCUs (Interface or Target) where they can wake up and continue operation |
| Power Down          | The lowest power state mode for individual MCUs (Interface or Target) |
| DAPLink             | [Interface firmware](https://github.com/ARMmbed/DAPLink) providing USB and programming capabilities |
| PC Connected        | When DAPLink on the Interface MCU is USB enumerated and/or has USB activity with a PC |
| On-board components | Motion sensors, speaker, microphone |
| I2C                 | [(Inter-Integrated Circuit](https://en.wikipedia.org/wiki/I%C2%B2C) bus |
| VBUS_ABSENT         | Signal connected to the Interface MCU indicate USB voltage presence (previously named WAKE_ON_EDGE)|
| COMBINED_SENSOR_INT | Interrupt signal shared between all the internal I2C devices in the micro:bit board |


## micro:bit Power Modes

We want to define 4 user-facing power modes for the micro:bit board. These board-level modes will be achieved via a combination of different subsystem power modes.

- **<a id="on-mode"></a>On Mode**: Normal running mode.
    - As far as the user is concerned, everything is running
    - The software in the Target MCU and Interface MCU can decide to go into Sleep if they are idle
        - If the Interface MCU is not PC Connected and is idle it must go into Sleep
            - When the Interface MCU is in Sleep it must automatically wake up when it receives I2C transactions from the Target MCU
        - If the Target MCU is idle it can automatically go into Sleep
            - When the Target MCU is in Sleep it must to wake up on the ticker to run background operations
            - When the Target MCU is in Sleep it must not drop any input data (buses, radio, gpio, etc)
        - When the Interface MCU or the Target MCU wake up they must resume where they left off
    - All events and background tasks must to be running during this mode
    - The Power (red) LED must be ON
    - If the board is PC connected the USB (orange) LED must be ON
    - If the board is not PC connected the USB (orange) LED must be OFF
- **<a id="deep-sleep-mode"></a>Deep Sleep Mode**: A mode triggered by the user code to go into a low-power state
    - The Interface MCU does not do anything different to the On Mode
    - The Target MCU must turn off the on-board components
    - The Target MCU must go into Sleep
    - When the Target MCU wakes up it must restore the on-board components to run again
    - When the Target MCU wakes up its software must resume where it left off
    - During the time the board is in Deep Sleep mode, it will not react to anything until it wakes up
        - Any data received via radio or other buses will be ignored and dropped
    - Entered via:
        - User code calling a CODAL `uBit` method
    - Woken up via:
        - Timer on the Target MCU
            - The user can configure the elapsed time to wake-up
        - Any pin in Target MCU
            - Edge connector pin
            - A or B buttons
            - The Interface MCU can wake up the Target MCU via COMBINED_SENSOR_INT
            - USB cable insertion (VBUS_ABSENT signal)
        - Pressing the reset button
            - The Interface MCU must halt the Target MCU while the button is pressed down
            - The Interface MCU must reset the Target MCU when the reset button is released
    - If the board is PC Connected the USB (orange) and Power (red) LEDs must be ON
    - If the board is not PC Connected the USB (orange) and Power (red) LEDs must be OFF
- **<a id="off-mode"></a>Off Mode**: Lowest possible power state for all components on the board
    - The board can be considered "off", although it will still consume power
    - This mode can only be reached if the micro:bit is powered via battery or USB bank (not USB Connected)
    - The Target MCU must turn off the on-board components
    - The Target MCU must go into Power Down
    - The Interface MCU must go into Power Down
    - The board will not react to anything except the wake up sources
    - Both microcontrollers must reset on wake
    - Entered via:
        - Long press of the reset button
        - User code calling a CODAL `uBit` method
    - Woken up via:
        - USB cable insertion (VBUS_ABSENT signal)
        - Pressing the reset button
    - The Power (red) LED must be OFF
    - The USB (orange) LED must be OFF
- **<a id="stand-by-mode"></a>Stand-by Mode**: User long presses the reset button, while the micro:bit is PC Connected, to stop their Target MCU programme
    - This is the same as the Off Mode, but this mode is activated when the micro:bit is PC Connected
    - The Interface MCU must not go into any sleep in this case, the rest behaves the same
    - The Power (red) LED must be blinking
    - The USB (orange) LED must be ON


## Interface MCU Power Modes

There is a large selection of sleep modes available in the Interface MCU KL27 microcontroller, two have been selected for the micro:bit requirements.

1) **Power Down**: VLLS0, Very Low-Leakage Stop 0
    - Lowest power mode available in the KL27
    - Wakes up in reset mode
    - Woken up via LLWU pins:
        - VBUS_ABSENT triggered by inserting the USB cable
        - BTN_NOT_PRESSED triggered by pressing the reset button
2) **Sleep**: VLPS0, Very Low Power Stop 0
    - The deepest sleep mode that can be woken up via I2C
    - On wake up it resumes where it left off
    - Can wake up from same sources as Deep Sleep, plus:
        - I2C transactions

Other notes:
- The Interface MCU must not enter either of these two modes when it is PC Connected
    - As both modes are not PC connected, the Interface MCU won't respond to any UART activity during Sleep or Power Down
- The Interface MCU must automatically go into Sleep mode when it's idle
    - This power mode is managed exclusively by the Interface MCU
- The Interface MCU must go into Power Down mode when requested by the Target MCU via I2C
    - If the Interface MCU is PC Connected it must ignore the request to go into Power Down

### LED behaviour

- In both modes the USB (orange) LED must be OFF
    - This is because this LED is only ON when the board is PC connected
- In both modes the Power (red) LED state will depend on a setting configured by the Target MCU via I2C
    - When the `Power LED Sleep state` setting is ON, the Interface MCU must set the Power (red) LED ON when it goes into Sleep or Power Down mode
        - In Sleep mode the Interface MCU is able to keep the LED brightness PWM controlled, so the LED must be dimmed
        - In Power Down mode the Interface MCU can only configure the pin high or low, so the LED will be in full brightness
    - When the `Power LED Sleep state` setting is OFF, the Interface MCU must turn OFF the Power (red) LED when it goes into Sleep or Power Down mode
    - On Interface MCU reset, the default `Power LED Sleep state` setting value must be set to ON
    - When the reset button is pressed, the Interface MCU must reset the `Power LED Sleep state` value to the default ON
    - When the Interface MCU wakes up the Target MCU, the Interface MCU must reset the `Power LED Sleep state` value to the default ON
    - When the Target MCU goes into Sleep or Power Down mode it must set the `Power LED Sleep state` setting to OFF
    - When the Target MCU wakes up from Sleep mode it must set the `Power LED Sleep state` setting to ON

### Waking Up The Interface MCU

Waking up via Target MCU:
- The Target MCU does not know the Interface MCU current power mode, but it needs the Interface MCU to always be responsive to I2C comms
- To wake up the Interface MCU the Target MCU can send an I2C message
    - When the Interface MCU for V2.00 (KL27) is woken up via I2C, it is unable to process the data from the transaction that woke it (e8777 errata)
    - Errata e8777: Address match wake-up from low-power mode cannot receive data
        - https://www.nxp.com/docs/en/errata/KINETIS_L_1N71K.pdf
        - So the data in the I2C transmission will be lost if it wakes up the Interface MCU
    - The Target MCU could use this workaround from e8777 errata: Send only the matching secondary address followed by a repeated start and then resend the matching secondary address including any subsequent data
        - At the time of writing the nrfx drivers used in CODAL makes this very difficult, so instead the wake up workaround is to start all I2C transactions with a "nop" I2C command that does nothing, so it doesn't matter if its lost and if it is received the Interface MCU will ignore it

Waking up from VBUS_ABSENT signal:
- A USB cable insertion will set the VBUS_ABSENT signal active (low)
- When the VBUS_ABSENT transitions from inactive to active the Interface MCU must wake up from any sleep mode

Waking up from the reset button:
- Pressing the reset button sets the BTN_RST signal active (low)
- When the BTN_RST signal transitions to active the Interface MCU must wake up from any sleep mode
- The Interface MCU must halt the Target MCU while the button is pressed down
- When the Interface MCU wakes up from Power Down it must start from a reset state
- When the Interface MCU wakes up from Power Down the DAPLink bootloader must not enter MAINTENANCE mode
- When the Interface MCU wakes up from Sleep the DAPLink interface must continue operation where it left off
- When the button is released the Interface MCU must reset the Target MCU

Setting up the COMBINED_SENSOR_INT signal:
- When the Interface MCU wakes up from any mode it must set the COMBINED_SENSOR_INT signal active
    - This signal is used to wake up the Target MCU
    - This signal is open drain and active low
        - To set the signal activate the Interface MCU must pull down its pin connected to this signal
        - To set the signal inactive the Interface MCU must set the pin to high impedance
- If the signal was already in active state the Interface MCU will pulse the signal
- The Interface MCU must be ready for the Target MCU to query via I2C the wake up source
    - If the Target MCU is in Power Down it might take a considerable amount of time for CODAL to start-up and service the Interface MCU interrupt
    - More information about the I2C protocol can be found in the [I2C Protocol Spec](https://github.com/microbit-foundation/spec-i2c-protocol)
- When the wake up source has been read by the Target MCU, the Interface MCU must release the COMBINED_SENSOR_INT to the inactive state


## Target MCU Power Modes

The nRF52833 just has two power modes (section 5.3 of the datasheet): System Off, where everything is off; and System On, where everything is running but individual components can be configured into a low power state.

1) **Power Down**: System Off
    - Lowest power mode available in the nRF52
    - The CPU stops and all peripherals are disabled
    - Wakes up in reset mode
    - Woken up via:
        - COMBINED_SENSOR_INT
        - SWD
2) **Sleep**: System On with the CPU in low power mode
    - The System On mode is the normal running mode, but we can configure components, like the CPU, in low power modes
    - On wake up it resumes where it left of
    - Can wake up from the same sources as Power Down, plus:
        - Internal timer
        - Microphone
        - Motion sensor events
        - Any user accessible nRF52 pin:
            - Edge connector pins
            - Face logo
            - A and B buttons
 
Other notes:
- The Target MCU will enter Sleep when:
    - Software is idle
    - User code calls a CODAL `uBit` method to sleep
- During Sleep the Target MCU must still be responsive to events, incoming data and run background tasks
- The Target MCU will enter Power Down when:
    - The Interface MCU indicates the user has long-pressed the reset button
    - User code calls a CODAL `uBit` method to go into the board Off mode

### Waking Up The Target MCU

- When the COMBINED_SENSOR_INT signal transitions from inactive to active the Target MCU must wake up from Sleep or Power Down
- When the Target MCU wakes up it must ensure the on-board components are enabled and running

Wake up from the reset button:
- The reset button is only connected to the Interface MCU
- The Interface MCU will set the COMBINED_SENSOR_INT signal active
- The specific Interface MCU behaviour when the reset button is pressed can be found in the [Waking Up The Interface MCU](#waking-up-the-interface-mcu) section

Wake up from USB cable insertion:
- The USB cable insertion signal is only connected to the Interface MCU
- The Interface MCU will set the COMBINED_SENSOR_INT active
- The specific Interface MCU behaviour when the USB cable is inserted can be found in the [Waking Up The Interface MCU](#waking-up-the-interface-mcu) section

Responding to the COMBINED_SENSOR_INT signal:
- The COMBINED_SENSOR_INT signal is open drain and active low
    - The Target MCU must configure its internal pull up in the pin connected to this signal
    - The Target MCU interrupt must be configured to wake up when the signal transitions from high to low
- The Target MCU must query the wake up source to the Interface MCU via I2C
    - More information about the I2C protocol can be found in the [I2C Protocol Spec](https://github.com/microbit-foundation/spec-i2c-protocol)


## Power Mode Transitions

### On -> Deep Sleep

- Target MCU turns off all on-board components
- Target MCU sends the I2C command to the Interface MCU to set the `power-led-sleep-state` to OFF
- Target MCU sets itself into Sleep mode
- Target MCU does not need to set the Interface MCU into a sleep mode, the Interface will manage its Sleep mode independently

### Deep Sleep -> On

- One of these events activates this transition:
    - Target MCU wakes up
    - USB insertion wakes up the Interface MCU (if asleep) and the Interface wakes up the Target via COMBINED_SENSOR_INT
    - Reset button wakes up the Interface MCU (if asleep) and the Interface wakes up the Target MCU via SWD reset
- Target MCU sends the I2C command to the Interface MCU to set the `power-led-sleep-state` to ON
- Target MCU turns-on all the required on-board components
- Target MCU does not need to wake up Interface MCU, as the Interface manages its own power state in Sleep mode

### On -> Off or On ->Stand-by

- User long-presses reset button
    - While the button is pressed down the Interface MCU halts the Target MCU
    - When the button is released the Interface MCU resets the the Target MCU
- Interface MCU sets the interrupt signal active
    - Or it pulses it, if it was already active
- Target MCU queries the Interface MCU status
- Interface MCU responds to the Target MCU that the reset button has been long-pressed
- Target MCU sends the command to the Interface MCU to go to Power Down
    - If the board is not PC connected, the Interface MCU goes into Deep Sleep
    - If the board is PC connected, the Interface MCU ignores the command and starts blinking the Power (red) LED
- Target MCU turns off all on-board components
- Target MCU sets itself to Deep Sleep

### Off -> On or Stand-by -> On

- One of these events activates this transition:
    - (Off mode only) USB insertion wakes up the Interface MCU (if asleep) and the Interface wakes up the Target MCU via COMBINED_SENSOR_INT
    - Reset button wakes up the Interface MCU (if asleep) and the Interface wakes up the Target MCU via SWD reset
- (Off mode only) Interface MCU wakes up in a reset state
- Interface MCU detects the wake up source
- If the wake up source is the USB insertion the Interface MCU sets the COMBINED_SENSOR_INT signal active (or pulses it if already active) to wake up the Target MCU
- If the wake up source is the reset button the Interface MCU halts the Target MCU while button is pressed, and resets the Target MCU when the button is released
- Target MCU starts running from reset
- The Target MCU software on start-up should configure all the on-board components to the expected "run mode" and not depend on the component default on-power-up state

### Deep Sleep -> Off or Stand-by

The only way to transition from Sleep to Stand-by is by the user pressing the reset button.

- User starts pressing the reset button
- Interface MCU wakes up **not** on reset mode
- Interface MCU detects the wake up source
- While the button is pressed down the Interface MCU halts Target MCU
- When the button is released the Interface MCU resets the Target MCU
- If the user releases the button before the long-press state is reached the Interface MCU continues with the reset process
- If the user releases the button after the long-press state is reached, it continues with the "On -> Off / Stand-by" process

### Off or Stand-by -> Deep Sleep

Not possible.


## Changelog

| Version      | Changes |
|--------------|---------|
| 1.0.0        | Initial release. |
| [1.0.1][101] | Added Table of Contents. |
|              | Reorganised sections for better presentation. |
|              | Added links to the power modes to the intro |
|              | Added markdown frontmatter metadata for tech.microbit.org website rendering. |
| [1.0.2][102] | Reformatted changelog |
|              | Fixed table of contents links |
| [1.0.3][103] | Replace references to "KL27" with "Interface MCU", and "nRF52" with "Target MCU" |

[101]: https://github.com/microbit-foundation/spec-power-management/commit/917a60dc70c2691abb5227d70829246f83ccfd53
[102]: https://github.com/microbit-foundation/spec-power-management/commit/bedda257b88bf032abe3cc8d1f05ba45f34fef41
[103]: https://github.com/microbit-foundation/spec-power-management/commit/????

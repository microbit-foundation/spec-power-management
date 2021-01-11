# Power Management

## Terminology

| Term                | Definition |
|---------------------|------------|
| Sleep               | A sleep mode for the KL27 or nRF52 where they can wake up and continue operation |
| Power Down          | Lowest power state mode for the KL27 or nRF52 |
| PC Connected        | When DAPLink on the KL27 is USB enumerated and/or has USB activity with a PC |
| On-board components | Motion sensors, speaker, microphone |


## micro:bit power modes

We want to define 4 user-facing power modes for the micro:bit board. These board-level modes will be achieved via a combination of different subsystem power modes.

- **On Mode**: Normal running mode.
    - As far as the user is concerned, everything is running
    - However the software in the nRF52 and KL27 can decide to go into Sleep if they are idle
        - If the KL27 is not PC Connected and is idle it can go into Sleep and automatically wake up when it receives I2C transactions from the nRF52
        - If the nRF52 is idle it can go automatically into Sleep, but it needs to wake up on the ticker to run background operations
        - When these systems wake up they must resume where they left off
    - All events and background tasks need to be running during this mode
    - The Power (red) LED will be ON
    - If the board is PC connected the USB (orange) LED will be ON
    - If the board is not PC connected the USB (orange) LED will be OFF
- **Deep Sleep Mode**: A mode triggered by the user code to go into a low-power state
    - The KL27 doesn't do anything differently
         - As defined in the "On Mode", the KL27 will manage its own Sleep if it's idle
    - The nRF52 turns off the on-board components
    - The nRF52 goes into Sleep mode
    - On wake the nRF52 will restore the on-board components to run again and the software will resume where it left off
    - During the time the board is in Deep Sleep, it will not react to anything until it wakes up. Any data received via radio or other buses will be ignored and dropped.
    - Entered via:
        - User code calling a CODAL uBit method
    - Woken up via:
        - Timer on nRF52
            - The user can configure the elapsed time to wake-up
        - Any nRF52 pin
            - Edge connector pin
            - COMBINED_SENSOR_INT by the KL27
            - A and B buttons
        - USB cable insertion (WAKE_ON_EDGE)
            - The KL27 will wake up the nRF52 via COMBINED_SENSOR_INT
        - Pressing the reset button
            - The KL27 will halt the nRF52 while the button is pressed down, and reset it on button release
    - If the board is PC Connected the USB (orange) and Power (red) LEDs will be ON
    - If the board is not PC Connected the USB (orange) and Power (red) LEDs will be OFF
- **Off Mode**: Lowest possible power state for all components on the board
    - The board can be considered "off", although it consumes a bit of power
    - This mode can only be reached if the micro:bit is powered via battery or USB bank (not USB Connected)
    - All on-board components turned off
    - Power Down mode for the KL27 and nRF52 microcontrollers
    - Device will not react to anything except the wake up sources
    - Both microcontrollers will reset on wake
    - Entered via:
        - Long press of the reset button
        - User code calling CODAL uBit method
    - Woken up via:
        - USB cable insertion (WAKE_ON_EDGE)
        - Pressing the reset button
    - The Power (red) LED will be OFF
    - The USB (orange) LED will be OFF
- **Stand-by Mode**: User long presses the reset button, while the micro:bit is PC Connected, to stop their nRF52 programme
    - This is the same as the Off Mode, but this mode is activated when the micro:bit is PC Connected
    - The KL27 does not go to sleep in this case, the rest behaves the same
    - The Power (red) LED will be blinking
    - The USB (orange) LED will be ON


## KL27 Power Modes

There is a large selection of sleep modes available in the KL27, two have been selected for the micro:bit requirements. 

1) **Power Down**: VLLS0, Very Low-Leakage Stop 0
    - Lowest power mode available in the KL27
    - Wakes up in reset mode
    - Woken up via LLWU pins:
        - WAKE_ON_EDGE triggered by inserting the USB cable
        - BTN_NOT_PRESSED triggered by pressing the reset button
2) **Sleep**: VLPS0, Very Low Power Stop 0
    - The deepest sleep mode that can be woken up via I2C
    - On wake up it resumes where it left off
    - Can wake up from same sources as Deep Sleep, plus:
        - I2C transactions

Other notes:
- The KL27 will only enter either of these two modes if it is **not** PC Connected
    - As both modes aren't PC connected, the KL27 won't respond to any UART activity
- The KL27 will automatically go into Sleep mode if it's idle
    - This power mode is managed exclusively by the KL27
- The KL27 will go into Power Down mode when requested by the nRF52 via I2C
    - If the KL27 is PC Connected it will ignore the request to go into Power Down

### LED behaviour

- In both modes the USB (orange) LED will be OFF
    - This is because this LED is only ON when PC connected
- In both modes the Power (red) LED will be ON or OFF depending on a setting configured by the nRF52 via I2C
    - When the `Power LED Sleep state` setting is ON, the KL27 leaves the Power (red) LED ON when it goes into Sleep or Power Down mode
        - In Sleep mode the KL27 is able to keep the LED brightness PWM controlled, so it will be dimmed
        - In Power Down mode the KL27 can only configure the pin high or low, so the LED will be in full brightness
    - When the `Power LED Sleep state` setting is OFF, the KL27 turns OFF the Power (red) LED when it goes into Sleep or Power Down mode
    - On KL27 reset, the default `Power LED Sleep state` setting value will be set to ON
    - When the reset button is pressed, the KL27 will reset the `Power LED Sleep state` value to the default ON
    - When the KL27 wakes up the nRF52, the KL27 will reset the `Power LED Sleep state` value to the default ON
    - When the nRF52 goes into Sleep or Power Down mode it will set the `Power LED Sleep state` setting to OFF
    - When the nRF52 wakes up from Sleep mode it will set the `Power LED Sleep state` setting to ON

### Waking Up The KL27

Waking up from the nRF52:
- The nRF52 does not know the KL27 current power mode, but it needs the KL27 to be responsive to I2C comms
- To wake up the KL27 device the nRF52 can send an I2C message
    - When the KL27 is woken up via I2C, it is unable to process the data from the transaction that woke it (e8777 errata)
    - Errata e8777: Address match wake-up from low-power mode cannot receive data
        - https://www.nxp.com/docs/en/errata/KINETIS_L_1N71K.pdf
        - So the data in the I2C transmission will be lost if it wakes up the KL27  
    - The nRF52 could use this workaround from e8777 errata: Send only the matching slave address followed by a repeated start and then resend the matching slave address including any subsequent data
        - At the time of writing the nrfx drivers CODAL uses make this very difficult, so instead the wake up workaround is to start all I2C transactions with a "nop" I2C command that does nothing, so it doesn't matter if its lost and if it is received the KL27 will ignore it

Waking up from WAKE_ON_EDGE:
- A USB cable insertion will set the WAKE_ON_EDGE signal active (high) and that will wake up the KL27 from either sleep mode
- The KL27 will also need to wake up the nRF52 via COMBINED_SENSOR_INT, even if the KL27 was not asleep

Waking up from the reset button:
- A reset button press will wake up the KL27 from either sleep mode
- The KL27 needs to halt the nRF52 while the button is pressed down, so the KL27 needs to wake up as soon the button is pressed
    - If the KL27 was in Power Down mode it will reset on wake up, and start execution from the DAPLink bootloader.
    - If the DAPLink bootloader sees the reset button is pressed down after a VLLS0 wake up it will not enter into MAINTENANCE mode
    - DAPLink needs to be able to quickly enter the Interface main loop to detect the reset button is still pressed down and keep the nRF52 halted
- When the button is released the KL27 will reset the nRF52


## nRF52 Power Modes

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
- The nRF52 will enter Sleep when:
    - Software is idle
        - It still needs to be responsive to events, incoming data and run background tasks
    - User code calls a uBit method to sleep
- The nRF52 will enter Power Down when:
    - The KL27 indicates the user has long-pressed the reset button
    - User code calls a uBit method to go into the board Off mode

### Waking Up The nRF52

Wake up from the reset button:
- While the button is pressed down the KL27 will halt the nRF52
- When the button is released the KL27 will reset the nRF52
- This will work the same no matter if the nRF52 is awake, in Sleep, or Power Down modes

Wake up from USB cable insertion, the KL27 will:
- Set the COMBINED_SENSOR_INT active
    - If the signal was already in active state the KL27 will pulse the signal
    - This signal is open drain and active low
    - The nRF52 has to configure its internal pull up in the pin connected to this signal
    - The nRF52 interrupt will be configured to wake up when the signal transitions from high to low
    - To set the signal activate the KL27 needs to pull down its pin connected to this signal
    - To set the signal inactive the KL27 needs to set the pin as high impedance
    - If the signal is already active by the time the KL27 needs to wake up the nRF52, it should toggle the signal state (inactive, then active again) to ensure the nRF52 interrupt is triggered
- Wait for the nRF52 to initiate an I2C read for the "interrupt event"
    - If the nRF52 is in Power Down it might take a considerable amount of time for the CODAL to start-up and service the KL27 interrupt
- When the "interrupt event" has been read, COMBINED_SENSOR_INT should be released and the signal be in the inactive state again

When the nRF52 wakes up, either in reset or continuing where it left off, CODAL might have turned off on-board components before it went sleep. Start-up and wake up routines will need to take this into consideration. 


## Power Mode Transitions

### On -> Deep Sleep

- nRF52 turns off all on-board components
- nRF52 sends the I2C command to the KL27 to set the `power-led-sleep-state` to OFF
- nRF52 sets itself into Sleep mode
- nRF52 does not need to set the KL27 into a sleep mode, the KL27 will manage its Sleep mode independently

### Deep Sleep -> On

- One of these events activates this transition:
    - nRF52 wakes up
    - USB insertion wakes up the KL27 (if asleep) and the KL27 wakes up the nRF52 via COMBINED_SENSOR_INT
    - Reset button wakes up the KL27 (if asleep) and the KL27 wakes up the nRF52 via SWD reset
- nRF52 sends the I2C command to the KL27 to set the `power-led-sleep-state` to ON
- nRF52 turns-on all the required on-board components
- nRF52 does not need to wake up KL27, as the KL27 manages its own power state in Sleep mode

### On -> Off or Stand-by

- User long-presses reset button
    - While the button is pressed down the KL27 halts the nRF52
    - When the button is released the KL27 resets the the nRF52
- KL27 sets the interrupt signal active
    - Or it pulses it, if it was already active
- nRF52 queries the KL27 status
- KL27 responds to the nRF52 that the reset button has been long-pressed
- nRF52 sends the command to the KL27 to go to Power Down
    - If the board is not PC connected, the KL27 goes into Deep Sleep
    - If the board is PC connected, the KL27 ignores the command and starts blinking the Power (red) LED
- nRF52 turns off all on-board components
- nRF52 sets itself to Deep Sleep

### Off or Stand-by -> On

- One of these events activates this transition:
    - (Off mode only) USB insertion wakes up the KL27 (if asleep) and the KL27 wakes up the nRF52 via COMBINED_SENSOR_INT
    - Reset button wakes up the KL27 (if asleep) and the KL27 wakes up the nRF52 via SWD reset
- (Off mode only) KL27 wakes up in a reset state
- KL27 detects the wake up source
- If the wake up source is the USB insertion the KL27 sets the COMBINED_SENSOR_INT signal active (or pulses it if already active) to wake up the nRF52
- If the wake up source is the reset button the KL27 halts the nRF52 while button is pressed, and resets the nR52 when the button is released
- nRF52 starts running from reset
- The nRF52 software on start-up should configure all the on-board components to the expected "run mode" and not depend on the component default on-power-up state

### Deep Sleep -> Off or Stand-by

The only way to transition from Sleep to Stand-by is by the user pressing the reset button.

- User starts pressing the reset button
- KL27 wakes up **not** on reset mode
- KL27 detects the wake up source
- While the button is pressed down the KL27 halts nRF52
- When the button is released the KL27 resets the nRF52
- If the user releases the button before the long-press state is reached the KL27 continues with the reset process
- If the user releases the button after the long-press state is reached, it continues with the "On -> Off / Stand-by" process

### Off or Stand-by -> Deep Sleep

Not possible.

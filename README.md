# General purpose

This is 'automatic' switch for a relay. An IR move detector trigs a signal
when it detect a move. A relay is switched on, if ambiant light is higger than a
value. A temporasitation will be set next. When the time gone out, the relay is
switched off.
This will provide a smart light, switching on when the dark rise and you still
present.
A potentiometer adjust the time and the light level.
A life led will tell you the device is alive. It display the difference between
ambiant light and threadsolve, in binary (long = 1, short = 0).

# Configuration
In the ten seconds of startup, turn the potentiometer to adjust time. It can be
beetween 1 and 17 minutes.
After, adjust against the potentiometer until light go on.

# General schematic
See doc repository

*Power supply*
- Usb transformateur

*Input*
- Potentiometer
- Photo-resistance
- IR move detector, trigger high signal when move

*Output*
- Signal to transtior, activation a relay
- Life led to monitor light intensity.

*Logical*
- Atmega85+ Microcontroller unit
- NPN Transitor
- Resitors

*Actuator*
- Relay 5v-230V
- Multiple plugs traffiqued


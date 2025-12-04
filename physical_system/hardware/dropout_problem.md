This document pertains to issues with Maggy V4.2, which as of December 2025 have been discovered to have been introduced somewhere after V2.7.

INTRO
=====
Maggy V4.2 was intended for use in student lab exercises during the fall semester of 2025. However, most of the units were dysfunctional to some degree.
The TLE493D hall effect sensor would experience frequent dropouts, making V4.2 hard or impossible to use. Significant work was put down to uncover the
cause of the problem during the semester, resulting in a several suggested solutions to be implemented in V4.3. The solutions are outlined in the Issues
section of this GitHub repository. This document gives context to the solutions, for the sake of posterity.

THE PROBLEM
===========
**No data is received from the TLV493D-A2B6 hall effect sensor.**
This was the starting point, determined by readouts from Teensy via the Arduino serial monitor.
The Problem then branched into two failure modes:
1) Failure to initalize the sensor on bootup, recognized by an error message output by the Arduino I2C library.
2) Failure during operation. No explicit error messages, but readouts of all axes would go to zero.

Efforts were concentrated on understanding failure mode 2, as no consistent trigger mechanism for mode 1 was found.

INITIAL DEBUGGING
=================




As a reminder, The Problem is that the sensors often do not work, and, secondarily, that the failure mode has been difficult to identify.
There is noise on the I2C and 3V3 lines, spiking when the coils are demagnetised. I assume it is back-emf from the coils.

The noise sometimes makes the signals cross VIH/VIL thresholds, but for such a short time that even the relatively sensitive I2C bus should be able to handle it. Spikes last for a few tens of nanoseconds.

Coil/trace proximity to I2C lines correlates poorly with noise levels. The coil with traces farthest from the I2C lines (X+) seems to produce the most noise.

When The Problem happens, the Teensy keeps sending data requests without getting an ACK or data back.

The Problem can happen immediately after power is plugged into the Maggy, in which case the initialisation function actually discovers it and throws an error which I think is related to configuration of the sensor. Notably, in this case The Problem cannot have been triggered by noise from the coils.

The Problem most commonly happens while the coils are (being) magnetized, presumably somehow triggered by noise. I have not been able to isolate the exact transition point from good sensor output to the sensor hanging.

If bootup is successful, The Problem does not occur so long as the coils are not magnetised; I've had the unit activated and sending data for over an hour uninterrupted.

Bootup is very rarely successful if the Teensy USB is plugged in before the USB-C.

Thus: Noise during operation seems to trigger The Problem, but does not seem to be the full picture.

The currently installed sensor, TLV493D-A1B6, is the cheap consumer version of the first generation of sensors in its family. There are several threads online about it being problematic under certain circumstances that I can't verify, test or mitigate with the software interface/driver I'm using.

Version 2.6 of the Maggy used TLE493D-W1B6, the (slightly more) expensive automotive version of the same sensor without problems.

Conlcusion: I want to buy three of each of TLE..W2B6, TLE..A2B6 and TLI..A2B6. They are the second generation of the same family, with the same footprint/pinout as the current TLV sensor and may be swapped with only minor changes to software. The W2B6 is particularly interesting, as it allows four sensors on the same I2C bus without a mux in between. Highly relevant for MIMO applications we've discussed. It's roughly twice as expensive as the TLV at 1.60$/pc. Noise reduction can and should be implemented as well on the longer term, but a marginally more expensive sensor (in the case of TLI at 0.91$/pc) could be the fast fix we've been looking for.

Additionally, Hans suggested that I look into the current sensors which were introduced in Maggy 4.2. I haven't done that yet. This, as well as power supply strangeness and noise reduction are next on the chopping block after sensor tests.

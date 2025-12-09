This document pertains to issues with Maggy V4.2, which as of December 2025 have been discovered to have been introduced somewhere after V2.7.

## INTRO
Maggy V4.2 was intended for use in student lab exercises during the fall semester of 2025. However, most of the units were dysfunctional to some degree.
The TLE493D hall effect sensor would experience frequent dropouts, making V4.2 hard or impossible to use. Significant work was put down to uncover the
cause of the problem during the semester, resulting in a several suggested solutions to be implemented in V4.3. The solutions are outlined in the Issues
section of this GitHub repository. This document gives context to the solutions, for the sake of posterity.

## THE PROBLEM
**No data is received from the TLV493D-A2B6 hall effect sensor.**  
This was the starting point, determined by readouts from Teensy via the Arduino serial monitor.
The Problem then branched into two failure modes:
1) Failure to initalize the sensor on bootup, recognized by an error message output by the Arduino I2C library.
2) Failure during operation. No explicit error messages, but readouts of all axes would go to zero.

Efforts were concentrated on understanding failure mode 2 (FM2), as no consistent trigger mechanism for failure mode 1 (FM1) was found.


## INITIAL DEBUGGING
Two Maggy 4.2 units were chosen for testing in the first phase: one which had been used in the TTK4111 lab, and one which hadn't.
No obvious or noteworthy differences were found between the two units during testing.
The Problem was found to not occur so long as the solenoids were not activated. This was tested by running the sensortest Arduino script.
The script ran for about an hour on both units without issue (so long as FM1 hadn't occurred), and oscilloscope readings of the I2C bus looked normal.
One of the hover magnets (50mm neodymium disk) was waved aggressively past the sensor occasionally in an attempt at triggering FM2, unsuccessfully.

=> FM2 is probably not triggered by problems inherent in the 5V, 3V3, USBC or Teensy circuits.  
=> FM2 is probably not triggered by the sensor itself being "overloaded" or similar.  
=> The bus has a rise time of about 200 ns, which indicates that bus resistance and capacitance are in line with the I2C specification.

The units were then flashed with the elektromagnettest Arduino script, which activates the solenoids individually at 50%DT for one second at a time.
Oscilloscope readings showed that the I2C and 3V3 lines were affected by spikes of about \pm1.5V at about 66kHz, which coincides with the falling
edges of the solenoids: the A3950 motor driver's PWM output frequency is 66.6kHz. In this configuration, both Maggy units would experience The Problem
occasionally. After The Problem had occured, some combination of repeated reboots and reprogrammings was necessary to resume normal operations,
that is, one instance of FM2 could be followed by multiple instances of FM1. After FM2, I2C scope readings look like one well-formed READ request
from the Teensy followed by a missing ACK bit from the sensor. Attempts at capturing FM2 (i.e. the transition from normal operations to a hung
sensor) in the oscilloscope were unsuccesful. Additionally, the 5V line is not notably affected by noise.

=> FM1 and FM2 may have a common root cause related to the spikes experienced on 3V3 and I2C.  
=> The noise levels are high enough to kick the I2C lines outside of their defined logical voltage levels (Vih=2.3V, Vil=0.99V).  
=> FM2 does probably not affect the Teensy, and **is probably triggered by back-emf from the solenoids**.


This final discovery motivated a closer inspection of the noise/spikes induced by the solenoids. In the list below, the voltage DeltaV represents
the magnitude of the spike relative to the voltage level of the I2C data line, while the frequency is the second order response of the spike itself.
X+): DeltaV = 2.4V @ 78MHz, causes garbled data in the Arduino plotter.  
Y+): DeltaV = 0.9V @ 50MHz  
Y-): DeltaV = 0.8V @ 83MHz  
X-): DeltaV = 1.3V @ 18MHz  
Readouts from the Arduino serial plotter had alredy indicated that X+ was particularly noisy, as data from that solenoid would be garbled in the plotter.
It was initially thought that this was caused by the relevant solenoid lines being close to the I2C lines, but this is not the case. X- and Y+ are
close to the I2C lines in Maggy V4.2, while X+ and Y- are not.

=> There does not seem to be a strong correlation between I2C-solenoid-distance and noise.

### Summary conclusions from the initial debugging campaign:
* FM2 is probably caused by noise induced by back-emf from solenoids.
* FM1 is uncertain.
* Next steps should involve taking measures to reduce noise besides just moving the I2C lines away from the solenoid lines,
  and/or finding a sensor more robust to noise.


## Interlude: New sensors
There was no time to make a new version of the circuit with noise reduction measures, so it was decided that resoldering the existing V4.2 with
other sensors of the same family might be worth the effort. V4.2 uses the TLV493D-A1B6 hall effect sensor, which is the cheaper consumer version
of a family of sensors. V2.7 had used TLE493D-W1B6, the more expensive "automotive" version with unique addressing, without issues.
Could The Problem stem from the sensors being underdimensioned for the noise associated with the solenoids?

As W1B6 was no longer in production, three other models of the next generation were considered:
* TLE493D-A2B6: Automotive version without unique addresses.
* TLE493D-W2B6: Automotive version with unique addresses (up to 4 units per bus), direct successor of W1B6.
* TLI493D-A2B6: Industrial version without unique addresses.

The greatest difference seems to be the price point, with TLI being only slightly more expensive than the original TLV. The datasheets did not
give a clear indication that either sensor would be more or less noise resistant than TLV, but the sensor range is slightly larger for all of them at
160mT vs 135mT for the TLV.

The W2B6's option of having multiple units on the same bus makes it very attractive. The alternative is to reintroduce the multiplexer, which was
in 4.1 on the suspicion that it caused The Problem, or only have one sensor, which is the case for 4.1 and 4.2.

**Three units of each prospective replacement were installed in 9 Maggy 4.2 units.** Poor hand soldering was, unfortunately, a significant source of
uncertainty in the next phase.


## SECOND DEBUGGING



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

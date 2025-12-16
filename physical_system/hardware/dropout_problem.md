This document pertains to issues with Maggy V4.2, which as of December 2025 have been discovered to have been introduced somewhere after V2.7.

## INTRO
Maggy V4.2 was intended for use in student lab exercises during the fall semester of 2025. However, most of the units were dysfunctional to some degree.
The [TLV493D](https://www.infineon.com/part/TLV493D-A1B6) hall effect sensor would experience frequent dropouts, making V4.2 hard or impossible to use.
Significant work was put down to uncover the cause of the problem during the semester, resulting in a several suggested solutions to be implemented 
in V4.3. The solutions are outlined in the Issues section of this GitHub repository. This document gives context to the solutions, for the sake of posterity.

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
* [TLE493D-A2B6](https://www.infineon.com/part/TLE493D-A2B6): Automotive version without unique addresses.
* [TLE493D-W2B6](https://www.infineon.com/part/TLE493D-W2B6-A0): Automotive version with unique addresses (up to 4 units per bus), direct successor of W1B6.
* [TLI493D-A2B6](https://www.infineon.com/part/TLI493D-A2B6): Industrial version without unique addresses.

The greatest difference seems to be the price point, with TLI being only slightly more expensive than the original TLV. The datasheets did not
give a clear indication that either sensor would be more or less noise resistant than TLV, but the sensor range is slightly larger for all of them at
160mT (vs 135mT for the TLV).

The W2B6's option of having multiple units on the same bus makes it very attractive. The alternative is to reintroduce the multiplexer, which was
removed in V4.1 on the suspicion that it caused The Problem, or to only use one sensor, which is the case for V4.1 and V4.2.

**Three units of each prospective replacement were installed in 9 Maggy V4.2 units.** Poor hand soldering was, unfortunately, a significant source of
uncertainty in the next phase.


## SECOND DEBUGGING

Two units which had been used for TTK4111 lab exercises, and one which had not, were resoldered with each of the new sensor types. In total, 9
units were used in this second round of debugging. As before, no differences between the "good" and the "bad" units were observed.
The elektromagnettest, sensortest and PD_v4x Arduino scripts were changed to accommodate the new sensors. This was simply a matter of changing
the sensorType variable to refer to them, rather than the old sensor type.

In brief, using the newer and more expensive sensors did not prove to be a silver bullet against The Problem. It changed characteristics somewhat,
but persisted.

* All new sensors seem to be somewhat more robust against FM2, but not FM1. FM2 now looks like the sensor responding to a READ request by sending
  24 bytes of 0xFF, which the Arduino sensor library interprets as a failure to "read consistent data". This difference is assumed to be related
  to the sensors being of the 2nd generation, but no wholehearted attempt was made to get to the bottom of it.
* All sensors exhibited a variant of FM2, in which the Z axis freezes at -132.99 while the other axes keep transmitting. This corresponds to two
  bytes of 0xFF towards the end of the I2C pulse train, consistent with the sensors' data sheets. Reflashing Maggy would occasionally fix the issue,
  while resetting it (by pressing the button on the Teensy) would not. No particular trigger could be found.
* The new sensors, in particular the TLI units, sometimes fail while running the sensortest script.

This might seem bleak, but some new information has been gained:  
=> The sensors are probably not the source of The Problem.  
=> The software is easily adapted to new sensor types, should it become relevant in the future.  
=> The TLE-W is almost certainly viable as a replacement (or rather, reinstatement), which good news for future MIMO applications.

At this point, it seems almost certain that The Problem, or at least FM2, is caused by disruptions from back-emf bleeding into the I2C and/or 3V3 lines.
The Issues opened in Q4 of 2025 reflect that conclusion, and largely concern strengthening the I2C bus and removing the 3V3 copper fill.

One component/assembly has not been checked out, or even investigated, as of December 2025: The current sensors. Probing them is impractical due to
their size and placement. They introduce 15 mOhms, and, presumably, some capacitance on the line between the drivers and the solenoids.



### Speculations about FM1

No consistent trigger mechanism could be found for FM1, but some observations were made which may strengthen the hypothesis that The Problem is
ultimately caused by disruptions on the 3V3 plane. In order for the solenoids to work at all, the USB-C must be plugged in before the Teensy.
This would seem irrelevant when the sensortest script is flashed, as it should not depend on power to the solenoids. Still, FM1 was observed
more frequently if the USB-C was not plugged in, or plugged in after the Teensy (including resetting the Teensy after USB-C plugin).
For reference, the USB-C power supply is a Raspberry 5 27W power supply.

While exploring this, it was discovered that the I2C and 3V3 lines would spike momentarily as the USB-C was plugged in. The event was
isolated, and discovered to occur the moment the power supply shield meets the shield of the USB-C receptacle on Maggy, irrespective of whether
the USB-C power lines were physically touching. The spike was measured to approximately 12V, lasting for approximately 700 ns. It was shaped
like a typical 2nd order RLC response with a period of approximately 40 ns. This response seems to be dependent on the Teensy **not** being
plugged in: in that case, the spike would last for about 200 ns. Rather than the typical RLC response shape, the voltage then jumps around
randomly (but not exceeding 12V) before stabilizing at 3.3V. Presumably, the randomness has something to do with the Teensy's internal
voltage control(ler) trying to stabilize the voltage.

The spike could **not** definitively be connected to FM1. However, it might seem plausible that a 12V spike could damage the sensor, even if it
only lasts for a few tens of nanoseconds. It does not immediately explain why FM1 happens more frequently if the Teensy is plugged in before the
USB-C, but it might give a clue about the underlying issue: the sensor is getting fried by instability in the 3V3 line, whether it comes from
the power supply assemblies or back-emf from the solenoids.

We have seen how the 3V3 line is affected by both sources. In the case of back-emf, the sensor experiences over- and undervoltages of as much as
1.5 V -- far outside of the absolute maximum rating of the sensor of 3.5V. All this to say that a more robust 3V3 assembly might solve The Problem.

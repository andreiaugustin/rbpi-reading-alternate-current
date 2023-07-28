# Reading alternate current with Raspberry Pi

### Requirements

---

**ACS712** is a Hall effect sensor that can measure alternate or direct current with versions that support up to 5A, 20A and 30A. In this example, a 20A version is used (difference is how many mV per Ampere are reported).

**MCP3008** is a 10-bit ADC (Analog to Digital Converter) that we can use to convert the analog signal produced by the **ACS712** to a digital signal that can be interpreted by a Raspberry Pi through its SPI interface. This is required because Raspberry Pi, at time for writing, cannot directly read analog signals.

**Raspberry Pi** is the single-board computer used for this example, more specifically a Raspberry Pi 3 Model B+.

**Breadboard** is a solderless construction base you can use to build semi-permanent prototypes of electronic circuits. This is probably the easiest way to build your hardware connections.

### Hardware Connections

---

**MCP3008 to Raspberry Pi**

- VDD (MCP3008 pin 16) to 3.3V (Pi pin 1)
- VREF (MCP3008 pin 15) to 3.3V (Pi pin 17)
- AGND (MCP3008 pin 14) to GND (Pi pin 6)
- CLK (MCP3008 pin 13) to GPIO11/SCLK (Pi pin 23)
- DOUT (MCP3008 pin 12) to GPIO9/MISO (Pi pin 21)
- DIN (MCP3008 pin 11) to GPIO10/MOSI (Pi pin 19)
- CS/SHDN (MCP3008 pin 10) to GPIO8/CE0 (Pi pin 24)
- DGND (MCP3008 pin 9) to GND (Pi pin 6)

**ACS712 to MCP3008**

- VCC (ACS712 pin 1) to 5V (Pi pin 2)
- OUT (ACS712 pin 2) to CH0 (MCP3008 pin 1)
- GND (ACS712 pin 3) to GND (Pi pin 6)

Remember to connect the high voltage line you're measuring the IP+ and IP- pins of ACS712. Be extremely cautious when doing this as you're dealing with high voltage.

### Python Code

---

```py title="Simple example"
import time
from gpiozero import MCP3008

# ACS712 20A version characteristics
mV_per_Amp = 100 # use 185 for 5A version and 66 for 30A version

# Create an object for analog channel 0
adc = MCP3008(channel=0)

while True:
    # Read the analog pin (returns a value between 0 (0V) and 1 (3.3V))
    voltage = adc.value * 3.3
    # Shift voltage to match ACS712 output
    shifted_voltage = voltage - 2.5
    # Calculate current
    current = shifted_voltage * 1000 / mV_per_Amp
    print("Current: {} A".format(current))
    time.sleep(0.5)
```

The above script will give us the Amperes going through the wire measured every half-second. Note the values will fluctuate between positive and negative numbers as alternate current changes direction.

To get positive values only, we need to calculate the RMS by taking multiple readings over one complete cycle of the AC waveform (approximately 20ms for a 50Hz system), square them, take the average and then take the square root of this average. However, keep in mind that this would require a sampling rate high enough to accurately represent the AC waveform, which might be challenging to achieve with a Raspberry Pi.

Here's a simplified version of how we could calculate the RMS (we also round to two decimal places when displaying the value):

```py
import time
import math
from gpiozero import MCP3008

# ACS712 20A version characteristics
mV_per_Amp = 100 # use 185 for 5A version and 66 for 30A version

# Number of samples per period
num_samples = 1000

# Sampling period (20ms for 50Hz system)
sampling_period = 20e-3

# Create an object for analog channel 0
adc = MCP3008(channel=0)

while True:
    current_sq_sum = 0
    for _ in range(num_samples):
        # Read the analog pin (returns a value between 0 (0V) and 1 (3.3V))
        voltage = adc.value * 3.3
        # Shift voltage to match ACS712 output
        shifted_voltage = voltage - 2.5
        # Calculate current
        current = shifted_voltage * 1000 / mV_per_Amp
        current_sq_sum += current ** 2
        time.sleep(sampling_period / num_samples)
    # Calculate RMS current
    current_rms = round(math.sqrt(current_sq_sum / num_samples), 2)
    print("Current RMS: {} A".format(current_rms))
```

The above example produces the following results when run with a kettle rated 1850W-2200W on a 230V home system for a few seconds:

```bash
Current RMS: 7.71 A
Current RMS: 7.77 A
Current RMS: 7.71 A
Current RMS: 7.78 A
Current RMS: 7.73 A
Current RMS: 7.67 A
Current RMS: 7.74 A
Current RMS: 7.8 A
Current RMS: 7.74 A
Current RMS: 7.75 A
Current RMS: 7.76 A
Current RMS: 7.78 A
Current RMS: 7.7 A
Current RMS: 7.8 A
Current RMS: 7.73 A
Current RMS: 7.77 A
Current RMS: 7.82 A
Current RMS: 7.75 A
Current RMS: 7.8 A
Current RMS: 7.77 A
Current RMS: 7.78 A
Current RMS: 7.7 A
Current RMS: 7.78 A
Current RMS: 7.8 A
Current RMS: 7.82 A
Current RMS: 7.75 A
Current RMS: 7.69 A
Current RMS: 7.77 A
Current RMS: 7.76 A
Current RMS: 7.75 A
Current RMS: 7.78 A
Current RMS: 7.81 A
Current RMS: 7.76 A
Current RMS: 7.79 A
Current RMS: 7.81 A
Current RMS: 7.73 A
Current RMS: 7.77 A
Current RMS: 7.81 A
```

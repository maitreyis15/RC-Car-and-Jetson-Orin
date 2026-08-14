# RC-Car-and-Jetson-Orin

# RC Car — Jetson Orin Nano Setup

Maitreyi Sarkar and Sayeed Azam

This project uses an **NVIDIA Jetson Orin Nano** to control an RC car using ROS 2. The Jetson will eventually receive data from the camera/LiDAR and an ML model, then directly command:

* **Steering servo**
* **ESC / drive motor**
---

## 1. Current Hardware Setup

### Main components

* NVIDIA Jetson Orin Nano
* RC steering servo
* Electronic Speed Controller (ESC)
* DC motor
* RC battery
* Camera / LiDAR for later autonomy work

### Power layout

```text
                 RC Battery
                    |
             +------+------+
             |             |
             v             |
            ESC            |
             |             |
             v             v
           Motor       Jetson Orin
```

The battery powers the **ESC** and **Jetson Orin** with 14.8 V in a parallel sequence.


**Do not power the drive motor or servo from the Jetson GPIO header.**

---

## 2. GPIO Wiring

### Steering Servo

Current steering signal:

```text
Servo Signal  -> Jetson physical pin 33
Servo Ground  -> Jetson GND
```

The steering servo successfully responds directly to the Jetson's **3.3 V PWM signal**.

### ESC

Current intended ESC control connection:

```text
ESC Signal -> Jetson GPIO
ESC Ground -> Jetson GND
ESC Power -> Battery Pack
```

Ground is extremely important. The ESC signal ground must be grounded using the pinout of the Jetson Orin.

Ground pins such as **physical pin 34 or pin 39** can be used.

Our current wiring diagram uses **physical pin 15 / GPIO 27** for the ESC control signal.

---

# 3. Steering Servo Test

We first tested the servo without ROS using `Jetson.GPIO`.

The servo uses approximately **50 Hz PWM**, where a roughly **1.5 ms pulse / 7.5% duty cycle** represents center.

```python
import Jetson.GPIO as GPIO
import time

SERVO_PIN = 33

GPIO.setmode(GPIO.BOARD)
GPIO.setup(SERVO_PIN, GPIO.OUT)

pwm = GPIO.PWM(SERVO_PIN, 50)

try:
    # 7.5% ~= 1.5 ms pulse at 50 Hz
    # This should place the steering near center.
    pwm.start(7.5)

    print("PWM running on pin 33 for 3 seconds...")
    time.sleep(3)

finally:
    pwm.stop()
    GPIO.cleanup()
    print("PWM stopped and GPIO cleaned up.")
```

This test worked successfully.

---

# 4. Steering Through ROS 2

After confirming the hardware worked directly, we created a ROS 2 steering node.

The node successfully starts with output similar to:

```text
[INFO] [steering_node]: Steering node started!
[INFO] [steering_node]: Servo PWM active on physical pin 33
```

We also tested steering commands such as:

```text
-0.25
```

and confirmed that the wheels responded correctly.

The steering center is already calibrated reasonably well.

The basic control flow is:

```text
ROS 2 Steering Command
        |
        v
 steering_node
        |
        v
PWM on Jetson Pin 33
        |
        v
 Steering Servo
```

---

# 5. ESC / Motor PWM Test

The ESC also expects an RC-style PWM signal at approximately **50 Hz**.

A neutral command is approximately:

```text
Pulse width: 1.5 ms
Duty cycle:  7.5%
Frequency:   50 Hz
```

A basic motor/ESC test follows the same idea:

```python
import Jetson.GPIO as GPIO
import time

# Current intended ESC signal pin
ESC_PIN = 15

GPIO.setmode(GPIO.BOARD)
GPIO.setup(ESC_PIN, GPIO.OUT)

pwm = GPIO.PWM(ESC_PIN, 50)

try:
    # Neutral ESC command
    pwm.start(7.5)

    print("Sending neutral PWM to ESC...")
    time.sleep(3)

finally:
    pwm.stop()
    GPIO.cleanup()
    print("ESC PWM stopped.")
```

**Do not test throttle with the wheels on the ground.** Raise the drive wheels before experimenting with motor commands.

---

# 6. Important ESC Discovery

We spent quite a bit of time troubleshooting the ESC because the PWM timing looked correct and was validated with the waveform application and DAD device, but the ESC would not properly recognize the Jetson command.

Using the oscilloscope, we confirmed a signal around:

```text
Positive duty: 7.42%
Positive width: 1.48 ms
```

That is essentially the expected **1.5 ms neutral signal**.

The important discovery was the **signal voltage**.

The Jetson GPIO outputs approximately:

```text
3.3 V HIGH
```

When we generated approximately the same **7.5% / 1.5 ms PWM signal at 5 V**, the ESC successfully recognized it as **neutral**.

Therefore, the main remaining ESC issue appears to be:

```text
Jetson PWM timing:  GOOD
Jetson signal:      3.3 V
ESC prefers:        ~5 V
```

### Planned solution

Add a **3.3 V → 5 V logic-level shifter / buffer** between the Jetson PWM output and ESC signal input.

The final signal chain should therefore look like:

```text
Jetson GPIO
   |
   | 3.3 V PWM
   v
3.3 V -> 5 V Level Shifter
   |
   | 5 V PWM
   v
ESC Signal Input
   |
   v
Motor
```

The Jetson and ESC grounds must still be connected.
The power for the buffer will be generate from the exposed PWM pins on the ESC.

---



# 7. Current Project Status

**Working:**

* Jetson boots and runs normally
* Network connection configured
* ROS 2 installed
* Jetson GPIO control working
* 50 Hz PWM verified
* Steering servo working
* Steering ROS 2 node working
* Steering commands tested
* Servo center calibrated
* ESC PWM timing verified with oscilloscope
* ESC recognizes a 5 V, ~7.5% duty-cycle signal as neutral

**Next steps:**

1. Add a **3.3 V → 5 V level shifter** to the ESC PWM line.
2. Verify neutral from the Jetson through the level shifter.
3. Carefully test forward/reverse throttle ranges.
4. Create/finalize the ROS 2 motor node.
5. Combine steering and throttle control.
6. Integrate LiDAR, camera, and autonomous driving logic.

---

## Useful PWM Reference

At **50 Hz**, one PWM period is:

```text
20 ms
```

Typical RC control values are approximately:

```text
1.0 ms pulse -> 5.0% duty
1.5 ms pulse -> 7.5% duty -> Neutral / Center
2.0 ms pulse -> 10.0% duty
```

Exact throttle limits should be determined carefully once the ESC is reliably receiving the 5 V control signal.

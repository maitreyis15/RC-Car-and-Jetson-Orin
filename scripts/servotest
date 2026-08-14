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

# Automatic guitar tuner (ESP32-S3)

My Bachelor's thesis in Robotics at Transilvania University of Brașov (2026).

You pick a string on a web page, pluck it, and the system tunes it. A microphone picks up the sound, the ESP32-S3 works out the pitch with an FFT, and a stepper motor turns the tuning key in small steps until the string is within ±5 cents of the target. It tunes one string at a time.


https://github.com/user-attachments/assets/b50fb336-e9bb-4ef3-86e0-c03244413f5c


<img width="974" height="531" alt="image" src="https://github.com/user-attachments/assets/4c95883b-36ac-498e-89cf-2cf0a76abe4a" />


## How it works

```
string -> MAX4466 mic -> ADC, 8 kHz -> Hamming window + FFT -> parabolic interpolation
                                                                        |
tuning key <- NEMA 17 <- Nanotec C5-01 <- step controller <- median -> IIR -> Kalman
```

The microphone is read by the ESP32's 12-bit ADC at 8 kHz, which covers the fundamentals of all six strings (82 to 330 Hz) with room to spare. I had to put it on an ADC1 pin, because ADC2 stops working on the ESP32-S3 while WiFi is on, and the web interface needs WiFi the whole time.

Each block of samples goes through a Hamming window and an FFT. A plain FFT peak is only accurate to one frequency bin, which isn't good enough for tuning, so the firmware fits a parabola through the peak and its two neighbours to get a finer estimate.

Even then the readings jumped by a few Hz from one block to the next. They go through three filters in a row: a median filter throws out the odd wild value, an IIR filter smooths what's left, and a scalar Kalman filter gives the final estimate.

The error is converted to cents and the controller decides how many steps to send. It's a stepped proportional controller: big corrections when the string is far off, small ones near the target. Each correction is a little smaller than the error on purpose, so the string creeps up on the target instead of overshooting it. The thesis explains why I went with this instead of a full PID.

The motor is a NEMA 17 (17HM19-2004S) driven by a Nanotec C5-01 industrial driver, connected to the tuning key through a coupling I designed in CATIA and 3D printed.

## Results

Measured on the real system, string A2 (110 Hz):

| | Raw FFT | After filtering |
|---|---|---|
| Standard deviation | 2.30 Hz (about 34 cents) | 0.32 Hz (about 5 cents) |
| Max deviation | 6.1 Hz | 0.5 Hz |

Once the string is in tune, the filtered reading stays within 1 cent, so the system doesn't flicker in and out of the "tuned" state around the ±5 cent limit.

Motor calibration, also measured: about 50 motor steps per cent. I set the delay between step pulses to 3500 µs. Faster than that and the motor skipped steps under the load of the tuning key.

Tuning all six strings, simulated in Simulink with the calibrated parameters:

| String | Target (Hz) | Final error (cents) | Time (s) | Corrections |
|---|---|---|---|---|
| E2 | 82.41 | +1.1 | 11.4 | 9 |
| A2 | 110.00 | −0.9 | 8.7 | 7 |
| D3 | 146.83 | +0.9 | 7.9 | 6 |
| G3 | 196.00 | −1.1 | 7.2 | 6 |
| B3 | 246.94 | +1.0 | 6.8 | 5 |
| E4 | 329.63 | +0.9 | 6.1 | 5 |

The thinner strings tune faster because the same turn of the key changes their pitch more.

The ESP32 sent its readings over serial, and I did the plots and statistics in MATLAB.

## Limitations

The 50 steps per cent figure belongs to this guitar and this coupling. A different instrument needs a new calibration. In a very noisy room the threshold that rejects false detections has to be raised. And with one motor, it tunes one string at a time.

## Hardware

- ESP32-S3 dev board: sampling, FFT, filtering, control and the web server
- MAX4466 microphone module
- NEMA 17 stepper motor, 17HM19-2004S
- Nanotec C5-01 driver (step/direction, phase current set to 1700 mA)
- LM2596 buck converter, 12 V to 5 V
- 3D-printed coupling between the motor and the tuning key

## Files

- `acordor_automat.ino`: the firmware, about 800 lines (acquisition, FFT, filters, controller, web server)
- `Documentatie_licenta_Acordor_FUGULIN_CLAUDIU-FLORIAN.docx`: the full thesis, in Romanian
- `Prezentare_Licenta_Acordor_Fugulin.pptx`: defense slides, in Romanian
- `Media1.mp4`: video of the prototype working

Claudiu Fugulin · [LinkedIn](https://www.linkedin.com/in/claudiufugulin)

https://github.com/user-attachments/assets/cb06d90b-43bc-4ac8-b656-3ec1be6e7d72


# Raspberry-Pi-Power-Button
Raspberry Pi Power Button (GPIO Shutdown + Wake)
This project documents how to wire and configure a momentary push-button on a Raspberry Pi to provide:
- Power on (wake from halt)
- Clean shutdown
- LED feedback using the ACT LED on a custom GPIO pin

It is based on the Raspberry Pi’s built-in `gpio-shutdown` overlay and works with DietPi, Raspberry Pi OS, and other Debian-based distros.

## Features

- Press button → clean, safe shutdown
- Press button again → boot / wake from halt
- Simple single-wire GPIO connection
- Very low resource usage (handled by the kernel via device tree)
- Optional: ACT LED remapped to the button LED for a heartbeat indicator

## Hardware Setup
### Button wiring

Your setup uses a simple normally-open momentary pushbutton:

| Pi Pin	Function	| Notes |
|-------------------|-------|
| Pin 5 (GPIO3)	  | Shutdown / power-on signal	Internal pull-up enabled by overlay |
| Pin 6 (GND)	    | Button ground |	

This is enough for both shutdown and wake-from-halt.
The Pi monitors GPIO3 during halt state and will power back up if grounded.

### Button LED

Your illuminated button’s LED is powered separately and tied to:

- GPIO14 (pin 8) — ACT LED output (via overlay)
- Ground (pin 6)

The LED brightness varies during boot, then follows the ACT heartbeat.

## Software Setup
### 1. Enable the shutdown + wake functionality

Edit `/boot/config.txt`:
```
dtoverlay=gpio-shutdown,gpio_pin=3,active_low=1,gpio_pull=up
```

Explanation:
- `gpio_pin=3` → use GPIO3 (mandatory for wake)
- `active_low=1` → button pulls the line to ground
- `gpio_pull=up` → enable internal pull-up resistor
- 
After reboot:
- Press button → system shuts down cleanly
- Once halted → press again → system powers on

No service or script needed — handled by the kernel.

### 2. Enable ACT LED heartbeat on GPIO14 (your button LED)

Add to `/boot/config.txt`:
```
dtoverlay=pi3-act-led,gpio=14
```

Then create the udev rule:
```
sudo nano /etc/udev/rules.d/99-led.rules
```

Put inside:
```
ACTION=="add", SUBSYSTEM=="leds", KERNEL=="ACT", ATTR{trigger}="heartbeat"
```

This sets the ACT LED to the “heartbeat” pattern once the system reaches multi-user target.

Apply the new rule:
```
sudo udevadm control --reload-rules
sudo udevadm trigger
```

## Testing

### Check shutdown handling:
```
sudo poweroff
```

Then press your button: system should power on.

### Check wake-from-halt:

Once halted (PWR LED solid), press the button.
GPIO3 is monitored by the PMIC and will trigger a wake event.

Check LED trigger:
```
cat /sys/class/leds/ACT/trigger
```

You should see `heartbeat` highlighted.

## Troubleshooting
### Button does not shut down

- Ensure wiring: GPIO3 to GND
- Ensure overlay is active:
```
grep -i gpio-shutdown /boot/config.txt
```
### Button does not power on after shutdown

- Wake-from-halt only works with GPIO3
- Do not move shutdown pin to any other GPIO

### ACT LED does not blink on GPIO14

- Ensure you disabled any conflicting LED overlays
- Ensure no service is overwriting LED triggers
- Check udev rule was applied:
```
sudo udevadm trigger
```

## Why GPIO3?

It is the only GPIO monitored during halted state.
The Pi’s PMIC keeps GPIO3 alive to detect a ground event and trigger power-up.
All other GPIOs are inactive while halted.

So if you want press button → Pi wakes up, GPIO3 is mandatory.

## Files Modified
| File |	Purpose |
|---|---|
| /boot/config.txt	| Enables gpio-shutdown overlay and ACT LED remapping |
| /etc/udev/rules.d/99-led.rules	| Applies heartbeat trigger to ACT LED |

No systemd services required.

## Example of Final `/boot/config.txt` section
```
# Power button on GPIO3
dtoverlay=gpio-shutdown,gpio_pin=3,active_low=1,gpio_pull=up

# ACT LED remapped to GPIO14
dtoverlay=pi3-act-led,gpio=14
```

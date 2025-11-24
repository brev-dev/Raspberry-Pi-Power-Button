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

My setup uses a simple normally-open momentary pushbutton:

| Pi Pin	Function	| Notes |
|-------------------|-------|
| Pin 5 (GPIO3)	  | Shutdown / power-on signal	Internal pull-up enabled by overlay |
| Pin 6 (GND)	    | Button ground |	

This is enough for both shutdown and wake-from-halt.
The Pi monitors GPIO3 during halt state and will power back up if grounded.

### Button LED

My illuminated button’s LED is powered separately and tied to:

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

### 2. Enable ACT LED heartbeat on GPIO14 (my button LED)

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

## Alternative LED Trigger Modes (ACT LED on GPIO14)

The ACT LED (which I remapped to GPIO14 to drive my button’s LED) supports several different trigger modes. These affect how the LED behaves and can be used to indicate different system states.

Once the LED is attached to `/sys/class/leds/ACT`, triggers can be viewed by running:
```
cat /sys/class/leds/ACT/trigger
```

You will see a list like:
```
none rc-feedback rfkill-any netdev timer heartbeat default-on mmc0 ...
```

One will be wrapped in brackets, e.g.:
```
heartbeat [default-on]
```

To select a trigger:
```
echo <trigger-name> | sudo tee /sys/class/leds/ACT/trigger
```

Below are the most practical and meaningful modes for a power button LED.

### 1. `heartbeat` (recommended)
```
echo heartbeat | sudo tee /sys/class/leds/ACT/trigger
```

- Pulses like a heartbeat (two short flashes, pause)
- Indicates the system is alive and not crashed
- Works well on LEDs that should still be active after boot

Ideal for:
Always-on devices, enclosure power button LEDs, servers.

### 2. `default-on`
```
echo default-on | sudo tee /sys/class/leds/ACT/trigger
```

- LED stays solid on once the system reaches userland
- Very simple “system is powered” indicator

Ideal for:
Users who want a clear, static “on” signal.

3. `mmc0`
```
echo mmc0 | sudo tee /sys/class/leds/ACT/trigger
```

- LED flashes on SD card / filesystem I/O
- On older Pis this was the classic ACT LED behaviour

Ideal for:
Debugging storage activity
Custom NAS setups
People wanting nostalgic Pi LED behaviour

### 4. `timer`
```
echo timer | sudo tee /sys/class/leds/ACT/trigger
```

You can configure period and duty cycle:
```
echo 500 | sudo tee /sys/class/leds/ACT/delay_on
echo 500 | sudo tee /sys/class/leds/ACT/delay_off
```

- LED blinks at a fixed rate
- Fully customizable blink speed

Ideal for:
Visual heartbeat alternative
Status signalling
Troubleshooting states

### 5. `none` (manual control)
```
echo none | sudo tee /sys/class/leds/ACT/trigger
```

Then set LED manually:
```
echo 1 | sudo tee /sys/class/leds/ACT/brightness
echo 0 | sudo tee /sys/class/leds/ACT/brightness
```

This allows you to script LED behaviour (Python, Bash, systemd services).

Ideal for:
Projects needing programmatic LED control.

### 6. `netdev` (network activity)
```
echo netdev | sudo tee /sys/class/leds/ACT/trigger
```

Then configure the interface:
```
echo eth0 | sudo tee /sys/class/leds/ACT/device_name
```

- LED flashes with network RX/TX
- Very useful for NAS or Pi-hole setups

Ideal for:
Network appliances
Routers
NAS devices
Pi-hole indicators

### Choosing the Best Trigger for a Power Button LED
|Trigger	|Meaningful On/Off?	|Noise?	|Use case|
|-|-|-|-|
|`heartbeat`|	✔|	Subtle pulses|	System is alive, not crashed|
|`default-on`|	✔|	None|	Simple "power on" indicator|
|`mmc0`|	Partial|	Busy blinking|	Storage activity|
|`netdev`|	Partial|	Busy blinking|	Network activity|
|`timer`|	✔|	Custom|	Status indicator|
|`none`|	N/A|	User-defined|	Full custom control|

For my NASPi enclosure:

- `heartbeat` gives the best crash-detection and “alive” indicator
- `default-on` is simpler if you just want a solid illumination
- `mmc0` can be nice if you want to know when the HDD/SD is being accessed
- `netdev` is good if you want “LAN activity lights”
- `timer` works for custom states or fault indicators



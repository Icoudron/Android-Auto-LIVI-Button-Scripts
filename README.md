# Android-Auto-LIVI-Button-Scripts
This repository contains scripts and instructions to program buttons to specific key binds that LIVI uses for controls in Android Auto

## Shutdown (GPIO 22) & Back Button (GPIO 13)

1. Open firmware configuration file
``` bash
    sudo nano /boot/firmware/config.txt
```
2. Add these two lines at the bottom under [all]:
``` ini
# Safe Shutdown on GPIO 22 (sends KEY_POWER to initiate clean shutdown)
dtoverlay=gpio-shutdown,gpio_pin=22,active_low=1,gpio_pull=up

# Dedicated Back Button on GPIO 13 (maps to KEY_BACKSPACE / keycode 14)
dtoverlay=gpio-key,gpio=13,keycode=14,label="AA_BACK"
```

## Display Brightness Toggle (GPIO 12)
For this project I am using raspberry pi Touch display
check how your screen controls brightness by listing your backlight device in the terminal

``` bash
ls /sys/class/backlight/
```
1. Create the Toggle Script
``` bash
sudo nano /usr/local/bin/brightness_toggle.py
```
2. update the following code (update BACKLIGHT_DIR, DAY_BRIGHTNESS, and NIGHT_BRIGHTNESS to match your setup):
``` python
#!/usr/bin/env python3
import os
from gpiozero import Button
from signal import pause

# --- CONFIGURATION ---
GPIO_PIN = 12
# Replace with the folder name from 'ls /sys/class/backlight/'
BACKLIGHT_DIR = "/sys/class/backlight/10-0045" 

# Set your two brightness levels (check max_brightness in that folder, usually 255 or 100)
DAY_BRIGHTNESS = "200"
NIGHT_BRIGHTNESS = "40"
# ---------------------

brightness_file = os.path.join(BACKLIGHT_DIR, "brightness")

def toggle_brightness():
    try:
        with open(brightness_file, "r") as f:
            current = int(f.read().strip())

        # Toggle between Day and Night values
        new_val = NIGHT_BRIGHTNESS if current > int(NIGHT_BRIGHTNESS) else DAY_BRIGHTNESS

        with open(brightness_file, "w") as f:
            f.write(new_val)
    except Exception as e:
        print(f"Brightness toggle error: {e}")

# bounce_time=0.2 prevents double-triggers from momentary switch bounce
btn = Button(GPIO_PIN, pull_up=True, bounce_time=0.2)
btn.when_pressed = toggle_brightness

pause()
```
3. Make it executable
```bash
sudo chmod +x /usr/local/bin/brightness_toggle.py
```
4. Run It Automatically on Boot
create the service file
```bash
sudo nano /etc/systemd/system/brightness-button.service
```
add this configuration
``` ini
[Unit]
Description=GPIO Brightness Toggle Button
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/brightness_toggle.py
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```
Enable the service and reboot to apply both the overlays and the service:
``` bash
sudo systemctl enable brightness-button.service
sudo reboot
```

# Android-Auto-LIVI-Button-Scripts
This repository contains scripts and instructions to program physical GPIO buttons to control power, display brightness, and LIVI key binds in Android Auto on a Raspberry Pi 5.

## Wiring Overview
Connect one leg of each momentary push button to its assigned GPIO pin and the other leg to any Ground (`GND`) pin on the 40-pin header:
* **Shutdown Button:** GPIO 22 (Physical Pin 15) + GND
* **LIVI Control Button:** GPIO 13 (Physical Pin 33) + GND
* **Display Brightness Toggle:** GPIO 12 (Physical Pin 32) + GND

---

## 1. LIVI Control Button (GPIO 13)
Instead of `gpio-shutdown` (which triggers the desktop shutdown confirmation popup on Raspberry Pi OS), GPIO 22 is handled by a dedicated Python service below, while GPIO 13 uses a `gpio-key` device tree overlay to send keyboard shortcuts to LIVI.

1. Open the firmware configuration file:
   ```bash
   sudo nano /boot/firmware/config.txt
   ```
2. Add your chosen `gpio-key` line at the bottom under `[all]` (do **not** add `dtoverlay=gpio-shutdown` on GPIO 22):
   ```ini
   # LIVI Control Button on GPIO 13
   # Common LIVI keycodes:
   #   50  = KEY_M (Google Assistant / Voice Command)
   #   164 = KEY_PLAYPAUSE (Play / Pause Media)
   #   35  = KEY_H (Home / Dashboard)
   #   14  = KEY_BACKSPACE (Back)
   dtoverlay=gpio-key,gpio=13,keycode=50,label="AA_VOICE"
   ```
3. Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## 2. Immediate Shutdown Button (GPIO 22)
Using a `systemd` service calling `systemctl poweroff` directly bypasses the Raspberry Pi OS desktop shutdown menu for an instant, clean shutdown on a single press.

1. Create the shutdown script:
   ```bash
   sudo nano /usr/local/bin/shutdown_button.py
   ```
2. Paste the following code:
   ```python
   #!/usr/bin/env python3
   import subprocess
   from gpiozero import Button
   from signal import pause

   def power_off():
       subprocess.run(["/usr/bin/systemctl", "poweroff"], check=False)

   # bounce_time=0.2 prevents accidental double triggers
   btn = Button(22, pull_up=True, bounce_time=0.2)
   btn.when_pressed = power_off

   pause()
   ```
3. Make it executable:
   ```bash
   sudo chmod +x /usr/local/bin/shutdown_button.py
   ```
4. Create the `systemd` service file:
   ```bash
   sudo nano /etc/systemd/system/shutdown-button.service
   ```
5. Add this configuration:
   ```ini
   [Unit]
   Description=GPIO Direct Shutdown Button
   After=multi-user.target

   [Service]
   Type=simple
   ExecStart=/usr/bin/python3 /usr/local/bin/shutdown_button.py
   Restart=always
   RestartSec=3

   [Install]
   WantedBy=multi-user.target
   ```
6. Enable the service:
   ```bash
   sudo systemctl enable shutdown-button.service
   ```

---

## 3. Display Brightness Toggle (GPIO 12)
For this project I am using a Raspberry Pi Touch Display. The script automatically detects the active backlight folder under `/sys/class/backlight/` (in case the I2C bus number changes across reboots) and clamps values to the display's `max_brightness`.

1. Create the toggle script:
   ```bash
   sudo nano /usr/local/bin/brightness_toggle.py
   ```
2. Paste the following code (adjust `DAY_BRIGHTNESS` and `NIGHT_BRIGHTNESS` to match your preference):
   ```python
   #!/usr/bin/env python3
   import glob
   import os
   from gpiozero import Button
   from signal import pause

   # --- CONFIGURATION ---
   GPIO_PIN = 12
   DAY_BRIGHTNESS = "200"
   NIGHT_BRIGHTNESS = "40"
   # ---------------------

   def get_brightness_file():
       paths = glob.glob("/sys/class/backlight/*/brightness")
       return next(iter(paths), None)

   def toggle_brightness():
       brightness_file = get_brightness_file()
       if not brightness_file:
           print("Error: No backlight device found in /sys/class/backlight/", flush=True)
           return

       try:
           with open(brightness_file, "r") as f:
               current = int(f.read().strip())

           max_file = os.path.join(os.path.dirname(brightness_file), "max_brightness")
           with open(max_file, "r") as f:
               max_val = int(f.read().strip())

           day_val = min(int(DAY_BRIGHTNESS), max_val)
           night_val = min(int(NIGHT_BRIGHTNESS), max_val)

           new_val = str(night_val if current > night_val else day_val)

           with open(brightness_file, "w") as f:
               f.write(new_val)
           print(f"Set brightness to {new_val} ({brightness_file})", flush=True)
       except Exception as e:
           print(f"Brightness toggle error: {e}", flush=True)

   # bounce_time=0.2 prevents double-triggers from momentary switch bounce
   btn = Button(GPIO_PIN, pull_up=True, bounce_time=0.2)
   btn.when_pressed = toggle_brightness

   pause()
   ```
3. Make it executable:
   ```bash
   sudo chmod +x /usr/local/bin/brightness_toggle.py
   ```
4. Create the `systemd` service file (configured to wait for `graphical.target` so the display driver is ready on boot):
   ```bash
   sudo nano /etc/systemd/system/brightness-button.service
   ```
5. Add this configuration:
   ```ini
   [Unit]
   Description=GPIO Brightness Toggle Button
   After=graphical.target systemd-backlight@.service
   Wants=graphical.target

   [Service]
   Type=simple
   ExecStartPre=/bin/sleep 3
   ExecStart=/usr/bin/python3 /usr/local/bin/brightness_toggle.py
   Restart=always
   RestartSec=3

   [Install]
   WantedBy=graphical.target
   ```
6. Enable the service and reboot to apply the overlay and both background services:
   ```bash
   sudo systemctl enable brightness-button.service
   sudo reboot
   ```

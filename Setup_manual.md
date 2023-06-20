# Install OS
1. Install Raspberry OS Lite on SD card. Configure user Pi, Wifi and SSH. _Raspberry Pi Imager is highly recommended - use Ctrl-Shift-X to open Settings._
1. Boot RPi. Find IP adress, on screen or DHCP server.
1. ssh pi@192.168.0.27 (replace with your IP). _To reset fingerprint for SSH: `ssh-keygen -R 192.168.0.27`_

All actions below are on/in the pi.

1. `sudo raspi-config` > System > Network at boot > No _Speeds up boot._

# WiringPi and safe shutdown button
1. Install WiringPi (for GPIO):
    - `wget https://project-downloads.drogon.net/wiringpi-latest.deb`
    - `sudo dpkg -i wiringpi-latest.deb`
    - `gpio -v` _Should show version 2.52 or later._
1. Attach a safe shutdown button:
    - Attach button: Pin 5 --> N/O push-button > GND.
    - `sudo nano /usr/local/bin/safeshutdown.sh` > Insert:
        ```bash
            #!/bin/bash
            #GPIO3 = Pin5. When Pi=Off: Shorting to GND --> On. This script: Shorting to ground --> Off.
            gpio -g mode 3 up #Pull-up
            gpio -g wfi 3 falling #Wait-for-edge (w/o looping)
            sudo poweroff
        ```
    - `sudo chmod a+x /usr/local/bin/safeshutdown.sh`
    - `sudo nano /etc/rc.local` > Insert before "exit 0": 
        `/usr/local/bin/safeshutdown.sh &` _Run script in backgrund._
    - Test shutdown script first time: `sudo /usr/local/bin/safeshutdown.sh &` > Press button to shut down.
    - Test startup: Push button to start pi. _rc.local is now running safeshutdown.sh in the background, so next press should power off._

# Build and test a simple PWM driver for DC vibrator
1. Attach and try PWM. _Gpio 18 is physical pin 12._
    - Attach gpio to modulate transistor: Pin 12 --> 2,2kohm --> B@BC547C --> E@BC547C --> fuse --> GND
    - Attach vibrator to transistor: +5V (not from pi!) --> Capacitor + Flyback-diode + Vibrator --> C@BC547C
    - `gpio -g mode 18 pwm` _Default pwm is Dynamic mode,  0...1024._
    - `gpio -g pwm 18 500 && sleep 1 && gpio -g pwm 18 0` _Run for 1s at about 50% Duty Cycle._

# Install Node-JS server and Node-Red
_Based on https://nodered.org/docs/getting-started/raspberrypi, including tweaks..._
1. Download and install:
    ```bash
        curl https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered --output noderedinstaller.sh
        bash noderedinstaller.sh --confirm-pi --allow-low-ports --confirm-install
    ```
1. _Be patient. Especially over SSH - connection might drop while install is actually going just fine._
1. Answer No to "customize settings now". 
1. Start Node-red manually one time to try it out: `node-red-pi --max-old-space-size=256`
1. Browse to http://192.168.0.27:1880 (your IP here). _You should now see the Node-red flow editor, the "backend"._
1. Stop Node-red with Ctrl-C
1. Configure HTTP to port 80 and format json properly: `nano ~/.node-red/settings.js` > Set 
    - `flowFilePretty: true` _Good for export/editing._
    - `uiPort: process.env.PORT || 80` _Default port for all browsers._
    - `httpAdminRoot: '/admin'` _More intuitive, since frontent is '/ui' by default_
    - `debugUseColors: true` _Small tweak, but helpful._
1. Autostart Node-red on next boot: `sudo systemctl enable nodered.service`
    - _Note: The parameter `--max-old-space-size=256` seems to be default for running as a service, at least after RPi install script._
1. Reboot RPi (using the fancy safe-shutdown-button).

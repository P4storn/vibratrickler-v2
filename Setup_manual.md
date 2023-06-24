# About this document
- I try to consistently use physical pin numbers on the RPi - i.e. how your would count them on the board.

# Install OS
1. Install Raspberry OS Lite on SD card. Configure user Pi, Wifi and SSH. _[Raspberry Pi Imager](https://downloads.raspberrypi.org/imager/imager_latest.exe) is highly recommended - use Ctrl-Shift-X to open Settings._
1. Boot RPi. Find IP adress, on screen or DHCP server.
1. ssh pi@192.168.0.27 (replace with your IP). _To reset fingerprint for SSH: `ssh-keygen -R 192.168.0.27`_

All actions below are on/in the pi.

1. `sudo raspi-config` > System > Network at boot > No _Speeds up boot._

# WiringPi and safe shutdown button
Creds to @drogon for this library! Lots more info at http://wiringpi.com/
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

# Connect scale (Load cell + Hx711)
Green/white wire from wheatstone bridge load cell beam will depend on type and how you mount it. The colors might also be completely different, but +/- should at least always be Red/Black. If you get negative values from Hx711: Mount your load cell 'upside down' or change green/white connections to Hx711. RPi GPIO-pins could be almost any, but below are the ones I know work.
1. Connect wheatstone type load cell to HX711 board,
    - Red wire --> Hx711:E+ _Excitement current +_
    - Black wire --> Hx711:E- _Excitement current -_
    - Green wire --> Hx711:A+ _Channel A +_
    - White wire --> Hx711:A- _Channel A -_
1. Connect Hx711 to RPi,
    - GND --> breadboard ground
    - DT --> RPi pin 29
    - SCK --> RPi pin 31
    - VCC --> breadboard 5V

# Import and test HX711 A/D conversion
Creds to Gandalf15 for the Hx711 library at https://github.com/gandalf15/HX711. More examplesfrom him [here](https://github.com/gandalf15/HX711/blob/master/python_examples/all_methods_example.py)
1. `sudo apt-get update` _Refresh Git hooks_
1. `sudo apt-get install git -y` _Install Git_
1. `pip3 install 'git+https://github.com/gandalf15/HX711.git#egg=HX711&subdirectory=HX711_Python3'` _Pip-install from Gandalf15's Git_
1. `python3` _Open a Python3 console_
    - `import RPi.GPIO as gpio`
    - `from hx711 import HX711`
    - `gpio.setmode(gpio.BOARD)` _Use physical board pin numbers. Hint/try: `gpio readall` for a full conversion table_
    - `hx = HX711(dout_pin=29, pd_sck_pin=31)`  _Create an hx711 object using the pins you connected before. Default settings include Channel=A and Gain=128/max, which suits this project just fine._
    - `hx.reset()` _False if OK (sic!). Hx711 board is working._
    - `hx.zero(30)` _False if OK (sic!). Take the mean of 50 readings and use this as 0/floor - "Tare" on a kitchen scale._
    - `hx.get_data_mean(5)` _Try getting some data! (5) will retreive 5 readings and return their average. If 4 readings fail - this will throw 'statistics.StatisticsError'._
    - Next, run a loop to get a bit closer to brass tacks:
        ```python
            while True:
                hx.get_data_mean(1)
        ```
    - Or log a lot of readings to a csv-readable file:
        ```python
            from datetime import datetime
            file = open("hx711.txt", "a")
            while True:
                timestring = datetime.now().strftime('%H:%M:%S')
                hxstring = hx.get_data_mean(1)
                outstring =  str(timestring) + ',' + str(hxstring) + '\n'
                file.write(outstring)
                print(outstring)
        ```
1. Always do `file.close()` after playing with files!
1. Always do `gpio.cleanup()` after playing with GPIO-pins!

Hint: Save the readings log file to Windows PC, over SSH, from RPi: Windows CMD.exe > `scp pi@192.168.0.27:~/hx711.txt .\hx711.txt`

## What output to expect? 
I'm using a 20g load cell (itsy, bitsy teeny weeny!) and about 45% of readings from the hx.get_data_mean(1) loop above is useful. Each grain on the scale gives approx 1950 increase in Hx711 output.

# Install Node-JS server and Node-Red
Based on https://nodered.org/docs/getting-started/raspberrypi, including some tweaks...
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
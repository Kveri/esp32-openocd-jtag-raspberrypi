# Guide
This is a guide how to make JTAG debugging of ESP-32 using OpenOCD via RaspberryPI for Visual Studio Code.

I had an unused RPi laying around and I wanted to use it for realtime debugging of ESP-32, because I was tired of debugging using just with Serial.print.

# Overview
We'll be using OpenOCD as a gdb server. OpenOCD will communicate with the ESP-32 board using JTAG protocol. VSCode gdb will talk to OpenOCD. Additionally we'll use esptool.py to flash the board. You can OpenOCD for flashing over JTAG, but I found it more difficult to setup.

# RPi setup
First we need to setup stuff on RPi.

1. flash standard ubuntu image onto RPi sdcard using native ubuntu instructions. I recommend using ubuntu server.
2. connect RPi to the network, I recommend using ethernet cable for stability
3. once in ubuntu, install required packages:
```
sudo apt-get update
sudo apt-get install -y git make pkg-config autoconf libtool libusb-1.0-0 libusb-1.0-0-dev zlib1g-dev
```

4. clone and compile openocd for ESP-32:
```
git clone https://github.com/espressif/openocd-esp32 ~/openocd-esp32
cd openocd-esp32/
./bootstrap 
./configure --enable-sysfsgpio --enable-bcm2835gpio
make -j4
sudo make install
```

5. install pip3 and esptool:
```
sudo apt-get install python3-pip
sudo pip3 install esptool
```

6. configure OpenOCD:
  - edit /usr/local/share/openocd/scripts/interface/raspberrypi-native.cfg and add `bindto 0.0.0.0` above the last line
  - edit /usr/local/share/openocd/scripts/interface/raspberrypi-gpio-connector.cfg:

change pin assigment as follows:
```
adapter gpio tck -chip 0 11
adapter gpio tms -chip 0 8
adapter gpio tdi -chip 0 10
adapter gpio tdo -chip 0 9
```
It is important to not use GPIO 25 (see note in the file why), so I instead used the recommend GPIO 8.

I don't use TRST as it is not required.

Add line `transport select jtag` below the pin mapping into this file to get rid of the warning.

7. sudo setup

do 'visudo' and add your account below 'root' so that sudo doesn't require password:
```
root    ALL=(ALL:ALL) ALL
<your account>   ALL=(ALL:ALL) NOPASSWD: ALL
```

# Initial ESP-32 setup

When using VSCode I flash just the application .bin and not things like booloader.bin, partitions.bin and boot_app0.bin. However, ESP-32 needs these in order to start our application.

Therefore I initially flash simple blink example directly onto ESP-32 using Arduino IDE. That flashes all required files and then we can use VSCode to flash just our application.

Run Arduino IDE, create this small sketch:
```
void setup() {
  pinMode (2, OUTPUT);
}
void loop() {
  digitalWrite(2, HIGH);
  delay(1000);
  digitalWrite(2, LOW);
  delay(1000);
}
```

and flash it to ESP-32. The result should be that the internal LED blinks once a second.

# Cabling

Make sure that your cables are as short as possible.

You don't necessarily need it, but I power my ESP-32 using external 5V power supply in addition to USB. If you do that then ensure that all GNDs are connected together (e.g. negative input from PSU to GND pin on ESP-32 and at the same time PIN 20 on RPi).

|Connection| RPi | ESP-32 |
|----------|-----|--------|
| GND      | GND/PIN 20 | GND |
| TDI      | GPIO 10/PIN 19 | GPIO 12/PIN 18 |
| TMS      | GPIO 8/PIN 24 | GPIO 14/PIN 17 |
| TDO      | GPIO 9/PIN 21 | GPIO 15/PIN 21 |
| TCK      | GPIO 11/PIN 23 | GPIO 13/PIN 20 |

Also connect USB between ESP-32 usb port and RPi USB port - this will be used for flashing the application firmware.

Here it is in graphical form:

![image](https://github.com/Kveri/esp32-openocd-jtag-raspberrypi/assets/3380181/234f8985-786c-4f2f-8090-9cc5be798e80)

It is absolutely crucial that these connections are good and stable. JTAG runs on MHz level and any issue with physical cabling will result in errors.

Errors like these:
- Error: JTAG scan chain interrogation failed: all ones
- Error: JTAG scan chain interrogation failed: all zeros
- Error: esp32.cpu0: IR capture error; saw 0x1f not 0x01
- cpu0: xtensa_resume (line 431): DSR (FFFFFFFF) indicates target still busy!
- cpu0: xtensa_resume (line 431): DSR (FFFFFFFF) indicates DIR instruction generated an exception!
- cpu0: xtensa_resume (line 431): DSR (FFFFFFFF) indicates DIR instruction generated an overrun!

Can result from:
- incorrect physical connections, not only mismatched pins, but also bad contact, bad cable, etc.
- too high adapter speed, try descreasing it
- your ESP-32 app using some of the JTAG pins, check your code and ensure that your app doesn't use any of the pins from the above table


# verification of OpenOCD JTAG and cabling setup
Before moving on let's verify that this basic setup works:
1. run `openocd -c 'set ESP_FLASH_SIZE 0' -s /usr/local/share/openocd/scripts -f interface/raspberrypi-native.cfg -f target/esp32.cfg -c "adapter speed 1000"` as root on RPi

ESP_FLASH_SIZE 0 is important to force OpenOCD to only use hardware breakpoints as I wasn't able to make software breakpoints to work (probably missing flash maps).

Also I use adapter speed 1000 (1MHz), as higher can result in more errors if cabling is not ideal. 1MHz is good enough for debugging.

You should see something like this:
```
Open On-Chip Debugger v0.12.0-esp32-20230921-103-gce96f2fa (2023-12-13-01:48)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
0
Warn : TMS/SWDIO moved to GPIO 8 (pin 24). Check the wiring please!
jtag
WARNING: ESP flash support is disabled!
WARNING: ESP flash support is disabled!
force hard breakpoints
adapter speed: 1000 kHz
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : BCM2835 GPIO JTAG/SWD bitbang driver
Info : clock speed 1000 kHz
Info : JTAG tap: esp32.cpu0 tap/device found: 0x120034e5 (mfg: 0x272 (Tensilica), part: 0x2003, ver: 0x1)
Info : JTAG tap: esp32.cpu1 tap/device found: 0x120034e5 (mfg: 0x272 (Tensilica), part: 0x2003, ver: 0x1)
Info : starting gdb server for esp32.cpu0 on 3333
Info : Listening on port 3333 for gdb connections
Info : [esp32.cpu0] Target halted, PC=0x400F0C8A, debug_reason=00000000
Info : [esp32.cpu0] Reset cause (3) - (Software core reset)
Info : Set GDB target to 'esp32.cpu0'
Info : [esp32.cpu1] Target halted, PC=0x400D145F, debug_reason=00000001
Info : [esp32.cpu1] Reset cause (14) - (CPU1 reset by CPU0)
```

Ensure that there are no errors like those above.

2. make sure that RPi listens on port 3333 on 0.0.0.0:
```
# netstat -ntaup | grep 3333
tcp        0      0 0.0.0.0:3333            0.0.0.0:*               LISTEN      8770/openocd
```

If you don't see this then probably you missed first step of point #6 above.

# Install VSCode, platformio, extensions, platform

1. On your Windows PC install Visual Studio Code, run it
2. Go to extension tab (on the left)
3. search for platformio and install PlatformIO IDE
4. also search for C/C++ and install C/C++ and C/C++ Extension Pack
5. You should see new icon on the left (PlatformIO), click on it
6. At the bottom you should see Quick Access, click 'Platforms'
7. search for Espressif 32 and install it

# Create task for uploading firmware

Download PuTTY installer and install at least putty, pcsp and plink. Add the Program Files PuTTY directory into your Windows PATH.

Go to C:\Users\<your account>\Documents\PlatformIO and create file named 'uploadOverRpi.ps1'. Put this in the file, don't forget to add your username, pw and IP.
```
$fn = $args[1]
Write-Output "Uploading: "$fn
pscp -pw <your RPi user password> $fn <your RPi user name>@<your RPi IP>:
plink -pw <your RPi user password> <your RPi user name>@<your RPi IP> sudo esptool.py -b 2000000 --port /dev/ttyUSB0 --chip esp32 write_flash 0x10000 /home/<your account>/firmware.bin
```

This PowerShell script will be used by VSCode after compiling the firmware to upload it to RPi and flash it using esptool.py to your ESP-32 over USB.

8. Now in Quick Access, click Open and then 'New project'
9. Select board based on your board, I have ESP32-WROVER-IE so I select Espressif ESP-WROVER-KIT
10. Once the project is created, put something simple into main.cpp, like our LED blink example, but with added Serial.print lines:

```
#include <Arduino.h>

void setup() {
  Serial.begin(115200);
  pinMode (2, OUTPUT);
}

void loop() {
  Serial.println("LED ON");
  digitalWrite(2, HIGH);
  delay(1000);
  Serial.println("LED OFF");
  digitalWrite(2, LOW);
  delay(1000);
}
```

11. Now, under your project, open .vscode directory and you should see launch.json file, open it
12. Under configurations, add these two - don't forget to replace placeholders with your info:
```
        {
            "name": "MY-Upload-Debug",
            "type": "cppdbg",
            "request": "launch",
            "preLaunchTask": "uploadOverRpi",
            "program": "${file}",
            "cwd": "${workspaceFolder}",
            "MIMode": "gdb",
            "targetArchitecture": "arm",
            "miDebuggerPath": "C:/Users/<your account>/.platformio/packages/toolchain-xtensa-esp32/bin/xtensa-esp32-elf-gdb.exe",
            "debugServerArgs": "",
            "customLaunchSetupCommands": [
                {
                    "text": "file C:/Users/<your account>/Documents/PlatformIO/Projects/<your project>/.pio/build/esp-wrover-kit/firmware.elf"
                },
                {
                    "text": "set remote hardware-watchpoint-limit 2"
                },
                {
                    "text": "target extended-remote <your RPi IP>:3333"
                },
                {
                    "text": "mon reset halt"
                },
                {
                    "text": "maintenance flush register-cache"
                },
                {
                    "text": "thb app_main"
                },
                {
                    "text": "c",
                    "ignoreFailures": true
                }
            ],
            "stopAtEntry": true,
            "serverStarted": "Info\\ :\\ [\\w\\d\\.]*:\\ hardware",
            "launchCompleteCommand": "exec-continue",
            "filterStderr": true,
            "args": []
        },
        {
            "name": "MY-DebugOnly",
            "type": "cppdbg",
            "request": "launch",
            "program": "${file}",
            "cwd": "${workspaceFolder}",
            "MIMode": "gdb",
            "targetArchitecture": "arm",
            "miDebuggerPath": "C:/Users/<your account>/.platformio/packages/toolchain-xtensa-esp32/bin/xtensa-esp32-elf-gdb.exe",
            "debugServerArgs": "",
            "customLaunchSetupCommands": [
                {
                    "text": "file C:/Users/<your account>/Documents/PlatformIO/Projects/<your project>/.pio/build/esp-wrover-kit/firmware.elf"
                },
                {
                    "text": "set remote hardware-watchpoint-limit 2"
                },
                {
                    "text": "target extended-remote <your RPi IP>:3333"
                },
                {
                    "text": "mon reset halt"
                },
                {
                    "text": "maintenance flush register-cache"
                },
                {
                    "text": "thb app_main"
                },
                {
                    "text": "c",
                    "ignoreFailures": true
                }
            ],
            "stopAtEntry": true,
            "serverStarted": "Info\\ :\\ [\\w\\d\\.]*:\\ hardware",
            "launchCompleteCommand": "exec-continue",
            "filterStderr": true,
            "args": []
        },
```

Couple of words about this config:
- these are configurations which you'll use to either upload&debug or just debug ESP-32 in realtime. The only difference between these two is the preLaunchTask, which flashes the firmware onto ESP-32 in case of Upload&Debug.
- vscode launches esp-32 gdb and then sends few commands to that gdb
- First command is for gdb to know about the symbols
- second is to make gdb/openocd aware that there are only 2 HW breakpoints
  - ESP-32 has only 2 HW breakpoints and around 63 SW breakpoints. I was not able to make the SW breakpoints work, so I needed to disable the FLASH (in openocd command) to force openocd to only use HW breakpoints
  - First HW breakpoint is always use to break at app_main, which is just before the entry point into your code. For some reason you need this first breakpoint when you run the code and if you don't have it then you can't break in your own code
- 3rd command tells gdb to connect to gdb server (openocd)
- 4th reset the ESP-32 and halts it at the start
- 5th is to flush the register cache in gdb, so that gdb always re-reads the registers
- 6th is the mentioned pre-entry brackpoint which we need in order for the 2nd breakpoint to work. Once your run the debugging and it breaks in app_main you can actually delete that breakpoint and use 2 HW breakpoints for your own code
- 7th command starts the execution

13. Next, you need to create file tasks.json in your .vscode directory and put this there:
```
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "uploadOverRpi",
            "type": "shell",
            "command": "powershell",
            "isShellCommand": true,
            "showOutput": "always",
            "args": [
               "C:\\Users\\<your account>\\Documents\\PlatformIO\\uploadOverRpi.ps1",
               "-FirmwareFileName ${workspaceFolder}\\.pio\\build\\esp-wrover-kit\\firmware.bin"
              ]
        }
    ]
}
```

14. Last in platformio.ini file at your project directory ensure you have this:
```
[env:esp-wrover-kit]
platform = espressif32
board = esp-wrover-kit
framework = arduino
build_type = debug
```

The 'esp-wrover-kit' can be anything, but you need to make sure that it's the same in your entire setup. It is used in tasks.json, launch.json, platformio.ini.

# Using it
Once you have everything set up this is how you normally use it:
1. start up your RPi, power on your ESP-32
2. ssh to RPi using two sessions
3. as root, on the first session run `openocd -c 'set ESP_FLASH_SIZE 0' -s /usr/local/share/openocd/scripts -f interface/raspberrypi-native.cfg -f target/esp32.cfg -c "adapter speed 1000"`
4. start your VScode, write your code
5. once ready, hit PlatformIO: Build icon at the bottom of VScode (small tick button), this will build the code, create firmware.elf and firmware.bin
6. On the left side click "Run and Debug" icon.
7. On top where you have the green Play button click the dropdown and select your configuration, make sure to select the right project if you have multiple projects open. Either select Upload&Debug or just Debug (if you have already latest code uploaded and just want to run it). Then click the green Play button

At this point what should happen is that in the console in VSCode you should see that firmware.bin has been uploaded to RPi and esptool.py flashed the firmware to ESP-32. You should also see some new messages in openocd and eventually VScode bottom bar should turn blue and your code execution should stop just below `externl "C" void app_main()` in main.cpp of esp32 code.

At this moment your ESP-32 hit the first HW breakpoint and is waiting. You can remove this breakpoint, then go to your code, setup your breakpoint and hit F5 to continue the execution. It should then work, including stepping (F10, etc).

8. Now open the second ssh session to RPi and run `sudo screen /dev/ttyS0 115200`. That will give you the serial output of your ESP-32, so you can see Serial.print messages and so on.

Please make sure that when you are uploading the code you close this screen (serial logging), otherwise esptool.py will fail to upload your code to ESP-32, because serial /dev/ttyS0 is used by screen. You can close the screen by hitting ctrl+a, releasing it and then hitting 'k' and confirming it with 'y'.

Important notes:
- you can only maximum of 2 breakpoints (including the one in app_main(), if you didn't delete it).
- 'Run to line' internally uses a breakpoint, so you can only use that if you have zero or one breakpoint set.
- I believe step over (F10) also internally uses a breakpoint, so I think that can be used also only if you have zero or one breakpoint set.
- any time you get a popup error in VSCode about gdb first step is to restart openocd (ctrl+c and run it again)


TODO:
- console output in VSCode
- SW breakpoints (+ flash support)

Related and useful links (helped me to learn and write this guide):
- https://docs.platformio.org/en/stable/tutorials/espressif32/arduino_debugging_unit_testing.html
- https://github.com/espressif/openocd-esp32/issues/84
- https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/jtag-debugging/tips-and-quirks.html
- https://blog.wokwi.com/gdb-debugging-esp32-using-raspberry-pi/
- https://chiptron.cz/articles.php?article_id=219 (old, outdated but provides good starting point)

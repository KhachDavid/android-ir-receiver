# Setting up Android TV from scratch with a hackable remote

This is for a Raspberry PI5 and LineageOS 20 setup.

Inspired from [this video](https://youtu.be/OqPmswpuBaU?si=Rro0DnSk3Kd0Nqlo) by leepspvideo.

OS and instructions are [here](https://konstakang.com/devices/rpi5/LineageOS20-ATV/), provided by konstakang.

## Setting up the custom LineageOS file system

mount -o remount,rw <CUSTOM DIRECTORY>

### Example of remounting to write files

mount -o remount,rw /boot

## Attaching IR Receiver on the PI

I used F-F cables to attach an IR receiver on the GPIO pin 18

![image](https://github.com/user-attachments/assets/b2a178cd-63d3-43fb-a193-6fb8fe63a4d6)

## Reading ir receivers

The below command resets the keys

`ir-keytable -c -w /boot/rc_keymap.txt`

To test them run

`ir-keytable -t`

## RC Map Setup

`cat /etc/rc_maps.cfg`

```bash
* * /boot/rc_keymap.txt
```

## Remote Receiver Codes

`cat /boot/rc_keymap.txt`

```bash
# table my_remote, type: nec
0x70765 KEY_LEFT
0x70761 KEY_DOWN
0x70760 KEY_UP
0x70762 KEY_RIGHT
0x70768 KEY_ENTER
0x70758 KEY_BACK
0x7072d KEY_HOME
```

## Power Off Automatically

`cat /data/local/tmp/power_listener.sh`

```bash
INPUT_DEVICE="/dev/input/event0"  # Replace this with your Power button's input device
SCANCODE="00070702"               # Replace this with the scancode for the Power button. Obtainable through getevent

# Monitor the input device for events
getevent -ql $INPUT_DEVICE | while read line; do
    echo "$line" | grep "$SCANCODE" > /dev/null
    if [ $? -eq 0 ]; then
        # Turn off the TV using HDMI-CEC
        echo "Turning off TV via HDMI-CEC..."
        cec-ctl --to 0 --standby

        # Shut down the Raspberry Pi safely
        echo "Shutting down Raspberry Pi..."
        reboot -p
    fi
done
```

## Go Home Listener

`cat /data/local/tmp/key_listener.sh`

```bash
#!/system/bin/sh

# Define the input device and scancode for the Home button
INPUT_DEVICE="/dev/input/event0"  # Input device identified as gpio_ir_recv
SCANCODE="0007072d"               # Scancode for the Home button
DEBOUNCE_DELAY=500  # Minimum delay between presses in milliseconds
LAST_PRESS_TIME=0   # Variable to track the last button press time

# Monitor the input device for events
getevent -ql $INPUT_DEVICE | while read line; do
    echo "$line" | grep "$SCANCODE" > /dev/null
    if [ $? -eq 0 ]; then
        # Get the current timestamp in milliseconds
        CURRENT_TIME=$(date +%s%3N)

        # Calculate the time difference since the last press
        TIME_DIFF=$(($CURRENT_TIME - $LAST_PRESS_TIME))

        # Check if enough time has passed
        if [ $TIME_DIFF -ge $DEBOUNCE_DELAY ]; then
            # Update the last press time
            LAST_PRESS_TIME=$CURRENT_TIME

            # Trigger the Home action
            /data/local/tmp/home_key.sh
        fi
    fi
done
```

## OK Button Listener

`cat /data/local/tmp/ok_button_listener.sh`

```
#!/bin/bash

# Input device and scancode configuration
INPUT_DEVICE="/dev/input/event0"     # Replace this with the actual input device for the remote
SCANCODE="00070768"                 # Replace this with the scancode for the desired button

# Debounce time in seconds
DEBOUNCE_TIME=1                     # Prevent double execution within this time frame

# Track the last execution time
LAST_EXECUTION=0

# Monitor the input device for events
getevent -ql $INPUT_DEVICE | while read line; do
    # Check if the scancode matches the desired one
    echo "$line" | grep "$SCANCODE" > /dev/null
    if [ $? -eq 0 ]; then
        # Get the current timestamp
        CURRENT_TIME=$(date +%s)

        # Check if the last execution was within the debounce period
        if (( CURRENT_TIME - LAST_EXECUTION >= DEBOUNCE_TIME )); then
            # Update last execution time
            LAST_EXECUTION=$CURRENT_TIME

            # Execute your desired action
            echo "Input detected: Scancode $SCANCODE"
            echo "Simulating 'input keyevent 96'..."
            input keyevent 96
        else
            echo "Debounced input ignored."
        fi
    fi
done
```

## Go home action

`cat /data/local/tmp/`

```bash
#!/system/bin/sh
input keyevent 3
```

## Launch the movies app

`cat /data/local/tmp/launch_movies_listener.sh`

```bash
#!/bin/bash

# Set variables
INPUT_DEVICE="/dev/input/event0"      # Replace this with the correct input device (e.g., event0)
SCANCODE="0007076c"                  # Scancode to listen for
DEBOUNCE_TIME=1                      # Debounce interval in seconds
LAST_EXECUTION=0                     # Variable to track the last execution time

# Monitor the input device for events
getevent -ql $INPUT_DEVICE | while read line; do
    # Check if the line contains the desired scancode
    echo "$line" | grep "$SCANCODE" > /dev/null
    if [ $? -eq 0 ]; then
        # Get the current timestamp
        CURRENT_TIME=$(date +%s)

        # Check if the debounce period has elapsed
        if (( CURRENT_TIME - LAST_EXECUTION >= DEBOUNCE_TIME )); then
            # Update last execution time
            LAST_EXECUTION=$CURRENT_TIME

            # Run the command to launch the app
            echo "Scancode $SCANCODE detected. Launching Movies Portal TV..."
            am start -n evdoo.mdotv/.core.activities.MainActivity
        else
            echo "Debounced input ignored."
        fi
    fi
done

```

## Start scripts on startup

`cat /system/etc/init/init.lineage.atv.adb.rc`

```bash
# Copyright (C) 2022 The LineageOS Project
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

on property:service.adb.tcp.port=5555
    stop adbd
    start adbd

on property:service.adb.tcp.port=-1
    stop adbd
    start adbd

service key_listener /system/bin/sh /data/local/tmp/key_listener.sh
    class main
    user root
    oneshot

service power_listener /system/bin/sh /data/local/tmp/power_listener.sh
    class main
    user root
    oneshot

service ok_listener /system/bin/sh /data/local/tmp/ok_button_listener.sh
    class main
    user root
    oneshot

service movies_listener /system/bin/sh /data/local/tmp/launch_movies_listener.sh
    class main
    user root
    oneshot
```

# Setting up Android TV from scratch with a hackable remote

This is for a Raspberry PI5 and LineageOS 20 setup.

Inspired from [this video](https://youtu.be/OqPmswpuBaU?si=Rro0DnSk3Kd0Nqlo) by leepspvideo.

OS and instructions are [here](https://konstakang.com/devices/rpi5/LineageOS20-ATV/), provided by konstakang.

## Setting up the custom LineageOS file system

mount -o remount,rw <CUSTOM DIRECTORY>

### Example of remounting to write files

mount -o remount,rw /boot

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

## Go home action

`cat /data/local/tmp/`

```bash
#!/system/bin/sh
input keyevent 3
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
```

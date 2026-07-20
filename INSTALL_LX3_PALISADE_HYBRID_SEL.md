Build 2026 Palisade HDA1 on a Comma 3x

The 2026 Palisade with HDA1 is not formally supported on Openpilot or
Sunnypilot.  The code has been modified using a fork from "kamdeva" that
got a good start, but more work was needed for the Palisade Hybrid SEL.
If your car has the same fingerprint, it will be detected automatically.

For the changes provided in the "kamdeva" fork, see
/data/openpilot/INSTALL\_LX3.md



In addition to he basic support, this code reduces the effort to move
the car within the lane, such as when a car in an adjacent lane is
hugging the line, and minimizes the effort to start the lane change
with the blinker when "nudge" has been selected.



The code is down-loaded from "github" to the Comma device (only 3x was
tested) and then built locally.  The commands are provided below.  This
will replace your current code, but you can either force a system reset
by tapping on the screen during power on, or going into the Software
menu and uninstalling.



Use ssh or adb to connect to the Comma.  Enable the connection method on
the Comma under Developer.  (ADB is the easier option) ADB is a windows
application.  After installing it, either "cd" to that directory or add
the directory to the path environment variable so the "adb" command can  
be found.



Connect the Comma to power, then connect a USB-C cable suitable for data
transfer to the port on the bottom.



From a dos command prompt, use "adb shell" to connect.  You will be working
in a Linux environment.



Copy and paste this block into the adb window, or run the commands one at a time.



echo "


cd /data


sudo chmod 777 /data/scons\_cache


sudo mount -o remount,rw /


sudo rm -rf openpilot


git config --global --add safe.directory /data/openpilot


git clone --recursive --branch master https://github.com/knhusker/sunnypilot.git openpilot


cd /data/openpilot


git lfs pull


source /usr/local/venv/bin/activate


export PYTHONPATH=/data/openpilot


scons -j4


sudo reboot" > /tmp/buildsp.sh


bash -x /tmp/buildsp.sh



After reboot, the Comma device will run with the LX3-patched code. Panda
firmware will be flashed automatically by pandad on first boot if the
firmware on the panda is older than what's compiled in this branch.

NOTE:  Go into the Cruise menu and turn on ICBM.  Logs have confirmed that
it is NOT sending any accel/decel messages so the car is still managing
the speed, but without it the car seems to "hunt" for the right speed
with quick acceleration and deceleration visible on the display on the
right side of the car console.



Do NOT enable "Use Lane Turn Desires".  After turning a corner, the
calculated track will want to keep right on turning, even though the
camera shows lane markers going straight.


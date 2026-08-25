Build code for 2026 Palisade SEL Premium with HDA1 on a Comma 3x.

The 2026 Palisade with HDA1 is not formally supported on Openpilot or Sunnypilot.  The code has been modified using a fork from "kamdeva" that got a good start, but more work was needed for the Palisade Hybrid SEL. If your car has the same fingerprint, it will be detected automatically and should work.

In addition to he basic support, this code reduces the effort to move the car within the lane, such as when a car in an adjacent lane is hugging the line, and minimizes the effort to start the lane change with the blinker when "nudge" has been selected.  There was a problem with abruptly ending steering on a slightly tight corner at higher speed, but that has been partially corrected.  Due to limitations in the HDA1 implementation, the car will drift toward the outside of the turn then a warning will be displayed to take control.  Pay attention in this situation and be ready to take over.  Hopefully HDA2 will be better! 

The code is down-loaded from "github" to the Comma device (only 3x was tested) and then built locally.  The commands are provided below.  This will replace your current code, but if you want to go back to some other fork you can either force a system reset by tapping on the screen during power on, or going into the Software menu and uninstalling.

Use ssh or adb to connect to the Comma.  An AI tool such as Claude can explain how to do either of these.  (adb is easier, assuming you have a compatible data cable.)

Copy and paste this block into the shell windows.  The commands will be displayed and executed one at a time.


```bash
cat > /tmp/build.sh << 'EOF'
cd /data
sudo chmod 777 /data/scons_cache
sudo mount -o remount,rw /
sudo rm -rf openpilot
git config --global --add safe.directory /data/openpilot
git clone --recursive --branch master https://github.com/knhusker/sunnypilot.git openpilot
cd /data/openpilot
git lfs pull
source /usr/local/venv/bin/activate
export PYTHONPATH=/data/openpilot
scons -j4
sudo reboot
EOF
bash -x /tmp/build.sh
```


After reboot, the Comma device will run with the LX3-patched code. Panda firmware will be flashed automatically by pandad on first boot if thef firmware on the panda is older than what's compiled in this branch.

When the vehicle is selected, it should show as HYUNDAI_PALISADE_HEV_LX3.

NOTE:  Go into the Cruise menu and turn on ICBM.  Logs have confirmed that it is NOT sending any accel/decel messages so the car is still managing the speed, but without it the car seems to "hunt" for the right speed with quick acceleration and deceleration visible on the display on the right side of the car console.

Do NOT enable "Use Lane Turn Desires".  After turning a corner, the calculated track will want to keep right on turning, even though the camera shows lane markers going straight.

The LKA or Lane Centering button will not work.  Press the Cruise control button to engage lane centering, and if cruise is not wanted press the toggle button or tap on the brake.  To disable lane centering, press the Cruise control button again.


# ddr-itgmania
Short guide how to set up a Dance Dance Revolution cabinet + linux PC with ITGMania.

<img src="/ddr.jpg" width="300" />

## Hardware

### What you need 
- DDR Arcade Cabinet - we've got Dancing Stage True Kiss Destination J-cab (there are many different https://jeffreyatw.com/blog/2019/03/types-of-ddr-cabinets-and-pcbs/)
- J-PAC (JAMMA interface for Pc to Arcade Controls) - https://www.ultimarc.com/control-interfaces/j-pac-en/j-pac-jamma-interface/
- LIT Board for lights to work - https://icedragon.io/lit/
- Dell U3014 Monitor (or similar) - if you want to replace the existing cabinet monitor. U3014 fits into the cabinet extremely well, has low refresh rate and 4:3mode
- Any PC that can have linux installed
- 2x USB > USB mini cables
- 3.5mm > RCA (red+white) audio cable

### Steps
- Follow LIT Board and J-PAC guides to plug them in to the existing cabinet wiring
- Plug USB > USB mini from your PC to LIT Board
- Plug USB > USB mini from your PC to J-PAC
- Plug 3.5mm > RCA cable from your PC to the cabinet amp. In our cabinet when you open the back board, there are two boxes: left - the amp and the right - original System 573 box. There should be already RCA > RCA cable from 573 to the amp but there are might be another set of RCA inputs on the amp. You ca either use them or disconnect existing cables and put yours instead.
- Optional. Rip off the original monitor - make the machine was unpowered for a while. The tube comes with two boards all entangled together. Just diconnect them all and it should be ok.

### Caveats
- There were 3 kill switches in the arcade cabinet we had (next to the PCU, next to the coin mechanism, next to the back board to prevent it running with the open back board)

## Software

### What you need 
- Any linux. We've had Ubuntu Desktop 24.04.3 LTS with xfce/lightdm installed later
- ITGMania for Linux (https://www.itgmania.com/)

### Intial Setup
- Install linux on your PC
- Install ITGMania (it should put it into /opt/itgmania and configs in ~/.itgmania/)
- Suprisingly, both j-pac and lit boards should not require any drivers and "just work". There are some tooling you can try if you need (https://github.com/katie-snow/Ultimarc-linux)
- Your config should be in ~/.itgmania/Save/Preferences.ini
- To make ITGmania fullscreen you need to change "Windowed=0" in the Preferences.ini
- change your window manager resoltion to something 4:3 (1600x1200 - worked for me). Adjust your monitor setting using its hardware buttons

### Controls
- ITGmania should pickup pad controls automatically, but you will need to bind < = > keys on the cabinet. For that you need to open ITGmania and go through Settings > Input Bindings (or whatever it called)

### Lights
So how it works is:
- ITGMania uses LightsDriver_SextetStreamToFile driver, puts binary information into a named piped 
- you redirect this pipe to the lit board interface

To do that:
- in ITGMania's Preferences.ini set `LightsDriver=SextetStreamToFile` and `SextetStreamOutputFilename=Data/StepMania-Lights-SextetStream.out`
- there are some mentiones that people have problems with writing into Data/* folders. You can use Save/* folder as a destination for StepMania-Lights-SextetStream.out
- use `socat` to pipe data to litboard interface. By default it is `/dev/ttyUSB0` so it looks something like this `socat "/home/dance/.itgmania/Save/StepMania-Lights-SextetStream.out" /dev/ttyUSB0,raw,echo=0,b115200` 
- socat needs to be running continiously for lights to work. You can run it with `&` (see the start script at the end)

More info here:
- https://github.com/stepmania/stepmania/blob/master/src/arch/Lights/LightsDriver_SextetStream.md
- https://icedragon.io/downloads/LIT-Board-Documentation.pdf

### Caveats
- Preferences.ini doesn't ignore trailing spaces in values (`LightsDriver=SextetStreamToFile ` - with the space at the end - will throw an error in logs and will not work)
- Logs in ~/.itgmania/Logs/ - can be useful for debugging 
- Not starting if there is nobody consumes the StepMania-Lights-SextetStream.out pipe

## Script to start socat + ITGMania automatically and shutdown when it was closed

### `/home/dance/launch-itgmania.sh` (make sure it's executable)
```
#!/bin/bash

# Start socat in the background to handle lights communication
socat "/home/dance/.itgmania/Save/StepMania-Lights-SextetStream.out" /dev/ttyUSB0,raw,echo=0,b115200 &

# Store the socat PID
SOCAT_PID=$!

# Give socat a moment to initialize
sleep 1

# Launch ITGmania (this will block until itgmania exits)
/opt/itgmania/itgmania

# When itgmania closes, kill socat
kill $SOCAT_PID 2>/dev/null

# Power off the system
sudo poweroff
```

### Allow dance user to run poweroff without password
```
sudo visudo
```
In the visudo editor, add this line at the end:
```
dance ALL=(ALL) NOPASSWD: /usr/sbin/poweroff
```

### `itgmania-launch.desktop` (put it into ~/.config/autostart/)
```
[Desktop Entry]
Type=Application
Name=ITGmania Launcher
Comment=Starts socat for lights and launches ITGmania
Exec=/home/dance/launch-itgmania.sh
Terminal=false
Hidden=false
X-GNOME-Autostart-enabled=true
```

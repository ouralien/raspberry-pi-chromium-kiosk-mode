# PI Setup

### Download Image & Install

`https://www.raspberrypi.org/downloads/raspbian/`

[Raspbian Latest](https://downloads.raspberrypi.org/raspbian_latest.torrent) (Jessie 8)

`sudo dd bs=1m if=path_of_your_image.img of=/dev/rdiskn`

Replace `n` with the disk number

### Setup Wifi

`sudo nano /etc/wpa_supplicant/wpa_supplicant.conf`

```
network={
    ssid="SSID"
    psk="password"
}
```

```
sudo ifdown wlan0
sudo ifup wlan
```

### Necessary Software for Kiosk
```
sudo apt-get update
sudo apt-get dist-upgrade
sudo apt-get install matchbox x11-xserver-utils ttf-mscorefonts-installer xwit sqlite3 libnss3
```

Then reboot : `sudo reboot`


### Installation of libgcrypt11 on Debian Linux 8 ( Jessie ) 64-bit

```
wget http://ftp.acc.umu.se/mirror/cdimage/snapshot/Debian/pool/main/libg/libgcrypt11/libgcrypt11_1.5.3-5_armhf.deb

dpkg -i libgcrypt11_1.5.3-5_armhf.deb
```

### Installation of Chromium browser 45

`https://launchpad.net/ubuntu/vivid/armhf/chromium-browser/45.0.2454.101-0ubuntu0.15.04.1.1183`

`https://launchpad.net/ubuntu/vivid/armhf/chromium-codecs-ffmpeg-extra/45.0.2454.101-0ubuntu0.15.04.1.1183`

```
dpkg -i chromium-codecs-ffmpeg-extra_45.0.2454.101-0ubuntu0.15.04.1.1183_armhf.deb \ 
	chromium-browser_45.0.2454.101-0ubuntu0.15.04.1.1183_armhf.deb
```

### Pi Config 

`/boot/config.txt`

```
# 1900x1200 at 32bit depth, DMT mode
disable_overscan=0
overscan_left=38
overscan_right=38
overscan_top=25
overscan_bottom=25

framebuffer_width=1900
framebuffer_height=1200
framebuffer_depth=32
framebuffer_ignore_alpha=1
hdmi_pixel_encoding=1
hdmi_group=2

#uncomment to overclock the arm. 700 MHz is the default.
arm_freq=1000

core_freq=500
sdram_freq=500
over_voltage=2
gpu_mem=128
```

### Startup Run Commands

`/etc/rc.local`

```
# Wait for the TV-screen to be turned on...
while ! $( tvservice --dumpedid /tmp/edid | fgrep -qv 'Nothing written!' ); do
	bHadToWaitForScreen=true;
	printf "===> Screen is not connected, off or in an unknown mode, waiting for it to become available...\n"
	sleep 10;
done;

printf "===> Screen is on, extracting preferred mode...\n"
_DEPTH=32;
eval $( edidparser /tmp/edid | fgrep 'preferred mode' | tail -1 | sed -Ene 's/^.+(DMT|CEA) \(([0-9]+)\) ([0-9]+)x([0-9]+)[pi]? @.+/_GROUP=\1;_MODE=\2;_XRES=\3;_YRES=\4;/p' );

printf "===> Resetting screen to preferred mode: %s-%d (%dx%dx%d)...\n" $_GROUP $_MODE $_XRES $_YRES $_DEPTH
tvservice --explicit="$_GROUP $_MODE"
sleep 1;

printf "===> Resetting frame-buffer to %dx%dx%d...\n" $_XRES $_YRES $_DEPTH
fbset --all --geometry $_XRES $_YRES $_XRES $_YRES $_DEPTH -left 0 -right 0 -upper 0 -lower 0;
sleep 1;

if [ -f /boot/xinitrc ]; then
	ln -fs /boot/xinitrc /home/pi/.xinitrc;
	su - pi -c 'startx' &
fi
```
### Xinit Script

`/boot/xinitrc`

```
#!/bin/sh
while true; do

	# Clean up previously running apps, gracefully at first then harshly
	killall -TERM chromium-browser 2>/dev/null;
	killall -TERM matchbox-window-manager 2>/dev/null;
	sleep 2;
	killall -9 chromium-browser 2>/dev/null;
	killall -9 matchbox-window-manager 2>/dev/null;

	# Clean out existing profile information
	rm -rf /home/pi/.cache;
	rm -rf /home/pi/.config;
	rm -rf /home/pi/.pki;

	# Generate the bare minimum to keep Chromium happy!
	mkdir -p /home/pi/.config/chromium/Default
	sqlite3 /home/pi/.config/chromium/Default/Web\ Data "CREATE TABLE meta(key LONGVARCHAR NOT NULL UNIQUE PRIMARY KEY, value LONGVARCHAR); INSERT INTO meta VALUES('version','46'); CREATE TABLE keywords (foo INTEGER);";

	# Disable DPMS / Screen blanking
	xset -dpms
	xset s off

	# Reset the framebuffer's colour-depth
	fbset -depth $( cat /sys/module/*fb*/parameters/fbdepth );

	# Hide the cursor (move it to the bottom-right, comment out if you want mouse interaction)
	xwit -root -warp $( cat /sys/module/*fb*/parameters/fbwidth ) $( cat /sys/module/*fb*/parameters/fbheight )

	# Start the window manager (remove "-use_cursor no" if you actually want mouse interaction)
	matchbox-window-manager -use_titlebar no -use_cursor no &

	# Start the browser (See http://peter.sh/experiments/chromium-command-line-switches/)
	chromium-browser  --app='https://google.com'

done;
```

### Credits

* [HOWTO: Boot your Raspberry Pi into a fullscreen browser kiosk](http://blogs.wcode.org/2013/09/howto-boot-your-raspberry-pi-into-a-fullscreen-browser-kiosk/)
* [Installing Google Chromium on ARM devices](http://blog.valitov.me/2014/06/installing-google-chromium-on-arm.html)
* [Installation of Spotify client on Debian Linux 8 ( Jessie ) 64-bit](http://linuxconfig.org/installation-of-spotify-client-on-debian-linux-8-jessie-64-bit)
* [How do I cross-compile Chromium for ARM?](http://unix.stackexchange.com/questions/176794/how-do-i-cross-compile-chromium-for-arm)
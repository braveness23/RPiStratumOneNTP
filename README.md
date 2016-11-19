# RPiStratumOneNTP

A project to turn a Rapberry Pi 3 into a Stratum 1 NTP server.

Turn this into an ansible playbook

Download RASPBIAN JESSIE LITE and install onto MicroSD card


	sudo umount /dev/sdb2
	sudo umount /dev/sdb1
	sudo dd bs=4M if=2016-09-23-raspbian-jessie.img of=/dev/sdb
	sync


For details: https://www.raspberrypi.org/documentation/installation/installing-images/linux.md


Update system

	sudo apt-get update
	sudo apt-get upgrade


Do basic configuration of RPi


	sudo raspi-config
	
Expand File System
Change User Password
Change Locale to en_US.UTF-8
Change Timezone to US EASTERN
Change Keyboard layout to Generic 105-key PC
Change WiFi country to US
Way more detail needed here

	sudo iwlist wlan0 scan
 
Install pps-tools

	sudo apt-get -y install pps-tools

Configure PPS


	sudo nano /etc/modules
	# /etc/modules: kernel modules to load at boot time.
	pps-gpio


	sudo nano /boot/config.txt
	# /boot/config.txt
	# GPIO 24 is pin 18 on Raspberry Pi.  Connect your PPS to pin 18
	dtoverlay=pps-gpio,gpiopin=24


Reboot

Test PPS


	dmesg | grep pps

	[    5.977541] pps_core: LinuxPPS API ver. 1 registered
	[    5.977560] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
	[    6.046277] pps pps0: new PPS source pps.-1
	[    6.046356] pps pps0: Registered IRQ 504 as PPS source


	sudo ppstest /dev/pps0

	trying PPS source "/dev/pps0"
	found PPS source "/dev/pps0"
	ok, found 1 source(s), now start fetching data...
	source 0 - assert 1475544227.003374325, sequence: 1965 - clear  0.000000000, sequence: 0
	source 0 - assert 1475544228.003372432, sequence: 1966 - clear  0.000000000, sequence: 0
	source 0 - assert 1475544229.003370392, sequence: 1967 - clear  0.000000000, sequence: 0
	source 0 - assert 1475544230.003367995, sequence: 1968 - clear  0.000000000, sequence: 0
	Install gpsd and gpsd-clients

	sudo apt-get -y install gpsd gpsd-clients


Configure gpsd


	sudo nano /boot/cmdline.txt


Remove: 
console=serial0,115200


	sudo systemctl stop serial-getty@ttyS0.service
	sudo systemctl disable serial-getty@ttyS0.service


	sudo nano /boot/config.txt


Append with: 
init_uart_baud=9600
enable_uart=1


Reboot


	ls /dev/tty*


Confirm that ttypS0 is listed


	sudo cat /dev/ttyS0


Confirm that GPS codes are output


	sudo nano /etc/default/gpsd

START_DAEMON="true"
GPSD_OPTIONS="-n"
DEVICES="/dev/ttyS0"
USBAUTO="false"
GPSD_SOCKET="/var/run/gpsd.sock"


Reboot


	cgps -s


	lqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqklqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqk
	x    Time:       2016-10-04T12:43:34.000Z   xxPRN:   Elev:  Azim:  SNR:  Used: x
	x    Latitude:    39.164391 N               xx  28    76    038    00      Y   x
	x    Longitude:   84.549300 W               xx  17    59    284    17      Y   x
	x    Altitude:   509.8 ft                   xx   1    42    053    17      Y   x
	x    Speed:      0.4 mph                    xx  19    38    263    00      Y   x
	x    Heading:    113.2 deg (true)           xx  30    36    184    24      Y   x
	x    Climb:      0.0 ft/min                 xx  11    27    052    21      Y   x
	x    Status:     3D FIX (23 secs)           xx  22    19    087    21      N   x
	x    Longitude Err:   +/- 43 ft             xx   3    17    108    00      N   x
	x    Latitude Err:    +/- 53 ft             xx   6    16    204    00      N   x
	x    Altitude Err:    +/- 65 ft             xx   7    11    169    00      N   x
	x    Course Err:      n/a                   xx  13    08    262    14      N   x
	x    Speed Err:       +/- 72 mph            xx  24    04    326    00      N   x
	x    Time offset:     0.587                 xx                                 x
	x    Grid Square:     EM79rd                xx                                 x
	mqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqjmqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqj


Install minicom and ntpstat


	sudo apt-get -y install minicom ntpstat


Test ntpstat:


	ntpstat
	synchronised to NTP server (129.6.15.28) at stratum 2
   	time correct to within 139 ms
   	polling server every 64 s


Configure /etc/ntp.config


	# https://www.eecis.udel.edu/~mills/database/brief/precise/precise.pdf
	# - PPS believed only if prefer peer correct and within 128 ms
	tinker  step 0.4  stepback 0.4  stepfwd 0.4


	# You do need to talk to an NTP server or two (or three).
	#server ntp.your-provider.example


	# Kernel-mode PPS ref-clock for the precise seconds
	server 127.127.22.0 minpoll 4 maxpoll 4
	fudge 127.127.22.0 refid PPS stratum 0


	# Server from shared memory provided by gpsd
	server 127.127.28.0 minpoll 4 maxpoll 4 prefer
	fudge 127.127.28.0 refid NMEA stratum 3 time1 0.510


	# pool.ntp.org maps to about 1000 low-stratum NTP servers.  Your server will
	# pick a different set every time it starts up.  Please consider joining the
	# pool: <http://www.pool.ntp.org/join.html>
	# server 0.debian.pool.ntp.org iburst
	# server 1.debian.pool.ntp.org iburst
	# server 2.debian.pool.ntp.org iburst
	# server 3.debian.pool.ntp.org iburst

Restart NTP

	sudo service ntp stop
	sudo service ntp start

Test that NTP is configured to talk to the type 28.0 shared memory driver, and can see the GPSD.

	ntpq -pn

Stop existing NTP service then download new, compile and replace it

	sudo service ntp stop
	sudo apt-mark hold ntp
	mkdir -p ~/ntp
	cd ~/ntp
	sudo apt-get install libcap-dev
	wget http://archive.ntp.org/ntp4/ntp-4.2/ntp-4.2.8p6.tar.gz
	tar xvfz ntp-4.2.8p6.tar.gz
	cd ntp-4.2.8p6/
	./configure --enable-linuxcaps
	make
	sudo make install
	sudo cp /usr/local/bin/ntp* /usr/bin/  && sudo cp /usr/local/sbin/ntp* /usr/sbin/
	sudo service ntp start

Sources:

http://www.satsignal.eu/ntp/Raspberry-Pi-NTP.html
https://learn.adafruit.com/adafruit-ultimate-gps-on-the-raspberry-pi/using-uart-instead-of-usb
https://www.raspberrypi.org/forums/viewtopic.php?f=29&t=140585



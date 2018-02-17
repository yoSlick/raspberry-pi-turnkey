# Raspberry Pi Turnkey

Have you ever wanted to startup a Raspberry Pi *without having to SSH or attach a keyboard* to add your WiFi credentials? This is particularly useful when you are making a Raspberry Pi that needs to be deployed somewhere where supplying the credentials via SSH or attaching a keyboard isn't an option. 

You can [follow the instructions below](#instructions-to-create-image) to create a turnkey image, or you can just download my latest one at [https://raspberry-pi-turnkey.schollz.com/2018-02-17-turnkey.img](https://raspberry-pi-turnkey.schollz.com/2018-02-17-turnkey.img) (3GB). 

# Usage 

Once you boot the Pi with this image, you will see a WiFi AP named "ConnectToConnect" (password same). Connect to it and navigate to `192.168.4.1` where you'll see a login form.

<p align="center">
  <img src="https://i.imgur.com/NeWmrlk.png"/>
</p>

When the WiFi credentials are entered onto the login form, the Pi will modify its internal `wpa_supplicant` to conform to them so that it will be connected to the net. The Pi will then reboot itself using those WiFi credentials. If the credentials are not correct, then the Pi will reboot back into the AP mode to allow you to re-enter them again.

_Note:_ The Raspberry Pi is **not** a fast computer. When you see the AP and connect to it, it may take up to a minute for the page at `192.168.4.1` to appear. Also, if you enter the wrong WiFi credentials, it will have to reboot twice to reset the Pi to allow you to enter the credentials again. So try to enter them right the first time!

# Instructions to create image

The following are the step-by-step instructions for how I create the turnkey image. If you don't want to download the image I created above (I don't blame you), then follow these to make one exactly the same.

These instructions assume you are using Ubuntu. You can use Windows/OS X for most of these steps, except step #4 which requires resizing.

## 1. Flash Raspbian Stretch Lite

Starting from version [Raspbian Stretch Lite](https://www.raspberrypi.org/downloads/raspbian/) version 2017-11-29.

```
sudo dd bs=4M if=2017-11-29-raspbian-stretch-lite.img of=/dev/mmcblk0 conv=fsync status=progress
```

Change `/dev/mmcblk0` to whatever your SD card is (find it using `fdisk -l`).

After flashing, for the first time use, just plug in ethernet and you can SSH into the Pi. To activate SSH on boot just do

```
touch /media/YOURUSER/boot/ssh
```

## 2. Install libraries onto the Raspberry Pi

### Basic libraries

```
sudo apt-get update
sudo apt-get dist-upgrade -y
sudo apt-get install -y dnsmasq hostapd vim python3-flask git
```

### Install node (optional)

```
wget https://nodejs.org/dist/v8.9.4/node-v8.9.4-linux-armv6l.tar.xz
sudo mkdir /usr/lib/nodejs
sudo tar -xJvf node-v8.9.4-linux-armv6l.tar.xz -C /usr/lib/nodejs 
rm -rf node-v8.9.4-linux-armv6l.tar.xz
sudo mv /usr/lib/nodejs/node-v8.9.4-linux-armv6l /usr/lib/nodejs/node-v8.9.4
echo 'export NODEJS_HOME=/usr/lib/nodejs/node-v8.9.4' >> ~/.profile
echo 'export PATH=$NODEJS_HOME/bin:$PATH' >> ~/.profile
source ~/.profile
```

### Install Go (optional)

```
wget https://dl.google.com/go/go1.10.linux-armv6l.tar.gz
sudo tar -C /usr/local -xzf go*gz
rm go*gz
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >>  ~/.profile
echo 'export GOPATH=$HOME/go' >>  ~/.profile
source ~/.profile
```

### Install turnkey

```
git clone https://github.com/schollz/raspberry-pi-turnkey.git
```

### Install Hostapd

```
sudo systemctl stop dnsmasq && sudo systemctl stop hostapd

echo 'interface wlan0
static ip_address=192.168.4.1/24' | sudo tee --append /etc/dhcpcd.conf
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig  

sudo systemctl daemon-reload
sudo systemctl restart dhcpcd

echo 'interface=wlan0
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h' | sudo tee --append /etc/dnsmasq.conf

echo 'interface=wlan0
driver=nl80211
ssid=ConnectToConnect
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=ConnectToConnect
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP' | sudo tee --append /etc/hostapd/hostapd.conf

echo 'DAEMON_CONF="/etc/hostapd/hostapd.conf"' | sudo tee --append /etc/default/hostapd

sudo systemctl start hostapd && sudo systemctl start dnsmasq
```

(_Sidenote:_ I save an image `intermediate.img` at this point so its easy to go back.)

Then open up the `root` crontab

```
$ sudo crontab -e
```

And add the following line:

```
@reboot cd /home/pi/raspberry-pi-turnkey && /usr/bin/sudo /usr/bin/python3 startup.py
```

### Reboot the Pi

```
$ sudo reboot now
```

The Pi will reboot itself twice, please be patient when it does this. At this point you should be able to see the "ConnectToConnect" AP address.

@# 3. Resize Raspberry Pi SD image

Put the newly made image into Ubuntu and do

```
$ xhost +local:
$ sudo gparted-pkexec
```

Right click on the SD card image and do "Unmount". Then right click on the image and do "Resize/Move" and change the size to `2400`. 

Then you can copy the image to your computer using the following command

```
$ sudo dd if=/dev/mmcblk0 of=~/2018-turnkey.img bs=1M count=2400 status=progress
```

Again `/dev/mmcblk0` is your SD card which you can determine using `fdisk -l`. 

## 4. Test the resized image

The new image will be in `~/2018-turnkey.img` which you can use to flash other SD cards. To test it out, get a new SD card and do:

```
$ sudo dd bs=4M if=~/2018-turnkey.img of=/dev/mmcblk0 conv=fsync status=progress
```

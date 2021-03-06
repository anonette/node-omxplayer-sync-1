Node.js OMXPlayer Synchronization
=================================

This repo contains code for using Raspberry Pis to create low-cost,
highly scalable video walls or multi-channel video installations. Setup
is simple, and the software takes care of the details for you:

1. Get a TV or computer monitor with an HDMI input. Better yet, get 10.
   Or 100.
2. Plug a Pi into each screen.
3. Connect the Pis to a network, load the software onto them, and copy
   your video files onto the Pis.
4. Start the Pis.

The Pis will talk to eachother over the network using a simple
peer-to-peer protocol to figure out how to stay in sync. You can remove
and add Pis while the cluster is running, and they will quickly fall
into sync with the other Pis.

## Setup

```
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash
#load nvm
. ~/.nvm/nvm.sh

nvm install 0.12
nvm alias default 0.12
echo 'default' > ~/.nvmrc
nvm use 0.12
sudo apt update
sudo apt install git omxplayer byobu -y
#d/l test video and rename to fit config.js
wget https://github.com/turingmachine/omxplayer-sync/raw/master/synctest.mp4
mv synctest.mp4 video.mp4
git clone https://github.com/anonette/node-omxplayer-sync.git 
cd node-omxplayer-sync
npm install
```
1. Edit `config.local.js` to use the filename of your video file. See
   `config.js` for a list of other available settings.

2. Start the service using `node main.js` 

### start on boot
add service file at `/etc/systemd/system/omx.service`

```
[Unit]
Description=node-omxplayer-sync
After=network.target

[Service]
ExecStart=/home/pi/node-omxplayer-sync/launcher.sh
WorkingDirectory=/home/pi/node-omxplayer-sync
StandardOutput=inherit
StandardError=inherit
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```
and a launcher.sh  
```
#!/bin/bash
. /home/pi/.nvm/nvm.sh
nvm run default main.js 
```
see [docs]
9https://www.raspberrypi.org/documentation/linux/usage/systemd.md) for more

## Video Format

Video files should not be temporally compressed (eg. MJPEG or Apple
PRORES), or they should have a very low GOP length (the interval between
i-frames or keyframes). You can use `script/get_keyframe_intervals` to
inspect the GOP of a file (depends on ffmpeg and ffprobe).

## Configuration

Default configuration resides in [config.js](config.js). The default
configuration can be overriden by creating a `config.local.js` file. See
documentation in [config.js](config.js) for details on each setting. The
configuration files are JavaScript to allow settings based on
calculations. This means that you need to be sure your configuration is
valid JavaScript, or the whole configuration file will be ignored.

An example `config.local.js` file:

    var FPS = 29.97;

    module.exports = {
      fps: FPS,
      toleranceSecs: 1 / FPS * 3,
      smoothingWindowMs: 1e3 / FPS * 15
    };


### Deploy
to deploy an edited img to all raspiberries. use kparttx to mount the img  


```
sudo pacman -S kaprtx 
sudo kpartx -av 2018-11-13-raspbian-stretch-lite.img
sudo mkdir -p /mnt/pi/boot && sudo mkdir -p /mnt/pi/rootfs
sudo mount /dev/mapper/loop0p1 /mnt/pi/boot
sudo mount /dev/mapper/loop0p2 /mnt/pi/rootfs
```
edit the stock img, add `ssh` to boot mount 
```
sudo touch /mnt/pi/boot/ssh
```
and `wpa_supplicant.conf` with your wifi conf (if you are going to use wifi)
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=«your_ISO-3166-1_two-letter_country_code»

network={
    ssid="«your_SSID»"
    psk="«your_PSK»"
    key_mgmt=WPA-PSK
}
```
add ssh-key to pi
```
mkdir -p /mnt/pi/rootfs/home/pi/.ssh
sudo cat ~/.ssh/anonette.pub >>  /mnt/pi/rootfs/home/pi/.ssh/authorized_keys
sudo chown -R 1000:1000 /mnt/pi/rootfs/home/pi/.ssh/

```

when done , unmount
```
sudo umount /mnt/pi/boot && sudo umount /mnt/pi/rootfs
sudo kpartx -d 2018-11-13-raspbian-stretch-lite.img
```
[source](https://www.raspberrypi.org/forums/viewtopic.php?t=28860#p254654)

included is a script that will mount/unmount and write the edited img to your SDcard, see `rsp-mount.sh`

## Management

Once the pis are running, you can access a control panel by opening a
web browser and navigating to any of the pis using their hostnames or
ips. For example, [http://player-1.local](http://player-1.local) or
[http://10.0.0.2](http://10.0.0.2). You can also try navigating to
[http://players.local](http://players.local), which should take you to
whichever player is currently the master.

The control panel will show you the status of the pis and allow you to
perform the following actions:

* Refresh: quickly restart all the players' software
* Stop: stop the software. There will be no way to restart it without
  logging into the players using SSH or a terminal.
* Restart: reboot all the players' hardware.
* Halt: turn off all the players' hardware. You will need to power cycle
  the players to get them to restart.

## Technical Overview

The control software is written in Node.js, uses OMXPlayer to play the
video, and OSC to talk over the network. The controller uses a
dumbed-down consensus algorithm to elect a peer as master, which all the
other Pis become slaves to, syncing their video by seeking, pausing, and
slightly tweaking their play speed.

To make management of the cluster easier, there is a [system automation
tool](https://github.com/heisters/node-omxplayer-sync-devops) that uses
Ansible to provision, deploy, and configure many Pis at the same time.

# A UPnP Bluetooth music player

## Introduction


I have an old Raspberry Pi B and I wanted to play the music I have on my NAS server and Linux desktop computer.
I wanted to control the media server and player from my cellphone, so not need for any display, web access, etc.

It connects using a Bluetooth USB dongle to a Bluetooth receiver connected to my music amplifier.
The memory and CPU limitations of the RPi B required for the bare minimum software. That ruled out media distros and packages.


## Hardware

- A working RPi, any model.
- Network connection.
- In my case, a USB Bluetooth dongle. Your RPi may already have Bluetooth.
- A bluetooth receptor for your audio equipment.


## Required knowledge

- How to install and configure a distro in a RPi.
- ssh
- Command line usage. I used `bash`.
- Intallation of packages through `apt`
- Editing files.
- A basic knowledge of:
  - `systemd` and `systemctl`.
  - `bluetoothctl`
  - Alsa
  - UPnP

  
## Architecture
  


## Installation steps

### Install and confugre the RPi OS

As the server won't have  a display, I used Raspbian Buster Lite.
- Download the last image.
- I Installed in SD card using Balena etcher.
  There are many other ways, like the little more dangerous but always effective:
  ```
  # If you don't know how perilous is 'dd', better don't use it.
  dd if={raspbian-image-here} of=/dev/{sdcard-block-dev-here} bs=100M
   ```
- Create an empty file called `ssh` in the root directory. It's required to be able to log through the `ssh` command.
- Connect the RPi to the LAN (the model B has only ethernet), the Bluetooth dongle (if needed), put the SD card in, and connect the RPi to a power source.
- There are many ways to get the IP address of the RPi. I just compared the output of:
  ```
  nmap -sP 192.168.0.0/24
  ```
  Before and after powering the RPi and from that I got the new IP in the LAN.
- Log in inmmediately to the RPi and change the passwrod to something more secure than the default "rasperry".
  ```
  ssh pi@{rpi-ip}
  $ passwd
  ```
- Become `root`. Everything you will do from now on requires `root` permission. IMO, it's the simplest way.
  ```
  sudo su -
  ```
- Update the distro. It will take some time that you could use in other way than watching the screen... Or not.
  ```
  apt get && apt update
  ```
  Reboot the server
  ```
  shutdown -r now
  ```
- Configure the RPi
  - Change the hostname (2 Network Options -> N1 Hostname)
  - I changed the locale to en_US.UTF-8. It took more time than I expected... (4 Localisation Options -> I1 Change Locale)
  - Set the timezone (4 Localisation Options -> I2 Change Timezone)
  - Extend filesystem to use the whole card (7 Advanced Options -> A1 Expand Filesystem)
  - Memory split to use the less possible memory for the video card. 0 doesn't to work. 1MB at least. (7 Advanced Options ->A3 Memory Split)
  ```
  raspi-config
  ```
- As I will always connect to the server from the same machine, I added the IP address to `/etc/hosts`.
  I also  added my computer public key to the list of authorized keys.
  That way I can connect without entering the password to the RPi by doing: `ssh pi@player`
  - In the local machine, edit `/etc/hosts`. You will need to use `sudo` because the file cannot be written by regular users.
    Add a line like:
    ```
    {your-RPi-ip-address}    player  # Raspberry Pi B (Eth)
    ```
    and save the file
  - In the RPi as the `pi` user.
	```
    mkdir ~/.ssh
    chmod 700 ~/.ssh
    touch ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys
    cat >> ~/.ssh/authorized_keys
    {Paste your public key. It's usually ~/.ssh/id_dsa.pub}
    Ctrl-D
    ```
- Storing temporary files in the SD card is not only slow but, in the long term, degrades the card (the available space decreases).
  A solution is to mount an in-memory filesystem (`tmpfs`) using part of the available RAM.
  The most written directories are `/var/cache` (by far) and `/var/log` (not as much as I expected).
  Therefore, become `root` again and add the following lines to `/etc/fstab`:
  ```
  tmpfs       /var/cache  tmpfs   defaults,noatime,nosuid,size=70m    0   0
  tmpfs       /var/log    tmpfs   defaults,noatime,nosuid,size=5m 0   0
  ```
  Remove whatever is in `var/cache` because it will be hidden by the mounted system.
  ```
  rm -rf /var/cache/*
  ```
  Mount the filesystems to check that the configuration is ok. If not, edit `/etc/fstab` and check.
  If that doesn't work, remove the changes. A malformed `/etc/fstab` can prevent the system to boot and you will need to start from scratch.
  ```
  mount /var/cache
  mount /var/log
  ```
  A 70MB of space is ok for the nomal use of `apt`.
  However, if you are going to update the distro, you may need to unmount `/var/cache` temporarily, because `apt` it may run out of space.
  ```
  # Only when you need to upgrade or dist-upgrade.
  umount /var/cache
  apt update && apt upgrade
  rm -rf /var/cache/*
  mount /var/cache
  ```

### Install the player (called "Renderer" in the UPnP world).

That part is fairly simple. The services have to be installed after their dependencies are met, e.g, `mpd` cannot be installed or configured before `blue-alsa`.

The steps are:

- Find the Bluetooth device to pair, trust and connect to it. Make the server reconnect to it when starts.
- Install and configure `blue-alsa`, `mpd`, and `upmpdcli`

#### Configure Bluetooth.

The default installation of the Bluetooth service complains about a missing `sap` plugin
If you don't want to see those errors, you need to pass a flag to the `bluetooth` executable when starts.
The command line that starts the service is in the `[Service]` section, under `ExecStart`.

- Edit the full service configuration:
  ```
  systemctl edit --full bluetooth.service
  ```
  Change the line
  ```
  ExecStart=/usr/lib/bluetooth/bluetoothd
  ```
  to
  ```
  ExecStart=/usr/lib/bluetooth/bluetoothd --noplugin=sap
  ```
  And save.

  Check the service configuration:
  ```
  systemctl cat bluetooth.service
  ```
  It outputs something like this:
  ```
  # /etc/systemd/system/bluetooth.service
  [Unit]
  Description=Bluetooth service
  Documentation=man:bluetoothd(8)
  ConditionPathIsDirectory=/sys/class/bluetooth

  [Service]
  Type=dbus
  BusName=org.bluez
  NotifyAccess=main
  WatchdogSec=10
  Restart=on-failure
  ExecStart=/usr/lib/bluetooth/bluetoothd --noplugin=sap
  CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
  LimitNPROC=1
  ProtectHome=true
  ProtectSystem=full

  [Install]
  WantedBy=bluetooth.target
  Alias=dbus-org.bluez.service
  #
  ```
  Reload the service configuration
  ```
  systemctl daemon-reload
  ```

- Connect to the Bluetooth device
  ```
  bluetoothctl
  ```
  Start scan
  ```
  [bluetooth]# power on
  [bluetooth]# agent on
  [bluetooth]# scan on
  Discovery started
  [CHG] Controller 00:11:22:33:44:55 Discovering: yes
  [NEW] Device 00:E5:D4:C3:B2:A1 MY BLUETOOTH DEVICE
  [bluetooth] scan off
  Discovery stopped
  [bluetooth] pair 00:E5:D4:C3:B2:A1
  Attempting to pair with 00:E5:D4:C3:B2:A1
  [bluetooth] trust 00:E5:D4:C3:B2:A1
  Changing 00:E5:D4:C3:B2:A1 trust succeeded
  [bluetooth] connect 00:E5:D4:C3:B2:A1
  Attempting to connect to 00:E5:D4:C3:B2:A1
  Connection successful
  [CHG] Device 00:E5:D4:C3:B2:A1 ServicesResolved: yes
  [MY BLUETOOTH DEVICE] info
  Device 00:E5:D4:C3:B2:A1 (public)
        Name: MY BLUETOOTH DEVICE
        Alias: MY BLUETOOTH DEVICE
        Class: 0x00240404
        Icon: audio-card
        Paired: yes
        Trusted: yes
        Blocked: no
        Connected: yes
        LegacyPairing: no
        UUID: Audio Sink                (0000110b-0000-1000-8000-00803729fa2d)
  [MY BLUETOOTH DEVICE] quit
  ```
  This will work until the server is reset.
  As the device is already paired and trusted, we will only have to connect to it at system start.
  This will be configured later.


#### Install and configure `blue-alsa`

- Install `blue-alsa`
  `alsa` lacks many features compared to `pulseaudio`, like mixing multiple audio sources, but it doesn't make much sense to use the latter because there is only one source (the music files) and one sink (the bluetooth device).
  ```
  apt install bluealsa
  systemctl start bluealsa
  ```
  The Bluetooth audio device has to be set as the default output in Alsa.
  In my case, `/etc/asound.conf` didn't exist, so I created it
  ```
  cat > /etc/asound.conf <<END
pcm.my_bluetooth_device {
    type plug
    slave.pcm {
        type bluealsa
        device "00:E5:D4:C3:B2:A1"
        profile "a2dp"
    }
    hint {
        show on
        description "MY BLUETOOTH DEVICE"
    }
}

pcm.!default "my_bluetooth_device"
END
  ```
  Now you can test playing a `wav` file. This can be done as the `pi` user, i.e., you do not need to be `root` for that.
  Do not use, `mp3`, `ogg` or whatever other type because the file will be played as-is and your ears and audio equipment will suffer.
  ```
  aplay sound.wav
  ```
  It should play the file.


#### Install and configure `mpd`
  ```
  apt install mpd
  ```
  - Configure `mpd`
    Edit `/etc/mpd.conf` and change the following values
      - There is no need to have a separate log file. Change the log to `syslog`.
      - Comment out `state_file`. It's not needed.
      - In the `audio_output` section, change the `mixer_type` to `"software"`
  - I want the `mpd` to start at medium volume:
    ```
    systemctl edit mpd.service
    ```
    Add:
    ```
    [Service]
    ExecStartPost=/usr/bin/mpc volume 50
    ```
    Of course, you can check the configuration by doing:
    ```
    systemctl cat mpd.service
    ...
    ```


#### Install and configure `upmpdcli`
  ```
  apt install mpd upmpdcli
  ```
- Configure `upmpdcli`
  Edit `/etc/upmpdcli.conf` and change `friendlyname` to whatever you like. It will be the name you see when you scan for renderers.



#### Connect to the Bluetooth audio device at start

As I mentioned, the RPi needs to connect to the Bluetooth device every time it starts.
I for a service that started after all the dependencieas have already started.
Due to my limited knowledge of `systemd`, the best candidate was `rc-local.service`.
So I edited ol' good `/etc/rc.local` and added the following line before the `exit 0` command.

```
bluetoothctl connect 00:E5:D4:C3:B2:A1 # MY BLUETOOTH DEVICE
```


#### Optizations

The RPi I used had very limited resources: A 100MB eth network, 512MB of RAM, and a 800MHz procesor.
So I looked for all processes that ran regurlarly and I moved them at a time I was pretty sure I would be sleeping.

These are the steps:

- Look for the configured timers:
  ```
  systemctl list-timers
  ```
  Outputs something like this:
  ```
  NEXT                         LEFT          LAST                         PASSED  UNIT                         ACTIVATES
  Thu 2020-03-26 04:00:00 -03  2h 56min left Wed 2020-03-25 04:00:04 -03  21h ago logrotate.timer              logrotate.service
  Thu 2020-03-26 05:00:43 -03  3h 57min left Wed 2020-03-25 05:01:54 -03  20h ago apt-daily.timer              apt-daily.service
  Thu 2020-03-26 05:30:00 -03  4h 26min left n/a                          n/a     systemd-tmpfiles-clean.timer systemd-tmpfiles-clean.service
  Thu 2020-03-26 06:02:53 -03  4h 59min left Wed 2020-03-25 06:07:29 -03  18h ago apt-daily-upgrade.timer      apt-daily-upgrade.service
  Thu 2020-03-26 06:30:00 -03  5h 26min left Wed 2020-03-25 06:30:01 -03  18h ago man-db.timer                 man-db.service

  5 timers listed.
  Pass --all to see loaded but inactive timers, too.
  ```

- Edit when the timers are triggered.
  I was user to `crontab`, so when I learnt about the many ways to configure the moment a timer is triggered, I was ovewhelmed because of the number of ways.
  I ended configuring the timers in similar way to `cron`, so it's quite possible for this to be done better.
  An example from the `apt-daily.timer`:
  ```
  systemctl cat apt-daily.timer
  ```
  Outputs:
  ```
  # /etc/systemd/system/apt-daily.timer
  [Unit]
  Description=Daily apt download activities

  [Timer]
  OnCalendar=*-*-* 6,18:00
  RandomizedDelaySec=12h
  Persistent=true

  [Install]
  WantedBy=timers.target
  ```
  Edit the configuration:
  ```
  systemctl edit --full apt-daily.timer
  ```
  And chane `OnCalendar` and `RandomizedDelaySec` to be:
  ```
  OnCalendar=*-*-* 5:00
  RandomizedDelaySec=10m
  ```
  To run it at some random time between 4:50 and 5:10.



That's all.


### Media provider (called "Library" in UPnP).


#### Install and configure the UPnP media servers.

I have a NAS that came with UPnP installed, so no issues there.
However, some of my music is in an Ubuntu machine, so I installed a UPnP server there:
  - Install `minidlna`:
    ```
    sudo apt install minidlna
    ```
    When started, it will run the service under the `minidlna` user.
    In my case, the music files are under my home directory. To allow the user `minilna` to access the files, either they have to be outside the home directory (as the permissions do not allow to read the contents of other user's home), or `minidlna` has access to it.
    Although the first option is the safest, I'm too lazy to reorganize my music folders, so I just added `minidlna` to my group:
    ```
    sudo adduser minidlna <my-username>
    ```
    After that, configure `/etc/minidlna.conf`:
    Edit `friendly_name` to be the name you want it to appear in the media provider list ("Libraries"). I will use "music".
    Edit the `media_dir` entry to point to the directory where the music is.
    In my case, as I have several directories, I just added one `media_dir` line per directory.

    Restart `minidlna`:
    ```
    sudo service minidlna restart
    ```


### Install a UPnP controller.

  - Install some UPnP app in your phone / tablet / whatever. I couldn't find an open source Android phone app that worked, so I tried the commercial ones until I found one I liked. Some of them are free (as beer, ads included).
  - Look under "Renderers" for the RPi friendly name. In this case, "player".
  - Browse the "Libraries" section. Select the DLNA server friendly name. In this case, "music".
  - Add an album and play!

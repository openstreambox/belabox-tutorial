# Setting up

These instructions were written for L4T 32.4.4 (Based on Ubuntu 18.4 LTS) and last tested on Dec 20th, 2020. Consider this a temporary guide to setting up BELABOX while a more convenient solution is being developed.

## Step -1

You'll need another Internet-connected machine to serve as the ingest for your srtla-bonded stream. This can be a low cost VPS or a Linux computer (even a low power RPi) at home. Follow the *receiver* instructions in the [srtla readme](https://github.com/BELABOX/srtla/) for setting it up.

## Step 1

Set up a micro SD card with L4T following the instructions from NVIDIA [for Jetson Nano 4GB](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit) or [for Jetson Nano 2GB](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-2gb-devkit). Note that you can do the initial setup either using a monitor, mouse & keyboard or in headless mode using the USB serial console. For the rest of the tutorial we'll assume that the system was set up correctly and that you have SSH access to the Jetson Nano via the Ethernet network.

## Step 2

Updating the system packages (this may take a while):

    sudo apt update
    sudo apt dist-upgrade

## Step 3

Installing the required dependencies:

    sudo apt install nano build-essential git tcl libssl1.0-dev nodejs npm usb-modeswitch libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev
    
## Step 4

Add the google DNS servers so you have some resolvers accessible through any Internet-connected interfaces, as opposed to the servers accessible through a single mobile operator that you may be getting from DHCP:

    printf "\nnameserver 8.8.8.8\nnameserver 8.8.4.4\n" | sudo tee -a /etc/resolvconf/resolv.conf.d/head

Let's set up source routing for any wired modems. This uses a dhclient hook, which will execute for dhcp entries in `/etc/network/interfaces` or if called manually, but not for NetworkManager-managed devices. `/etc/network/interfaces` takes over configuring the ethX and usbX network interfaces from NetworkManager

    sudo wget https://raw.githubusercontent.com/BELABOX/tutorial/main/dhclient-source-routing -O /etc/dhcp/dhclient-exit-hooks.d/dhclient-source-routing
    sudo wget https://raw.githubusercontent.com/BELABOX/tutorial/main/interfaces -O /etc/network/interfaces
    printf "100 usb0\n101 usb1\n102 usb2\n103 usb3\n104 usb4\n" | sudo tee -a /etc/iproute2/rt_tables
    printf "110 eth0\n111 eth1\n112 eth2\n113 eth3\n114 eth4\n" | sudo tee -a /etc/iproute2/rt_tables

Also set up source routing for WiFi with NetworkManager:

    sudo wget https://raw.githubusercontent.com/BELABOX/tutorial/main/nm-source-routing -O /etc/NetworkManager/dispatcher.d/nm-source-routing
    sudo chmod 755 /etc/NetworkManager/dispatcher.d/nm-source-routing
    printf "120 wlan0\n121 wlan1\n122 wlan2\n123 wlan3\n124 wlan4\n" | sudo tee -a /etc/iproute2/rt_tables

If you have a WiFi adapter fitted, you can connect to a WiFi network with `sudo nmcli device wifi connect <AP NAME> password <WPA password>` after rebooting.

We use the `/etc/network/interfaces` configuration because it seems more reliable than Networkmanager at always bringing up all the interfaces. It also brings up all the modems even when they use the same MAC address (which is the case for several Huawei models), unlike NetworkManager.

## Step 5

Disable the virtual Ethernet interface as it will cause naming conflicts if you use modems that get enumerated as `usbX` devices.

    sudo systemctl disable nv-l4t-usb-device-mode.service
    sudo reboot

## Step 6

Installing (the BELABOX fork of) SRT:

    cd
    git clone https://github.com/BELABOX/srt.git
    cd srt
    ./configure --prefix=/usr/local
    make -j4
    sudo make install
    sudo ldconfig

## Step 7

Building belacoder:

    cd
    git clone https://github.com/BELABOX/belacoder.git
    cd belacoder
    make
    
## Step 8

Building srtla:

    cd
    git clone https://github.com/BELABOX/srtla.git
    cd srtla
    make

## Step 9

Setting up belaUI:

    cd
    git clone https://github.com/BELABOX/belaUI.git
    cd belaUI
    git checkout ws_nodejs

Install the Node.js dependencies:

    npm install

Edit `setup.json` with the paths to the `belacoder` and `srtla` directories.

You can start the web interface to test it with:

    sudo nodejs belaUI.js

At this point BELABOX is ready to use, assuming that you have a capture card / other v4l2 input connected. Open `http://address_of_the_jetson` in a web browser. It will first ask you to set a password for access.

See the [belacoder readme](https://github.com/BELABOX/belacoder) for information about the available pipelines you can select in the *Encoder settings* menu of the web interface. Configure the *srtla settings* with the data for your ingest configured at *Step -1*.

After setting up and confirming that everything is working correctly, you can install belaUI as a system service that starts automatically at boot by running:

    sudo ./install_service.sh

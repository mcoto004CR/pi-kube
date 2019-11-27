# pi-kube
Setup K8 on raspberry
Note: you need to be familiar with Linux/Debian and Raspberry.

Get a copy of https://www.raspberrypi.org/downloads/raspbian/
Flash the a 64Gb min SD Card using fletcher or other image tool

1.- Configure you wifi (I recommend Ethernet if you can)
  Use ifconfig, to get your current ip
  Copy your IP

  Below will enable a static IP so you can easy SSH
  sudo nano /etc/dhcpcd.conf and enter the following data
    interface wlan0
    static ip_address=192.168.XX.XXX/24   (XX.XX is the IP you copied in the previous point)
    static routers=192.168.X.X  (this is your router IP gateway)
    static domain_name_servers=8.8.8.8  (setup this to you prefered DNS or Google DNS)

   Below you can setup your WiFI SSID
   sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
    network={
        ssid="your SSID"
        psk="your WiFi Password"
    }
  
  Reboot
  To check if it's working
    ifconfig
    iwconfig
    or ping google.com

2.- Enable SSH on Pi
  sudo raspi-config
  



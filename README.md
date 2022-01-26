# rhf2s308_experience
Experiences dealing with rhf2s308 for helium mining


# Connecting to Wifi
The stock box only uses wifi for diagnostics/debugging/admin.  Thus it creates an access point. If you attempt to connect to Wifi using the Helium app, you might get lucky and do so before this service has created the access point, making it seem like it's working, but the hotspot wifi will not be consistent as the access point services will be constantly battling for access to the network interface.  Here I attempt to disable this access point so that I can use the Wifi interface as a network connection. 

First, we must disable the two services that initialize the access point:
``` bash
systemctl stop init_wifi
systemctl disable init_wifi
systemctl mask init_wifi

systemctl stop create_ap
systemctl disable create_ap
systemctl mask create_ap
```

The networking on this device is managed by `connman`, but if you try to connect to wifi using `connmanclt` right now, you will get a `No Carrier` error. This is because (for some reason) there are two instances of the `wpa_supplicant` service:

``` bash 
root@rhf2s308:# ps -aux | grep wpa_supplicant
root       536  0.0  0.1  12564  1704 ?        Ss   14:11   0:00 /sbin/wpa_supplicant -u -s -O /run/wpa_supplicant
root       795  0.0  0.0  12712   836 ?        Ss   14:11   0:00 wpa_supplicant -B -c/etc/wpa_supplicant/wpa_supplicant.conf -iwlan0 -Dnl80211,wext
```

The first one is launched by `systemd` and is good.  The second one is rogue.  Let's hunt down its parent.



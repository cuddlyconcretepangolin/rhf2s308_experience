# rhf2s308_experiences
Experiences dealing with rhf2s308 for helium mining. 

I am documenting this for my own use (when inevitably I need to flash the firmware and start over).  I would not recommend anyone execute these commands unless you know what you are doing. I take no responsibility if you mess up your miner.

# Obtaining shell access

It seems that RisingHF has begun to lock us out of the devices.  The credentials in the manual (https://risinghf-official-website.oss-cn-shenzhen.aliyuncs.com/static/file/product/GX5CI5KeGI7Dl7fFvTrLrA==.pdf) are stated as username/password: `rxhf`/`risinghf`.  These work for a few moments after a firmware flash, but suddenly change moments later. A solution is to quickly create your own username and password, and give yourself sudo access.

1) Flash firmware (https://risinghf-pkg.oss-cn-shenzhen.aliyuncs.com/risinghf/firmware/rhf2s308-1.0.4-20220107-helium.img.zip)
2) Unplug USB.
3) Disconnect power.
4) Reconnect power.
6) Immediately start attempting to log in with `rxhf`/`risinghf` username/password combo (via serial (i.e. USB). Use PuTTY on Windows. Refer to the manual above for this as it's accurate this far). Remember that this is Linux so the password field will not show placeholders as you type.
8) As soon as you're logged in, create a new user (username is your choice): `sudo adduser username`
9) Then give your new account sudo access: `sudo adduser username sudo`

(Aside: several moments later, if you try to log in with `rxhf/risinghf` you likely will not be able to.  If you don't want a user account with mystery credentials on *your own hardware*, then you can change the password with `sudo pw rxhf` since you now have your own account with root access).

## Turn on `ssh` access

Run `sudo openssh`.  It seems that the ssh service gets periodically masked again. I add
```
./usr/local/sbin/openssh & 
```
to my `/etc/rc.local` file to unmask on boot, or to crontab to run periodically.


# Accessing the web dashboard

``` sudo openweb``` will start the web server and give access on port 80 to the dashboard discussed in the manual. The information is generic, and mostly irrelevant to Helium mining.


# Enabling Wifi for Internet

The stock box only uses wifi for diagnostics/debugging/admin.  Thus it creates an access point. If you attempt to connect to Wifi using the Helium app, you might get lucky and do so before this service has created the access point, making it seem like it's working, but the hotspot wifi will not be consistent as the access point services will be constantly battling for access to the wlan0 network interface.  Here I attempt to disable this access point so that I can use the Wifi interface as an internet connection. 

First, we must disable the two services that initialize the access point:
``` bash
systemctl stop init_wifi
systemctl disable init_wifi
systemctl mask init_wifi

systemctl stop create_ap
systemctl disable create_ap
systemctl mask create_ap
```

The networking on this device is managed by `connman`, but if you try to connect to wifi using `connmanctl` right now, you will get a `No Carrier` error. This is because (for some reason) there are two instances of the `wpa_supplicant` service started. 

``` bash 
root@rhf2s308:# ps -aux | grep wpa_supplicant
root       536  0.0  0.1  12564  1704 ?        Ss   14:11   0:00 /sbin/wpa_supplicant -u -s -O /run/wpa_supplicant
root       795  0.0  0.0  12712   836 ?        Ss   14:11   0:00 wpa_supplicant -B -c/etc/wpa_supplicant/wpa_supplicant.conf -iwlan0 -Dnl80211,wext
```

The first one is launched by systemd (you can verify by comparing this command to the `ExecStart` entry in `/usr/lib/systemd/system/wpa_supplicant.service`) and the second is launched by dhcpcd via `/lib/dhcpcd/dhcpcd-hooks/10-wpa_supplicant`.  It sounds like this is an issue with Raspberry Pis in general (https://forums.raspberrypi.com/viewtopic.php?t=292401) and it is safe to disable the systemd service, as dhcpcd will take care of launching wpa_supplicant. In fact, if you look at the systemd service status (`systemctl status wpa_supplicant`), you might even see that it's full of complaints that the driver is already in use ("nl80211: kernel reports: Match already configured").  So we stop it and mask it as well:

``` bash 
systemctl stop wpa_supplicant
systemctl disable wpa_supplicant
systemctl mask wpa_supplicant
```

Now, all that's left to do is add our network credentials to the wpa_supplicant configuration file.  Modify `/etc/wpa_supplicant/wpa_supplicant.conf` to look like the following: (the first two lines should already be present; obviously put your own network's credentials. Use `key_mgmt=NONE` for unsecured Wifi (untested)). 
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
	ssid="MY-WIFI-NETWORK-NAME"
 	psk="My-Password"
}
```
After doing all this and rebooting, I was automatically connected to wifi when the system came back up:

``` bash 
username@rhf2s308:~ $ ifconfig
...
wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.163  netmask 255.255.255.0  broadcast 192.168.0.255
        inet6 fe80::84b9:6e67:6246:417e  prefixlen 64  scopeid 0x20<link>
        ether 20:50:e7:10:9d:4b  txqueuelen 1000  (Ethernet)
        RX packets 2600  bytes 2244628 (2.1 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2220  bytes 467805 (456.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
...

```

My concern was that the Helium miner would have the `eth0` interface hard coded and would not pick up on `wlan0` being connected, but after pulling the ethernet plug, the `peer_book` went relayed as expected (`docker exec miner miner peer book -s`) as I do not have port forwarding set up for the Wifi IP.  Since reboot, I have successfully heard and submitted (over wifi) a witness, so it seems to be working:

```bash
root@rhf2s308:~# docker exec miner cat /var/data/log/console.log | grep --text witness

2022-01-26 15:54:17.375 8 [info] <0.1642.0>@miner_onion_server:decrypt:{372,13} could not decrypt packet received via radio: treating as a witness
2022-01-26 15:54:17.377 8 [info] <0.3142.0>@miner_onion_server:send_witness:{188,13} sending witness at RSSI: -99, Frequency: 904.7, SNR: -2.75
2022-01-26 15:54:17.573 8 [warning] <0.3142.0>@miner_onion_server:send_witness:{243,37} failed to dial challenger "/p2p/redacted": not_found
2022-01-26 15:54:47.592 8 [info] <0.3142.0>@miner_onion_server:send_witness:{246,37} re-sending witness at RSSI: -99, Frequency: 904.7, SNR: -2.75
2022-01-26 15:54:49.193 8 [info] <0.3142.0>@miner_onion_server:send_witness:{251,37} successfully sent witness to challenger "/p2p/redacted" with RSSI: -99, Frequency: 904.7, SNR: -2.75
```

(i.e. it heard a witness, then ~30 seconds later was successful in dialing the challenger)


## Frozen sync
On Wednesday, Feb 2, my miner started to fall behind the blockchain and the logs were full of messages along the lines of `no plausible blocks in batch`. I do not have a fix for this; I do not know why it happened, but I have a workaround. It amounts to installing the miner on another computer, letting it sync, then copying over the entire blockchain directory. This worked for me, and my miner is back witnessing and earning rewards.  I will note that if you don't have some basic knowledge of Unix, this is probably not going to be possible for you to undertake, as it's not a straight copy/paste of my commands.

You need another computer (not sure requirements, but I used a Ubuntu server I have at home).  Using this other computer (I'll call it "B"):
1) Install the Helium miner: https://github.com/helium/devdocs/blob/master/blockchain/run-your-own-miner.md.
2) Start the miner and let it sync.  It will download a snapshot and sync. Mine took about 4 hours, but it's a pretty powerful computer. It was much faster than the low-powered hotspot. Keep an eye on the logs and move to 3) once it's caught up.
3) Once it's caught up, stop the miner. On B, run: `sudo docker stop miner`.
4) Stop the miner on the RisingHF hotspot: `sudo docker stop miner`.
5) Remove the miner data directory: On the hotspot: `sudo rm -r /opt/helium/miner_data`
6) Copy over the contents of `miner_data` folder from Computer B to hotspot. This exact command depends on where you chose to locally store the miner data. In the link in step (1) above, they chose `~/miner_data`. I used rsync to copy over the network. I ran the following command _from the hotspot_ as root is needed to write to `/opt/helium`. On the hotspot: 
```bash 
# user = username on Computer B 
# 192.168.0.333 = Local IP Address of Computer B (yours will be different)

rsync -av --info=progress2 user@192.168.0.333:/home/user/miner_data/ /opt/helium/miner_data
```
7) Start the miner on the hotspot: ```sudo docker start miner```
8) The hot spot should now only be several minutes behind (however long the copy operation took). 


The nice thing about this solution is that now I have another miner running on Computer B (that I will just allow to continue running so it's always in sync), and I can copy over the blockchain if it ever falls behind again, or if I need to resync firmware in the future.

# Migrating to Light Hotspots
During the update to light Hotspots, several patches have been released to improve POC. RisingHF has not pushed these updates (to be fair, I have only given them hours to do so...this is admittedly unreasonable expectations on my part).  Nonetheless, I have upgraded my miner manually by updating the docker-compose.yaml to pull the updated miner images from Helium. First, though, we must disable RisingHF's agent services that will overwrite our changes.  I do not know why there are two of these, but I disabled them both and have not experienced any troubles:

```bash
systemctl stop agent-upgrade2
systemctl mask agent-upgrade2

systemctl stop agent
systemctl mask agent
```
Now we will edit `/opt/helium/docker-compose.yaml` and change the two references to the helium miner image to be consistent with the most recent miner release from https://quay.io/repository/team-helium/miner?tab=tags. At time of writing, this is `2022.05.11.0`.

There are two references: one in the `image:` line, and one in the `volumes:` section (for the sys.config).

Once this file has been updated, we need to ensure we are in the same directory as this file, and then build the new miner:

```bash
cd /opt/helium/
docker-compose up -d
```

Once it's finished, verify that the new version is active by running

```bash 
docker exec miner miner versions
```

*Note that doing this (more specifically disabling the agent services) will prevent you from receiving future OTA (over the wire?) miner updates.  My plan is to keep it like this for a while until after the official migration and see how it plays out.  Then once it seems like RisingHF is on top of everything again, I will enable the agent again. *

#


# wpa-supplicant-pmk
custom version of wpa_supplicant (v1.0, 2012-05-10), written **FOR RESEARCH PURPOSES ONLY**. 

it prints the PMK during a WPA2 authentication procedure. this PMK can then be used on Wireshark to decrypt WiFi data frames. 

this custom version of wpa_supplicant was tested w/ the following platforms:
* raspberry pi model B+, V1 2, running Raspbian GNU/Linux 7 (wheezy)
* wireshark v2.2.3-0-g57531cd, running on Mac OSX El Capitan 10.11.5 (15F34)

# usage

after compilation, run the resulting wpa_supplicant binary as follows, to initiate a connection to a WPA2 Enterprise WiFi network:

```
pi@raspberrypi $ sudo /<custom-wpa-supplicant-dir>/wpa_supplicant -Dwext -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf 
```

the above assumes that you have previously setup the correct configuration on `/etc/wpa_supplicant/wpa_supplicant.conf`. for details regarding .conf files, see [this link](https://raspberrypi.stackexchange.com/questions/22875/connecting-to-wpa2-enterprise-wifi-network).

if the WPA2 authentication procedure is successful, the result should be similar to this:

```
wlan0: Trying to associate with xx:xx:xx:xx:xx:xx (SSID='some-ssid' freq=2462 MHz)
ioctl[SIOCSIWFREQ]: Device or resource busy
wlan0: Association request to the driver failed
wlan0: Associated with xx:xx:xx:xx:xx:xx
wlan0: CTRL-EVENT-EAP-STARTED EAP authentication started
wlan0: CTRL-EVENT-EAP-PROPOSED-METHOD vendor=0 method=25
wlan0: CTRL-EVENT-EAP-METHOD EAP vendor 0 method 25 (PEAP) selected

(...)

EAP-MSCHAPV2: Authentication succeeded
EAP-TLV: TLV Result - Success - EAP-TLV/Phase2 Completed
wlan0: CTRL-EVENT-EAP-SUCCESS EAP authentication completed successfully
wlan0: WPA: Key negotiation completed with xx:xx:xx:xx:xx:xx:     
  PMK=<PMK-AS-HEX-STRING-WILL-BE-HERE>    
  [PTK=CCMP GTK=CCMP]
wlan0: CTRL-EVENT-CONNECTED - Connection to xx:xx:xx:xx:xx:xx completed (auth) [id=0 id_str=]
```
# cross-compilation details

this custom version of wpa_supplicant was cross-compiled to run on a raspberry pi model B+, V1 2, running Raspbian GNU/Linux 7 (wheezy). i've added a special [defconfig-raspberry-pi](https://github.com/adamiaonr/wpa-supplicant-pmk/blob/master/wpa_supplicant/defconfig-raspberry-pi) file with the configurations to compile wpa_supplicant.

the cross compilation was based [on this guide](http://www.fabriziodini.eu/posts/cross_compile_tutorial/), and requires part of the target's filesystem (in my case, a raspberry pi) to be present your host machine. for me, this included:
* copying `/usr` and `/lib` from the raspberry pi into a directory on the host machine. this directory - with path `<SYSROOT_DIR>` - then becomes the so-called `sysroot`.
* copying the header files of `openssl` ([v1.0.1c](https://www.openssl.org/source/old/1.0.1/openssl-1.0.1c.tar.gz)) and `libnl-3` (Netlink Protocol Library Suite, [v3.2.5](http://www.infradead.org/~tgr/libnl/files/libnl-3.2.5.tar.gz)) into `<SYSROOT_DIR>/usr/include` (i.e. on the host machine).
* there was no need to copy `.so` files from cross-compiled versions of `openssl` or `libnl-3` into the `sysroot`, because these were already present in the copied files from the raspberry pi (`/usr` and `/lib`). **however** i had to create symlinks for `libssl.so`, `libnl-3.so`, `libnl-genl-3.so` and `libcrypto.so`, as follows:

```
cd <SYSROOT_DIR>/usr/lib/arm-linux-gnueabihf
ln -s libssl.so.1.0.0 libssl.so
ln -s libcrypto.so.1.0.0 libcrypto.so
cd <SYSROOT_DIR>/lib/arm-linux-gnueabihf
ln -s libnl-3.so.200.5.2 libnl-3.so
ln -s libnl-genl-3.so.200.5.2 libnl-genl-3.so
```
# usage on wireshark

based on [this guide](https://wiki.wireshark.org/HowToDecrypt802.11):

* say you have a `.pcap` file with the exchange of WiFi frames in-between your raspberry pi (client) and some AP in a WPA2 Enterprise WiFi network (like eduroam). this `.pcap` file **must** include the WPA2 Enterprise authentication process (namely the [4-way handshake](https://mrncciew.com/2014/08/19/cwsp-4-way-handshake/)). 
* say you used this customized version of `wpa_supplicant` to connect the client to the WiFi network, and as such have the PMK (available under the `WPA: Key negotiation completed with` message).

on wireshark v2.2.3-0 (Mac OSX), go to *Wireshark* > *Preferences...* > *Protocols* (left pane) > *IEEE 802.11*. Click on the `Edit...` button, to the side of `Decryption Keys`. By clicking on `+`, you should add a new `wpa-psk` key, with contents equal to the hex string given by `wpa_supplicant`. after this step, open the `.pcap` file, and you should be able to inspect the contents of WiFi data frames exchanged during that session.

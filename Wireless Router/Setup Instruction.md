#Setup Instruction
When setting up a wireless router here, we connect the Internet access on `eth0`, and plan to use `wlan0` as the WiFi transmitter. If you want to receive WiFi signals with another NIC, just modify the configuration file a little.

##Get Proper Driver for NIC
I have a high-power NIC card chipped with RaLink 3070. It is supportted by Raspbian itself, so the step for finding the driver can be skipped. If you get an unsupported NIC card, maybe compiling the driver by yourself is necessary. Good luck.

##Change NIC settings
We need to assign a static IP address for the wireless NIC card.

* Open `/etc/network/interfaces` with root authorization.
* Comment all about `wlan0` with `#`.
* Add the following new contents:

		iface wlan0 inet static
		address 10.0.16.1
		netmask 255.255.255.0
	The assignment of IP address the subnet mask is at your wish. For normal use, a C-type IP range(192.168.0.0/24-192.168.255.0/24) can fulfill our needs.
* Reastart the networking service to verify that IP is properly assigned.

##Install necessary packages
###hostapd
*It seems that hostapd only supports 802.11a/b/g mode, how to make it support 802.11n? This is to be discovered...*

* Run the following command to install `hostapd`:

		sudo apt-get install hostapd
* Edit the default configuration file of `hostapd`
	* Find `#DAEMON_CONF=""` and modify it to:
	
			DAEMON_CONF="/etc/hostapd/hostapd.conf"
	* Create `/etc/hostapd/hostapd.conf` and add the following contents:

			#Function wlan0 as AP
			interface=wlan0suc
			#Identify the driver
			driver=nl80211
			#Disable the SSID broadcast
			ignore_broadcast_ssid=1
			ssid=RaspberryPi Router
			#Working mode
			hw_mode=g
			channel=6
			#Encryption settings 
			wpa=2
			#Password
			wpa_passphrase=********
			wpa_key_mgmt=WPA-PSK
			wpa_pairwise=CCMP
			rsn_pairwise=CCMP
			beacon_int=100
			auth_algs=3
			wmm_enabled=1
* Run `sudo service hostapd restart` to verify the configuration file. Go ahead if there is no error message.

###DHCP server
* Run the following command to install DHCP server for Raspberry Pi:

		sudo apt-get install isc-dhcp-server
* Edit the configuration file `/etc/dhcp/dhcpd.conf`. If necessary, make a backup file before modifying it.  
All properties have clear names and functions. So, I won't explain them.

default-lease-time 43200;
max-lease-time 86400;

subnet 10.0.16.0 netmask 255.255.255.0 {
range 10.0.16.1 10.0.16.254;
option routers 10.0.16.1;
option broadcast-address 10.0.16.255;
option domain-name-servers 61.177.7.1 223.5.5.5;
default-lease-time 43200;
max-lease-time 86400;
}

##Settings of iptables
* To enable wireless clients to have access to the Internet, we still need to configure something about packet forwarding.

		sudo iptables -F				//Delete all existing rules in all chains
		sudo iptables -X				//Delete all user-defined chains
		//The core NAT rule
		sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE 
		//Save the iptables we just configured
		sudo bash 
		iptables-save > /etc/iptables.up.rules 
		exit
* We can let `iptables` load our configuration every time the Raspberry Pi powers on. In `/etc/network/if-pre-up.d/iptables`, add the following shell codes:

		#!/bin/bash
		/sbin/iptables-restore < /etc/iptables.up.rules
* Save the modification and do a little `chmod`:

		sudo chmod 755 /etc/network/if-pre-up.d/iptables

##Enable forwarding in Linux kernel
* Open file `/etc/sysctl.conf`.
* Uncomment the following text in the file above:

		net.ipv4.ip_forward=1
* Run `sudo sysctl -p`.

##Let services run automatically when powered on
This is where `chkconfig` works. Run the following commands to add `hostapd` and `isc-dhcp-server` into the start list:

	sudo chkconfig --add hostapd
	sudo chkconfig --add isc-dhcp-derver

If necessary, you may define in which levels these services start up.
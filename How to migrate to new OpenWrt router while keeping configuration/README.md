# Introduction
This guide is more written for my own sake as of 2025.02.15 but could help others out.

It explains how to migrate from one OpenWrt router to another while keeping your configurations such as your custom: Port Forwards, Static IPs, and DNS hostnames.

# Backing up configuration files
All config files are located in `/etc/config`.

The most important ones to backup are
* `/etc/config/dhcp`
* `/etc/config/firewall`
* `/etc/config/network`

## config: dhcp
Only copy the bottom half which contains `config host` and `config domain`. Ignore anything on top as they are the default.
```
config dnsmasq
	option domainneeded '1'
	option localise_queries '1'
	option rebind_protection '1'
	option rebind_localhost '1'
	option local '/lan/'
	option domain 'lan'
	option expandhosts '1'
	option authoritative '1'
	option readethers '1'
	option leasefile '/tmp/dhcp.leases'
	option resolvfile '/tmp/resolv.conf.d/resolv.conf.auto'
	option localservice '1'
	option ednspacket_max '1232'
	option confdir '/tmp/dnsmasq.d'

config dhcp 'lan'
	option interface 'lan'
	option start '100'
	option limit '150'
	option leasetime '12h'
	option dhcpv4 'server'
	option dhcpv6 'server'
	option ra 'server'
	list ra_flags 'managed-config'
	list ra_flags 'other-config'

config dhcp 'wan'
	option interface 'wan'
	option ignore '1'

config odhcpd 'odhcpd'
	option maindhcp '0'
	option leasefile '/tmp/hosts/odhcpd'
	option leasetrigger '/usr/sbin/odhcpd-update'
	option loglevel '4'

############ Copy stuff below this line!!!	############
config host
	blahblahblah
config domain
	blahblahblah
```
Paste these sections below the existing dhcp config of your new router.
* The config host section
	* contains your static IP assignments
* The config domain section
	* contains your local DNS assignments


## config: firewall transfer
Only copy the bottom half which contains all the `config redirect` and append and paste it under the same firewall file in your new router. These contain your port forwards. Such as:

```
config redirect
	option target 'DNAT'
	option src 'wan'
	option dest 'lan'
	option proto 'tcp'
	option src_dport '6112'
	option name 'StarCraftPortForward'
	option dest_ip '192.168.1.101'
	option dest_port '6112'
```
Paste these sections below the existing firewall config of your new router.

## config: network
No need to copy anything here. Just note what subnet you used before for your router and do the same on the new router. The area of interest is `config interface 'lan'` and the area of modification is `option ipaddr`. On your new router change to your preferred subnet. Mine is `192.168.2.1` in this example.

```
config interface 'lan'
	option device 'br-lan'
	option proto 'static'
	option netmask '255.255.255.0'
	option ip6assign '60'
	option ipaddr '192.168.2.1'
```
This is all it takes to migrate all the configs over. Reboot to apply configs.

# Check actual DNS service is updated
* For me that means checking my namecheap.com to see if things are pointing to the right public IP. I have DDNS so not much to change.

# Install opkgs
```
opkg update 
```
The ones I usually need are
```
luci-app-sqm
htop
nano
luci-app-nlbwmon
luci-app-wol
etherwake
```

Optionally
```
btop
luci-app-dockerman  
docker-compose
```

Reboot after install

# Setup SQM
Network > SQM QoS
* Enable SQM Instance
* Set DL Speed to 90% of top speed and test. Increase %tage until bufferbloat happens. 
	* Xfinity offers 1.2 Gbps but usually goes past 1.4 Gbps. To be safe I set mine at 1150000 kbps.
		* Yes 1.15 Gbps is not 90% but it's my best case.
* Set UL Speed to 90% of top speed and test. Increase %tage until bufferbloat happens
	* Xfinity offers 40 Mbps here. To be safe I set mine at 36000 kbps
* Anytime bufferbloat gets worst reduce %tage until it gets better. This is part of fine tuning. There's a balance between max bandwidth and minimum ping.

Queue Discipline Tab
* cake
* piece_of_cake.qos

Advanced section as mentioned in this [article](https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://openwrt.org/docs/guide-user/network/traffic-shaping/sqm-details&ved=2ahUKEwj-lo3Rr8aLAxUTGDQIHTtxDqUQFnoECBoQAQ&usg=AOvVaw2eCdj6T4m1QmGWGyk3auV5)
* Checkmark `Advanced Configuration`
* Checkmark `Dangerous Configuration`
	* Qdisc options (ingress) set to `nat dual-dsthost ingress`
	* Qdisc options (egress) set to `nat dual-srchost`

Linked Layer Adaptation Tab
* For my own Xfinity connection. Since it's DOCSIS 3.0 Cable Modem with over 760Mbps the correct value to set is 
	* `Ethernet with overhead` with
	* Per Packet Overhead (bytes) set as `42`

# Enable Packet Steering to go past 700 Mbps SQM
Finally, enable Packet Steering on all CPUS to go past 700 Mbps SQM without the CPU Performance tweak
* This is done under Network > Interfaces > Global network options tab

# CPU Affinity Reference
The official OpenWrt image for the R6S already has manual cpu affinity adjustments done for this Rockchip SoC.

This can be viewed with
```
nano /etc/hotplug.d/net/40-net-smp-affinity
```
OpenWrt's default config for the R6S is as follows
```
friendlyarm,nanopi-r6s)
        set_interface_core 2 "eth0"
        set_interface_core 4 "eth1"
        set_interface_core 8 "eth2"
        ;;
```
I modified mine to. It doesn't have xps_cpus on the tx for eth1 and eth2 like on FriendlyWrt because it doesn't seem to exist.
```
friendlyarm,nanopi-r6s)
        set_interface_core 1 "eth0"
        echo c0 > /sys/class/net/eth0/queues/rx-0/rps_cpus
        echo 30 > /sys/class/net/eth0/queues/tx-0/xps_cpus
        set_interface_core 2 "eth1"
        echo c0 > /sys/class/net/eth1/queues/rx-0/rps_cpus
        set_interface_core 4 "eth2"
        echo c0 > /sys/class/net/eth2/queues/rx-0/rps_cpus
        ;;
```
However it might be experimenting changing that to `20`, `40` and `80` so that the performance cores are assigned instead of the slower `2`, `4`, and `8` cores if it seems packet steering isn't working that well.
```
## Performance Tweaks Quick Reference R6S
Binary   = hex ## = cpu core
00000001 = hex 1 = cpu core 0 (A55) selected
00000010 = hex 2 = cpu core 1 (A55) selected
00000100 = hex 4 = cpu core 2 (A55) selected
00001000 = hex 8 = cpu core 3 (A55) selected
00010000 = hex 10 = cpu core 4 (A76) selected
00100000 = hex 20 = cpu core 5 (A76) selected
01000000 = hex 40 = cpu core 6 (A76) selected
10000000 = hex 80 = cpu core 7 (A76) selected
```
Confirm by running htop to watch CPU usage while bandwidth is under full load with speedtest.

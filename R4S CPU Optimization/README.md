# Performance Tweak for R4S

This guide is based on my [main guide for the R6S](https://github.com/StarWhiz/NanoPi-R6S-CPU-Optimization-for-Gigabit-SQM) except this guide is for the R4S instead. This applies to FriendlyWrt firmware only. Official OpenWrt 24.10.0 released on Feb 03, 2025 is already optimized and only needs packet steering (Enabled All CPUs) checked under Network > Interfaces > Global network options tab.

Please read that for full understanding. Otherwise let's just get on to it! This won't be as detailed as my main guide, but you sure can speed run this!

Before this modification I was only able to push 630 Mbps cake SQM. After the mod I was able to push 780 Mbps on cake SQM.

This assumes you installed nano already
```
opkg update
opkg install nano
```

## Script for checking current affinity
Run these commands
```
touch checkaffinity.sh
chmod +x checkaffinity.sh
nano checkaffinity.sh
```
Paste in this script. CTRL+O to save.
```
#!/bin/sh

# Save the output of /proc/interrupts in a variable
interrupts=$(cat /proc/interrupts)

# Extract the numbers associated with eth1-0 and eth2-0 using grep and awk

eth1_0=$(echo "$interrupts" | grep "eth0" | awk '{print $1}' | tr -d ':')
eth1_1=$(echo "$interrupts" | grep "eth1" | awk '{print $1}' | tr -d ':')

# Display current CPU cores assigned to current IRQs and queues


echo "CPU Affinity for ETH0 1gbps WAN is $(cat /proc/irq/$eth1_0/smp_affinity)"
echo "CPU Affinity for ETH2 1gbps LAN is $(cat /proc/irq/"$eth1_1"/smp_affinity)"
echo "CPU cores assigned to ETH0 queue rx-0 is: $(cat /sys/class/net/eth0/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH1 queue rx-0 is: $(cat /sys/class/net/eth1/queues/rx-0/rps_cpus)"
```
Run the script to check current affinity
```
./checkaffinity.sh
```

## Performance Mod
```
nano /etc/hotplug.d/net/40-net-smp-affinity
```

Edit the section starting with
```
friendlyarm,nanopi-r4s|\
friendlyelec,nanopi-r4s)
        blar
        blar
        blar
        ;;
```

And replace it with the folowing below. 
```
friendlyarm,nanopi-r4s|\
friendlyelec,nanopi-r4s)
        set_interface_core 4 "eth0"
        set_interface_core 8 "eth1"
        echo -n 10 > /sys/class/net/eth0/queues/rx-0/rps_cpus
        echo -n 20 > /sys/class/net/eth1/queues/rx-0/rps_cpus
        ;;
```
Unlike the R6S which has options. I believe this is the most optimized settting. There's only two ports. We assign the two faster cores on the eth0 and eth1 queues and 2 slower cores on the IRQs.

Reboot.

Then run `./checkaffinity.sh`
The output should look like below now.

```
CPU Affinity for ETH0 1gbps WAN is 04
CPU Affinity for ETH2 1gbps LAN is 08
CPU cores assigned to ETH0 queue rx-0 is: 10
CPU cores assigned to ETH1 queue rx-0 is: 20
```
Congratulations. Enjoy your 780-800 Mbps cake SQM!

Credits to [choppyc](https://forum.openwrt.org/t/nanopi-r6s-with-openwrt/167611/94?u=starwhiz) on the openWrt forums. I wouldn't have found out the R4S could be further improved otherwise.

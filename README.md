R4S Owners Read [here](https://github.com/StarWhiz/NanoPi-R6S-CPU-Optimization-for-Gigabit-SQM/tree/main/R4S%20CPU%20Optimization) instead.
To follow this tutorial you just need to understand how to SSH into your router. If you're on Windows this is very easy. Just download and install Putty. Then type in the router's IP in "Hostname" and click "Open". Login with the same password as the web GUI.

# Introduction
The NanoPi R6S's default setting for some reason overutilizes the 4 slower A55 cores for both queues and IRQs.
This causes cake SQM to not be able to push past 800 Mbps.

You can figure this out by running the [Waveform Bufferbloat Test] (https://www.waveform.com/tools/bufferbloat)
And monitoring your CPU Cores usage with `htop` you'll see some cores hitting close to 100%.

If you don't have htop you can install it with
```
opkg update
opkg install htop
htop
```

This tutorial helps you fix that by separating all individual 4 faster A76 cores into each queue and slower A55 cores into each IRQ on your interfances

For reference the faster A76 Cores are CPU# 5, 6, 7, and 8. While the slower A55 Cores are CPU # 0, 1, 2, and 3

You can check that with
```
cat /sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_max_freq
```
You'll see that CPU0-3 on top are the slower CPU cores.
```
1800000
1800000
1800000
1800000
2304000
2304000
2304000
2304000
```

## Installing Nano text editor
To begin, SSH into your NanoPi R6S. 
If you don't know how to use vi please install the nano text editor with
```
opkg update
opkg  install nano
```

From here you can skip to this [section](https://github.com/StarWhiz/NanoPi-R6S-CPU-Optimization-for-Gigabit-SQM/blob/main/README.md#the-actual-fix-to-optimize-nanopi-r6s-to-go-beyond-1400-mbps-w-cake-sqm) if you're not interested in checking if the solution worked or not and want to speedrun this. Otherwise, continue reading.

## How to check the current IRQ CPU affinitys
You don't need to do this step but it helps you confirm if your CPU affinity is indeed fixed.

First create an executable script with the commands below
```
touch checkaffinity.sh
chmod +x checkaffinity.sh
nano checkaffinity.sh
```

A new text editor will open up. Paste in this checkaffinity.sh script below.
```
#!/bin/bash

# Save the output of /proc/interrupts in a variable
interrupts=$(cat /proc/interrupts)

# Extract the numbers associated with eth1-0 and eth2-0 using grep and awk
irqs=$(grep "eth0" /proc/interrupts | awk '{print $1}' | tr -d ':')
eth1_0=$(echo "$interrupts" | grep "eth1-0" | awk '{print $1}' | tr -d ':')
eth2_0=$(echo "$interrupts" | grep "eth2-0" | awk '{print $1}' | tr -d ':')

# Display current CPU cores assigned to current IRQs and queues
for irq in $irqs; do
    echo "CPU Affinity for ETH0 1gbs LAN $irq is $(cat /proc/irq/$irq/smp_affinity)"
done

echo "CPU Affinity for ETH1 2.5gbs LAN was $(cat /proc/irq/"$eth1_0"/smp_affinity)"
echo "CPU Affinity for ETH2 2.5gbs WAN was $(cat /proc/irq/"$eth2_0"/smp_affinity)"
echo "CPU cores assigned to ETH0 queue rx-0 is: $(cat /sys/class/net/eth0/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH1 queue rx-0 is: $(cat /sys/class/net/eth1/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH2 queue rx-0 is: $(cat /sys/class/net/eth2/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH0 queue tx-0 is: $(cat /sys/class/net/eth0/queues/tx-0/xps_cpus)"
echo "CPU cores assigned to ETH1 queue tx-0 is: $(cat /sys/class/net/eth1/queues/tx-0/xps_cpus)"
echo "CPU cores assigned to ETH2 queue tx-0 is: $(cat /sys/class/net/eth2/queues/tx-0/xps_cpus)"
```
Press CTRL+O to save and exit nano.

You can now run the script with
```
./checkaffinity.sh
```

**The default output from ./checkaffinity.sh**
```
CPU Affinity for ETH0 1gbs LAN 63 is 02
CPU Affinity for ETH0 1gbs LAN 64 is ff
CPU Affinity for ETH1 2.5gbs LAN was 04
CPU Affinity for ETH2 2.5gbs WAN was 08
CPU cores assigned to ETH0 queue rx-0 is: fe
CPU cores assigned to ETH1 queue rx-0 is: fe
CPU cores assigned to ETH2 queue rx-0 is: fe
CPU cores assigned to ETH0 queue tx-0 is: 00
CPU cores assigned to ETH1 queue tx-0 is: 00
CPU cores assigned to ETH2 queue tx-0 is: 00
```

The output tells you your current CPU Affiniity for IRQs on all interfaces!

Now you're ready to see the change!

## The actual fix to optimize NanoPi R6S to go beyond 1400 Mbps w/ cake SQM
Okay let's begin!

Edit the file in /etc/hotplug.d/net/40-net-smp-affinity with

```
nano /etc/hotplug.d/net/40-net-smp-affinity
```
From there scroll down till you see the section that begins with

```
friendlyelec,nanopi-r6s)
        set_interface_core 2 "eth0"
        echo fe > /sys/class/net/eth0/queues/rx-0/rps_cpus
        set_interface_core 4 "eth1-0"
        set_interface_core 4 "eth1-16"
        set_interface_core 4 "eth1-18"
        echo fe > /sys/class/net/eth1/queues/rx-0/rps_cpus
        set_interface_core 8 "eth2-0"
        set_interface_core 8 "eth2-16"
        set_interface_core 8 "eth2-18"
        echo fe > /sys/class/net/eth2/queues/rx-0/rps_cpus
        ;;
```

You have a few options to replace the above with.

**Option #1** This is a standard option if you're using all the eth ports: eth0, eth1, eth2. When in doubt pick this one.
```
friendlyelec,nanopi-r6s)
	set_interface_core 1 "eth0"
	echo c0 > /sys/class/net/eth0/queues/rx-0/rps_cpus
	echo 30 > /sys/class/net/eth0/queues/tx-0/xps_cpus
	set_interface_core 2 "eth1-0"
	set_interface_core 2 "eth1-16"
	set_interface_core 2 "eth1-18"
	echo c0 > /sys/class/net/eth1/queues/rx-0/rps_cpus
	echo 30 > /sys/class/net/eth1/queues/tx-0/xps_cpus
	set_interface_core 4 "eth2-0"
	set_interface_core 4 "eth2-16"
	set_interface_core 4 "eth2-18"
	echo c0 > /sys/class/net/eth2/queues/rx-0/rps_cpus
	echo 30 > /sys/class/net/eth2/queues/tx-0/xps_cpus
	;;
```

**Option #2a** This is a non-standard option for if you're only using the 2.5gbps ports: eth1 (LAN) and eth2 (WAN)... and not the 1gbps port: eth0

I believe it is more optimized if you only want to use eth1 and eth2.
```
friendlyelec,nanopi-r6s)
	set_interface_core 1 "eth0"
	echo 2 > /sys/class/net/eth0/queues/rx-0/rps_cpus
	echo 2 > /sys/class/net/eth0/queues/tx-0/xps_cpus
	set_interface_core 4 "eth1-0"
	set_interface_core 4 "eth1-16"
	set_interface_core 4 "eth1-18"
	echo 10 > /sys/class/net/eth1/queues/rx-0/rps_cpus
	echo 20 > /sys/class/net/eth1/queues/tx-0/xps_cpus
	set_interface_core 8 "eth2-0"
	set_interface_core 8 "eth2-16"
	set_interface_core 8 "eth2-18"
	echo 40 > /sys/class/net/eth2/queues/rx-0/rps_cpus
	echo 80 > /sys/class/net/eth2/queues/tx-0/xps_cpus
	;;
```

**Option #2b**: This is a variation of 2a for asymmetrical WAN. (Example: 1200Mbps Down / 40 Mbps from your ISP)
It assumes you're only using the 2.5gbps ports: eth1 (LAN) and eth2 (WAN)
```
friendlyelec,nanopi-r6s)
	set_interface_core 1 "eth0"
	echo 2 > /sys/class/net/eth0/queues/rx-0/rps_cpus
	echo 2 > /sys/class/net/eth0/queues/tx-0/xps_cpus
	set_interface_core 4 "eth1-0"
	set_interface_core 4 "eth1-16"
	set_interface_core 4 "eth1-18"
	echo 10 > /sys/class/net/eth1/queues/rx-0/rps_cpus
	echo 20 > /sys/class/net/eth1/queues/tx-0/xps_cpus
	set_interface_core 8 "eth2-0"
	set_interface_core 8 "eth2-16"
	set_interface_core 8 "eth2-18"
	echo c0 > /sys/class/net/eth2/queues/rx-0/rps_cpus
	echo 80 > /sys/class/net/eth2/queues/tx-0/xps_cpus
	;;
```

After you are done editing. Press CTRL+O to save and exit nano.

Reboot your NanoPI R6s. That's all you need to do!

If you followed the previous section and want to check if the CPU affinities did change.

SSH back into your NanoPi R6s and run

```
./checkaffinity.sh
```

It should be different from before and similar to the values you had just set in eithe option 1, 2a or 2b!

**output of option 1**
```
CPU Affinity for ETH0 1gbs LAN 63 is 01
CPU Affinity for ETH0 1gbs LAN 64 is ff
CPU Affinity for ETH1 2.5gbs LAN was 02
CPU Affinity for ETH2 2.5gbs WAN was 04
CPU cores assigned to ETH0 queue rx-0 is: c0
CPU cores assigned to ETH1 queue rx-0 is: c0
CPU cores assigned to ETH2 queue rx-0 is: c0
CPU cores assigned to ETH0 queue tx-0 is: 30
CPU cores assigned to ETH1 queue tx-0 is: 30
CPU cores assigned to ETH2 queue tx-0 is: 30
```

That's it! Congratulations. Go ahead and test out cake SQM at higher bandwidths!

# Optimizing NanoPi R4S
If you want to optimize a NanoPi R4S and not a R6S please read [here](https://github.com/StarWhiz/NanoPi-R6S-CPU-Optimization-for-Gigabit-SQM/tree/main/R4S%20CPU%20Optimization).
I was able to push from 630 Mbps cake SQM to almost 800 Mbps cake SQM with similar mods.

> [!Important]  
> As of 2024.01.08... I had an [older version of the tutorial](https://github.com/StarWhiz/NanoPi-R6S-CPU-Optimization-for-Gigabit-SQM/blob/main/OldREADME.md) where the cpu affinity kept geting reverted back to the old values.
> The new update you're reading above fixes all the reverting issues.


# Further Explanations
If you are interested in more information please check out my wiki at [the deprecated section here](https://wiki.stoplagging.com/books/technical-guides/page/nanopi-r6s-r4s-for-gigabit-sqm-with-openwrt#bkmrk-about-performance-tw)

# Credits
Credits to the following people who helped made the openwrt document linked here: https://openwrt.org/docs/guide-user/advanced/load_balancing_-_tuning_smp_irq

* mercygroundabyss
* moeller0
* walmartshopper
* xShARkx

Credits to [choppyc](https://forum.openwrt.org/t/nanopi-r6s-with-openwrt/167611/87?u=starwhiz) for the alternative smp-affinities

This allowed me to come up with a better solution for the R6S. Thank you!

# Introduction
For the NanoPi R6S's firmware, the default setting assigned 4x slower A55 cores for IRQs on ETH2. 
2x slower A55 cores for IRQs on ETH1. And 1x slower A55 slow core for IRQs on Eth1.
This causes cake SQM to not be able to push past 800 Mbps.

This tutorial helps you fix that and by assigning the faster A76 cores for IRQs in ETH0 (1gbps LAN) ETH1 (2.5gbps LAN) and ETH2 (2.5gbps WAN).
AFter doing so you should be able to easily push cake SQM beyond 1400 Mbps.

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
echo "CPU cores assigned to ETH0 queue rx-0 was: $(cat /sys/class/net/eth0/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH1 queue rx-0 was: $(cat /sys/class/net/eth1/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH2 queue rx-0 was: $(cat /sys/class/net/eth2/queues/rx-0/rps_cpus)"
```
Press CTRL+O to save and exit nano.

You can now run the script with
```
./checkaffinity.sh
```

The output tells you your current CPU Affiniity for IRQs on all interfaces!

Now you're ready to see the change!

## The actual fix to optimize NanoPi R6S to go beyond 1400 Mbps w/ cake SQM
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

You want to modify the numbers 2 4 and 8 above to, f0, 30, and c0 as shown below.
Then do ff for all the queues. Alternatively just copy paste the code below to replace
the original code above.

```
friendlyelec,nanopi-r6s)
        set_interface_core f0 "eth0"
        echo ff > /sys/class/net/eth0/queues/rx-0/rps_cpus
        set_interface_core 30 "eth1-0"
        set_interface_core 30 "eth1-16"
        set_interface_core 30 "eth1-18"
        echo ff > /sys/class/net/eth1/queues/rx-0/rps_cpus
        set_interface_core c0 "eth2-0"
        set_interface_core c0 "eth2-16"
        set_interface_core c0 "eth2-18"
        echo ff > /sys/class/net/eth2/queues/rx-0/rps_cpus
        ;;
```

After you are done editing. Press CTRL+O to save and exit nano.

Reboot your NanoPI R6s. That's all you need to do!

If you followed the previous section and want to check if the CPU affinities did change.

SSH back into your NanoPi R6s and run

```
./checkaffinity.sh
```
It should be different now!

> [!Important]  
> As of 2024.01.08... I had an [older version of the tutorial](https://github.com/StarWhiz/NanoPi-R6S-CPU-Optimization-for-Gigabit-SQM/blob/main/OldREADME.md) where the cpu affinity kept geting reverted back to the old values.
> The new update you're reading above fixes all the reverting issues.


# Further Explanations
If you are interested in more information please check out my wiki at https://wiki.stoplagging.com/books/technical-guides/page/sqm-with-nanopi-for-1-gbps-lines-with-openwrt#bkmrk-about-performance-tw

# Credits
Credits to the following people who helped made the openwrt document linked here: https://openwrt.org/docs/guide-user/advanced/load_balancing_-_tuning_smp_irq

* mercygroundabyss
* moeller0
* walmartshopper
* xShARkx

This allowed me to come up with a better solution for the R6S. Thank you!



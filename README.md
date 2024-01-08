# Introduction
For the NanoPi R6S's firmware, the default setting to use all 4 slower A55 cores for irq on ETH2. While only using 2 of the slower A55 cores for irq on ETH1.
This causes cake SQM to not be able to push past 800 Mbps.

This script fixes that and assigns the faster A76 cores to ETH1 (2.5gbps LAN) and ETH2 (2.5gbps WAN)
Now you can push cake SQM beyond 1400 Mbps!


To use this script... SSH into your NanoPi R6S. Install nano with

```
opkg update
opkg  install nano
```

Then do the follow commands below

```
touch performancetweak.sh
chmod +x performancetweak.sh
nano performancetweak.sh
```

Then you'll be in the text editor.

Copy paste in the following script below.

```
#!/bin/bash

# Save the output of /proc/interrupts in a variable
interrupts=$(cat /proc/interrupts)

# Extract the numbers associated with eth1-0 and eth2-0 using grep and awk
eth1_0=$(echo "$interrupts" | grep "eth1-0" | awk '{print $1}' | tr -d ':')
eth2_0=$(echo "$interrupts" | grep "eth2-0" | awk '{print $1}' | tr -d ':')

# Display current CPU cores assigned to current IRQs and queues
echo "CPU Affinity for ETH1 2.5gbs LAN was $(cat /proc/irq/"$eth1_0"/smp_affinity)"
echo "CPU Affinity for ETH2 2.5gbs WAN was $(cat /proc/irq/"$eth2_0"/smp_affinity)"
echo "CPU cores assigned to ETH0 queue rx-0 was: $(cat /sys/class/net/eth0/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH1 queue rx-0 was: $(cat /sys/class/net/eth1/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH2 queue rx-0 was: $(cat /sys/class/net/eth2/queues/rx-0/rps_cpus)"

# Set the CPU affinity for IRQs using variables
echo -n ff > /sys/class/net/eth2/queues/rx-0/rps_cpus
echo -n ff > /sys/class/net/eth1/queues/rx-0/rps_cpus
echo -n ff > /sys/class/net/eth0/queues/rx-0/rps_cpus
echo -n 30 > /proc/irq/"$eth2_0"/smp_affinity
echo -n c0 > /proc/irq/"$eth1_0"/smp_affinity

# Display new CPU cores assigned to new IRQs and queues
echo "CPU Affinity for ETH1 2.5gbs LAN is now $(cat /proc/irq/"$eth1_0"/smp_affinity)"
echo "CPU Affinity for ETH2 2.5gbs WAN is now $(cat /proc/irq/"$eth2_0"/smp_affinity)"
echo "CPU cores assigned to ETH0 queue rx-0 is now: $(cat /sys/class/net/eth0/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH1 queue rx-0 is now: $(cat /sys/class/net/eth1/queues/rx-0/rps_cpus)"
echo "CPU cores assigned to ETH2 queue rx-0 is now: $(cat /sys/class/net/eth2/queues/rx-0/rps_cpus)"
```

Once pasted. Press Ctrl+O to save and exit out of nano

To run the script do the following command.
```
./performancetweak.sh
```

That's all!

Note: If you restart Smart Queue Management or change SQM settings, it will reset the CPU affinity changes
and you will need to run the script again with `./performancetweak.sh`


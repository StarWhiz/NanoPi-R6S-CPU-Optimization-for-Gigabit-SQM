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

Lastly you might want to have the script run on reboot. 

You can do this by going to System > Startup > Local Startup

and put in the command to run the script above exit 0 pictured below.

```
/root/./performancetweak.sh
```

![Start script on boot](/AddingScriptToStartOnReboot.png?raw=true "Start script on boot")

> [!IMPORTANT]  
> Even though we have the local start up script.... If you restart Smart Queue Management or change SQM settings,
> it will reset the CPU affinity and you will need to run the script again with ./performancetweak.sh to solve this
> continue reading to modify the init.d script of SQM

To solve the problem above we will need to modify sqm's init.d so that it starts the /root/performancetweak.sh script
each time a change is made in sqm. To begin...

```
nano /etc/init.d/sqm
```

The original defaults were

```
#!/bin/sh /etc/rc.common

START=50
USE_PROCD=1

service_triggers()
{
        procd_add_reload_trigger "sqm"
}

reload_service()
{
        stop "$@"
        start "$@"
}

start_service()
{
        /usr/lib/sqm/run.sh start "$@"
}

stop_service()
{
        /usr/lib/sqm/run.sh stop "$@"
}

boot()
{
        export SQM_VERBOSITY_MIN=5 # Silence errors
        start "$@"
}
```

We will add 

```
/root/performancetweak.sh
```

To all the functions like so below

```
#!/bin/sh /etc/rc.common

START=50
USE_PROCD=1

service_triggers()
{
        procd_add_reload_trigger "sqm"
}

reload_service()
{
        stop "$@"
        start "$@"
        /root/performancetweak.sh
}

start_service()
{
        /usr/lib/sqm/run.sh start "$@"
        /root/performancetweak.sh
}

stop_service()
{
        /usr/lib/sqm/run.sh stop "$@"
        /root/performancetweak.sh
}

boot()
{
        export SQM_VERBOSITY_MIN=5 # Silence errors
        start "$@"
        /root/performancetweak.sh
}
```

Finished! Now everytime you reboot or change SQM settings the performance tweak is retained!

# Further Explanations
If you are interested in more information please check out my wiki at https://wiki.stoplagging.com/books/technical-guides/page/sqm-with-nanopi-for-1-gbps-lines-with-openwrt#bkmrk-about-performance-tw

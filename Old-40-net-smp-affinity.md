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
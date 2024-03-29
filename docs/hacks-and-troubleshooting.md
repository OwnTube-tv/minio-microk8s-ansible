
Hacks and Troubleshooting
=========================

This page will be extended with funny quirks that needs to be addressed reproducibly, as they are discovered.


Realtek `r8125` Network Drivers for ASUS PN52/PN53 Hardware
-----------------------------------------------------------

The servers with hostnames `v1517a.mabl.online` and `v1517b.mabl.online` have 2.5 GbE NICs that Ubuntu do not
include good drivers for, after a number of days of uptime they stop working and the server need to be rebooted.

Since the network drivers that comes with Ubuntu does not fly well with Realtek r8125, the
[2.5G Ethernet LINUX driver](https://www.realtek.com/ja/component/zoo/category/network-interface-controllers-10-100-1000m-gigabit-ethernet-pci-express-software)
need to be installed, and then re-installed on each restart or else the NIC will freeze and drop all packages.

As the root user, extract the `r8125-9.012.04.tar.bz2` driver in `/root/r8125-9.012.04/`:

    sudo su -
    cd /root/
    cp -v /path/to/download/r8125-9.012.04.tar.bz2 .
    bunzip2 r8125-9.012.04.tar.bz2
    tar xvf r8125-9.012.04.tar

Create a new service `/etc/systemd/system/ar9708-r8125-hack.service`:

```systemd
[Unit]
Description=ar9708-install-r8125-driver
After=network.target

[Service]
Type=oneshot
WorkingDirectory=/root/r8125-9.012.04/
ExecStart=/root/r8125-9.012.04/autorun.sh
TimeoutStopSec=20
KillMode=process
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

Reload and enable the `ar9708-r8125-hack` service, run a manual test to verify:

    systemctl daemon-reload
    systemctl enable ar9708-r8125-hack.service
    systemctl start ar9708-r8125-hack

When the driver is installed as intended, the driver should be listed by `lsmod`:

    lsmod | grep r8125
    'r8125                 233472  0'

If the `r8125` driver is not listed, then we have problems.

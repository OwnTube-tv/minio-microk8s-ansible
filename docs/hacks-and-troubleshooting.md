
Hacks and Troubleshooting
=========================

This page will be extended with funny quirks that needs to be addressed reproducibly, as they are discovered.


Realtek `r8125` Network Drivers for ASUS PN52/PN53 Hardware
-----------------------------------------------------------

The servers with hostnames `a12c.mabl.online` and `a12d.mabl.online` have 2.5 GbE NICs
that Ubuntu do not include good drivers for, after a number of days of uptime they stop working and the server
need to be rebooted.

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


WiFi Disabled on All Cluster Nodes
----------------------------------

All four nodes (`a12a`–`a12d`) have built-in WiFi that was originally kept as an
out-of-band fallback onto the `A12-local` SSID (`192.168.0.0/24`). Interface names
differ per model: `wlo1` on a12a/a12b, `wlp3s0` on a12c, `wlp4s0` on a12d.

**As of 2026-08-14 the WiFi is disabled on every node.** Two reasons:

1. Kubernetes picks node identity independently of the routing table. Even with the
   wired NIC on a lower DHCP route metric (100 vs 600), `kubelet` autodetection chose
   the WiFi address on a12d and registered `192.168.0.19` as the node `InternalIP`,
   sending pod traffic over the fallback network. Calico's default `first-found` IP
   autodetection can do the same.
2. Since all four nodes were consolidated onto site s3 (`192.168.3.207`–`.210`), the
   wired path is the only correct one — the WiFi reaches a different site's client
   network entirely.

Disabling is done declaratively, by moving the netplan WiFi configuration out of the
way. The file — including the SSID credentials — is preserved, so nothing is lost:

    sudo mv /etc/netplan/00-installer-config-wifi.yaml /etc/netplan/00-installer-config-wifi.yaml.disabled
    sudo netplan apply

Netplan only reads `*.yaml`, so after this the radio is left unconfigured and the
interface stays `DOWN` across reboots. The wired configuration lives in a separate
file (`00-installer-config.yaml`) and is untouched.

### Re-enabling WiFi

Reverse the rename and re-apply. Run `netplan apply` detached, because applying the
configuration can briefly interrupt the SSH session you are connected over:

    sudo mv /etc/netplan/00-installer-config-wifi.yaml.disabled /etc/netplan/00-installer-config-wifi.yaml
    sudo systemd-run --no-block --unit=netplan-wifi netplan apply

Verify with `ip -br a` — the wireless interface should go `UP` with a `192.168.0.x`
address, while the wired interface keeps its `192.168.3.x` address and the default
route (metric 100 beats metric 600).

**Before re-enabling, make sure the node identity is pinned**, or the problem above
comes straight back:

* `--node-ip=<wired IP>` in `/var/snap/microk8s/current/args/kubelet`
  (note: `microk8s join` wipes this file, so set it *after* joining)
* `--advertise-address=<wired IP>` in `/var/snap/microk8s/current/args/kube-apiserver`
* optionally pin Calico with `IP_AUTODETECTION_METHOD: "cidr=192.168.3.0/24"`

Restart `snap.microk8s.daemon-kubelite` after changing either argument file.

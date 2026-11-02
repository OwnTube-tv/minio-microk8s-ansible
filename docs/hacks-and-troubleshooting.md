
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

`autorun.sh` unloads the in-kernel `r8169` driver *before* it builds `r8125` for the running
kernel, and exits without loading anything if the build fails. A node whose build breaks after a
kernel update therefore boots without network. On the console, `modprobe r8169` brings the link
back on the in-kernel driver, which does support the chip (`10ec:8125`); booting the previous
kernel from GRUB's "Advanced options for Ubuntu" works too. Build output goes to
`/root/r8125-9.012.04/log.txt` (stdout only; compiler warnings land in the unit's journal). The
build has succeeded on 6.8.0-107, -124, -137 and -147.


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


Power Saving Disabled for 24/7 Operation
----------------------------------------

The nodes are consumer mini PCs (ASUS NUC 13 Pro on a12a/a12b, ASUS PN52 on a12c, ASUS PN53 on
a12d), and both their firmware and the Ubuntu kernel default to laptop-style power saving. On
servers that run around the clock that saves a watt or two and adds failure modes, so it is
turned off where Ansible can reach it and should be turned off in the BIOS where it cannot.

### What prompted it

On 2026-11-02, during its first boot on kernel 6.8.0-147, a12c's Samsung 870 QVO SATA SSD logged
29 interface CRC errors on queued writes within three seconds. The kernel hard-reset the link
four times and then limited it from 6.0 to 3.0 Gbps:

    ata2.00: exception Emask 0x10 SAct 0x19ff8018 SErr 0xd0000 action 0x6 frozen
    ata2: SError: { PHYRdyChg CommWake 10B8B }
    ata2.00: error: { ICRC ABRT }
    ata2: limiting SATA link speed to 3.0 Gbps

No write failed, XFS reported nothing and SMART stayed clean (`CRC_Error_Count` 0). It still
matters: each node keeps all eight of its MinIO drives as LVM volumes on that one SSD, so a link
that drops for good takes the node's whole share of the cluster with it.

a12c is the only node whose SATA controller (AMD FCH, `1022:7901`) the kernel treats as a mobile
chipset. Ubuntu builds with `CONFIG_SATA_MOBILE_LPM_POLICY=3`, so a12c's ports came up as
`med_power_with_dipm` with device-initiated power management enabled on the drive; the other
three nodes report `keep_firmware_settings`. `CommWake` in the SError is the signal that wakes a
link from a low-power state, which fits link power management, but a poor cable or connector
produces similar errors, so LPM is a suspect rather than a proven cause. Upstream libata already
disables NCQ and LPM for Samsung 860/870 SSDs behind ATI SATA controllers
(`ATA_QUIRK_NO_LPM_ON_ATI`, kernel 6.14), but that quirk matches PCI vendor `0x1002` only and
does not cover a12c's newer AMD controller.

### Enforced by Ansible (role `common`)

**SATA link power management off.** `/etc/udev/rules.d/60-sata-link-power.rules` sets
`max_performance` on every SATA host when it appears. At boot that happens during udev coldplug,
about five seconds in on a12c, before the errors above started. The playbook also triggers the
rule on the running system, so no reboot is needed. A port without a device (`host0` on a12c)
rejects the write with "Operation not supported"; udev logs it and carries on. Verify with:

    grep . /sys/class/scsi_host/host*/link_power_management_policy
    sudo hdparm -I /dev/sda | grep 'Device-initiated'   # no leading '*' means DIPM is off
    grep . /sys/class/ata_link/link*/sata_spd

A link speed the kernel has limited stays limited until the next reboot. 3.0 Gbps is still
well above what the 1 GbE network can carry.

**Energy Efficient Ethernet off on the Realtek NICs.** `/etc/modprobe.d/r8125.conf` sets
`options r8125 eee_enable=0`. The Realtek driver is built with `ENABLE_EEE = y` and advertises
EEE; the switch does not negotiate it today (`ethtool --show-eee` reports `enabled - inactive`),
and the option keeps it off if a switch profile ever starts offering it. The Intel `igc` NICs on
a12a/a12b already report EEE disabled. The option applies whenever `r8125` loads, which
`ar9708-r8125-hack` does on every boot, so the running link is left alone until the next reboot.
After that, `sudo ethtool --show-eee enp2s0` on a12c (`enp3s0` on a12d) should report
`EEE status: disabled`.

**No suspend.** `sleep.target`, `suspend.target`, `hibernate.target` and `hybrid-sleep.target`
are masked. Nothing suspends the nodes today (no desktop, logind defaults, no power daemon such
as `tlp` or `power-profiles-daemon` installed), but logind's default `HandleSuspendKey=suspend`
would act on a keyboard's sleep key. `systemctl is-enabled suspend.target` reports `masked`.

### Left at the kernel defaults

**PCIe ASPM and NVMe APST** are laptop-style too: the ASPM policy is `default`, which leaves L1
enabled on the NVMe and NIC links of several nodes, and NVMe APST allows power states with up to
100 ms exit latency (`nvme_core.default_ps_max_latency_us=100000`). They concern the OS disks
and the NICs, not MinIO's SATA SSD, and as of 2026-11-02 nothing in the logs points at them.
Turning them off takes kernel parameters and a reboot of every node, so it is held back until
there is a reason: NVMe "controller is down; will reset" messages or NIC link drops. The
parameters are `pcie_aspm=off` and `nvme_core.default_ps_max_latency_us=0` in
`GRUB_CMDLINE_LINUX_DEFAULT`.

**CPU frequency scaling** needs nothing. `intel_pstate` and `amd-pstate-epp` run in active mode,
where the `powersave` governor is the drivers' dynamic governor rather than a fixed low clock;
the energy/performance preference is `balance_performance` on the Intel nodes and `performance`
on the AMD nodes.

### BIOS settings to check on site

Ansible can neither see nor change firmware settings. Check these on each node at the next site
visit and record the result in the table below. The NUC and ASUS BIOS name them differently; the
usual names are given.

- **Power on after power loss**, often "After Power Failure" or "Restore AC Power Loss": set to
  *Power On*. The SATA SSDs count 37 to 50 unclean power-offs each (SMART attribute 235,
  `POR_Recovery_Count`), so the nodes do lose power, and they must come back without someone
  pressing the button.
- **ErP, EuP or Deep S5/Deep Sleep**: off. These cut standby power so far that Wake-on-LAN and
  power-loss recovery can stop working.
- **SATA aggressive link power management (ALPM)** and **PCIe ASPM**: off, where the BIOS has
  them.
- **Fan or thermal profile**: *Standard* or *Performance* rather than *Quiet*. a12d's NVMe
  controllers already run at around 70 °C (see [hardware.md](hardware.md#thermal-sensors)).

| Node | Power on after power loss | ErP / deep sleep | ALPM / ASPM | Fan profile |
|------|---------------------------|------------------|-------------|-------------|
| a12a | not checked               | not checked      | not checked | not checked |
| a12b | not checked               | not checked      | not checked | not checked |
| a12c | not checked               | not checked      | not checked | not checked |
| a12d | not checked               | not checked      | not checked | not checked |

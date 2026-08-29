
Hardware Details for MinIO Servers
==================================

The MinIO servers have a similar baseline with 16-thread CPUs (Intel i5 or AMD Ryzen 7/9), 64 GB memory, and 16 TB of
SSD storage each. The storage is configured as two LVM Volume Groups, a 8 TB "`ubuntu-vg`" with M.2 storage and
a 8 TB "`minio-vg`" with SATA storage. See additional details below.


Site Overview
-------------

All 4 servers are co-located in a server rack across two sub-sites sharing the same
Bredband2 fiber connection and UPS power chain.

- **Site a12a** (192.168.1.0/24, VLAN 2): minio1 (a12a), minio2 (a12b) — ASUS NUC 13 Pro Tall
- **Site a12b** (192.168.3.0/24): minio3 (a12c), minio4 (a12d) — ASUS PN52/PN53
- Inter-site connectivity: IPsec VPN (IKEv2) between ER7206 routers (.206 ↔ .207)


Power Supply
------------

All 4 servers and networking equipment share a dual-UPS chain:

    Primary UPS: EcoFlow RIVER 3 Plus Max
      - Built-in battery: 286 Wh (LiFePO4)
      - Extra Battery 1929: 572 Wh (LiFePO4)
      - Total capacity: 858 Wh
      - Transfer time: <10 ms
      - Output: pure sine wave, 600W (1200W X-Boost)

    Secondary UPS: APC BX750MI-GR
      - Capacity: 750 VA / 410 W
      - Type: Line-Interactive with AVR
      - Battery: RBC17 (12V/9Ah)
      - Provides protection during EcoFlow maintenance/restarts

    Total continuous load: ~140 W (4 servers + networking)
    Estimated runtime on battery: ~5.5 hours (858 Wh at ~140 W,
      accounting for ~10-15% inverter efficiency loss)


Server Details for `minio1`
---------------------------

Server setup:

    Site: a12a
    Hostname: a12a.mabl.online
    Model: ASUS NUC 13 Pro Tall
    OS: Ubuntu 24.04
    CPU: Intel i5-1340P, 13th Gen (16 threads)
    Memory: 64 GB (DDR4 SDRAM)
    Hard drives:
      - 8 TB SSD (Corsair MP600 PRO M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 1 GbE LAN (Intel), address 192.168.1.6/24, public 83.233.237.206
      - 802.11ax Wi-Fi (Intel iwlwifi), address 192.168.0.16/24


Server Details for `minio2`
---------------------------

Server setup:

    Site: a12a
    Hostname: a12b.mabl.online
    Model: ASUS NUC 13 Pro Tall
    OS: Ubuntu 24.04
    CPU: Intel i5-1340P, 13th Gen (16 threads)
    Memory: 64 GB (DDR4 SDRAM)
    Hard drives:
      - 8 TB SSD (Corsair MP600 PRO M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 1 GbE LAN (Intel), address 192.168.1.8/24, public 83.233.237.208
      - 802.11ax Wi-Fi (Intel iwlwifi), address 192.168.0.18/24


Server Details for `minio3`
---------------------------

Server setup:

    Site: a12b
    Hostname: a12c.mabl.online
    Model: ASUS PN52
    OS: Ubuntu 24.04
    CPU: AMD Ryzen 9 5900HX (16 threads)
    Memory: 64 GB (DDR4 SDRAM)
    Hard drives:
      - 4 TB SSD (Corsair MP600 PRO M.2)
      - 4 TB SSD (Kingston NV2 M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 2.5 GbE LAN (Realtek r8125), address 192.168.3.7/24, public 83.233.237.207
      - 802.11ax Wi-Fi (MediaTek mt7921e), address 192.168.0.17/24


Server Details for `minio4`
---------------------------

Server setup:

    Site: a12b
    Hostname: a12d.mabl.online
    Model: ASUS PN53
    OS: Ubuntu 24.04
    CPU: AMD Ryzen 7 7735HS (16 threads)
    Memory: 64 GB (DDR5 SDRAM)
    Hard drives:
      - 4 TB SSD (Kingston NV2 M.2)
      - 4 TB SSD (Kingston NV2 M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 2.5 GbE LAN (Realtek r8125), address 192.168.3.9/24, public 83.233.237.209
      - 802.11ax Wi-Fi (MediaTek mt7921e), address 192.168.0.19/24



Thermal Sensors
---------------

The temperature on the SSH login banner comes from `landscape-sysinfo`, which prints one number
and does not say which component it belongs to. Across these four nodes that is not the same
component twice in a row. Read simultaneously with the underlying hwmon entries on 2026-08-29,
cluster otherwise idle, the banner value matched the hottest sensor on each machine:

    a12a  banner 61.0 °C  <- coretemp Package id 0 61,  acpitz 56,  nvme Composite 47
    a12b  banner 58.0 °C  <- acpitz 58,  coretemp Package id 0 52,  nvme Composite 41
    a12c  banner 63.5 °C  <- k10temp Tctl 63,  nvme Sensor 2 59,  amdgpu edge 53,
                             nvme Composite 52 and 47,  mt7921_phy0 42
    a12d  banner 71.8 °C  <- nvme Sensor 2 71 and 68,  k10temp Tctl 59,
                             nvme Composite 57 and 55,  acpitz 56,  amdgpu edge 51,
                             mt7921_phy0 50

`a12d` is the reading worth remembering. It runs more than ten degrees hotter than any other node
in the banner, and that number is one of its two Kingston NV2 controllers; its processor sits at
59 °C, the second coolest of the four. `a12a` shows a calmer-looking 61.0 °C and that one *is* the
CPU. The higher banner number belongs to the cooler processor, and the banner gives no way to know
it. That has caught us out before: a node showed `80+` at login for ten days, which turned out to
be the CPU package at 86 °C behind two forgotten `btop` processes, while a neighbour showing `70+`
had a perfectly healthy CPU at 54 °C and a warm NVMe controller.

`sensors` (from `lm-sensors`, part of the common-role baseline) prints every source at once, each
labelled with the chip it belongs to. The sets differ per machine:

    a12a, a12b (ASUS NUC 13 Pro, Intel i5-1340P)
      coretemp     CPU package and 14 cores
      acpitz       ACPI thermal zone
      nvme         Corsair MP600 PRO, Composite
      iwlwifi_1    Intel WiFi; fails to read while the radio is idle

    a12c (ASUS PN52, AMD Ryzen 9 5900HX)
      k10temp      Tctl
      nvme         Corsair MP600 PRO, Composite
      nvme         Kingston NV2, Composite and Sensor 2
      amdgpu       integrated GPU, edge
      mt7921_phy0  MediaTek WiFi
      no /sys/class/thermal zones at all on this machine

    a12d (ASUS PN53, AMD Ryzen 7 7735HS)
      k10temp      Tctl
      acpitz       ACPI thermal zone
      nvme         two Kingston NV2, Composite and Sensor 2 each
      amdgpu       integrated GPU, edge
      mt7921_phy0  MediaTek WiFi

Reading this by hand is worse than it looks. `/sys/class/thermal/` is not a common path — a12a and
a12b have three zones, a12d has one, a12c has none. The two NVMe controllers on a12c and a12d both
label themselves `Composite`, only the PCI address separates them, and the hwmon index order does
not follow the `nvmeN` numbering (on a12d, `hwmon1` is `nvme1` and `hwmon2` is `nvme0`). `sensors`
appends the bus and address to each chip name, which is what tells the pair apart. a12d exposes ten
hwmon directories in all, two of them USB-C power supplies and two ASUS WMI platform devices.

None of this polls or alerts. It makes the numbers readable on demand, not noticed on their own.

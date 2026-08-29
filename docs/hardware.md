
Hardware Details for MinIO Servers
==================================

The MinIO servers have a similar baseline with 16-thread CPUs (Intel i5 or AMD Ryzen 7/9), 64 GB memory, and 16 TB of
SSD storage each. The storage is configured as two LVM Volume Groups, a 8 TB "`ubuntu-vg`" with M.2 storage and
a 8 TB "`minio-vg`" with SATA storage. See additional details below.


Site Overview
-------------

All 4 servers are co-located in one server rack on a single subnet behind the `a12-s3` gateway,
sharing the same Bredband2 fiber connection and UPS power chain. Until the consolidation on
2026-08-14 they were split across two sub-sites on separate subnets joined by an IPsec tunnel;
that split is gone and no tunnel sits in the cluster's path any more.

- **Subnet:** 192.168.3.0/24, gateway 192.168.3.1
- **Addressing:** the last octet of each LAN address mirrors its public one — a12a `.207`,
  a12b `.208`, a12c `.209`, a12d `.210`
- **Public IPs:** 1:1 NAT for 83.233.237.207-.210. `83.233.237.206` stays with the `a12-s1` site,
  which remains active for the office and client network but hosts no MinIO servers.
- **Hardware:** minio1 (a12a), minio2 (a12b) — ASUS NUC 13 Pro Tall; minio3 (a12c),
  minio4 (a12d) — ASUS PN52/PN53
- **Link speed:** all four have 2.5 GbE NICs and all four currently negotiate 1 Gb/s
- **WiFi:** built-in on every node, but deliberately disabled since 2026-08-14 — the netplan
  config sits as `00-installer-config-wifi.yaml.disabled` and every radio is down. It is not an
  available fallback path despite the adapters being present. See
  [hacks-and-troubleshooting.md](hacks-and-troubleshooting.md) for why and how to reverse it.


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

    Site: a12-s3
    Hostname: a12a.mabl.online
    Model: ASUS NUC 13 Pro Tall
    OS: Ubuntu 24.04
    CPU: Intel i5-1340P, 13th Gen (16 threads)
    Memory: 64 GB (DDR4 SDRAM)
    Hard drives:
      - 8 TB SSD (Corsair MP600 PRO M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 2.5 GbE LAN (Intel igc), interface enp86s0, address 192.168.3.207/24,
        public 83.233.237.207, currently linked at 1 Gb/s
      - 802.11ax Wi-Fi (Intel iwlwifi), interface wlo1 — disabled, radio down


Server Details for `minio2`
---------------------------

Server setup:

    Site: a12-s3
    Hostname: a12b.mabl.online
    Model: ASUS NUC 13 Pro Tall
    OS: Ubuntu 24.04
    CPU: Intel i5-1340P, 13th Gen (16 threads)
    Memory: 64 GB (DDR4 SDRAM)
    Hard drives:
      - 8 TB SSD (Corsair MP600 PRO M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 2.5 GbE LAN (Intel igc), interface enp86s0, address 192.168.3.208/24,
        public 83.233.237.208, currently linked at 1 Gb/s
      - 802.11ax Wi-Fi (Intel iwlwifi), interface wlo1 — disabled, radio down


Server Details for `minio3`
---------------------------

Server setup:

    Site: a12-s3
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
      - 2.5 GbE LAN (Realtek r8125), interface enp2s0, address 192.168.3.209/24,
        public 83.233.237.209, currently linked at 1 Gb/s
      - 802.11ax Wi-Fi (MediaTek mt7921e), interface wlp3s0 — disabled, radio down


Server Details for `minio4`
---------------------------

Server setup:

    Site: a12-s3
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
      - 2.5 GbE LAN (Realtek r8125), interface enp3s0, address 192.168.3.210/24,
        public 83.233.237.210, currently linked at 1 Gb/s
      - 802.11ax Wi-Fi (MediaTek mt7921e), interface wlp4s0 — disabled, radio down



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

`sensors` also pairs each chip with its own high and critical thresholds, which the raw hwmon walk
does not do for you. That turned out to be the more useful half. Read on 2026-08-29 with the
cluster idle:

    Disk                       Composite now   high / crit   Fitted in
    Corsair MP600 PRO NH       42.9-53.9 °C    83.8 / 88.8   a12a, a12b, a12c
    Kingston NV2 SNV2S4000G    48.9-56.9 °C    76.8 / 78.8   a12c, a12d (two)

The Kingston drives carry a critical limit ten degrees below the Corsair ones, and a12d's pair are
the warmest in the cluster. `Composite` is the sensor those thresholds apply to and it has better
than 20 °C of headroom everywhere, so nothing is close to a limit at idle — but a12d is the node to
re-read under load rather than assume. Its `Sensor 2`, the reading the login banner picks up at
71.8 °C, publishes no thresholds at all: both `high` and `low` come back as +0.0 °C, so that number
cannot be judged against anything the drive tells us.

CPU limits are uniform and generous by comparison. `coretemp` reports high and crit both at
100.0 °C on the Intel nodes, and `k10temp` publishes neither on the AMD ones.

A cosmetic wart worth knowing about: on a12c and a12d every `sensors` run writes two lines to
stderr, because the Kingston firmware does not answer for its third sensor's min/max. The readings
themselves are unaffected.

    ERROR: Can't get value of subfeature temp3_min: I/O error
    ERROR: Can't get value of subfeature temp3_max: I/O error

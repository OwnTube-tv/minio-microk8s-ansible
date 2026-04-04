
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


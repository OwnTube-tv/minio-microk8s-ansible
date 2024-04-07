
Hardware Details for MinIO Servers
==================================

The MinIO servers have a similar baseline with 8-core CPUs (Intel i5 or AMD Ryzen 7/9), 64 GB memory, and 16 TB of
SSD storage each. The storage is configured as two LVM Volume Groups, a 8 TB "`ubuntu-vg`" with M.2 storage and
a 8 TB "`minio-vg`" with SATA storage. See additional details below.


Server Details for `minio1`
---------------------------

Server setup:

    Site name: a12
    Hostname: a12a.mabl.online
    OS: Ubuntu 22.04
    CPU cores: 16 (Intel i5-1340P, 13th Gen)
    Memory: 64 GB (DDR4 SDRAM)
    Hard drives:
      - 8 TB SSD (Corsair MP600 PRO M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 1 GbE LAN (Intel), address 192.168.1.6/24
      - 802.11ax Wi-Fi, address 192.168.0.X/24


Server Details for `minio2`
---------------------------

Server setup:

    Site name: a12
    Hostname: a12b.mabl.online
    OS: Ubuntu 22.04
    CPU cores: 16 (Intel i5-1340P, 13th Gen)
    Memory: 64 GB (DDR4 SDRAM)
    Hard drives:
      - 8 TB SSD (Corsair MP600 PRO M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 1 GbE LAN (Intel), address 192.168.1.8/24
      - 802.11ax Wi-Fi, address 192.168.0.X/24


Server Details for `minio3`
---------------------------

Server setup:

    Site name: v1517
    Hostname: v1517a.mabl.online
    OS: Ubuntu 22.04
    CPU cores: 16 (AMD Ryzen 9)
    Memory: 64 GB (DDR4 SDRAM)
    Hard drives:
      - 4 TB SSD (Corsair MP600 PRO M.2)
      - 4 TB SSD (Kingston NV2 M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 2.5 GbE LAN (Realtek r8125), address 192.168.3.7/24
      - 802.11ax Wi-Fi, address 192.168.2.123


Server Details for `minio4`
---------------------------

Server setup:

    Site name: v1517
    Hostname: v1517b.mabl.online
    OS: Ubuntu 22.04
    CPU cores: 16 (AMD Ryzen 7)
    Memory: 64 GB (DDR5 SDRAM)
    Hard drives:
      - 4 TB SSD (Kingston NV2 M.2)
      - 4 TB SSD (Kingston NV2 M.2)
      - 8 TB SSD (Samsung 870 QVO SATA)
    Network:
      - 2.5 GbE LAN (Realtek r8125), address 192.168.3.9/24
      - 802.11ax Wi-Fi, address 192.168.2.124


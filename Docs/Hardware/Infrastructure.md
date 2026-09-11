# Hardware
This section contains all the physical network appliances of my homelab.

## Server Rack
My homelab is housed in a 8U 10" server rack with the following specifications:

- **Model**: GeeekPi DeskPi RackMate T1
- **Dimensions**: 20W x 28,1D x 45,1H cm

## Main Server

The main server on which most of my virtualized workloads are to run:

- **Model**: Lenovo m920q tiny
- **CPU**: Intel Core i7 8700T (6 cores, 12 threads)
- **RAM**: 32 GB DDR4 ECC (2x16GB)
- **Storage**: 1x 512 GB NVMe SSD for OS/VM storage
- **Network**: 1x 1GbE Intel NIC

## Networking Equipment

| Device | Model | Specifications | Location |
|--------|-------|----------------|----------|
| Core Switch | TP-Link TL-SG108E | Managed switch, 8-port Gigabit | Top of rack |
| Router | OPNsense Appliance | Intel Core i5 9500T (6 cores, 6 threads), 16 GB RAM, 256 GB NVMe SSD, 3x 1GbE ports | Middle of rack |
| Access Points | TP-Link EAP610 | Wi-Fi 6 | Ceiling mounted |

## Dedicated Service Hosts
Planned migration to Proxmox after the main server is operational.

| Service | Model | Specifications | Location |
|--------|-------|----------------|----------|
| DNS-Filter | Raspberry Pi 5 | 4 GB RAM, 128 GB NVMe SSD | Bottom of rack |
| Home Automation | Raspberry Pi 5 | 4 GB RAM, 128GB NVMe SSD | Bottom of rack |
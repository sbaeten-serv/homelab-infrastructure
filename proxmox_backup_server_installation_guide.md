The main guide is based on the following video tutorial: [How to Setup and Use Proxmox Backup Server](https://www.youtube.com/watch?v=33ubleU4OFc&list=PLT98CRl2KxKHnlbYhtABg6cF50bYa8Ulo&index=19&pp=iAQB)

> **Important**: Set a quota to prevent the ZFS filesystem from locking up when the storage pool approaches full capacity (recommended at around 95% of total pool capacity):
> ```bash
> zfs set quota=890G zfs-pool
> ```
# Attach LV to VM

## Host

```bash
sudo lvcreate -L 10G -n disk01 vg_vm                              # create a 10G logical volume named disk01 inside volume group vg_vm — this is the raw "disk" you will attach to the VM
sudo virsh attach-disk jack /dev/vg_vm/disk01 vdb \               # attach the LV /dev/vg_vm/disk01 to the running libvirt VM 'jack' as device vdb
  --targetbus virtio --persistent --live
OR
sudo virsh attach-disk jack\
  --source /dev/vg_vm/disk01 \
  --target vdb \
  --targetbus virtio \
  --persistent \
  --live                                                          # use VirtIO bus for speed, save to VM XML so it survives reboots, and apply live without restarting the VM
sudo virsh domblklist jack                                        # list all disks currently attached to VM 'jack' to verify the new vdb entry appears
```

## VM

```bash
lsblk                                                             # list all block devices inside the VM to confirm the new disk (vdb) is visible and check its size
sudo blkid /dev/vdb                                               # check if vdb already has a filesystem — blank output means it's safe to format
sudo mkfs.ext4 /dev/vdb                                           # format vdb with the ext4 filesystem — WARNING: this erases everything on the disk, only run on a brand-new LV
sudo mkdir -p /mnt/jack01                                         # create the mount point folder /mnt/jack01 (and any missing parent folders) where the disk's files will appear
sudo mount /dev/vdb /mnt/jack01                                   # temporarily attach the filesystem on vdb to /mnt/jack01 in the running kernel — this mount will disappear on reboot
df -h /mnt/jack01                                                 # verify the disk is mounted and see total/used/free space in human-readable format
```

## Persist

```bash
sudo blkid /dev/vdb                                               # get the UUID of vdb — this unique identifier is how fstab will reliably find the disk even if the device name (/dev/vdb) changes between reboots
sudo cp /etc/fstab /etc/fstab.bak                                 # back up the fstab file before editing — if your changes break boot, you can restore this copy from recovery mode
sudo nano /etc/fstab                                              # open the filesystem table and add a line: UUID=<uuid>  /mnt/jack01  ext4  defaults  0  2  — this tells Linux to auto-mount the disk on every boot
sudo umount /mnt/jack01 && sudo mount -a                          # unmount the disk, then re-mount everything listed in fstab — this tests your new entry WITHOUT rebooting so you can fix typos safely while the system is still running
df -h /mnt/jack01                                                 # verify the disk remounted correctly from fstab — if it shows up, the fstab entry is valid
sudo reboot                                                       # reboot the VM to prove the mount survives a fresh boot — Linux will read fstab on startup and auto-mount the disk without any manual command
df -h /mnt/jack01                                                 # after reboot, run this to confirm the disk mounted automatically — success here means persistence is working
```

### fstab line — field breakdown

```
UUID=<uuid>  /mnt/jack01  ext4  defaults  0  2
```

| # | Field | Meaning |
|---|-------|---------|
| 1 | `UUID=<uuid>` | Which filesystem to mount (identified by UUID, not `/dev` path) |
| 2 | `/mnt/jack01` | Where in the filesystem tree to attach it |
| 3 | `ext4` | Filesystem type (must match what `mkfs` created) |
| 4 | `defaults` | Standard mount options (`rw,suid,dev,exec,auto,nouser,async`) |
| 5 | `0` | Dump flag (legacy; always `0`) |
| 6 | `2` | fsck order (`2` = check after root, `0` = skip) |

## Notes

- Use `vdc`/`vdd` if `vdb` is taken.
- Always use `sudo` for `lvs`, `virsh`, `lvcreate`.
- CloudStack won't track this volume — no snapshots, no migration support.

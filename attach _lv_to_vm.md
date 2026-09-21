# Attach LV to VM

## Host

```bash
sudo lvcreate -L 10G -n disk01 vg_vm                              # create 10G LV named disk01 in vg_vm
sudo virsh attach-disk jack /dev/vg_vm/disk01 vdb \               # attach LV to VM 'jack' as vdb
  --targetbus virtio --persistent --live                          # virtio bus, survive reboot, apply live
sudo virsh domblklist jack                                        # list disks attached to VM 'jack'
```

## VM

```bash
lsblk                                                             # list block devices, confirm new disk (vdb)
sudo blkid /dev/vdb                                               # check filesystem on vdb (blank = ok)
sudo mkfs.ext4 /dev/vdb                                           # format vdb as ext4
sudo mkdir -p /mnt/jack01                                         # create mount point
sudo mount /dev/vdb /mnt/jack01                                   # mount vdb at /mnt/jack01
df -h /mnt/jack01                                                 # verify mount and free space
```

## Persist

```bash
sudo blkid /dev/vdb                                               # get UUID of vdb
sudo cp /etc/fstab /etc/fstab.bak                                 # backup fstab before editing
sudo nano /etc/fstab                                              # add: UUID=<uuid> /mnt/jack01 ext4 defaults 0 2
sudo umount /mnt/jack01 && sudo mount -a                          # unmount, remount all from fstab (test)
sudo reboot                                                       # reboot to confirm persistence
```

## Notes

- Use `vdc`/`vdd` if `vdb` is taken.
- Always use `sudo` for `lvs`, `virsh`, `lvcreate`.
- CloudStack won't track this volume — no snapshots, no migration support.

# Create a VM Manually on KVM/CloudStack Host

Full workflow: create LV → build cloud-init ISO → launch VM with `virt-install`.

---

## Step 1 — Create the LV (Host)

```bash
sudo lvcreate -L 10G -n dell vg_vm
```
Creates a 10G logical volume named `dell` inside volume group `vg_vm` — this is the raw disk the VM will use.

---

## Step 2 — Create the Autoinstall Directory (Host)

```bash
sudo mkdir -p /var/lib/libvirt/autoinstall-dell
cd /var/lib/libvirt/autoinstall-dell
```
Creates and enters a working directory to hold the cloud-init files before packaging them into an ISO.

---

## Step 3 — Create `user-data` (Host)

```bash
sudo vi user-data
```

Paste the following:

```yaml
#cloud-config
disable_root: false

autoinstall:
  version: 1

  locale: en_US.UTF-8

  keyboard:
    layout: us

  network:
    network:
      version: 2
      ethernets:
        enp1s0:
          critical: true
          addresses: [172.16.x.xx/21]
          routes:
            - to: default
              via: 172.xx.x.x
          nameservers:
            addresses: [8.8.8.8, 1.1.1.1]

        enp2s0:
          addresses: [10.xx.xx.xx/24]

  storage:
    wipe: superblock
    layout:
      name: lvm

  identity:
    hostname: dell
    username: dell
    password: '$6$hash.password.here'   # generate with: openssl passwd -6

  ssh:
    install-server: true
    allow-pw: true

  late-commands:
    - echo 'dell ALL=(ALL) NOPASSWD:ALL' > /target/etc/sudoers.d/dell
    - chmod 440 /target/etc/sudoers.d/dell
```

Replace:
- `172.16.x.xx/21` → static IP for the primary NIC
- `172.xx.x.x` → gateway
- `10.xx.xx.xx/24` → secondary NIC IP
- `$6$hash.password.here` → password hash, generated with:
  ```bash
  openssl passwd -6
  ```

---

## Step 4 — Create `meta-data` (Host)

```bash
sudo vi meta-data
```

Paste:

```yaml
instance-id: dell-1
local-hostname: dell
```

---

## Step 5 — Create `vendor-data` (Host)

```bash
sudo touch vendor-data
```
Creates an empty file — required by the cloud-init ISO format even if unused.

---

## Step 6 — Build the Autoinstall ISO (Host)

```bash
sudo genisoimage \
  -output autoinstall-dell.iso \
  -volid cidata \
  -joliet \
  -rock \
  user-data \
  meta-data \
  vendor-data
```

| Flag | Purpose |
|------|---------|
| `-output` | Name of the resulting ISO |
| `-volid cidata` | Volume label cloud-init looks for |
| `-joliet -rock` | Adds long-filename support |

Copy it to the libvirt images folder:

```bash
sudo cp autoinstall-dell.iso /var/lib/libvirt/images/autoinstall-dell.iso
sudo chown qemu:qemu /var/lib/libvirt/images/autoinstall-dell.iso
```

---

## Step 7 — Launch the VM (Host)

```bash
sudo virt-install --connect qemu:///system \
  --name dell \
  --memory 4096 \
  --vcpus 4 \
  --disk path=/dev/vg_vm/dell,bus=virtio \
  --location /var/lib/libvirt/images/ubuntu-24.04.3-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
  --disk path=/var/lib/libvirt/images/autoinstall-dell.iso,device=cdrom
```

| Flag | Meaning |
|------|---------|
| `--name dell` | libvirt domain name |
| `--memory 4096` | 4 GB RAM |
| `--vcpus 4` | 4 virtual CPUs |
| `--disk path=/dev/vg_vm/dell,bus=virtio` | Use the LV as the VM's disk over VirtIO |
| `--location ...ubuntu-24.04.3-live-server-amd64.iso,...` | Boot installer from Ubuntu ISO and extract kernel/initrd |
| `--disk ...,device=cdrom` | Attach the cloud-init ISO as a CD-ROM |

---

## Step 8 — Monitor the Install

```bash
sudo virsh console dell
```
Connects to the VM's serial console to watch cloud-init run — exit with `Ctrl+]`.

```bash
sudo virsh list --all
```
Shows the VM state (`running`, `shut off`, etc.).

```bash
sudo virsh domblklist dell
```
Lists all disks attached to the VM — confirm both the LV and the cloud-init ISO appear.

---

## Step 9 — Verify After Install

Once the installer finishes and the VM reboots, log in with the credentials defined in `user-data`, then confirm:

```bash
hostname
ip a
df -h
```

- `hostname` → should be `dell`
- `ip a` → check the static IPs from `user-data` are applied
- `df -h` → root filesystem should be on the LVM layout

---

## Notes

- Replace IPs, gateway, username, and password hash in `user-data` before building the ISO.
- The `-volid cidata` label is **required** — cloud-init scans for it at boot.
- After install, detach the cloud-init ISO from the VM:
  ```bash
  sudo virsh detach-disk dell sda --persistent --live
  ```
  (adjust the target device name via `virsh domblklist dell`)
- To remove the VM completely:
  ```bash
  sudo virsh destroy dell
  sudo virsh undefine dell
  sudo lvremove /dev/vg_vm/dell
  ```

Attach cloud-init drive via SCSI, not IDE


## Context

VM 100 (`ubuntu-22-04-template`, Ubuntu 22.04 KVM-optimised cloud image)
had a cloud-init drive attached as an IDE CD-ROM:

    ide1: local-zfs:vm-100-cloudinit,media=cdrom

This followed the example in Proxmox's own Cloud-Init wiki page, which
uses an IDE device. Proxmox's Hardware tab showed the drive as attached
and configured correctly.

Despite that, cloud-init never ran. Inside the guest, `lsblk` showed only
the main disk (`sda`, `sda1`, `sda14`, `sda15`) — no `sr0`, no CD-ROM, no
second device of any kind. Cloud-init's own log confirmed it:

    ISO9660_DEVS=
    No ds found [mode=search, notfound=disabled]. Disabled cloud-init

Because cloud-init found no datasource, it disabled itself entirely. This
cascaded into a full chain of failures: no netplan config generated, NIC
stayed down, no IP, SSH key never installed, SSH host keys never
generated, SSH refused to start.

The main disk, by contrast, was attached via `scsi0` and was detected by
the guest without issue.

## Options considered

**Keep IDE, investigate further** — rejected. No evidence the IDE path
would ever be reliably detected by this image/kernel combination; would
mean re-debugging the same failure on every future template built from
this image.

**Move cloud-init to SCSI (`scsi1`), matching the main disk's bus** —
tested directly:

    qm stop 100
    qm set 100 --delete ide1
    qm set 100 --scsi1 local-zfs:cloudinit
    qm cloudinit update 100
    qm start 100

After this change, `lsblk` showed the new device:

    sr0      11:0    1     4M  1 rom

and cloud-init correctly detected its datasource:

    status: running
    boot_status_code: enabled-by-generator
    detail: DataSourceNoCloud [seed=/dev/sr0]

Network, SSH key injection, and SSH host key generation all completed
correctly once the datasource was found.

## Decision

Attach the cloud-init drive on SCSI (e.g. `scsi1`), matching the bus the
main disk already uses — not IDE, despite IDE being what Proxmox's own
documentation example shows.

## Consequences

- Every future VM/template built from this same base image (or with the
  same VirtIO SCSI controller config) should default to SCSI for
  cloud-init, not copy the wiki's IDE example without verifying it.
- A device showing as "attached" in Proxmox's Hardware tab is not proof
  the guest OS can see it. Any future cloud-init build should be
  verified from inside the guest (`lsblk`, `cloud-init status --long`)
  before assuming the config applied — not just checked from the
  Proxmox side.
Context: Single 256GB SSD, used, unknown write history. 16GB RAM, both
slots full — RAM is the binding constraint.
Options: ext4 + LVM-thin, or ZFS RAID0.
Both provide thin provisioning and snapshots, so snapshots do not decide it.
Decision: ZFS, lz4, arc_max 1.6GB.
Rationale: block checksumming detects silent corruption on a drive of
unknown history; ZFS clones make VM templating cheap for the K3s nodes in W8.
arc_max set to 1.6GB to fit inside the 1–2GB host allocation already budgeted.
Consequences: On a single disk ZFS detects corruption but cannot repair it —
this is not backup or DR. Recovery path remains rebuild-from-code. Costs
~1.6GB RAM against a 16GB ceiling.
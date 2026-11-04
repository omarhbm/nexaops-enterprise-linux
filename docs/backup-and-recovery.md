# Backup & Recovery — NexaOps V2

## Goal

Protect Nginx configuration files on `web01`, keep a copy on a separate lab VM (`storage01`), and demonstrate recovery from an archive.

## What was implemented in the lab

1. A root-run Bash script on `web01` archives `/etc/nginx` into `/var/backups/nexaops/nginx-YYYY-MM-DD.tar.gz`.
2. A local `systemd` timer schedules the Nginx archive job. The script applies a local retention rule to dated archives.
3. A dedicated `backupsvc` account uses SSH key authentication. ACLs allow reading the source archives; destination permissions allow writing to `/data/backups` on `storage01`.
4. The included `backup/remote-backup.yml` transfers a dated archive through `ctrl01` to `storage01`, using Ansible `fetch` and `copy` without sudo elevation during the transfer.
5. The lab included a restore test into a separate directory, including recovery of a deleted test copy.

## Current automation boundary

**Local backup creation is scheduled. Remote transfer is manual/on-demand** (`ansible-playbook -i inventory.ini backup/remote-backup.yml`). The playbook expects today's archive to exist before execution; it does not create the archive or schedule itself.

The local backup script and its `systemd` unit/timer files were not present in the uploaded source archive and therefore are not reproduced here as source files.

## Verify the remote archive

On `storage01`, after the transfer:

```bash
sudo ls -lh /data/backups/
sudo tar -tzf /data/backups/nginx-$(date +%F).tar.gz | head
```

Use a **test extraction path** to avoid overwriting a live service configuration:

```bash
sudo mkdir -p /root/restore-test
sudo tar -xzf /data/backups/nginx-$(date +%F).tar.gz -C /root/restore-test
sudo ls -l /root/restore-test/nginx/
```

These commands assume an archive for the current date exists. In a real recovery, select the intended dated file and verify its integrity and contents before restoration.

## Limitations / next improvements

- Backup archives contain configuration data; protect both SSH credentials and stored files.
- `backupsvc` has write access to the destination in this lab and may modify or remove files there. For stronger protection, use a separate restricted ingestion area and root-owned promotion/retention.
- Remote scheduling, offsite copies, integrity checksums, alerting, and automated restore drills are not implemented in the included source.

NexaOps V2 — Security & Patch Management

Overview

In Phase 5 of NexaOps V2, I worked on securing the Linux infrastructure and setting up automatic security updates.

I used storage01, running Red Hat Enterprise Linux 9.8, to practice SSH hardening, sudo permissions, ACLs, SELinux, firewall management, service auditing, and patch management.

The goal was to improve server security and make sure security updates could be installed automatically.

1. SSH Hardening

I started by securing SSH access on storage01.

I disabled direct root login and password authentication, limited login attempts to three, and disabled X11 forwarding.

I tested the configuration and reloaded SSH:

sudo sshd -t
sudo systemctl reload sshd

I also confirmed that the ansible user could still connect using an SSH key.

This helped secure remote access without breaking Ansible connectivity.

2. Sudo Permissions

I created an auditor user and gave it permission to run only one administrative command without a password.

sudo useradd auditor
sudo visudo -f /etc/sudoers.d/auditor

The sudo rule was:

auditor ALL=(root) NOPASSWD: /usr/bin/systemctl is-active sshd

I tested the allowed command and checked that the user could not restart SSH without additional authorization.

I kept the ansible user's administrative permissions because they are needed for automation.

3. File Permissions and ACLs

I created a directory for permission testing:

sudo mkdir /srv/security-lab
sudo chmod 700 /srv/security-lab

Then I gave the auditor user access using an ACL:

sudo setfacl -m u:auditor:rwx /srv/security-lab

I confirmed that auditor could create a file inside the directory while nobody received a permission-denied error.

4. SELinux

I checked that SELinux was enforcing:

getenforce

I practiced managing file contexts using semanage and restorecon.

sudo semanage fcontext -a -t httpd_sys_content_t '/srv/security-lab(/.*)?'
sudo restorecon -Rv /srv/security-lab

After testing, I removed the temporary custom rule and restored the default context.

5. Firewall Configuration

I checked the active firewall zone and allowed services:

sudo firewall-cmd --list-all

The storage server needed SSH and NFS access.

I temporarily opened HTTP to practice firewall management, then removed it because storage01 does not host a web server.

This helped me understand the importance of allowing only necessary services.

6. Auditing Running Services

I checked listening ports:

sudo ss -tulnp

I also reviewed running services:

sudo systemctl --type=service --state=running

During the audit, I confirmed that NFS was running and configured the server to use NFSv4 instead of NFSv3.

I verified that web01 could still access its NFS mount after restarting the NFS server.

7. Security Patch Management

Before configuring automatic updates, I checked which packages needed updates.

sudo dnf check-update

Then I checked security advisories:

sudo dnf updateinfo list security

I found pending security updates for curl and libcurl under advisory RHSA-2026:92907.

I previewed the changes without installing anything:

sudo dnf upgrade --security --assumeno

Configuring Automatic Updates

I installed dnf-automatic:

sudo dnf install -y dnf-automatic

Then I edited its configuration:

sudo vim /etc/dnf/automatic.conf

Under [commands], I used:

upgrade_type = security
random_sleep = 0
download_updates = yes
apply_updates = yes

I enabled the timer:

sudo systemctl enable --now dnf-automatic.timer

And verified it:

sudo systemctl status dnf-automatic.timer

The timer was enabled and waiting for its scheduled run.

The update schedule is daily at 6:00 AM, with up to one hour of randomized delay.

Manual Patch Procedure

If I need to install security updates manually during a maintenance window, I can use:

sudo dnf upgrade --security

Before installing patches on an important server, I would review the updates, arrange a maintenance window, and make sure a backup or recovery option is available.

Verifying Updates

After patching, I can review the installation history:

sudo dnf history

Check for remaining security updates:

sudo dnf updateinfo list security

And inspect automatic update logs:

sudo journalctl -u dnf-automatic.service

These commands help confirm whether updates were installed or whether something went wrong.

8. System Health Checks

I checked the server before the first automatic patch run.

uptime
df -h
systemctl --failed

The results showed:

Very low CPU load.

Root filesystem at 18% usage.

/data filesystem at 1% usage.

One failed service: mcelog.service.

Troubleshooting mcelog

I investigated the failure:

sudo systemctl status mcelog.service

The logs showed that mcelog did not support the AMD processor exposed to the VMware virtual machine.

I checked for the recommended AMD kernel module, but it was not loaded.

Since this was a VMware lab and the service was incompatible with the virtual CPU, I disabled it:

sudo systemctl disable --now mcelog.service
sudo systemctl reset-failed mcelog.service

Then I verified:

systemctl --failed

The result was:

0 loaded units listed.

This resolved the failed-service status in my lab. On physical production hardware, I would investigate an appropriate hardware error-monitoring solution instead.

9. Post-Patch Verification

After the scheduled updates run, I will check:

sudo dnf history
sudo dnf updateinfo list security
systemctl --failed

I will also verify the main services:

systemctl status sshd
systemctl status nfs-server
systemctl status firewalld

Finally, I will check the NFS mount on web01:

findmnt /mnt/nexaops

If the updates install successfully and the services remain healthy, the patching process will be considered complete.

Current Status

The security configuration and baseline health checks are complete on storage01.

Automatic security updates are enabled, but the first scheduled installation has not yet been verified.

The remaining step is to confirm the pending security patches were installed and repeat the health checks.

Screenshots

36-sudo-least-privilege.png

37-file-permissions-acl.png

38-selinux-contexts.png

39-firewalld-zones-services.png

40-open-ports-services-audit.png

41-dnf-security-patch-management.png

42-system-health-validation.png

Project: NexaOps V2
Phase: 5 — Linux Security & Patch Management
Deliverable: Secured Linux infrastructure with documented patch procedures.

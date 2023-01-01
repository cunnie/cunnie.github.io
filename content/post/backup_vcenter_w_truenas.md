---
title: The Least Secure Way to Back Up vCenter 8.0 with TrueNAS 13.0
date: 2023-01-02T16:34:10-08:00
draft: false
---

We're going to set up automated backups for a vCenter which we were forced to
rebuild over the winter break because the unexpected reboot of the file server
hosting the iSCSI datastore backing the vCenter's disk drive caused
unrecoverable database corruption, and we had no backups.

- Log into your [TrueNAS](https://www.truenas.com/) server via its web
  interface, e.g. <https://nas.nono.io>
- Browse to "Services"
- Start FTP (by toggling the "Running" slider) and configure it to start
  automatically

{{< figure src="https://user-images.githubusercontent.com/1020675/210378500-cee6db21-5ec0-40f4-a58a-a7e6484e2018.png" alt="TrueNAS Services Configuration Screen"  caption="Remember to start the FTP service and configure it to start automatically. Once that's done, you can configure it by clicking on the ✎ icon in the Actions column.">}}

- Click on the ✎ (pencil icon) in the actions tab to configure FTP
  - Bump "Connections" to 20 to avoid the error on the TrueNAS console,
    "proftpd[63793]: 127.0.0.1 (10.9.9.128[10.9.9.128]) - Connection refused
    (MaxConnectionsPerHost 2)"
  - Check "Allow Anonymous Login" to enable anonymous FTP
  - Browse to the path where you want the backups stored

{{< figure src="https://user-images.githubusercontent.com/1020675/210401600-0714afc1-77b2-44d6-baec-a180df88b3bd.png" alt="TrueNAS FTP Configuration Screen"  caption="Remember to bump the number of connections 2 → 20, enable anonymous FTP, and browse to the directory where you want your backups stored.">}}

Let's configure the vCenter to backup to the TrueNAS server via anonymous FTP:

- Browse to your vCenter Server Appliance Web Console (VAMI) on port 5480, e.g.
  <https://vcenter-80.nono.io:5480>
- Log in
- Browse to "Backup"
- Click "Activate"
- Click "Edit"
- Type the backup location, protocol (FTP) followed by the TrueNAS server's
  hostname, e.g. "ftp://nas.nono.io"
- Username: "ftp"
- Password: "administrator@vsphere.local". Anonymous FTP expects this to be an
  identifier, usually an email address (by convention). Don't put a real
  password here because it may appear in the logs
- Encrypt backup: enter a password to encrypt the backup; we use the same
  password as our "administrator@vsphere.local" account so we can remember it.
  Yes, we know that by setting a password here we're no longer the "The Least
  Secure Way" to backup our vCenter. What can we say? We wanted a catchy title
- We choose to retain 60 backups. Choose as many as you'd like
- Click "Save"

{{< figure src="https://user-images.githubusercontent.com/1020675/210406682-cd74c4a7-3615-4c34-bc29-f718c5d71863.png" alt="vCenter Backup Configuration Screen"  caption="">}}

- Click "Backup Now" to test our configuration
- Check "Use backup location and user name from backup schedule."
- Enter the encryption password
- Click "Start"; make sure the backup successfully completes

### Security Considerations

Don't back up your vCenter this way unless you don't care about security. Don't
use FTP, certainly not anonymous FTP. Use one of the more secure protocols,
e.g. FTPS, HTTPS, SFTP, SMB.

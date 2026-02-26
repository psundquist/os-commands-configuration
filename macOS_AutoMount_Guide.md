# macOS Network Share Auto-Mount

## Configuration Guide

**Using Autofs for Persistent SMB and NFS Mounts via SSH**

*February 2026*

---

# Overview

This guide documents how to configure a macOS system to automatically mount network file shares (SMB and NFS) using autofs. This method is designed for headless (SSH-only) environments where no user will log in via the GUI. Mounts are triggered on demand when the path is accessed, require no sudo from end users after initial setup, and survive reboots without any additional scripts or LaunchDaemons.

## Why Autofs?

macOS includes several mechanisms for mounting network shares, but most are unreliable in headless SSH-only scenarios. Autofs is the recommended approach for the following reasons:

- Built into macOS as a system service — no third-party software required
- Starts automatically at every boot and reads its configuration from /etc/auto_master
- Mounts shares on demand when the path is accessed (lazy mounting)
- Automatically remounts if the server becomes temporarily unavailable
- End users do not need sudo privileges to trigger a mount
- No GUI login, LaunchDaemons, cron jobs, or startup scripts required
- Supports both SMB (Windows shares) and NFS mounts using the same framework

> **Important:** Do not use /Volumes as a mount point. macOS manages /Volumes internally, and autofs cannot create mount triggers there. This guide uses /smb and /nfs as base paths instead.

# Prerequisites

- macOS system with SSH access enabled (tested on macOS 26.3)
- Administrative (sudo) access for initial one-time setup
- For SMB: A Windows PC sharing a folder via SMB at a known IP address
- For NFS: A server exporting a directory via NFS to your Mac's IP address
- SMB credentials: username, password, and workgroup/domain name

# How Autofs Works

Autofs is a macOS system service that starts at every boot. It reads /etc/auto_master, which lists map files. Each map file defines one or more network shares and where to mount them. Here is the behavior to expect:

**On boot:** Autofs starts automatically, reads auto_master, and creates trigger directories at the configured base paths (e.g., /smb, /nfs). No shares are mounted yet.

**On first access:** When any user or process accesses a mount path (e.g., via ls, cd, or any file operation), autofs transparently mounts the share. No sudo is required.

**During idle periods:** If a mount is unused for approximately 60 seconds, autofs may unmount it. It will remount automatically on the next access.

**If the server is unavailable:** If the server is offline or unreachable, the access attempt will fail. Autofs will retry on the next access attempt once the server is back online.

---

## Part 1: SMB (Windows File Share)

### SMB Configuration Details

| Parameter | Value |
|---|---|
| SMB Server IP | 192.168.86.65 |
| Share Name | Recordings |
| Username | username |
| Workgroup / Domain | WORKGROUP |
| Password | password |
| Mount Point | /smb/Recordings |

> **Security Note:** The password is stored in plain text in /etc/auto_smb. File permissions are set to 600 (root read/write only) to restrict access. If the password contains special characters, they must be URL-encoded (e.g., ! becomes %21).

# SMB Configuration Steps

All steps require sudo and are performed once. After setup, no further administrative access is needed.

## Step 1: Edit /etc/auto_master

The auto_master file tells autofs which map files to load. Add a line that defines /smb as the base directory and points to a custom map file.

```
sudo vi /etc/auto_master
```

Add the following line at the bottom of the file:

```
/smb    auto_smb
```

This tells autofs: manage the /smb directory using the map defined in /etc/auto_smb. The /smb directory will be created and managed by autofs automatically.

## Step 2: Create /etc/auto_smb

This map file defines the specific SMB share to mount. Autofs combines the base path from auto_master (/smb) with the key in this file (Recordings) to produce the final mount point /smb/Recordings.

```
sudo vi /etc/auto_smb
```

Enter the following as the entire contents of the file:

```
Recordings -fstype=smbfs ://WORKGROUP;username:password%21@192.168.86.65/Recordings
```

Breaking down the syntax:

| Component | Description |
|---|---|
| Recordings | Key name — appended to /smb to form mount path |
| -fstype=smbfs | Specifies the SMB filesystem type |
| WORKGROUP | Windows workgroup or domain name |
| username | SMB username |
| passwords | Password with ! URL-encoded as %21 |
| 192.168.86.65 | IP address of the Windows PC |
| /Recordings | Name of the shared folder on the server |

## Step 3: Secure the Map File

Since the map file contains credentials in plain text, restrict permissions so only root can read it:

```
sudo chmod 600 /etc/auto_smb
```

## Step 4: Reload Autofs

Apply the new configuration without rebooting:

```
sudo automount -vc
```

You should see output confirming the new map was loaded. Any errors will be displayed here.

## Step 5: Test the SMB Mount

Access the mount point. Autofs will mount the share on demand:

```
ls /smb/Recordings
```

You should see the contents of the Windows shared Recordings folder. No sudo required.

---

## Part 2: NFS (Network File System)

### NFS Configuration Details

| Parameter | Value |
|---|---|
| NFS Server IP | 192.168.86.30 |
| Export Path | /Scratch/ASL-Audio/554780 |
| Mount Point | /nfs/ASL-Audio |

> NFS does not require credentials in the autofs map file. Access control is handled on the NFS server side via its export configuration (e.g., /etc/exports). Ensure the server exports the path to your Mac's IP address.

# NFS Configuration Steps

## Step 1: Edit /etc/auto_master

Add a second line to auto_master for the NFS map. If you already have the /smb line from Part 1, add this below it.

```
sudo vi /etc/auto_master
```

Add the following line:

```
/nfs    auto_nfs
```

Your auto_master file should now contain both entries (among any default entries):

```
/smb    auto_smb
/nfs    auto_nfs
```

## Step 2: Create /etc/auto_nfs

Create the NFS map file. This is simpler than SMB because no credentials are embedded.

```
sudo vi /etc/auto_nfs
```

Enter the following as the entire contents of the file:

```
ASL-Audio -fstype=nfs,resvport 192.168.86.30:/Scratch/ASL-Audio/554780
```

Breaking down the syntax:

| Component | Description |
|---|---|
| ASL-Audio | Key name — appended to /nfs to form mount path |
| -fstype=nfs | Specifies the NFS filesystem type |
| resvport | Required on macOS — uses a reserved port (<1024) |
| 192.168.86.30 | IP address of the NFS server |
| /Scratch/ASL-Audio/554780 | Full export path on the NFS server |

> **Critical:** The resvport option is required on macOS. Without it, NFS mounts will fail because macOS defaults to requiring a reserved (privileged) source port for NFS connections.

## Step 3: Reload Autofs

Apply the new configuration:

```
sudo automount -vc
```

## Step 4: Test the NFS Mount

Access the mount point:

```
ls /nfs/ASL-Audio
```

You should see the contents of the NFS export. No sudo required.

## Optional: Create Shortcut Links

For convenience, create symbolic links in your home directory:

```
ln -s /smb/Recordings ~/Recordings
ln -s /nfs/ASL-Audio ~/ASL-Audio
```

---

# Troubleshooting

## Verify SMB Credentials Manually

If the SMB autofs mount does not work, verify credentials by mounting manually:

```
mkdir -p /tmp/testmount
mount_smbfs '//WORKGROUP;username:password%21@192.168.86.65/Recordings' /tmp/testmount
ls /tmp/testmount
umount /tmp/testmount
```

If this manual mount fails, the issue is with credentials, network connectivity, or the Windows share configuration rather than autofs.

## Verify NFS Export Manually

If the NFS autofs mount does not work, verify the export by mounting manually:

```
mkdir -p /tmp/testnfs
sudo mount -t nfs -o resvport 192.168.86.30:/Scratch/ASL-Audio/554780 /tmp/testnfs
ls /tmp/testnfs
sudo umount /tmp/testnfs
```

If this fails, check the NFS server's export configuration to ensure your Mac's IP address is permitted.

## Check Autofs Logs

View recent autofs activity in the system log:

```
log show --predicate 'subsystem == "com.apple.automount"' --last 5m
```

## Reload After Configuration Changes

If you modify any map file or auto_master, reload autofs:

```
sudo automount -vc
```

## Common Issues

**"mountpoint unavailable" error:** This occurs when the mount point path is inside a macOS-managed directory such as /Volumes. Use /smb, /nfs, or another non-system path as described in this guide.

**"Permission denied" on automount:** The automount command requires sudo. End users do not need to run this command — they only need to access the /smb or /nfs path.

**SMB domain/workgroup not recognized:** If the Windows PC is not part of a domain, try removing the workgroup prefix from the auto_smb file:

```
Recordings -fstype=smbfs ://username:password%21@192.168.86.65/Recordings
```

**NFS mount fails without resvport:** macOS requires the resvport mount option for NFS. Ensure `-fstype=nfs,resvport` is specified in the auto_nfs map file.

## Special Character URL Encoding (SMB)

If the SMB password contains special characters, they must be URL-encoded in the auto_smb file. Common encodings:

| Character | URL Encoding |
|---|---|
| ! | %21 |
| @ | %40 |
| # | %23 |
| $ | %24 |
| % | %25 |
| ^ | %5E |
| & | %26 |
| * | %2A |
| (space) | %20 |

# Configuration File Summary

## Complete /etc/auto_master Example

For reference, here is what the relevant additions to /etc/auto_master should look like with both SMB and NFS configured:

```
# (existing default entries above)
/smb    auto_smb
/nfs    auto_nfs
```

| File | Purpose |
|---|---|
| /etc/auto_master | Master autofs configuration — defines base paths and map files |
| /etc/auto_smb | SMB map file — defines Windows share details and credentials |
| /etc/auto_nfs | NFS map file — defines NFS export paths and mount options |
| /smb/Recordings | SMB mount point — created and managed by autofs |
| /nfs/ASL-Audio | NFS mount point — created and managed by autofs |

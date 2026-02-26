# macOS SMB Auto-Mount Configuration Guide

*Using Autofs for Persistent, Headless SMB Mounts via SSH*
*February 2026*

## Overview

This guide documents how to configure a macOS system to automatically mount a Windows SMB file share using autofs. This method is designed for headless (SSH-only) environments where no user will log in via the GUI. The mount is triggered on demand when the path is accessed, requires no sudo from end users after initial setup, and survives reboots without any additional scripts or LaunchDaemons.

## Why Autofs?

macOS includes several mechanisms for mounting network shares, but most are unreliable in headless SSH-only scenarios. Key advantages:

- Built into macOS as a system service
- Starts automatically at every boot, reads config from `/etc/auto_master`
- Mounts shares on demand (lazy mounting)
- Automatically remounts if the server becomes temporarily unavailable
- No GUI login, LaunchDaemons, cron jobs, or startup scripts required

> **Important:** Do not use `/Volumes` as a mount point. This guide uses `/smb` instead.

## Prerequisites

- macOS system with SSH access enabled (tested on macOS 26.3)
- Administrative (`sudo`) access for initial one-time setup
- A Windows PC sharing a folder via SMB at a known IP address
- SMB credentials: username, password, and workgroup/domain name

## Example Configuration

| Parameter         | Value            |
|-------------------|------------------|
| SMB Server IP     | 192.168.86.65    |
| Share Name        | Recordings       |
| Username          | username         |
| Workgroup/Domain  | WORKGROUP        |
| Password          | password         |
| Mount Point       | /smb/Recordings  |

## Configuration Steps

### Step 1: Edit `/etc/auto_master`

Add the following line at the bottom of `/etc/auto_master`:

```
/smb    auto_smb
```

### Step 2: Create `/etc/auto_smb`

Create the file `/etc/auto_smb` with the following contents:

```
Recordings -fstype=smbfs ://WORKGROUP;username:password%21@192.168.86.65/Recordings
```

### Step 3: Secure the Map File

```bash
sudo chmod 600 /etc/auto_smb
```

### Step 4: Reload Autofs

```bash
sudo automount -vc
```

### Step 5: Test the Mount

```bash
ls /smb/Recordings
```

## Troubleshooting

### URL Encoding for Special Characters in Passwords

If your password contains special characters, they must be URL-encoded in the autofs map file. For example:

| Character | Encoded |
|-----------|---------|
| `!`       | `%21`   |
| `@`       | `%40`   |
| `#`       | `%23`   |
| `$`       | `%24`   |
| `%`       | `%25`   |

## Configuration File Summary

| File              | Purpose                                      |
|-------------------|----------------------------------------------|
| `/etc/auto_master` | Master autofs map; defines mount namespaces |
| `/etc/auto_smb`   | Map file defining individual SMB mounts      |

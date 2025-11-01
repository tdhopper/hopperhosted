# Jellyfin

### Architecture Overview

```
┌──────────────────┐        NFS/SMB         ┌─────────────────┐
│   Mac Mini       │◄──────────────────────►│  Synology NAS   │
│   (dobro)        │                        │  (192.168.68.89)│
│                  │                        │                 │
│  Jellyfin Server │                        │  Media Storage  │
│  Hardware Trans. │                        │  /volume1/docker│
└──────────────────┘                        └─────────────────┘
```

**Strategy:**
- Mac Mini: Runs Jellyfin, handles transcoding (M-series HW acceleration)
- Synology: Stores media files, serves via NFS

---

### Step 1: Configure Synology NFS Export

=== "Enable NFS Service"

    1. Open **Control Panel** → **File Services** → **NFS**
    2. Check **Enable NFS Service**
    3. Set **Maximum NFS Protocol:** NFSv3
    4. Click **Apply**

=== "Configure NFS Permissions"

    1. Go to **Control Panel** → **Shared Folder**
    2. Select `docker` folder → **Edit** → **NFS Permissions**
    3. Create new rule:
       - **Hostname/IP:** `192.168.68.101` (Mac Mini IP)
       - **Privilege:** Read/Write
       - **Squash:** Map all users to admin
       - ✅ Allow connections from non-privileged ports
       - ✅ Allow users to access mounted sub-folders
       - ✅ Enable asynchronous
    4. Click **Apply**

=== "Restart NFS Service"

    SSH into Synology:
    ```bash
    sudo synoservice --restart nfsd
    ```

    Verify service is running:
    ```bash
    sudo synoservice --status nfsd
    ```

!!! note
    macOS requires "Allow connections from non-privileged ports" for NFSv3 mounts

### Step 2: Mount NFS Share on macOS

=== "Create Mount Point"

    ```bash
    sudo mkdir -p /Users/Shared/media/docker
    ```

=== "Test Mount (Temporary)"

    ```bash
    sudo mount_nfs -o vers=3,tcp,nolocks,async \
      192.168.68.89:/volume1/docker \
      /Users/Shared/media/docker
    ```

    **Verify:**
    ```bash
    ls -la /Users/Shared/media/docker
    df -h | grep docker
    ```

=== "Make Persistent (Auto-Mount)"

    Edit `/etc/fstab`:
    ```bash
    sudo nano /etc/fstab
    ```

    Add this line:
    ```
    192.168.68.89:/volume1/docker /Users/Shared/media/docker nfs vers=3,tcp,nolocks,async 0 0
    ```

    Reload automounter:
    ```bash
    sudo automount -vc
    ```

=== "Alternative: SMB Mount"

    If NFS has issues, use SMB instead:
    ```bash
    mount_smbfs //tdhopper@192.168.68.89/docker \
      /Users/Shared/media/docker
    ```

!!! tip
    Use wired Ethernet for both NAS and Mac for best performance

### Step 3: Install Jellyfin on Mac Mini

=== "Install via Homebrew"

    ```bash
    brew install --cask jellyfin
    ```

    **Verify installation:**
    ```bash
    brew list --cask | grep jellyfin
    ```

=== "Configure Hardware Transcoding"

    1. Open `http://localhost:8096`
    2. Complete initial setup wizard
    3. Go to **Dashboard** → **Playback** → **Transcoding**
    4. Enable **Hardware Acceleration: VideoToolbox**
    5. Set directories:
       - **Transcode temp:** `/Users/jellyfin/transcode`
       - **Cache:** `/Users/jellyfin/cache`

    Create directories:
    ```bash
    sudo mkdir -p /Users/jellyfin/transcode
    sudo mkdir -p /Users/jellyfin/cache
    sudo chown -R $(whoami) /Users/jellyfin
    ```

=== "Add Media Libraries"

    In Jellyfin dashboard:

    **Movies:**
    ```
    /Users/Shared/media/docker/media/Movies
    ```

    **TV Shows:**
    ```
    /Users/Shared/media/docker/media/TV
    ```

    **Music:**
    ```
    /Users/Shared/media/docker/media/Music
    ```

!!! tip
    M-series Mac should use VideoToolbox for near-zero CPU usage during 4K transcoding

### File Structure Reference

| Component | Purpose | Path |
|-----------|---------|------|
| NAS Media | Persistent storage | `/volume1/docker/media/` |
| Mac Mount | Local NFS access | `/Users/Shared/media/docker` |
| Transcoding | Temp scratch space | `/Users/jellyfin/transcode` |
| Jellyfin Config | App settings | `/Users/<user>/.config/jellyfin` |

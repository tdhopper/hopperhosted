# OPERATIONS // RUNBOOK DATABASE

!!! warning "RESTRICTED ACCESS"
    These runbooks contain critical system procedures. Handle with care.

---

## 🐳 DOCKER CONTAINER MANAGEMENT

### OpenAudible Container (Synology)

**Purpose:** Audiobook management and conversion
**Host:** Synology NAS
**Port:** 3000

=== "Start Container"

    ```bash
    sudo docker run -d --rm -it \
      -v /volume1/docker/openaudible/:/config/OpenAudible \
      -p 3000:3000 \
      -e PGID=`id -g` \
      -e PUID=`id -u` \
      --name openaudible \
      openaudible/openaudible:latest
    ```

    **Verification:**
    ```bash
    docker ps | grep openaudible
    curl -I http://localhost:3000
    ```

=== "Stop Container"

    ```bash
    docker stop openaudible
    ```

=== "View Logs"

    ```bash
    docker logs -f openaudible
    ```

!!! note "DATA PERSISTENCE"
    Configuration stored in `/volume1/docker/openaudible/`

---

### Pi-hole Container (Synology)

**Purpose:** Network-wide ad blocking
**Host:** Synology NAS
**Port:** 8765 (Web UI)

=== "Start Container"

    ```bash
    sudo docker run -d \
      --name pihole \
      --network=host \
      -e TZ="America/New_York" \
      -e WEBPASSWORD="" \
      -e WEB_PORT=8765 \
      -e PIHOLE_DNS_="8.8.8.8;8.8.4.4" \
      -v /volume1/docker/pihole:/etc/pihole \
      -v /volume1/docker/pihole/dnsmasq.d:/etc/dnsmasq.d \
      --restart=unless-stopped \
      -e DNSMASQ_USER=root \
      -e DNSMASQ_LISTENING=local \
      pihole/pihole:latest
    ```

    !!! warning "NETWORK MODE"
        Using `--network=host` for DNS functionality. Container shares host networking.

=== "Check Status"

    ```bash
    docker ps | grep pihole
    docker exec pihole pihole status
    ```

=== "Access Admin Panel"

    **URL:** `http://[NAS-IP]:8765/admin`
    **Password:** Set via `WEBPASSWORD` env var

=== "Update Gravity"

    ```bash
    docker exec pihole pihole -g
    ```

---

## 🎬 JELLYFIN SETUP (MAC MINI + SYNOLOGY)

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

### STEP 1: Configure Synology NFS Export

**Priority:** CRITICAL
**Time Required:** 10 minutes

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

!!! note "MAC COMPATIBILITY"
    macOS requires "Allow connections from non-privileged ports" for NFSv3 mounts

---

### STEP 2: Mount NFS Share on macOS

**Host:** Mac Mini (dobro)
**Priority:** CRITICAL

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

!!! warning "NETWORK REQUIREMENT"
    Use wired Ethernet for both NAS and Mac for best performance

---

### STEP 3: Install Jellyfin on Mac Mini

**Host:** dobro
**Priority:** HIGH

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

!!! tip "PERFORMANCE OPTIMIZATION"
    M-series Mac should use VideoToolbox for near-zero CPU usage during 4K transcoding

---

### File Structure Reference

| Component | Purpose | Path |
|-----------|---------|------|
| NAS Media | Persistent storage | `/volume1/docker/media/` |
| Mac Mount | Local NFS access | `/Users/Shared/media/docker` |
| Transcoding | Temp scratch space | `/Users/jellyfin/transcode` |
| Jellyfin Config | App settings | `/Users/<user>/.config/jellyfin` |

---

## 🌐 CADDY SERVER MANAGEMENT

### Overview

**Host:** dobro (100.80.29.92)
**Purpose:** Reverse proxy with automatic HTTPS for `*.hopperhosted.com`
**DNS Challenge:** Cloudflare

---

### Build Caddy with Cloudflare Plugin

**Required:** Custom build includes Cloudflare DNS-01 challenge support
**Frequency:** When updating Caddy or plugins

=== "Prerequisites"

    ```bash
    brew install go caddy
    ```

    Add Go bin to PATH:
    ```bash
    echo 'export PATH="$PATH:$(go env GOPATH)/bin"' >> ~/.zshrc
    source ~/.zshrc
    ```

=== "Install xcaddy"

    ```bash
    go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
    ```

=== "Build Custom Caddy"

    ```bash
    PREFIX="$(brew --prefix)"
    sudo mkdir -p "$PREFIX/libexec/caddy"

    xcaddy build --with github.com/caddy-dns/cloudflare@latest \
      --output "$PREFIX/libexec/caddy/caddy-cloudflare"
    ```

    **Verify Cloudflare module:**
    ```bash
    "$PREFIX/libexec/caddy/caddy-cloudflare" list-modules | grep cloudflare
    ```

    Expected output:
    ```
    dns.providers.cloudflare
    ```

=== "Configure brew services"

    ```bash
    # Stop existing service
    brew services stop caddy
    brew services start caddy && sleep 1 && brew services stop caddy

    # Update LaunchAgent plist
    PLIST=~/Library/LaunchAgents/homebrew.mxcl.caddy.plist

    /usr/libexec/PlistBuddy -c "Set :ProgramArguments:0 $PREFIX/libexec/caddy/caddy-cloudflare" "$PLIST"
    /usr/libexec/PlistBuddy -c "Add :EnvironmentVariables dict" "$PLIST" 2>/dev/null || true
    /usr/libexec/PlistBuddy -c "Set :EnvironmentVariables:CLOUDFLARE_API_TOKEN YOUR_TOKEN_HERE" "$PLIST"
    /usr/libexec/PlistBuddy -c "Add :KeepAlive bool true" "$PLIST" 2>/dev/null || true

    # Reload service
    launchctl unload "$PLIST"
    launchctl load "$PLIST"
    brew services start caddy
    ```

!!! warning "API TOKEN REQUIRED"
    Get Cloudflare API token from: https://dash.cloudflare.com/profile/api-tokens

---

### Important File Locations

| Purpose | Path |
|---------|------|
| **Active Config** | `/usr/local/etc/Caddyfile` |
| **Working Copy** | `~/Caddyfile` |
| **Binary** | `/usr/local/opt/caddy/bin/caddy` |
| **Logs** | `/usr/local/var/log/caddy.log` |
| **Data Directory** | `/usr/local/var/lib/caddy/` |
| **LaunchAgent** | `~/Library/LaunchAgents/homebrew.mxcl.caddy.plist` |

!!! note "CONFIG LOCATION"
    LaunchAgent is hardcoded to use `/usr/local/etc/Caddyfile`, not `~/Caddyfile`

---

### Common Operations

=== "Check Status"

    ```bash
    # Via SSH
    ssh dobro "ps aux | grep caddy | grep -v grep"

    # LaunchAgent status
    ssh dobro "launchctl list | grep caddy"

    # Check listening ports
    ssh dobro "netstat -an | grep -E ':(80|443).*LISTEN'"
    ```

=== "View Logs"

    ```bash
    ssh dobro "tail -f /usr/local/var/log/caddy.log"
    ```

=== "Edit Configuration"

    ```bash
    # Edit working copy
    vim ~/Caddyfile

    # Copy to active location
    ssh dobro "sudo cp ~/Caddyfile /usr/local/etc/Caddyfile"
    ```

=== "Validate Config"

    ```bash
    ssh dobro "/usr/local/opt/caddy/bin/caddy validate --config /usr/local/etc/Caddyfile"
    ```

=== "Reload Config"

    ```bash
    ssh dobro "sudo /usr/local/opt/caddy/bin/caddy reload --config /usr/local/etc/Caddyfile"
    ```

=== "Restart Service"

    ```bash
    ssh dobro "launchctl stop homebrew.mxcl.caddy && launchctl start homebrew.mxcl.caddy"
    ```

---

### Example Caddyfile

=== "Basic Structure"

    ```
    {
      email you@example.com
    }

    example.com {
      tls {
        dns cloudflare {
          token {env.CLOUDFLARE_API_TOKEN}
        }
      }
      reverse_proxy 127.0.0.1:3000
    }
    ```

=== "Jellyfin (HTTP Backend)"

    ```
    jellyfin.hopperhosted.com {
      tls {
        dns cloudflare {
          token {env.CLOUDFLARE_API_TOKEN}
        }
      }
      reverse_proxy 127.0.0.1:8096
    }
    ```

    !!! note "NO TLS TRANSPORT"
        Jellyfin runs plain HTTP, so no TLS transport config needed

=== "Synology (HTTPS Backend)"

    ```
    synology.hopperhosted.com {
      tls {
        dns cloudflare {
          token {env.CLOUDFLARE_API_TOKEN}
        }
      }
      reverse_proxy https://192.168.68.89:5001 {
        transport http {
          tls_insecure_skip_verify
        }
      }
    }
    ```

    !!! warning "BACKEND TLS"
        Synology uses self-signed cert, requires `tls_insecure_skip_verify`

---

### Troubleshooting

=== "Caddy Won't Start"

    **Check logs:**
    ```bash
    ssh dobro "tail -50 /usr/local/var/log/caddy.log"
    ```

    **Common issues:**

    - ❌ Caddyfile not found at `/usr/local/etc/Caddyfile`
    - ❌ Syntax errors in Caddyfile
    - ❌ Missing Cloudflare DNS module
    - ❌ Port 80/443 already in use

    **Verify ports are free:**
    ```bash
    ssh dobro "lsof -i :80"
    ssh dobro "lsof -i :443"
    ```

=== "502 Bad Gateway"

    **Checklist:**

    1. Is backend service running?
       ```bash
       ssh dobro "curl -I http://127.0.0.1:8096"
       ```

    2. Check reverse_proxy IP/port

    3. TLS mismatch?
       - HTTP backend → no transport config
       - HTTPS backend → add TLS transport

=== "Certificate Issues"

    **Check ACME logs:**
    ```bash
    ssh dobro "grep -i acme /usr/local/var/log/caddy.log | tail -20"
    ```

    **Common causes:**

    - ❌ Invalid Cloudflare API token
    - ❌ DNS records not pointing to 100.80.29.92
    - ❌ Cloudflare API token lacks DNS edit permissions

    **Verify DNS:**
    ```bash
    dig +short jellyfin.hopperhosted.com
    ```

=== "Can't Access Externally"

    **Verification steps:**

    1. Is Caddy listening?
       ```bash
       ssh dobro "netstat -an | grep -E ':(80|443).*LISTEN'"
       ```

    2. Is Caddy running?
       ```bash
       ssh dobro "ps aux | grep caddy"
       ```

    3. Tailscale connectivity?
       ```bash
       ping 100.80.29.92
       ```

    4. Test from dobro itself:
       ```bash
       ssh dobro "curl -I https://jellyfin.hopperhosted.com"
       ```

---

### Workflow: Making Configuration Changes

**Standard procedure for updating Caddyfile:**

1. **Edit working copy**
   ```bash
   vim ~/Caddyfile
   ```

2. **Validate syntax** (optional but recommended)
   ```bash
   caddy validate --config ~/Caddyfile
   ```

3. **Copy to active location**
   ```bash
   ssh dobro "sudo cp ~/Caddyfile /usr/local/etc/Caddyfile"
   ```

4. **Reload Caddy**
   ```bash
   ssh dobro "sudo /usr/local/opt/caddy/bin/caddy reload --config /usr/local/etc/Caddyfile"
   ```

5. **Test changes**
   ```bash
   curl -I https://yourservice.hopperhosted.com
   ```

6. **Monitor logs** (if issues occur)
   ```bash
   ssh dobro "tail -f /usr/local/var/log/caddy.log"
   ```

!!! tip "AUTO-START"
    LaunchAgent is configured to:

    - ✅ Start automatically at login (`RunAtLoad`)
    - ✅ Restart if it crashes (`KeepAlive`)
    - ✅ Run in background sessions

---

## 🚨 INCIDENT RESPONSE

### Service Down

**Response Time:** Immediate

1. **Identify** the affected service
2. **Check** service status
   ```bash
   docker ps -a
   launchctl list | grep [service]
   ```
3. **Review** recent logs
4. **Attempt** service restart
5. **Escalate** if issue persists

---

### High Resource Usage

**Alert Threshold:** CPU > 80% or Memory > 90% sustained

```bash
# Check resource usage
top -l 1 | head -20

# Identify resource hogs
ps aux --sort=-%mem | head -10
ps aux --sort=-%cpu | head -10

# Check Docker stats
docker stats --no-stream
```

---

## 📞 EMERGENCY CONTACTS

| ROLE | CONTACT | AVAILABILITY |
|------|---------|--------------|
| System Admin | admin@hopperhosted.com | 24/7 |
| Network Lead | network@hopperhosted.com | Business Hours |
| Security Team | security@hopperhosted.com | 24/7 |

---

## 📊 QUICK REFERENCE

### System IPs

| Host | IP Address | Purpose |
|------|------------|---------|
| dobro | 100.80.29.92 | Mac Mini - Jellyfin, Caddy |
| Synology | 192.168.68.89 | NAS - Media storage, Docker |

### Port Map

| Service | Host | Port | Protocol |
|---------|------|------|----------|
| Jellyfin | dobro | 8096 | HTTP |
| Caddy | dobro | 80, 443 | HTTP/HTTPS |
| Pi-hole Admin | Synology | 8765 | HTTP |
| OpenAudible | Synology | 3000 | HTTP |
| Synology DSM | Synology | 5001 | HTTPS |

---

!!! tip "AUTOMATION REMINDER"
    Most services auto-start via Docker restart policies or LaunchAgents. Manual intervention should only be necessary for configuration changes or emergencies.

*Runbook version 2.0 - Last updated: 2025-11-01*
*SYSTEM STATUS: OPERATIONAL*

# Caddy Server Management

Reverse proxy with automatic HTTPS for `*.hopperhosted.com` domains.

**Host:** dobro (100.80.29.92)
**DNS Provider:** Cloudflare

### Build Caddy with Cloudflare Plugin

Custom build required for Cloudflare DNS-01 challenge support.

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

    Build in temporary location, then move to `/usr/local/libexec/caddy/` (keeps custom binaries separate from Homebrew-managed directories):

    ```bash
    # Build caddy with Cloudflare DNS plugin
    cd /tmp
    ~/go/bin/xcaddy build --with github.com/caddy-dns/cloudflare@latest \
      --output caddy-cloudflare

    # Move to permanent location
    sudo mkdir -p /usr/local/libexec/caddy
    sudo mv caddy-cloudflare /usr/local/libexec/caddy/
    sudo chmod +x /usr/local/libexec/caddy/caddy-cloudflare
    ```

    **Verify Cloudflare module:**
    ```bash
    /usr/local/libexec/caddy/caddy-cloudflare list-modules | grep cloudflare
    ```

    Expected output:
    ```
    dns.providers.cloudflare
    ```

    **Create required directories with correct permissions:**
    ```bash
    sudo mkdir -p /usr/local/var/log /usr/local/var/lib
    sudo chown -R $(whoami):staff /usr/local/var/lib /usr/local/var/log
    ```

=== "Configure brew services"

    ```bash
    # Initialize plist (creates the file)
    brew services start caddy && sleep 1 && brew services stop caddy

    # Update LaunchAgent plist
    PLIST=~/Library/LaunchAgents/homebrew.mxcl.caddy.plist

    # Set custom binary path to /usr/local/libexec/caddy/caddy-cloudflare
    /usr/libexec/PlistBuddy -c "Set :ProgramArguments:0 /usr/local/libexec/caddy/caddy-cloudflare" "$PLIST"

    # Add Cloudflare API token environment variable
    /usr/libexec/PlistBuddy -c "Add :EnvironmentVariables:CLOUDFLARE_API_TOKEN string YOUR_TOKEN_HERE" "$PLIST"

    # Reload service with new configuration
    launchctl unload "$PLIST"
    launchctl load "$PLIST"
    ```

    !!! warning "Important: Avoid brew services commands"
        After initial setup, **DO NOT** use `brew services restart/start/stop` as it regenerates the plist and erases custom settings. Use `launchctl` directly:
        ```bash
        launchctl unload "$PLIST"
        launchctl load "$PLIST"
        ```

!!! info
    Get Cloudflare API token from: https://dash.cloudflare.com/profile/api-tokens

=== "Secret Management with yadm"

    The Caddyfile uses an environment variable for the Cloudflare API token, keeping secrets out of version control.

    ```bash
    # Add plist and working copy to yadm ignore
    mkdir -p ~/.config/yadm
    cat >> ~/.config/yadm/ignore <<EOF
    Library/LaunchAgents/homebrew.mxcl.caddy.plist
    Caddyfile
    EOF

    # Commit the ignore file
    yadm add ~/.config/yadm/ignore
    yadm commit -m "Ignore Caddy plist and working copy"
    ```

    The Caddyfile references the token via `{env.CLOUDFLARE_API_TOKEN}`:
    ```
    *.hopperhosted.com hopperhosted.com {
      tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
      }
    }
    ```

    !!! note
        ~/Caddyfile should not be tracked in version control since it contains environment-specific configuration.

### Important File Locations

| Purpose | Path |
|---------|------|
| **Active Config** | `~/Caddyfile` (not tracked - local only) |
| **Custom Binary** | `/usr/local/libexec/caddy/caddy-cloudflare` |
| **Standard Binary** | `$(brew --prefix)/bin/caddy` (unused) |
| **Logs** | `/usr/local/var/log/caddy.log` |
| **Data Directory** | `/usr/local/var/lib/caddy/` |
| **LaunchAgent** | `~/Library/LaunchAgents/homebrew.mxcl.caddy.plist` (not tracked - contains secret) |

!!! note
    Our LaunchAgent uses `~/Caddyfile` directly. The plist watches this file and automatically reloads Caddy when it changes.


### Common Operations

=== "Check Status"

    ```bash
    # Check if caddy is running
    ps aux | grep caddy-cloudflare | grep -v grep

    # LaunchAgent status
    launchctl list | grep caddy

    # Check listening ports
    lsof -nP -iTCP -sTCP:LISTEN | grep -E ':(80|443)'

    # Verify correct binary is running
    ps aux | grep caddy | grep -v grep
    # Should show: /usr/local/libexec/caddy/caddy-cloudflare
    ```

=== "View Logs"

    ```bash
    tail -f /usr/local/var/log/caddy.log
    ```

=== "Edit Configuration"

    ```bash
    # Edit Caddyfile directly - changes are auto-detected by WatchPaths
    vim ~/Caddyfile
    ```

    !!! tip
        The LaunchAgent is configured with `WatchPaths` to monitor `~/Caddyfile`. Caddy will automatically reload when you save changes.

=== "Validate Config"

    ```bash
    # Requires CLOUDFLARE_API_TOKEN for validation
    CLOUDFLARE_API_TOKEN=xxx /usr/local/libexec/caddy/caddy-cloudflare validate --config ~/Caddyfile
    ```

=== "Reload Config"

    ```bash
    # Manual reload without full restart (usually not needed due to WatchPaths)
    /usr/local/libexec/caddy/caddy-cloudflare reload --config ~/Caddyfile

    # Or full restart via launchctl
    launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.caddy.plist
    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.caddy.plist
    ```

=== "Restart Service"

    ```bash
    # Use launchctl (NOT brew services - it overwrites custom plist)
    launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.caddy.plist
    launchctl load ~/Library/LaunchAgents/homebrew.mxcl.caddy.plist
    ```

    !!! warning
        Avoid `brew services restart caddy` - it regenerates the plist and removes custom binary path and environment variables

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

    !!! note
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

    !!! warning
        Synology uses self-signed cert, requires `tls_insecure_skip_verify`


### Troubleshooting

=== "Caddy Won't Start"

    **Check logs:**
    ```bash
    ssh dobro "tail -50 /usr/local/var/log/caddy.log"
    ```

    **Common issues:**

    - ❌ Caddyfile not found at `/usr/local/etc/Caddyfile`
    - ❌ Syntax errors in Caddyfile
    - ❌ Missing Cloudflare DNS module (using wrong binary)
    - ❌ Missing CLOUDFLARE_API_TOKEN environment variable
    - ❌ HTTP backend with TLS transport config (causes crash)
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
       - HTTP backend → no transport config (e.g., `reverse_proxy 127.0.0.1:8096`)
       - HTTPS backend → add TLS transport with `tls_insecure_skip_verify` if using self-signed cert

       !!! danger "Critical Error"
           Using `tls_insecure_skip_verify` with HTTP backends causes Caddy to crash with:
           ```
           upstream address scheme is HTTP but transport is configured for HTTP+TLS
           ```

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

### Workflow: Making Configuration Changes

**Standard procedure for updating Caddyfile:**

1. **Edit Caddyfile**
   ```bash
   vim ~/Caddyfile
   ```

2. **Validate syntax** (optional but recommended)
   ```bash
   CLOUDFLARE_API_TOKEN=xxx /usr/local/libexec/caddy/caddy-cloudflare validate --config ~/Caddyfile
   ```

3. **Save file**

   Caddy will automatically reload due to `WatchPaths` configuration in the LaunchAgent. No manual reload needed!

4. **Test changes**
   ```bash
   curl -I https://yourservice.hopperhosted.com
   ```

5. **Monitor logs** (if issues occur)
   ```bash
   tail -f /usr/local/var/log/caddy.log
   ```

!!! tip
    The LaunchAgent `WatchPaths` monitors `~/Caddyfile` and automatically reloads Caddy when changes are detected. No need to manually restart the service!

!!! info
    LaunchAgent is configured to start automatically at login, restart if it crashes, and run in background sessions.

## Incident Response

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

## Emergency Contacts

| ROLE | CONTACT | AVAILABILITY |
|------|---------|--------------|
| System Admin | admin@hopperhosted.com | 24/7 |
| Network Lead | network@hopperhosted.com | Business Hours |
| Security Team | security@hopperhosted.com | 24/7 |

## Quick Reference

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

!!! info "Automation"
    Most services auto-start via Docker restart policies or LaunchAgents. Manual intervention should only be necessary for configuration changes or emergencies.

---

*Last updated: 2025-11-02*

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

!!! info
    Get Cloudflare API token from: https://dash.cloudflare.com/profile/api-tokens


### Important File Locations

| Purpose | Path |
|---------|------|
| **Active Config** | `/usr/local/etc/Caddyfile` |
| **Working Copy** | `~/Caddyfile` |
| **Binary** | `/usr/local/opt/caddy/bin/caddy` |
| **Logs** | `/usr/local/var/log/caddy.log` |
| **Data Directory** | `/usr/local/var/lib/caddy/` |
| **LaunchAgent** | `~/Library/LaunchAgents/homebrew.mxcl.caddy.plist` |

!!! note
    LaunchAgent is hardcoded to use `/usr/local/etc/Caddyfile`, not `~/Caddyfile`


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

*Last updated: 2025-11-01*

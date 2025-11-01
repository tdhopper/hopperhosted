# Pi-hole

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

    !!! warning
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


# OpenAudible

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

!!! note
    Configuration stored in `/volume1/docker/openaudible/`

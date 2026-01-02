# docker-compose-servarr-with-gluetun

This is a Docker compose stack to deploy Sonarr, Radarr, Prowlarr, flaresolverr, and qBittorrent. With Prowlarr, flaresolverr, and qBittorrent connecting through Gluetun container.

All apps connect to a shared volume that connects to a NAS SMB share using CIFS.

**Previous Versions**: [docker-arrs-with-nordvpn](https://github.com/pjortiz/docker-arrs-with-nordvpn)

## Sources/Referances

- Servarr Wiki: [https://wiki.servarr.com/](https://wiki.servarr.com/)
- Gluetun: [https://github.com/qdm12/gluetun](https://github.com/qdm12/gluetun)
  - Gluetun-wiki: [https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers)
- qBittorrent: [https://hub.docker.com/r/linuxserver/qbittorrent](https://hub.docker.com/r/linuxserver/qbittorrent)
- Prowlarr: [https://hub.docker.com/r/linuxserver/prowlarr](https://hub.docker.com/r/linuxserver/prowlarr)
- Sonarr: [https://hub.docker.com/r/linuxserver/sonarr](https://hub.docker.com/r/linuxserver/sonarr)
- Radarr: [https://hub.docker.com/r/linuxserver/radarr](https://hub.docker.com/r/linuxserver/radarr)
- Overseerr: [https://hub.docker.com/r/linuxserver/overseerr](https://hub.docker.com/r/linuxserver/overseerr)
- autoheal: [https://github.com/willfarrell/docker-autoheal](https://github.com/willfarrell/docker-autoheal)
- docker-autoheal: [https://github.com/willfarrell/docker-autoheal](https://github.com/willfarrell/docker-autoheal)
- linuxserver-io-mod-vuetorrent: [https://github.com/arafatamim/linuxserver-io-mod-vuetorrent](https://github.com/arafatamim/linuxserver-io-mod-vuetorrent)

## Setup

The bellow setup is using [Proton VPN](https://pr.tn/ref/16AH5RAM) with Wiregard. Check [Gluetun-wiki](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers) for your specific VPN provider setup.

This setup assumes your VPN provider suports Port-Fowarding.

### Docker Environment Variables

| Variable | Notes |
| ----------- | ----------- |
| nasUser | The username on NAS host |
| nasPass | The password on NAS host |
| WIREGUARD_PRIVATE_KEY | Your Wiregard private key |
| UPDATER_PROTONVPN_EMAIL | Your Proton user email |
| UPDATER_PROTONVPN_PASSWORD | Your Proton passord |
| nasMediaPath | Network path to your SMB share Example: //192.168.1.18/Media |
| timezone | Whatever you timezone is, Example: America/New_York |


### Docker Compose File
```yaml:docker-compose.yml
networks:
  proxy-network:   # configure your reverse proxy network if needed
    external: true

volumes:
  qbit_config:
  prowlarr_config:
  flaresolver_config:
  sonarr_config:
  radarr_config:
  overseerr_config:
  media:
    driver_opts:
      type: cifs
      o: username=$nasUser,password=$nasPass,file_mode=0777,dir_mode=0777,noperm
      device: $nasMediaPath

services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - TZ=${timezone:-UTC}
      - VPN_SERVICE_PROVIDER=protonvpn                                        # set your vpn provider
      - VPN_TYPE=wireguard                                                    # set your vpn protocol
      - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY:?Private key required}  # set your wireguard private key
      - SERVER_COUNTRIES=United States                                        # set your preferred server country
      - FIREWALL_OUTBOUND_SUBNETS=172.18.0.0/16,192.168.0.0/16                # allow access to you local and docker networks
      - UPDATER_PERIOD=24h                                                    # frequency of updater checks
      - UPDATER_PROTONVPN_EMAIL=${UPDATER_PROTONVPN_EMAIL}                    # set your protonvpn account email for automatic server updates if needed
      - UPDATER_PROTONVPN_PASSWORD=${UPDATER_PROTONVPN_PASSWORD}              # set your protonvpn account password for automatic server updates if needed
      - PORT_FORWARD_ONLY=on                                                  # required to filter server list to only those that support port forwarding
      - VPN_PORT_FORWARDING=on                                                # required to enable port forwarding
      # Required: Commands to run when port forwarding is set up or taken down. see documentation for details.
      - VPN_PORT_FORWARDING_UP_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":{{PORT}},\"current_network_interface\":\"{{VPN_INTERFACE}}\",\"random_port\":false,\"upnp\":false}" http://127.0.0.1:9080/api/v2/app/setPreferences 2>&1' 
      - VPN_PORT_FORWARDING_DOWN_COMMAND=/bin/sh -c 'wget -O- --retry-connrefused --post-data "json={\"listen_port\":0,\"current_network_interface\":\"lo"}" http://127.0.0.1:9080/api/v2/app/setPreferences 2>&1'
    ports:
      - 8000:8000 #gluetun http server
      - 9080:9080 #qbittorrent UI
      - 9696:9696 #prowlarr
      - 8191:8191 #flaresolverr
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=1     # optional, disable ipv6; recommended if using ipv4 only
    networks:
      - proxy-network 
    restart: unless-stopped
    labels:
      - autoheal=true                        # optional, for willfarrell/docker-autoheal. Gluetun has its own healthcheck and restart mechanism, may be redundant.

  qbittorrent:
    image: linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: service:gluetun            # IMPORTANT: use gluetun's network stack
    depends_on:
      gluetun:
        condition: service_healthy           # wait for gluetun to be healthy before starting
    environment:
      - PUID=1000
      - PGID=997
      - TZ=${timezone:-UTC}
      - WEBUI_PORT=9080
      - DOCKER_MODS=arafatamim/linuxserver-io-mod-vuetorrent # optional, for vuetorrent mod
    volumes:
      - qbit_config:/config
      - media:/media
    restart: unless-stopped
    labels:
      - autoheal=true                        # required, for willfarrell/docker-autoheal
    healthcheck:
      # using the gluetun healthcheck to determine if vpn is up, since qbittorrent won't work without it
      test: curl -sL --retry-connrefused -w "%{http_code}" -o /dev/null http://localhost:9999 || exit 1
      interval: 10s
      timeout: 10s
      retries: 3

  prowlarr:
    image: linuxserver/prowlarr:latest
    container_name: prowlarr
    network_mode: service:gluetun            # IMPORTANT: use gluetun's network stack
    depends_on:
      gluetun:
        condition: service_healthy           # wait for gluetun to be healthy before starting
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=${timezone:-UTC}
    volumes:
      - prowlarr_config:/config
    restart: unless-stopped
    labels:
      - autoheal=true                        # required, for willfarrell/docker-autoheal
    healthcheck:
      # using the gluetun healthcheck to determine if vpn is up, since prowlarr won't work without it
      test: curl -sL --retry-connrefused -w "%{http_code}" -o /dev/null http://localhost:9999 || exit 1
      interval: 10s
      timeout: 10s
      retries: 3

  flaresolverr:
    image: flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    network_mode: service:gluetun            # IMPORTANT: use gluetun's network stack
    depends_on:
      gluetun:
        condition: service_healthy           # wait for gluetun to be healthy before starting
    environment:
      - LOG_LEVEL=info
      - LOG_HTML=false
      - LOG_FILE=${LOG_FILE:-none}
      - CAPTCHA_SOLVER=none                  # Warning At this time none of the captcha solvers work.
      - TZ=${timezone:-UTC}
    volumes:
      - flaresolver_config:/config
    restart: unless-stopped
    labels:
      - autoheal=true
    healthcheck:
      # using the gluetun healthcheck to determine if vpn is up, since flaresolverr won't work without it
      test: curl -sL --retry-connrefused -w "%{http_code}" -o /dev/null http://localhost:9999 || exit 1
      interval: 10s
      timeout: 10s
      retries: 3

  sonarr:
    image: linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=997
      - TZ=${timezone:-UTC}
    volumes:
      - sonarr_config:/config
      - media:/media
    ports:
      - 8989:8989
    networks:
      - proxy-network
    restart: unless-stopped
    labels:
      - autoheal=true
    healthcheck:
      test: curl -sL --retry-connrefused -w "%{http_code}" -o /dev/null --ipv4 1.1.1.1 || curl -sL --retry-connrefused -w "%{http_code}" -o /dev/null --ipv4 8.8.8.8  || exit 1
      interval: 10s
      timeout: 5s
      retries: 3
      
  radarr:
    image: linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=997
      - TZ=${timezone:-UTC}
    volumes:
      - radarr_config:/config
      - media:/media
    ports:
      - 7878:7878
    networks:
      - proxy-network
    restart: unless-stopped
    labels:
      - autoheal=true
    healthcheck:
      test: curl -sL --retry-connrefused -w "%{http_code}" -o /dev/null --ipv4 1.1.1.1 || curl -sL --retry-connrefused -w "%{http_code}" -o /dev/null --ipv4 8.8.8.8  || exit 1
      interval: 10s
      timeout: 5s
      retries: 3
      
  overseerr:
    image: linuxserver/overseerr:latest
    container_name: overseerr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=${timezone:-UTC}
    volumes:
      - overseerr_config:/config
    ports:
      - 5055:5055
    networks:
      - proxy-network
    restart: unless-stopped
    labels:
      - autoheal=true
    healthcheck:
      test: curl -sL --retry-connrefused -w "%{http_code}" -o /dev/null --ipv4 1.1.1.1 || curl -sL --retry-connrefused -w "%{http_code}" -o /dev/null --ipv4 8.8.8.8  || exit 1
      interval: 10s
      timeout: 5s
      retries: 3
```
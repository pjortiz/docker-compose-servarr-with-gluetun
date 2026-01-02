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
${DOCKER_COMPOSE}
```
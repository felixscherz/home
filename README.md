# homeassistant

## Setting up homekit bridge

In order to pair iOS devices with homeassistant I need to setup a [HomeKit
Bridge](https://www.home-assistant.io/integrations/homekit/) integration. iOS devices will try to discover
homeassistant using [mDNS](https://en.wikipedia.org/wiki/Multicast_DNS) querying `homeassistant.local`. If your network
setup does not support those kind of queries you won't be able to integrate iOS devices with homeassistant.

Since my network uses an [OpenWRT](https://openwrt.org) router, I had to setup an `avahi-daemon` in order to support
mDNS queries on my local network. After following this guide: https://blog.christophersmart.com/2020/03/30/resolving-mdns-across-vlans-with-avahi-on-openwrt/
I finally was able to call `ping homeassistant.local` and resolve to my homeassistant instance.

## Network Architecture

```mermaid
flowchart TD
    Internet([Internet Traffic])

    subgraph cloud[Cloud VM - Hetzner]
        direction TB
        nginx[nginx Reverse Proxy<br/>SSL Termination]
        wg_server[WireGuard Server<br/>10.0.0.1:51820]
        certbot[Let's Encrypt<br/>SSL Certificates]
    end

    subgraph home_net[Home Network - 192.168.0.x]
        direction TB
        router[Home Router]

        subgraph homeserver[Home Server - 192.168.0.182]
            direction TB
            wg_client[WireGuard Client<br/>10.0.0.8]
            docker[Docker Engine]

            subgraph services[Services]
                ha[Home Assistant<br/>:8123]
                paperless[Paperless<br/>:8000]
                grocy[Grocy<br/>:9283]
                mealie[Mealie<br/>:9925]
            end
        end
    end

    Internet --> nginx
    nginx -.->|home.felixscherz.me<br/>10.0.0.8:8123| ha
    nginx -.->|paperless.felixscherz.me<br/>10.0.0.8:8000| paperless
    nginx -.->|grocy.felixscherz.me<br/>10.0.0.8:9283| grocy
    nginx -.->|mealie.felixscherz.me<br/>10.0.0.8:9925| mealie

    wg_server ===|Encrypted VPN Tunnel| wg_client
    docker --> services
    homeserver --> router

    style nginx fill:#e1f5fe
    style wg_server fill:#f3e5f5
    style wg_client fill:#f3e5f5
    style services fill:#e8f5e8
```

### Key Components

- **Cloud VM**: Runs nginx reverse proxy with SSL termination and WireGuard VPN server
- **Home Server**: Hosts all services in Docker containers, connected via WireGuard client
- **VPN Tunnel**: Encrypted connection (10.0.0.0/24) allows proxy to reach home services
- **No SSH Tunnels**: Eliminated complex SSH reverse tunnels for simplified, reliable connectivity

All application services run locally on the home server, accessible to the internet through the cloud proxy via secure VPN tunnel.

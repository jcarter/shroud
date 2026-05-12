# Network Architecture

```mermaid
flowchart TD
    subgraph INET["Internet"]
        Users(["Users"])
        Admin(["Admin"])
    end

    subgraph CF["Cloudflare"]
        CFProxy["Proxy · TLS · WAF\nyour-domain.com"]
    end

    subgraph TS_NET["Tailscale"]
        TS_CTRL["Coordination Server\n(key exchange only)"]
    end

    subgraph LAN["Local Network"]
        subgraph SERVER["Debian Server"]
            subgraph UFW["UFW Firewall"]
                PUB443[":443 HTTPS — eth0, public"]
                TS22[":22 SSH — tailscale0 only"]
                TS3000[":3000 Dokploy — tailscale0 only"]
            end
            subgraph DOCKER["Docker  —  Dokploy-managed"]
                TRAEFIK["Traefik\nreverse proxy"]
                DOKPLOY["Dokploy UI"]
                SVCA["Service A"]
                SVCB["Service B"]
                SVCN["Service …"]
            end
            SSHD["sshd"]
        end
    end

    Users -- ":443 HTTPS" --> CFProxy
    CFProxy -- ":443" --> PUB443
    PUB443 --> TRAEFIK
    TRAEFIK -- "route by hostname" --> SVCA & SVCB & SVCN

    Admin -. "key exchange" .-> TS_CTRL
    SERVER -. "key exchange" .-> TS_CTRL
    Admin == "WireGuard tunnel" ==> TS22 & TS3000
    TS22 --> SSHD
    TS3000 --> DOKPLOY
```

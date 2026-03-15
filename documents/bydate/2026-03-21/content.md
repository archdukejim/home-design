# 2026-03-21

```mermaid
    flowchart TD
    %% Top Level Objects
        eero((                              eero <br> Router                        ))
        ubiquiti1{{                         unmanaged switch <br> 2.5 gb            }}

    %% Subgraphs
        subgraph pi-vpn [Raspberry Pi 5 <br> VPN EntryPoint]
            pi-vpn-nic1{{                       eth0                                    }}
            pi-vpn-ip2{                        192.168.4.2                             }
            pi-vpn-cerbot
            subgraph docker-pi-vpn [Docker Daemon]
                pi-vpn-docker-network((             Default <br> Docker <br> Network        ))
                pi-vpn-portainer-agent[             Portainer Agent                         ]
                subgraph docker-pi-vpn-stack [ VPN Stack ]
                    pi-vpn-nginx
                    adguard-dns[                        ADGuardHome                             ]
                    pi-vpn-ddns[                        DDNS updater                            ]
                    pi-vpn-openvpn

                end
            end

        end

        subgraph pi-core [Raspberry Pi Core - Internal Services]
            pi-core-nic{{                       eth0                                    }}
            pi-core-ip10{                      192.168.4.10                            }
            subgraph pi-core-containers
                pi-core-docker-net((                Default <br> Docker <br> Network        ))
                pi-core-portainer-agent[            Portainer Agent                         ]
                subgraph core-stack
                    pi-core-nginx[                      NGINX Proxy                             ]
                    Bind9
                    StepCA
                    OpenLDAP
                    Keycloak
                    pi-core-pg-keycloak[(               PostgresDB <br> for <br> Keycloak       )]
                end
            end
        end

        subgraph nas25 [nas25.j-j.family]
        %% Assign Objects to nas25.j-j.family        
            nas25-nic1{{                        enp5s0                                  }}
            nas25-ip3{                         192.168.4.3                             }
            nas25-ip4{                         192.168.4.4                             }
            nas25-ip5{                         192.168.4.5                             }
            nas25-webui[                        WebUI for nas25                         ]
            subgraph docker-nas25 [Docker Daemon]
                media-backend((App-Backend))
                subgraph proxy-stack
                    nas25-nginx[                        NGINX-Proxy-Manager                     ]
                end
                subgraph automated-media-stack
                    qbittorrent
                    jellyfin
                    seerr
                    jellyfin
                    seerr
                    prowlarr
                    sabnzbd
                    sonarr
                    radarr
                    LazyLibrarian

                end
                portainer-server[                   WebUI for Portainer                     ]
            end
        end

        %% Top Level Connections
            eero                        --- | Cat 7                             | ubiquiti1
            ubiquiti1                   --- | Cat 7                             | nas25-nic1
            ubiquiti1                   --- | Cat 7                             | pi-vpn-nic1
            ubiquiti1                   --- | Cat 7                             | pi-core-nic

        %% Connections in pi-vpn
            pi-vpn-nic1             ---      |                               | pi-vpn-ip2
            pi-vpn-cerbot           ---      |                               | pi-vpn-ip2
            pi-vpn-ip2              --->     | TCP 443                       | pi-vpn-nginx
            pi-vpn-ip2              ---      |                               | pi-vpn-docker-network
            pi-vpn-docker-network   ----     |                               | pi-vpn-nginx
            pi-vpn-docker-network   --->     | TCP 53 <br> UDP 53            | adguard-dns
            pi-vpn-docker-network   --->     | TCP 443 for Managment         | adguard-dns
            pi-vpn-docker-network   ----     |                               | pi-vpn-ddns
            pi-vpn-docker-network   --->     | unknown                       | pi-vpn-openvpn
            pi-vpn-docker-network   --->     | TCP 9001                      | pi-vpn-portainer-agent
            pi-vpn-ip2              --->     | TCP 9001                      | pi-vpn-portainer-agent

        %% Draw Connections within pi-core
            pi-core-nic         ---     |                                   | pi-core-ip10
            pi-core-ip10        ---     |                                   | pi-core-docker-net
            pi-core-ip10        --->    | TCP 443 / 53 / 636                | pi-core-nginx
            pi-core-docker-net  ----    |                                   | pi-core-nginx
            pi-core-docker-net  --->    | TCP 5432                          | pi-core-pg-keycloak
            pi-core-docker-net  --->    | TCP 8443                          | Keycloak
            pi-core-docker-net  --->    | TCP 636                           | OpenLDAP
            pi-core-docker-net  --->    | TCP/UDP 53                        | Bind9
            pi-core-docker-net  --->    |                                   | StepCA
            pi-core-docker-net  ---     |                                   | pi-core-portainer-agent
            pi-core-ip10        ---->   | TCP 9001                          | pi-core-portainer-agent


        %% Draw Connections within nas25
            nas25-nic1          ---     |                                   | nas25-ip3
            nas25-ip3           -->     | TCP 443                           | nas25-webui
            nas25-nic1          ---     |                                   | nas25-ip4
            nas25-ip4           --->    | TCP 443                           | portainer-server
            nas25-nic1          ---     |                                   | nas25-ip5

            nas25-ip5           ---->   | TCP 51413 <br> UDP 51413 <br> Torrent Stream | qbittorrent
            media-backend       --->    | TCP 30024 <br> Management         | qbittorrent
            media-backend       --->    | TCP 81 <br> Management            | nas25-nginx
            nas25-ip5           ---->   | TCP 80 / 443 <br> Host Resolution | nas25-nginx
            nas25-ip5           ---     |                                   | media-backend

            media-backend       -->     | TCP 8096                          | jellyfin
            media-backend       -->     | TCP 5055                          | seerr
            media-backend       -->     | TCP 9696                          | prowlarr
            media-backend       -->     | TCP 9090                          | sabnzbd
            media-backend       -->     | TCP 8989                          | sonarr
            media-backend       -->     | TCP 7878                          | radarr
            media-backend       -->     | TCP 5299                          | LazyLibrarian



```

## Notes:

- Ports listed are the exposed or open ports
- Lines represent IP layer connectivity
- Lines do not necessarily guarantee routes
- Hexagons are Physical Connections
- Circles represent subnet translation points
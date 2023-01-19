# The Best Self-Hosted DNS Server
Example Ansible role configures Unbound local & recursive DNS + PiHole setup, generating internal network DNS zone files for all types of DNS configurations (root A record, DNS round-robin,...) making it simple to manage declariative DNS for your private network needs.  This is not fully spit and polished, feel free to PR or issue if you would like to suggest enhancements.

## Configuration notes
PiHole is configured to listen on port 53, with forward resolution to local Unbound recursive DNS service listening on port 5353.  Using jinj2 templates, and variable, this allows for declarative DNS zones.  Below are examples for a few different types of zone usages.

**Security note**: No password is configured for PiHole, but can be configured in setupVars.conf.j2 template, using `pihole -a -p`.  I don't see a very good idempotent way to manage its password generation, but also didn't dig very hard.

## Usage examples
You can have multiple sub-domains/zones for different purposes (e.g. K8s resolving `*.apps.mydomain.local`) for reverse proxy/traffic management services.

```yaml
# Private network address spaces for your network
recursive_dns_local_networks:
   - { network: "192.168.0.0/24", access: "allow"}
   - { network: "192.168.0.1/24", access: "allow"}
   - { network: "192.168.0.2/24", access: "allow"}
   - { network: "10.0.0.0/24", access: "allow"}

# Your root lan domain
private_domain_root: "home.lan"

# Zone/subdomain examples within your network
local_zones:
  - { name: "media.home.lan",
    A: [
          { name: "@", ip: "192.168.0.1" },   # Zone root A record only zone
    ]}
  - { name: "services.home.lan",
    A: [
          { name: "@", ip: "192.168.1.1" },   # Zone root A record

          { name: "www", ip: "172.16.1.1" },  # Round-robin dns example
          { name: "www", ip: "172.16.1.2" },  #
          { name: "www", ip: "172.16.1.3" },  #
      ],
    CNAME: [
          { name: "*", data: "services.home.lan." } # All other CNAMEs (besides www)
                                                    # will resolve to the Zone root A record @
                                                    # Note the trailing period (.) needed
                                                    # for all CNAME references
      ]
    }
  - { name: "hwe.home.lan",                      # Mix of A records and CNAM
      A: [
          { name: "dns1", ip: "192.168.2.2" },
          { name: "dns2", ip: "192.168.2.3" },
          { name: "dns3", ip: "192.168.2.4" },
          { name: "proxmox11", ip: "192.168.2.11" },
          { name: "proxmox12", ip: "192.168.2.12" },
          { name: "proxmox13", ip: "192.168.2.13" },
          { name: "proxmox11-idrac", ip: "192.168.2.21" },
          { name: "laptop", ip: "192.168.2.14" },
          { name: "office-printer", ip: "192.168.2.15" },
          { name: "hdhomerun", ip: "192.168.2.16" },
         ],
      CNAME: [
          { name: "tv", ip: "hdhomerun.hwe.home.lan." },
      ],
    }
  - { name: "plex.direct",
      A: [ { name: "@", ip: "192.168.0.1" }         # Plex server IP for direct connection resolution workaround
      ],
      CNAME: [
      { name: "*", data: "plex.direct." }
      ]
    }
```
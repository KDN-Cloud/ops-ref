# CrowdSec Reference

CrowdSec Distributed Mesh: Central LAPI & Remote Bouncer Operations.

## Decisions & Alerts
| Action | Command |
| :--- | :--- |
| **List active decisions** | `cscli decisions list` |
| **Ban an IP** | `cscli decisions add --ip <IP> --reason "manual ban" --duration 24h` |
| **Unban an IP** | `cscli decisions delete --ip <IP>` |
| **Ban a range** | `cscli decisions add --range <CIDR> --reason "manual ban" --duration 24h` |
| **List alerts** | `cscli alerts list` |
| **Inspect alert** | `cscli alerts inspect <ID>` |
| **Flush all decisions** | `cscli decisions delete --all` |

## Bouncers & Machines
| Action | Command |
| :--- | :--- |
| **List bouncers** | `cscli bouncers list` |
| **Delete bouncer** | `cscli bouncers delete <name>` |
| **List agents/machines** | `cscli machines list` |
| **Delete machine** | `cscli machines delete <name>` |
| **Enroll in Cloud Console** | `cscli console enroll <ID>` |

## Collections & Parsers
| Action | Command |
| :--- | :--- |
| **List installed** | `cscli collections list` |
| **Install collection** | `cscli collections install crowdsecurity/nginx` |
| **Update all** | `cscli hub update && cscli hub upgrade` |
| **List parsers** | `cscli parsers list` |
| **List scenarios** | `cscli scenarios list` |

## Metrics & Observability
| Action | Command |
| :--- | :--- |
| **View metrics** | `cscli metrics` |
| **LAPI status** | `cscli lapi status` |
| **Version info** | `cscli version` |

## Docker exec Variants (Gatekeeper)
Prefix any `cscli` command with `docker exec crowdsec` when running from the host shell:

| Action | Command |
| :--- | :--- |
| **List decisions** | `docker exec crowdsec cscli decisions list` |
| **Ban an IP** | `docker exec crowdsec cscli decisions add --ip <IP> --reason "manual ban" --duration 24h` |
| **Unban an IP** | `docker exec crowdsec cscli decisions delete --ip <IP>` |
| **List bouncers** | `docker exec crowdsec cscli bouncers list` |
| **List machines** | `docker exec crowdsec cscli machines list` |
| **View metrics** | `docker exec crowdsec cscli metrics` |
| **Update hub** | `docker exec crowdsec cscli hub update && docker exec crowdsec cscli hub upgrade` |
| **Tail CrowdSec logs** | `docker logs -f crowdsec` |

## Node Enrollment (LAPI Central)
*Run these on the Gatekeeper (NUC) to register a new remote node (e.g., Raspberry Pi).*

| Action | Command |
| :--- | :--- |
| **Register Agent (Machine)** | `docker exec crowdsec cscli machines add <node-name> --auto -f -` |
| **Register Bouncer (Key)** | `docker exec crowdsec cscli bouncers add <node-name>` |

> **Note:** For Machine Registration, use the generated `password` in the remote node's `AGENT_PASSWORD` environment variable. For Bouncer Registration, use the `api_key` in the remote node's `.yaml` config.

## Native Bouncer
| Action | Command |
| :--- | :--- |
| **Check status** | `sudo systemctl status crowdsec-firewall-bouncer` |
| **Start / Stop / Restart** | `sudo systemctl <start\|stop\|restart> crowdsec-firewall-bouncer` |
| **Verify Config & LAPI Connection** | `sudo crowdsec-firewall-bouncer -c /etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml -t` |
| **Tail logs** | `sudo journalctl -u crowdsec-firewall-bouncer -f` |
| **Check nftables rules** | `sudo nft list ruleset \| grep crowdsec` |

---

### Quick Ref Notes
* **Connection Testing:** The `-t` flag is your best friend after an Ansible run. It verifies that the VPS can actually reach the **Gatekeeper (NUC)** over the WireGuard tunnel at `10.69.0.1`.
* **Ruleset Verification:** If the service is running but you suspect it isn't blocking, the `sudo nft list ruleset` command will confirm if the active drop list is actually populated with the IP sets from the LAPI.
* **Recommended Collections:** For distributed nodes, include: `crowdsecurity/linux`, `crowdsecurity/sshd`, `crowdsecurity/whitelist-good-actors`, `crowdsecurity/iptables`, and `crowdsecurity/docker`.


---
title: "How I monitor my firewall and homelab containers"
date: "2026-08-11"
summary: "My small monitoring setup for Proxmox, Dokploy containers, OPNsense and Suricata: what it checks, how alerts reach Telegram."
# description: ""
tags: ["Home lab", "Proxmox", "Dokploy", "Opnsense", "Suricata", "Telegram", "Monitoring"]
author: "Liviu Iancu"
weight: 1
series: ["Devops"]
# categories: [""]
# aliases: [""]
# cover:
#   image: images/img.png
#   caption: "Description [text](https://link.somewhere/)"
---

My home lab is no longer just a Proxmox host with a few things running on it. Applications deployed through Dokploy, a firewall doing more than basic routing, and home automation services all create a simple question: how do I know something is wrong before I discover it by accident?

I did not want a large observability stack for this. No Prometheus, Grafana, database cluster or a dashboard that becomes another service to maintain. I built two small Python monitors instead: one for the machines and Docker containers, and one for the OPNsense firewall. Both send only useful alerts to Telegram.

The goal is not to record every event. It is to get a short message when I need to do something.

## The layout

The monitoring flow looks like this:

```text
Proxmox host
  Homelab monitor LXC
  -> SSH to application VMs
  -> Docker and host checks

  Firewall monitor LXC
  -> read-only API queries for firewall, system and Suricata logs

  Both monitors
  -> SSH relay to Mac mini
  -> Telegram bridge API
  -> Telegram alert
```

```text
                    +----------- PROXMOX -----------+
                    |                               |
                    |   +--------------------+      |
                    |   | Homelab monitor    |      | 
                    |   | LXC                | SSH  |
                    |   |                    |-------------->+
                    |   | priv: app SSH key  |      |        |
                    |   +--------------------+      |        |
                    |            |                  |        |
                    |            | SSH              |        |
                    |            v                  |        |
                    |    +-----------------+        |        |
                    |    | Application VMs |        |        |
                    |    |                 |        |        |
                    |    | pub: SSH key    |        |        |
                    |    | Docker/host     |        |        |
                    |    +-----------------+        |        |
                    |                               |        |
                    |                               |        |
                    |   +--------------------+      |        |
+-----------+  HTTP |   | Firewall monitor   |      |        |
| OPNsense  |---------->| LXC                |      |        |
+-----------+       |   |                    |  SSH |        |
                    |   | API key + secret   |-------------->+
                    |   | root-only env file |      |        |
                    |   +---------+----------+      |        |
                    |                               |        |
                    +-------------------------------+        |
                                                             |
                    +--MAC MINI ----------------------+      |
                    |                                 |      |
                    |   Public monitor keys           |      |
                    |                                 |      |
                    |   +-------------------------+   |      |
                    |   | Telegram bridge         |   |      |
                    |   | 127.0.0.1               |<---------+
                    |   | API key                 |   |
                    |   +---------------+---------+   |
                    |                   |             |
                    +-------------------+-------------+
                                        | Telegram Bot API
                                        v
                                 +--------------+
                                 | Telegram     |
                                 | alert        |
                                 +--------------+
                                          
                                          
```

The monitors run in their own containers, separate from the VMs they inspect. This is important. A Dokploy VM can be down and the monitor can still report that fact.

## Homelab and Dokploy checks

The homelab monitor reaches each application VM over SSH using a non-root account that belongs to the Docker group. Root SSH is not enabled on those VMs. The private key exists only in the monitor container and the public key is installed only on the hosts it needs to inspect.

This gives the monitor enough access to run `docker ps`, `docker inspect`, `docker logs`, `df` and a few small host checks, without giving it blanket root access to every VM.

Checks:

- whether the VM accepts a TCP connection and SSH is reachable;
- Docker containers that are exited, unhealthy, restarting or killed by the OOM killer;
- restart counts compared with the previous run;
- disk usage, memory pressure and pending reboot markers.

The container restart check is more useful than I expected. A container can recover quickly enough that it looks healthy when I open Dokploy, while repeated restarts usually point to a bad environment variable, an unavailable dependency or memory pressure.

Every thirty minutes the monitor searches recent container logs for errors. Raw logs do not go to Telegram. A stack trace can contain request IDs, URLs, application data and far too much noise for a phone notification. Instead, matching lines are reduced to short facts like `app-vm / api: HTTP 500` or `app-vm / worker: timeout`.

There is a limit of ten alert lines and a length limit for each one. This is not only for Telegram. It forces the alert to be an alert, not a remote log viewer.

## Avoiding alert fatigue

The first version sent me the same thing every five minutes. Not exactly useless but it becomes background noise fast.

The monitor persists a small JSON state file. Each alert line is normalized before it is hashed: timestamps, request IDs, UUIDs, changing disk percentages and elapsed times are removed. A disk at 91% and then 97% is still the same disk-full problem; a new container state is not.

After successful delivery, that alert is suppressed for thirty days. If the Telegram bridge is unavailable, the alert is deliberately not marked as sent, so it is retried on the next run. This detail matters more than the hash. Recording an alert before delivery would create the worst kind of failure: everything looks quiet while the notification path is broken.

This state file is enough for the job. PostgreSQL would make sense if I needed history, queries, multiple users or reporting. For a single monitor with a few deduplication keys, it would be extra infrastructure and another failure mode.

## Monitoring OPNsense and Suricata

The firewall monitor runs independently in another LXC and talks to OPNsense over its API. It makes only `GET` and search/query `POST` requests. The latter is slightly confusing at first, but those endpoints are read queries, not configuration writes.

It checks five things:

- repeated inbound blocks on the WAN interface, especially against sensitive ports;
- unusual outbound behavior, such as one internal host reaching many different external addresses in the log window;
- new high and critical Suricata alerts;
- new OPNsense administrator or API authentication failures and service crashes;
- configuration changes to firewall aliases, automation rules and IDS settings.

Suricata alerts are deduplicated with the flow ID and timestamp. System events receive an initial baseline instead of sending every old log entry as a fresh alert after a restart. Firewall configuration is different: the monitor creates a stable SHA-256 hash of each JSON response, with sorted keys, and alerts when a later snapshot changes.

That does not tell me *who* changed a rule. It tells me that something changed and gives me a reason to open OPNsense while the change is still recent.

The OPNsense API account has its own key and secret, stored outside the source repository in a root-readable environment file. 

The script does not call write, delete, apply or reload routes. I am still treating the account as broader than ideal because some OPNsense privileges are grouped more widely than the specific read endpoints require.

## Telegram is the last hop

Neither monitor talks directly to the Telegram bot API. They SSH to my Mac mini using dedicated keys, then send a short JSON report to a bridge listening on loopback only. The bridge validates an API key before it forwards the message to Telegram.

Keeping the HTTP listener on the loopback interface means endpoint only is accesible from the host, not from LAN. SSH is the only path into it.

The bridge sends monitor reports directly as short Telegram messages. An AI model does not decide whether an alert is important. Security and availability decisions remain deterministic and reviewable.

Only critical Suricata detections, firewall configuration changes, and relevant OPNsense authentication or service failures are sent from the firewall monitor. Inbound blocks and possible beaconing remain in local logs for now. Internet scans are common; receiving a Telegram notification for each one would make the critical alerts useless.

## Failure handling and deployment

The firewall monitor is a systemd service that starts after the network is online and restarts after a failure. API requests have short timeouts, failed API calls are logged, and an unavailable response does not overwrite the last known configuration hash.

The homelab monitor writes its state atomically: it saves a temporary JSON file and replaces the old file only after the write completes. A power interruption should not leave a half-written state file that makes deduplication unpredictable.

I do not have CI/CD for these two small monitors. Homelab monitor changes are copied to its LXC through Proxmox and checked in dry-run mode before I rely on them. It is manual, but intentional: the deployment path is short, obvious and does not require a third system with credentials to my network.

This setup is not a complete SOC, and it is not meant to be. It gives me a useful picture of the machines I run, the containers behind Dokploy, and the firewall protecting the network. More importantly, it gives me a phone notification only when there is a reasonable chance I need to look at something.

That is enough monitoring for now. Hope you learned something useful from this setup.

# Monitoring and Alerting

This deployment includes a host-level monitoring script that checks the health of Pi-hole, Unbound, Tailscale, and internet connectivity. Failures and recoveries are reported through Telegram.

## Overview

| Component              | Check                                                          |
| ---------------------- | -------------------------------------------------------------- |
| Pi-hole                | Docker container is running and healthy                        |
| Unbound                | systemd service is active                                      |
| Unbound DNS            | DNS query to `127.0.0.1:5335` succeeds                         |
| Pi-hole DNS            | DNS query to `127.0.0.1:53` succeeds                           |
| Tailscale service      | `tailscaled` is active                                         |
| Tailscale connectivity | Tailscale reports this device as online                        |
| Exit-node capability   | Tailscale reports exit-node capability enabled                 |
| Internet connectivity  | HTTPS request to Cloudflare's DNS-over-HTTPS endpoint succeeds |

The monitor runs on the Pi-hole host and uses a systemd timer.

## Components and Paths

| Item                       | Path or name                        |
| -------------------------- | ----------------------------------- |
| Monitoring script          | `/usr/local/sbin/pihole-monitor.sh` |
| Telegram configuration     | `/etc/pihole-alerts/telegram.env`   |
| Monitoring state directory | `/var/lib/pihole-alerts`            |
| State file                 | `/var/lib/pihole-alerts/status`     |
| systemd service            | `pihole-monitor.service`            |
| systemd timer              | `pihole-monitor.timer`              |

The Telegram configuration file supplies `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID`. Keep this file restricted to privileged access; do not commit it or expose its contents.

The script creates the state directory with permissions `0700` and the state file with permissions `0600`. The systemd service runs as root with `UMask=0077`.

## Check Details

### Pi-hole

The monitor inspects the Docker container named `pihole`. The check passes only when the container state is `running` and its health status is `healthy`.

### Unbound and DNS

The monitor checks that the `unbound` systemd service is active. It also sends DNS queries directly to:

* Unbound at `127.0.0.1:5335`
* Pi-hole at `127.0.0.1:53`

Each query uses a timestamp-based name. A response with `NOERROR` or `NXDOMAIN` is considered a successful DNS response. This checks DNS responsiveness; it does not validate the correctness of a particular DNS answer.

### Tailscale

The monitor checks that:

* The `tailscaled` systemd service is active.
* Tailscale reports the local device as online.
* Tailscale reports that exit-node capability is enabled.

The exit-node check verifies the local capability flag. It does not test end-to-end client traffic through the exit node.

### Internet connectivity

The monitor makes an HTTPS request to:

`https://cloudflare-dns.com/cdn-cgi/trace`

The request uses a direct address override for `1.1.1.1`, avoiding dependence on the host's configured DNS for this check.

### Telegram delivery

For Telegram notifications, the script resolves `api.telegram.org` using `1.1.1.1` and uses curl's `--resolve` option to connect to the resolved address. This avoids relying on the host's configured DNS to deliver an alert about a DNS failure.

The bot token and chat ID are loaded from the local Telegram environment file. They must not be stored in this repository.

## Alert and Recovery Behavior

The script records the result of each check in the state file.

* **Failure:** When the monitor detects a failure while the previous recorded state had no failures, it sends one Telegram alert listing the checks that failed.
* **Recovery:** When all checks pass after a previously recorded failure, it sends a recovery notification.
* **Repeated failures:** The script does not send a new alert on every run while a failure remains active.
* **Notification delivery failure:** If sending an alert or recovery message fails, the script retains the previous state so that the notification can be retried on a subsequent run.

The monitor aggregates failures. If one check is already failing and another check subsequently fails, the new failure is not separately reported while the overall system remains in a failed state. A recovery notification is sent only after all monitored checks pass.

## systemd Scheduling

The timer is configured to:

* Run once approximately one minute after boot.
* Run again every two minutes after the previous activation.
* Allow up to 15 seconds of timer accuracy adjustment.

The service is a `oneshot` unit that executes the monitoring script as root.

Inspect the timer:

```bash
sudo systemctl status pihole-monitor.timer
sudo systemctl list-timers pihole-monitor.timer
```

Inspect the most recent service result:

```bash
sudo systemctl status pihole-monitor.service
sudo journalctl -u pihole-monitor.service -n 50 --no-pager
```

Run the monitor manually:

```bash
sudo systemctl start pihole-monitor.service
```

Inspect the recorded check results:

```bash
sudo cat /var/lib/pihole-alerts/status
```

Check individual services:

```bash
docker inspect --format '{{.State.Status}} {{if .State.Health}}{{.State.Health.Status}}{{else}}no-healthcheck{{end}}' pihole

sudo systemctl is-active unbound
sudo systemctl is-active tailscaled

tailscale status
```

Test DNS resolution directly:

```bash
dig @127.0.0.1 -p 5335 example.com A
dig @127.0.0.1 -p 53 example.com A
```

## Limitations

* **No independent external monitoring:** The script runs on the Pi-hole host. If the host is powered off, unreachable, or unable to execute the monitor, it cannot report that condition itself.
* **No end-to-end exit-node test:** The monitor checks Tailscale's local service, online status, and exit-node capability flag. It does not verify client routing through the exit node.
* **No independent client DNS test:** The DNS checks run locally on the server. They do not confirm that every Tailscale client is using Pi-hole.
* **No per-check repeat alerts during an ongoing incident:** The monitor sends notifications on aggregate failure and recovery transitions, rather than sending a new alert for every failing check on every run.
* **Telegram delivery depends on external connectivity:** If Telegram cannot be reached, notifications cannot be delivered immediately. The script retains the previous state when a notification fails, allowing a later retry.

## Security Notes

* Keep `/etc/pihole-alerts/telegram.env` out of version control.
* Do not publish bot tokens, chat IDs, private Tailscale addresses, or other sensitive configuration.
* Keep the monitoring script and state files restricted to privileged users.
* The monitoring script uses the host's Docker and systemd interfaces to inspect service health.

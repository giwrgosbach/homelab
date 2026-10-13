# Lab log

## 2026-09-22 - no lab (work day)
Expected: -
Saw:      -
Note:     map → 134 cards. New: scheduler, signals, terminal/session,
          caching, indirection. Gap audit written; SSH hardening next.


## Sunday 2026-09-27 - Full lab
Summary: changed the static IP on fred, using nmcli.

First grabbed the connection name of the NIC:

```bash
CON="$(nmcli -g GENERAL.CONNECTION device show enp3s0f3u4c2)"
```

Then changed it with `nmcli connection modify "$CON" ...`

Before applying, I set up an automatic rollback with systemd, so if I lose
the connection it goes back to DHCP after 5 minutes:

```bash
sudo systemd-run --on-active=300 --unit=net-rollback /bin/sh -c \
"nmcli connection modify '$CON' ipv4.method auto ipv4.addresses '' ipv4.gateway '' ipv4.dns '' && nmcli connection up '$CON'"
```

If the new IP works, cancel the rollback before the 5 min are up:

```bash
sudo systemctl stop net-rollback.timer
```

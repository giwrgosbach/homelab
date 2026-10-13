# The Journey

A home lab I'm building to learn Linux infrastructure and work toward the RHCSA.

## Machines

- **bach**: my main workstation
- **fred**: the server (Fedora)

## What's in here

| Path | What it is |
|------|------------|
| [`lab-log.md`](lab-log.md) | running log of lab sessions: what I did, what I expected, what I saw |
| [`ssh_hardening/`](ssh_hardening/ssh_hardening.md) | locking SSH on fred down to key-only logins |
| [`metrics/cpu-probe`](metrics/cpu-probe) | small Python script that prints total CPU usage once per second from `/proc/stat` |

## Next up

- Ansible against fred
- more monitoring

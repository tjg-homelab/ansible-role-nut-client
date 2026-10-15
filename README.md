# Ansible Role: nut-client

[![CI](https://github.com/tjg-homelab/ansible-role-nut-client/actions/workflows/ci.yml/badge.svg)](https://github.com/tjg-homelab/ansible-role-nut-client/actions/workflows/ci.yml)

Installs a [Network UPS Tools](https://networkupstools.org/) client that
monitors a UPS served by a **remote** `upsd`, and shuts the host down after it
has been on battery for a configurable number of seconds.

Each host gets its own delay, so a fleet on one UPS can shed load in a
deliberate order: heavy, dependent boxes first, and the hosts you want to keep
alive last or never.

## How it works

- `upsmon` runs in `netclient` mode and monitors the UPS as a **`secondary`**.
  A secondary never touches the UPS hardware and cannot set forced-shutdown
  (FSD) on the server, so one client can never take down the server or its
  neighbours.
- `ONBATT` and `ONLINE` events go to `upssched`. `ONBATT` starts a timer of
  `nut_client_shutdown_delay` seconds and `ONLINE` cancels it, so outages
  shorter than the delay cost nothing.
- When the timer fires, the command script runs `upsmon -c fsd`, which shuts
  this host down via `SHUTDOWNCMD` after `FINALDELAY`.
- NUT's own triggers stay active as a backstop. The host also shuts down
  early on **low battery**, on **FSD from the server's primary**, or if the UPS
  was last seen on battery and the server then becomes unreachable for
  `nut_client_deadtime` seconds.

### No root needed for the shutdown trigger

`upsmon` splits into a root parent (which runs `SHUTDOWNCMD`) and an
unprivileged child running as `nut`. The pidfile names the **child**, and the
child is the process that handles the forced-shutdown signal. So the command
script, which `upssched` runs as `nut`, can call `upsmon -c fsd` directly: it
signals a process it owns, and the child asks the parent over their private
pipe to shut down. No sudo rule and no `RUN_AS_USER root`. Checked against the
NUT source, 2.8.0 through 2.8.5.

### Conflicting packages

Debian's `nut-client` declares `Conflicts: apcupsd`. The role **purges**
`apcupsd` (and anything else in `nut_client_conflicting_packages`) before it
installs, so two UPS daemons never coexist on a host.

## Requirements

- Debian 12/13 or Ubuntu 24.04/26.04
- A reachable NUT server with an `upsmon secondary` user in `upsd.users`
  (on Home Assistant's NUT app this is an entry under `users`)

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `nut_client_server` | `""` — **required** | Hostname/IP of the `upsd` server |
| `nut_client_password` | `""` — **required** | Password of the monitor user. Keep it in vault. Must not contain `"` |
| `nut_client_ups_name` | `ups` | UPS name on the server |
| `nut_client_server_port` | `3493` | `upsd` port |
| `nut_client_user` | `monuser` | Monitor user on the server |
| `nut_client_power_value` | `1` | Power supplies this UPS feeds on the host |
| `nut_client_shutdown_delay` | `300` | Seconds on battery before shutdown. `0` disables the timer (low battery / FSD only) |
| `nut_client_shutdown_timer` | `onbatt-shutdown` | Timer name, visible in logs |
| `nut_client_cmdscript_path` | `/usr/local/sbin/nut-upssched-cmd` | Script `upssched` runs |
| `nut_client_shutdown_cmd` | `/sbin/shutdown -h +0` | `SHUTDOWNCMD` |
| `nut_client_final_delay` | `5` | `FINALDELAY` — seconds between warning and shutdown |
| `nut_client_poll_freq` | `5` | `POLLFREQ` |
| `nut_client_poll_freq_alert` | `5` | `POLLFREQALERT` (while on battery) |
| `nut_client_deadtime` | `15` | `DEADTIME` — seconds before the UPS is declared stale |
| `nut_client_packages` | `nut-client` | Packages installed |
| `nut_client_conflicting_packages` | `apcupsd` | Packages purged first |
| `nut_client_run_as_user` | `nut` | Account the `upsmon` child runs as |
| `nut_client_upsmon_path` | `/sbin/upsmon` | Path the command script calls |
| `nut_client_service_name` | `nut-monitor` | systemd unit |
| `nut_client_service_state` | `started` | Service runtime state |
| `nut_client_service_enabled` | `true` | Enable at boot |

## Example Playbook

Staggered shutdown: one group default, with per-host overrides.

```yaml
# group_vars/nut_client/vars.yml
nut_client_server: 192.168.2.128
nut_client_ups_name: ups01
nut_client_user: fleet
nut_client_password: "{{ vault_nut_monitor_password }}"
nut_client_shutdown_delay: 300

# host_vars/bigbox/vars.yml — goes first
nut_client_shutdown_delay: 90
```

```yaml
- hosts: nut_client
  become: true
  roles:
    - role: nut-client
```

## Testing a host without pulling the plug

```sh
# Is the client talking to the server?
upsc ups01@192.168.2.128

# Exercise the real shutdown path on ONE host. This shuts that host down.
sudo upsmon -c fsd
```

On a secondary, `upsmon -c fsd` only shuts down the local host; it cannot set
FSD on the server.

## License

MIT

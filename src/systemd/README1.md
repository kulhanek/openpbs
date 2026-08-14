# OpenPBS native systemd split

This layout replaces the monolithic `pbs_init.d` lifecycle with separate systemd units.
`/etc/pbs.conf` remains the source of `PBS_EXEC`, `PBS_HOME`, and `PBS_START_*` settings.

## Units

- `pbs.target` — aggregate start/stop point.
- `pbs-prepare.service` — validates configuration/hostnames, runs `pbs_habitat` when needed,
  rotates stale core files, and applies MOM's `restrict_user_maxsysid` default.
- `pbs-comm.service` — `pbs_comm`.
- `pbs-mom.service` — `pbs_mom -p`.
- `pbs-dataservice.service` — data service control process; active only on a server host.
- `pbs-server.service` — `pbs_server`, with the legacy exit-code-4 one-time retry and graceful `qterm` stop.
- `pbs-sched.service` — `pbs_sched`, ordered after the server.

## Installation

Adjust `/usr/libexec` if your distribution uses a different local helper directory.

```sh
install -m 0755 pbs-systemd-helper /usr/libexec/pbs-systemd-helper
install -m 0644 pbs-*.service pbs.target /etc/systemd/system/

systemctl daemon-reload
systemctl disable --now pbs.service 2>/dev/null || true
systemctl enable --now pbs.target
```

Do not enable the component units individually unless you intentionally want them independently
pulled into boot. `pbs.target` is the normal entry point; `ExecCondition` honors `PBS_START_*`.

## Useful commands

```sh
systemctl status pbs.target pbs-server pbs-sched pbs-mom pbs-comm pbs-dataservice
systemctl restart pbs-sched
systemctl restart pbs-mom
journalctl -u pbs-server -u pbs-dataservice
```

## Notes

1. Each PBS daemon is launched in its own systemd service/cgroup. This avoids the monolithic
   `pbs.service` cgroup used by the upstream compatibility unit.
2. The data service is ordered before the server. On shutdown, systemd reverses that ordering,
   so the server is stopped before `pbs_dataservice stop`; this matches the dataservice script's
   explicit refusal to stop while a PBS server is running.
3. `pbs_sched` is ordered after `pbs_server`, unlike the historical init-script sequence. This is
   intentional: the scheduler is a client of the server and should not be considered ready first.
4. MOM does not require the server unit, allowing execution-only hosts to boot even when the PBS
   server is remote or temporarily unavailable.

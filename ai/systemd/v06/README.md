# OpenPBS native systemd bootstrap split

The server startup path is split into four ordered phases:

    pbs-prepare
        -> pbs-datastore-init
        -> pbs-dataservice
        -> pbs-server-init
        -> pbs-server
        -> pbs-sched

`pbs-prepare` is now DB-independent.  It validates the host configuration,
ensures the PBS_HOME tree exists using `pbs_postinstall`, rotates stale cores,
and applies MOM defaults.

`pbs-datastore-init` runs `pbs_db_utility install_db` (or `upgrade_db` for an
existing datastore) while PostgreSQL is stopped.  A marker in server_priv is
created for a newly-created datastore.

`pbs-dataservice` then starts PostgreSQL as the native systemd-owned foreground
process and waits until it accepts connections.

`pbs-server-init` consumes the new-datastore marker.  On first installation it
runs `pbs_server -t create`, creates/enables the default `workq`, optionally
creates the local node on a combined server+MOM host, stops the temporary
server, writes `PBS_HOME/pbs_version`, and removes the marker.

Finally, the normal `pbs-server.service` starts.  It is also ordered after
`pbs-comm.service`.

## Important compatibility note

This helper implements the normal Linux/OpenPBS bootstrap path explicitly
instead of running `pbs_habitat` as one monolithic operation.  Less-common
`pbs_habitat` branches (notably Cray-specific setup, legacy licensing-file
migration, and PostgreSQL binary backup handling) are not reproduced here.
If those behaviours are required, they should be split out as separate helper
functions as well rather than reintroducing `pbs_habitat` into the ordered
systemd startup path.

## Fresh-install fix

`pbs-prepare` must consider a server PBS_HOME incomplete when
`server_priv` or `server_priv/db_user` is missing, even when
`pbs_environment` already exists.  In that case it reruns
`pbs_postinstall` before `pbs-datastore-init`.

## DB-user helper portability fix

`prepare-datastore` no longer assumes that `pbs_db_env` defines
`get_db_user` or `chk_dataservice_user`.  It reads the configured account
directly from `$PBS_HOME/server_priv/db_user`, validates it, and confirms the
account exists with `getent passwd` before exporting `PBS_DATA_SERVICE_USER`.

## Datastore bootstrap PostgreSQL

`pbs_db_utility` now starts the short-lived PostgreSQL instance needed for
`install_db` and `upgrade_db` directly with `pg_ctl` under `setpriv`, instead
of calling `pbs_dataservice start/status/stop`.  This avoids the legacy
`su`/PAM/`pbs_dataservice.bin` path during bootstrap and keeps the temporary
postmaster in `pbs-datastore-init.service`'s cgroup.  The persistent database
is still started afterwards by `pbs-dataservice.service` using
`pbs_dataservice systemd-start`.

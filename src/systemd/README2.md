# Native systemd OpenPBS data service

## Why the old implementation fails

The legacy OpenPBS database start path is not systemd-native:

1. `pbs_dataservice.bin -s start` starts `pbs_ds_monitor`, which daemonizes.
2. It runs `pg_ctl ... start` through `su - DBUSER -c ...`.
3. `pg_ctl` daemonizes PostgreSQL.
4. It subsequently invokes `pbs_ds_systemd`, which writes PostgreSQL PIDs into
   a cgroup `tasks` file by hand.

The `su -` login/PAM transition and manual cgroup migration make lifetime
ownership dependent on implementation details outside systemd.  They are not
needed when PostgreSQL is started as the actual service process.

## New model

`pbs-dataservice.service` runs:

    pbs_dataservice systemd-start

The script:

* reads `/etc/pbs.conf` and `pbs_db_env`;
* validates the datastore and DB user;
* starts `pbs_ds_monitor monitor` to preserve the existing `pbs_dblock`
  locking/failover semantics;
* drops privileges with `setpriv`, without `su`, PAM, or a login session;
* `exec`s `${PGSQL_BIN}/postgres` in the foreground.

Consequently PostgreSQL is the unit MainPID and its worker processes remain in
`pbs-dataservice.service`'s cgroup.  `pbs_ds_systemd` is no longer used.

`ExecStartPost=... systemd-wait` waits until PostgreSQL is ready, so
`pbs-server.service` does not race database startup.

## Shutdown order

The data-service unit declares:

    Before=pbs-server.service

and the server declares:

    Requires=pbs-dataservice.service
    After=pbs-dataservice.service

systemd reverses ordering during shutdown, therefore `pbs-server.service` is
stopped before `pbs-dataservice.service`.

The datastore uses `KillSignal=SIGINT`, PostgreSQL's fast-shutdown signal,
which corresponds to the previous `pg_ctl stop -m fast`.  The monitor observes
the postmaster exit and releases `pbs_dblock`.

## Source cleanup

Apply `db_common-systemd.patch` (or make the equivalent source change) so the
old `pbs_ds_systemd` cgroup-migration hook is no longer called by
`pbs_dataservice.bin`.  Ideally remove `pbs_ds_systemd` from installation as
well.

The native unit itself does not call `pbs_dataservice.bin -s start`, so this
patch is not needed for the normal native-systemd startup path, but removing it
prevents accidental use of the obsolete cgroup manipulation elsewhere.

## Installation

Example for the paths used by the refactored units:

    install -m 0755 pbs_dataservice /opt/pbs/sbin/pbs_dataservice
    install -m 0644 pbs-dataservice.service /etc/systemd/system/
    mkdir -p /etc/systemd/system/pbs-server.service.d
    install -m 0644 pbs-server-dependency.conf \
        /etc/systemd/system/pbs-server.service.d/dataservice.conf
    systemctl daemon-reload
    systemctl restart pbs-server.service

Adjust `/opt/pbs` if `PBS_EXEC` is installed elsewhere.

## Expected process ownership

After startup:

    systemctl status pbs-dataservice.service
    systemctl show -p MainPID -p ControlGroup pbs-dataservice.service
    systemd-cgls /system.slice/pbs-dataservice.service

The MainPID should be the PostgreSQL `postgres` process and the monitor plus
all PostgreSQL backend processes should appear below the same service cgroup.

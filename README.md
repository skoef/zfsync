# zfsync
**remote ZFS replication made easy**

Using ZFS opens up the really cool possibility to send snapshots to remote ZFS enabled servers, but managing synchronization of your snapshots can turn out more difficult. `zfsync` makes keeping these snapshots in sync easy.

TL;DR:
 
`zfsync -H remote.host -T tank/backups/myhost zroot/precious/data`

syncs all filesystems from zroot/precious/data to the remote host.

**Usage:**
```
zfsync [-dhimMPq] -H target_host -T target_zfs [-U target_user] [-I identity_file] source_zfs

    source_zfs is the filesystem that needs syncing to a remote host

Options:
    -d                  enable debugging (negates -q)
    -h                  show you this help page
    -H target_host      host to send snapshots to [required]
    -i                  send incremental stream between local and remote snapshot
                        instead of a stream package of all intermediary snapshots
    -I identity_file    use this key for authentication
    -m                  only show matching snapshots and exit (mutually exclusive with -M)
    -M                  only show missing snapshots and exit (mutually exclusive with -m)
    -P                  do not set user properties on remote end
    -q                  be more quiet
    -T target_zfs       ZFS on target host to send snapshots to [required]
    -U target_user      alternative user to authenticate as on remote host
```

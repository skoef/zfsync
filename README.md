# zfsync
**remote ZFS replication made easy**

Using ZFS opens up the really cool possibility to send snapshots to remote ZFS enabled servers, but managing synchronization of your snapshots can turn out more difficult. `zfsync` makes keeping these snapshots in sync easy.

It offers a purely in shell written script, and works without any dependencies on FreeBSD and \*Solaris systems.

##TL;DR:

`zfsync -H remote.host -T tank/backups/myhost zroot/precious/data`

syncs all filesystems from `zroot/precious/data` to the remote host.

##Usage:
```
zfsync [-dBhimMPq] -H target_host -T target_zfs [-U target_user] [-I identity_file] source_zfs

    source_zfs is the filesystem that needs syncing to a remote host

Options:
    -d                  enable debugging (negates -q)
    -B                  even when mbuffer is detected, don't buffer
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

##Preparing remote host

Currently, `zfsync` should just work as long as the remote host is running FreeBSD or \*Solaris and you have access to it over SSH. While it is certainly possible, you really shouldn't use the `root` user for receiving the ZFS snapshots (arguably for no SSH access at all, actually). Instead, set up a non-privileged user, most preferably with public key access, on the remote host. Here is a FreeBSD example:

**remote host:**
```
pw useradd -n zfsync -d /home/zfsync -m -w random -s /bin/sh
mkdir ~zfsync/.ssh/
ssh-keygen -f ~zfsync/.ssh/id_rsa
chown -R zfsync ~zfsync/.ssh
zfs allow -u zfsync create,mount,receive tank/backups/myhost
```
to create non-privileged user `zfsync`, set up a public/private key and give user `zfsync` minimal permissions for receiving ZFS snapshots `tank/backups/myhost` (`zfsync` will warn you otherwise). The public key `~zfsync/.ssh/id_rsa.pub` should be copied to the source host via a secure channel, so it could be pointed to with `-I` while using `-U zfsync` on the next sync. You should add a strong passphrase to the key file, and use ssh-agent so you can cron syncing your filesystems.

##Known issues
- has to be tested on GNU/Linux with ZFSOnLinux
- zfsync does not support multiple `source_zfs` arguments yet
- on \*Solaris, `/bin/sh` is symlinked to `ksh` on some systems, which breaks `zfsync`

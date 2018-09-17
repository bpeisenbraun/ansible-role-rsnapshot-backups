# Ansible Role: rsnapshot Backups

# !!! Caution: Menhirs Turning !!!

This is not fully baked yet. Proceed with caution.

# Description

This role can configure both the rsnapshot server and backup clients. It can
also configure an intermediate SSH proxy host that allow server -> proxy ->
client connections for backups.

Terminology: the "server" is the machine that will use rsync to pull the
backups from the "clients". A "proxy" is an intermediate host between the two
endpoints.

# How It Works

On the rsnapshot server, the role will create:

  - a backup directory hierarchy, e.g. at /backup
  - a directory at /var/log/rsnapshot for the backup logs
  - a backup configurations directory at /etc/rsnapshot.d
  - a cron job to schedule backups

On the rsnapshot clients, the role will create:

  - an 'rsnapshot' user with sudo privileges to run an rsync wrapper script

If an intermediate SSH proxy host is configured for the role, it will also create:

  - an 'sshproxy' account on that host
  - an SSH configuration file on the server that specifies the path through the proxy to the client

# Role Variables

### Required Server Configuration

Enable the server role in a host_vars file by specifying the filesystem path
where backups will be stored:

```
rsnapshot_backup_root: /backup
```

### Required Client Variables

You must specify the rsnapshot backup server and its IP address in a group or
host variable file:

```
backup_server: the-ansible-inventory-hostname
backup_server_ip: ip.add.re.ss  # This will be the proxy server IP when using one
```

You should specify the client paths to be backed up in an appropriate group or 
host variable file:

```
backup_dirs: [ '/etc', '/root', '/opt' ]
```

### Optional Client Configuration

The following variables can be set on a per-host basis.

Set the default backup retention settings:

```
# Defaults
rsnapshot_keep_daily: 7
rsnapshot_keep_weekly: 4
rsnapshot_keep_monthly: 3
```

Set custom include/exclude lists:

```
rsnapshot_exclude: [ 'exclude1', 'exclude2' ]
rsnapshot_include: [ 'include1', 'include2' ]
```

### Required Proxy Configuration

To use the proxy functionality, add a backup server entry to the inventory for
this application environment you are working with:

```
[backup]
bsn-backup301
```

Use that server name and the IP address of the proxy host as the entries for 
rsnapshot clients:

```
backup_server_ip: ip-of-the-proxy
backup_server: bsn-backup301
backup_user_key: "ssh-rsa ..."
rsnapshot_backup_root: /backup
```

For the proxy host you need to specify the IP address of the backup server in
its native silo. You should also create a list of the hosts for which it will
proxy connections.

```
backup_server_ip: 10.72.1.71
backup_proxy:
  - hostname1
  - hostname2
```

You will need to add the rsnapshot role to both the playbooks for machines
acting as proxies and those that are backup clients.

## Testing and Troubleshooting Your Configuration

It's a good idea to run your first backup manually. This is particularly true
for hosts that are not directly reachable from the backup host, since the
ssh-keyscan command will not honor your SSH configuration file and the play
automatically adding the SSH key to the known_hosts file will fail.

To do a dry run of the backup:

```
rsnapshot -tvc /etc/rsnapshot.d/HOSTNAME.conf sync
```

If it complains about an error in the configuration file, check the file syntax:

```
rsnapshot -c /etc/rsnapshot.d/HOSTNAME.conf configtest
```

If the check passes, run the first backup manually in a screen(1) or tmux(1)
session:

```
rsnapshot -c /etc/rsnapshot.d/HOSTNAME.conf sync
```

Accept the host key if necessary and monitor the progress to ensure it
completes successfully.
